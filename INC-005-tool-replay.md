# INC-005 - Tool Replay

| Field | Detail |
|---|---|
| **Incident** | #005 |
| **Title** | Tool Replay |
| **Tagline** | The Hidden Killer Of Agent Systems |
| **Severity** | Critical |
| **Date Documented** | 2026-09-14 |
| **Provider** | LangGraph / Any |
| **Tags** | tool replay, duplicate execution, idempotency, journaling |

---

## Symptoms

- Potential duplicate tool execution - the same tool is called multiple times with identical inputs.
- Side effects fire more than once (API calls, DB writes, chargeable operations).
- Debugging becomes impossible - the tool execution history is ambiguous.
- Checkpoint restore can replay a tool call that already completed successfully.

## Evidence

```
Tool execution journal (before fix):
  req_123 → get_weather(location="SF") → "72°F"       (timestamp: 14:02:11)
  req_123 → get_weather(location="SF") → "72°F"       (timestamp: 14:02:15, DUPLICATE)
  req_123 → charge_user(amount=9.99) → charged        (timestamp: 14:02:16)
  req_123 → charge_user(amount=9.99) → ALREADY CHARGED (timestamp: 14:02:17, DUPLICATE)

Total duplicate tool calls in 48 hours: 342
Total duplicate charges: 12
```

## Root Cause

**Missing execution journal**: The system had no durable record of which tools had already been executed for a given request. When LangGraph checkpoints were restored (e.g., after a crash or timeout), the framework would re-invoke tools that had already produced results.

Key issues:
1. No `ToolExecutionJournal` persisted tool execution records.
2. No `(request_id, tool_call_id)` deduplication was performed.
3. Non-idempotent tools (payment, email, API writes) could not detect prior execution.
4. Checkpoint replay would blindy re-execute all pending tools.

## Code Path

```
graph.checkpoint_resume(thread_id, checkpoint_id)
  → checkpoint.load()                    # restores all messages

## Fix

### 1. ToolExecutionJournal

```python
class ToolExecutionJournal:
    """Durable journal of tool execution (request_id, tool_call_id)."""

    def __init__(self, store: PostgresStore):
        self.store = store

    def record(self, request_id: str, tool_call_id: str,
               tool_name: str, args: dict, result: Any,
               status: str = "success") -> None:
        self.store.execute(
            """INSERT INTO tool_execution_journal
              (request_id, tool_call_id, tool_name, args, result, status, executed_at)
            VALUES (%s, %s, %s, %s, %s, %s, NOW())""",
            (request_id, tool_call_id, tool_name, json.dumps(args),
             json.dumps(result), status)
        )

    def has_executed(self, request_id: str, tool_call_id: str) -> bool:
        row = self.store.query_one(
            "SELECT 1 FROM tool_execution_journal WHERE request_id = %s AND tool_call_id = %s",
            (request_id, tool_call_id)
        )
        return row is not None

    def get_result(self, request_id: str, tool_call_id: str) -> Any | None:
        row = self.store.query_one(
            "SELECT result FROM tool_execution_journal WHERE request_id = %s AND tool_call_id = %s AND status = 'success'",
            (request_id, tool_call_id)
        )
        return json.loads(row["result"]) if row else None
```

### 2. Idempotency Controls

```python
class IdempotentToolExecutor:
    """Wraps tool execution with replay protection."""

    def __init__(self, journal: ToolExecutionJournal):
        self.journal = journal

    async def execute(self, ctx, tool_call):
        # Check journal first - if already executed, return cached result
        if self.journal.has_executed(ctx.request_id, tool_call.id):
## Validation

### Manual validation

1. **Journal check**: Verify `tool_execution_journal` table has entries for every tool call.
2. **Replay test**: Restore from a checkpoint and re-invoke - confirm tools are NOT re-executed.
3. **Idempotency**: Run the same request twice - confirm side effects fire only once.

### Automated validation

```bash
python -m pytest tests/test_tool_replay_prevention.py -v
```

## Regression Tests

See `prevention-scanner/tests/test_incident_005.py`:

```python
async def test_tool_not_re_executed_on_checkpoint_restore():
    """Restoring from a checkpoint must not re-execute tools."""
    journal = ToolExecutionJournal(store)
    executor = IdempotentToolExecutor(journal)
    ctx = ExecutionContext(request_id="req_123")

    # First execution
    result1 = await executor.execute(ctx, ToolCall(id="tc_1", name="get_weather", args={"loc": "SF"}))
    assert not result1.replayed

    # Simulate checkpoint restore + re-invoke
    result2 = await executor.execute(ctx, ToolCall(id="tc_1", name="get_weather", args={"loc": "SF"}))
    assert result2.replayed  # returned from journal, not re-executed
    assert result1.result == result2.result

async def test_journal_records_all_executions():
    """Every tool execution must be recorded in the journal."""
    executor = IdempotentToolExecutor(journal)
    await executor.execute(ctx, ToolCall(id="tc_1", name="search", args={"q": "test"}))
    assert journal.has_executed("req_123", "tc_1")
```

## Prevention Checklist

- [ ] Every tool execution is recorded in a durable `ToolExecutionJournal`.
- [ ] Journal keys on `(request_id, tool_call_id)` - never on tool name alone.
- [ ] Checkpoint restore checks the journal before re-executing tools.
- [ ] Non-idempotent tools (payments, emails) require explicit idempotency keys.
- [ ] CI runs a test that simulates checkpoint restore and asserts no duplicate tool calls.
- [ ] Production alerts fire when `replayed=True` count exceeds 0.

## Prompt For Copilot

```
Review the agent codebase for tool replay protection:

1. Does every tool execution get recorded in a journal with (request_id, tool_call_id)?
   If not → "CRITICAL: ToolExecutionJournal missing - tool replay risk"

2. Is the journal checked before re-executing a tool on checkpoint restore?
   If not → "CRITICAL: No replay check before tool execution"

3. Do non-idempotent tools use idempotency keys?
   If not → "HIGH: Non-idempotent tool without idempotency key - <tool_name>"

Output: CRITICAL/HIGH/WARNING/OK per file
```

## Related Incidents

- [INC-001 - Goal Loop](./INC-001-goal-loop.md): Without replay protection, goal-loop retries re-execute tools.
- [INC-004 - ToolResult Ordering](./INC-004-toolresult-ordering.md): Replay can reorder tool results.
- [INC-010 - State Divergence](./INC-010-state-divergence.md): No journal means no reconciliation source of truth.

## Scanner Rule

Maps to scanner rules **R-02: Missing Tool Execution Journal**, **R-03: Missing Replay Protection**, **R-11: Tool Replay Risk**.

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->

            cached = self.journal.get_result(ctx.request_id, tool_call.id)
            logger.info("tool_replay_avoided", tool_call_id=tool_call.id)
            return ToolResult(tool_call_id=tool_call.id, result=cached, replayed=True)

        # Execute tool for the first time
        result = await tool_call.fn(**tool_call.args)

        self.journal.record(
            request_id=ctx.request_id, tool_call_id=tool_call.id,
            tool_name=tool_call.name, args=tool_call.args, result=result
        )
        return ToolResult(tool_call_id=tool_call.id, result=result)
```

  → agent.llm.invoke(messages)           # sees old tool_use blocks
  → tool_executor.execute(tool_calls)    # RE-EXECUTES tools already done
  → charge_user(amount=9.99)             # DUPLICATE CHARGE
  → get_weather(location="SF")           # DUPLICATE CALL
```

### File locations

| Component | File | Function |
|---|---|---|
| Tool execution | `app/tools/executor.py` | `execute()` |
| Checkpoint | `app/checkpoint/store.py` | `load()`, `resume()` |
| Request tracking | `app/request/tracker.py` | `track_request()` |
| Cost accounting | `app/billing/ledger.py` | `record_charge()` |
