# INC-010 - State Divergence

| Field | Detail |
|---|---|
| **Incident** | #010 |
| **Title** | State Divergence |
| **Tagline** | Multiple sources of truth disagree |
| **Severity** | High |
| **Date Documented** | 2026-09-14 |
| **Provider** | LangGraph / Any |
| **Tags** | state divergence, event sourcing, reconciliation |

---

## Symptoms

Multiple sources of truth all **disagree**:

- **Usage ledger** says 5 tool calls were made.
- **Checkpoints** show 7 tool calls.
- **Tool execution** log records 6 tool calls.
- **Run history** shows 4 tool calls.

The numbers don't match - making debugging, billing, and auditing impossible.

## Evidence

```
Usage ledger (PostgreSQL):
  request_id=req_abc → tool_calls=5, cost=$0.042

Checkpoint state (langgraph checkpoint):
  messages contain 7 tool_use blocks

Tool execution journal:
  6 entries for request_id=req_abc

Run history (audit log):
  4 tool calls recorded for request_id=req_abc

Audit query:
  SELECT COUNT(*) FROM tool_calls WHERE request_id='req_abc';
  → 5 (but 3 sources disagree)
```

## Root Cause

**Multiple sources of truth**: The system maintained state in multiple independent stores:
1. **Usage ledger** - for billing/audit.
2. **Checkpoints** - for graph state restoration.
3. **Tool execution journal** - for replay protection.
4. **Run history** - for observability.

These stores were updated independently, with no reconciliation mechanism. When a tool call succeeded in one store but failed in another (e.g., DB write succeeded but checkpoint save failed), the stores diverged permanently.

The fix: **Durable event sourcing** - all state changes are recorded as a single immutable event log. All other stores are materialized views derived from this log, and a **reconciliation** process periodically verifies they match.

## Code Path


## Fix

### 1. Durable Event Sourcing

```python
class EventStore:
    """Single source of truth - all state changes are immutable events."""

    def append(self, event: DomainEvent) -> None:
        """Append an event to the durable log. All writes go through here."""
        with self.db.transaction() as tx:
            tx.execute(
                """INSERT INTO events (event_id, request_id, type, data, created_at)
                   VALUES (%s, %s, %s, %s, NOW())""",
                (event.id, event.request_id, event.type, json.dumps(event.data))
            )
            # All materialized views are derived FROM this event log
        self._emit_to_subscribers(event)

    def replay(self, request_id: str) -> list[DomainEvent]:
        """Replay all events for a request to reconstruct state."""
        rows = self.db.query(
            "SELECT type, data FROM events WHERE request_id = %s ORDER BY created_at",
            (request_id,)
        )
        return [DomainEvent(r["type"], json.loads(r["data"])) for r in rows]


class ReconciliationEngine:
    """Periodically verifies that all stores agree with the event log."""

    def reconcile(self, request_id: str) -> ReconciliationResult:
        """Compare event log against all materialized views."""
        events = self.event_store.replay(request_id)

        # Derive expected state from events
        expected_tool_calls = sum(1 for e in events if e.type == "tool_called")
        expected_cost = sum(e.data.get("cost", 0) for e in events)
        expected_messages = len(events)

        # Compare against each store
        violations = []

        ledger_count = self.usage_ledger.get_tool_call_count(request_id)
        if ledger_count != expected_tool_calls:
            violations.append(
                f"Usage ledger mismatch: {ledger_count} vs {expected_tool_calls}"
            )

        checkpoint_count = self.checkpoint.get_tool_call_count(request_id)
        if checkpoint_count != expected_tool_calls:
            violations.append(
                f"Checkpoint mismatch: {checkpoint_count} vs {expected_tool_calls}"
            )

        return ReconciliationResult(violations=violations, clean=len(violations) == 0)
```

### 2. Two-Phase State Updates

```python
# BEFORE: 4 independent writes
tool_executor.execute(...)       # journal
usage_ledger.record(...)          # ledger
checkpoint.save(...)              # checkpoint
audit_log.append(...)             # history
# If any fails → stores diverge

# AFTER: single event log, materialized views
event = ToolExecutedEvent(request_id=req_id, tool_name=..., cost=...)
event_store.append(event)          # atomic - all views derived

# Materialized views update from the event log

```python
def test_replay_reproduces_expected_state():
    """Replaying events must reproduce the expected state."""
    events = [
        ToolCalledEvent(tool_name="get_weather", cost=0.01),
        ToolSucceededEvent(tool_name="get_weather", result="72°"),
        ChargeEvent(amount=0.99, cost=0.01),
    ]
    for e in events:
        event_store.append(e)

    replayed = event_store.replay("req_123")
    assert len(replayed) == 3
    assert sum(e.data.get("cost", 0) for e in replayed) == 0.02

def test_reconciliation_detects_divergence():
    """Reconciliation must detect when stores disagree."""
    # Event log says 3 tool calls
    # Usage ledger says 2 (one was dropped)
    result = ReconciliationEngine.reconcile("req_123")
    assert not result.clean
    assert any("ledger mismatch" in v for v in result.violations)

def test_two_phase_commit_atomicity():
    """All state stores must update from a single event log."""
    # Before fix: 4 independent writes could diverge
    # After fix: 1 event log append, 4 derived projections
    event = ToolExecutedEvent(request_id="req_123", tool_name="read_file", cost=0.01)
    event_store.append(event)

    # All stores must reflect this event
    assert usage_ledger.get_tool_count("req_123") == 1
    assert checkpoint.get_tool_count("req_123") == 1
    assert audit_log.get_tool_count("req_123") == 1
    assert journal.get_tool_count("req_123") == 1
```

## Prevention Checklist

- [ ] A single `EventStore` is the system of record for all state changes.
- [ ] All other stores (ledger, checkpoint, audit, journal) are materialized projections.
- [ ] `ReconciliationEngine` runs periodically to detect divergence.
- [ ] Two-phase commits ensure atomicity across stores.
- [ ] Production alerts fire when reconciliation detects violations.
- [ ] CI runs `test_state_divergence_prevention.py` with intentional divergence injection.

## Prompt For Copilot

```
Review the codebase for state divergence risks. Check:

1. Are there multiple independent stores for the same state
   (usage ledger, checkpoint, tool journal, audit log)?
   If yes → "HIGH: Multiple sources of truth - state divergence risk"

2. Is there a single event log that all stores derive from?
   If not → "CRITICAL: No event source of truth - stores can diverge"

3. Is there a reconciliation process that verifies stores agree?
   If not → "WARNING: No reconciliation - divergence goes undetected"

4. Are state updates atomic (all stores or none)?
   If not → "MEDIUM: Non-atomic state updates - partial failures cause divergence"

Output: CRITICAL/HIGH/WARNING/MEDIUM/OK per file
```

## Related Incidents

- [INC-002 - Context Explosion](./INC-002-context-explosion.md): Divergence includes message count mismatch.
- [INC-005 - Tool Replay](./INC-005-tool-replay.md): No journal means no source of truth for replay.
- [INC-006 - Checkpoint Isolation](./INC-006-checkpoint-isolation.md): Wrong checkpoint = wrong state.

## Scanner Rule

Maps to scanner rules **R-05: Missing Replay Protection**, **R-08: Missing Event Sourcing**, and **R-09: Missing Reconciliation**.

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->

class UsageLedgerProjection(EventSubscriber):
    def on(self, event: ToolExecutedEvent):
        self.insert(event.to_usage_row())

class CheckpointProjection(EventSubscriber):
    def on(self, event: ToolExecutedEvent):
        self.update_checkpoint(event)
```

## Validation

### Manual validation

1. **Event log**: Verify all state changes are recorded as events.
2. **Replay**: Verify `reconcile()` confirms all stores agree.
3. **Divergence**: Intentionally corrupt one store and verify reconciliation detects it.

### Automated validation

```bash
python -m pytest tests/test_state_divergence_prevention.py -v
```

## Regression Tests

```
tool_executor.execute(tool_call)
  → execute tool               # tool execution journal
  → update usage_ledger        # usage ledger
  → update checkpoint          # checkpoint state
  → update run_history         # run history
  → 4 independent writes, no atomicity
  → if any fails → stores diverge
```

### File locations

| Component | File | Function |
|---|---|---|
| Usage ledger | `app/billing/ledger.py` | `record()` |
| Checkpoint | `app/checkpoint/store.py` | `save()` |
| Tool journal | `app/tools/journal.py` | `record()` |
| Run history | `app/audit/history.py` | `append()` |
| Event sourcing | `app/events/store.py` | `append_event()` |
| Reconciliation | `app/reconciliation/engine.py` | `reconcile()` |
