# INC-009 - Retry Storm Risk

| Field | Detail |
|---|---|
| **Incident** | #009 |
| **Title** | Retry Storm Risk |
| **Tagline** | Unbounded retries amplify every failure |
| **Severity** | High |
| **Date Documented** | 2026-09-14 |
| **Provider** | Any / LangGraph |
| **Tags** | retry, storm, budget, idempotency |

---

## Symptoms

- **Repeated provider attempts** - the same request is sent to the LLM provider multiple times.
- **Duplicated side effects** - non-idempotent operations fire multiple times.
- Exponential retry behavior that escalates costs and rate-limit pressure.
- Provider returns 429 (rate limited) or 503 (unavailable) under load.

## Evidence

```
[2026-09-11T10:00:01Z] LLM call attempt 1 → OpenAI → 429 rate limit
[2026-09-11T10:00:02Z] retry attempt 2 → Backoff 1s
[2026-09-11T10:00:03Z] retry attempt 3 → Backoff 2s
[2026-09-11T10:00:05Z] retry attempt 4 → Backoff 4s
[2026-09-11T10:00:09Z] retry attempt 5 → Backoff 8s
[2026-09-11T10:00:17Z] retry attempt 6 → Backoff 16s
...
[2026-09-11T10:01:45Z] retry attempt 22 → gave up
Total retries: 22
Total API calls: 42
Cost multiplier: 8.4x
```

## Root Cause

**Unbounded retries**: The retry logic had no upper bound on:
1. Number of retry attempts per request.
2. Total cost/token budget for retries.
3. Time budget for a single operation.
4. Idempotency - non-idempotent tools were retried without protection.

When a provider returned rate-limit errors (429), the system would retry indefinitely with exponential backoff. If multiple concurrent requests hit the same rate limit, all would retry simultaneously, creating a **retry storm** that further overwhelmed the provider.

## Code Path

```
llm.chat(request)
  → provider returns 429 (rate limited)
  → retry_with_backoff()
  → attempt 1, 2, 3, ... N (N = infinity)

## Fix

### 1. Budget Governor

```python
class BudgetGovernor:
    """Enforces retry, cost, and time budgets to prevent retry storms."""

    def __init__(self, max_attempts=3, max_cost_usd=1.0, max_duration_seconds=120):
        self.max_attempts = max_attempts
        self.max_cost_usd = max_cost_usd
        self.max_duration_seconds = max_duration_seconds
        self._start_time = None
        self._attempt = 0
        self._cost = 0.0

    def can_retry(self, failure: FailureInfo) -> bool:
        """Check if a retry is allowed under all budgets."""
        self._attempt += 1

        if self._attempt > self.max_attempts:
            return False

        if self._cost > self.max_cost_usd:
            return False

        if self._duration() > self.max_duration_seconds:
            return False

        if failure.is_rate_limit() and self._is_concurrent_storm():
            return False  # All requests backing off

        return True

    def record_cost(self, cost: float):
        self._cost += cost

    def _duration(self) -> float:
        if self._start_time is None:
            self._start_time = time.time()
        return time.time() - self._start_time
```

### 2. Idempotency Controls for Retries

```python
# Non-idempotent operations get a retry-after header check
# Idempotent operations can be retried freely
class IdempotentTool(Tool):
    """Tools that are safe to retry."""
    idempotent = True

class ChargeTool(Tool):
    """Non-idempotent - must not retry."""
    idempotent = False

```python
def test_retry_budget_enforced():
    """Retries must stop when max_attempts is exceeded."""
    governor = BudgetGovernor(max_attempts=3)
    failure = FailureInfo(type="rate_limit")
    for _ in range(3):
        assert governor.can_retry(failure) is True
    assert governor.can_retry(failure) is False

def test_cost_budget_enforced():
    """Retries must stop when cost budget is exceeded."""
    governor = BudgetGovernor(max_attempts=10, max_cost_usd=1.0)
    failure = FailureInfo(type="rate_limit")
    governor.record_cost(1.5)  # exceeds budget
    assert governor.can_retry(failure) is False

def test_duration_budget_enforced():
    """Retries must stop when time budget is exceeded."""
    governor = BudgetGovernor(max_attempts=10, max_duration_seconds=120)
    failure = FailureInfo(type="rate_limit")
    governor._start_time = time.time() - 130  # 130s elapsed
    assert governor.can_retry(failure) is False

def test_non_idempotent_tool_not_retried():
    """Non-idempotent tools must not be retried without idempotency key."""
    tool = ChargeTool(idempotency_key=None)
    executor = RetryableToolExecutor()
    assert executor.can_retry_tool(tool, failure=FailureInfo()) is False
```

## Prevention Checklist

- [ ] `BudgetGovernor` enforces max attempts, max cost, and max duration.
- [ ] Non-idempotent tools require an idempotency key before retrying.
- [ ] Concurrent rate-limit failures trigger coordinated backoff (not simultaneous retries).
- [ ] Circuit breaker trips when budget is exceeded.
- [ ] Retry counts and costs are logged for every request.
- [ ] CI runs `test_retry_storm_prevention.py` with concurrent load simulation.

## Prompt For Copilot

```
Review the retry logic for retry storm risks. Check:

1. Is there a BudgetGovernor that limits max_attempts, max_cost, and max_duration?
   If not → "HIGH: Unbounded retries - retry storm risk"

2. Are non-idempotent tools (charge_user, send_email, etc.) retried?
   If yes → "CRITICAL: Non-idempotent tool retry - duplicated side effects"

3. Is there a circuit breaker that trips on failure?
   If not → "WARNING: No circuit breaker - failures cascade"

4. Are concurrent retries coordinated (backoff + jitter)?
   If not → "WARNING: No jitter on retry backoff - thundering herd"

Output: CRITICAL/HIGH/WARNING/OK per file
```

## Related Incidents

- [INC-001 - Goal Loop](./INC-001-goal-loop.md): Goal loops with retries cause storm behavior.
- [INC-009 - Retry Storm](./INC-009-retry-storm.md): Same incident class; this doc focuses on the storm.

## Scanner Rule

Maps to scanner rule **R-12: Retry Storm Risk**.

    idempotency_key_required = True

# In retry logic:
if not tool.idempotent and not tool.idempotency_key:
    breaker.trip()  # don't retry non-idempotent tools without keys
```

## Validation

### Manual validation

1. **Budget test**: Set `max_attempts=3` and trigger a rate limit. Confirm retries stop at 3.
2. **Cost test**: Set `max_cost_usd=0.50` and make expensive tool calls. Confirm billing stops at budget.
3. **Concurrency test**: Run 20 concurrent requests that all hit rate limits. Confirm staggered retries, not a storm.

### Automated validation

```bash
python -m pytest tests/test_retry_storm_prevention.py -v
```

## Regression Tests

  → each retry consumes tokens and budget
  → if tools are involved, each retry re-executes tools
  → storm amplifies across concurrent requests
```

### File locations

| Component | File | Function |
|---|---|---|
| Retry logic | `app/providers/base.py` | `retry_with_backoff()` |
| Budget | `app/billing/budget.py` | `check_budget()`, `deduct()` |
| Tool execution | `app/tools/executor.py` | `execute_with_retry()` |
| Circuit breaker | `app/circuit/breaker.py` | `should_retry()` |
