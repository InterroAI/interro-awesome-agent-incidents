# INC-006 - Checkpoint Tenant Isolation

| Field | Detail |
|---|---|
| **Incident** | #006 |
| **Title** | Checkpoint Isolation Problems |
| **Tagline** | State contamination across users |
| **Severity** | Critical |
| **Date Documented** | 2026-09-14 |
| **Provider** | LangGraph / PostgreSQL |
| **Tags** | checkpoint, tenant isolation, namespace, state contamination |

---

## Symptoms

- **State contamination**: User B sees state from User A's conversation.
- **Incorrect checkpoint reuse**: Checkpoints from one user are used for another.
- User A's data leaks into User B's agent responses.
- Resume/cancel operations target the wrong user's graph.

## Evidence

```
User A creates conversation: thread_id = "conv_abc"
  → checkpoints: {user_a_data, plan, findings}

User B creates conversation: thread_id = "conv_abc"  # COLLISION
  → checkpoints: {user_a_data, plan, findings}  ← SAME CHECKPOINTS!
  → User B sees User A's plan and findings in their response
```

Debug trace from instrumented InMemorySaver:
```
GET thread: user-b-thread ns: ''
first_b: ['a1', 'b1']     # expected ['b1'] - USER A's state leaked
```

## Root Cause

**Checkpoint namespace issues**: The system relied on client-supplied `thread_id` as the primary checkpoint key without additional tenant scoping. When two users happened to receive the same `thread_id` (or reused a stale one), their checkpoints collided in the checkpoint store.

Worse, `checkpoint_ns` (the namespace component) was being set by the application but **the LangGraph framework discards top-level `checkpoint_ns` values** - treating them as internal subgraph nesting identifiers. This meant the tenant-isolation namespace was silently ignored.

## Code Path

```
user_request(user_b, thread_id="conv_abc")

## Fix

### 1. Checkpoint Identity Improvements - Owner-Scoped Thread IDs

```python
import hashlib

class CheckpointIdentity:
    """
    Deterministic, tenant-isolated checkpoint identity.
    Replaces the framework's discarded checkpoint_ns approach.
    """

    @staticmethod
    def checkpoint_thread_id(user_id: str, graph_id: str, thread_id: str) -> str:
        """
        Build a deterministic, collision-free thread_id that:
        - Is unique per user (no cross-tenant leakage)
        - Is deterministic (same inputs → same output, for resume)
        - Does not contain PII (hashed)
        """
        raw = f"{user_id}:{graph_id}:{thread_id}"
        hash_hex = hashlib.sha256(raw.encode()).hexdigest()[:16]
        return f"{hash_hex}:{graph_id}:{thread_id}"

# Usage in runner.py:
configurable = RunnableConfig(
    configurable={
        "thread_id": CheckpointIdentity.checkpoint_thread_id(
            user_id=caller.user_id,
            graph_id="career_goal",
            thread_id=client_supplied_thread_id
        ),
        "checkpoint_ns": "user-scoped",  # for ops traceability only
    }
)
```

### 2. Checkpoint Audit Events

```python
class CheckpointAuditor:
    """Emit audit events for every checkpoint save/load."""

    def on_save(self, event: CheckpointEvent) -> None:
        self._emit(
            "checkpoint.save",
            thread_id=event.thread_id,
            user_id=event.user_id,
            graph_id=event.graph_id,
            checkpoint_id=event.checkpoint_id,
            message_count=len(event.messages),
            namespace=event.namespace,
            owner_hash=event.owner_hash,
        )

    def on_load(self, event: CheckpointEvent) -> None:
        self._emit(
            "checkpoint.load",
            thread_id=event.thread_id,
            user_id=event.user_id,
            graph_id=event.graph_id,
            checkpoint_id=event.checkpoint_id,
            # Alert if user_id doesn't match the owner of the checkpoint
            owner_verified=self._verify_owner(event),
        )
```

## Validation

### Manual validation

1. **Cross-user isolation**: Create two users with the same `thread_id`. Confirm User B cannot see User A's state.
2. **Resume**: Resume a conversation after 7 days. Confirm the correct user's state is loaded.
3. **Audit trail**: Check the audit log for every checkpoint save/load event.

### Automated validation

```bash
python -m pytest tests/test_checkpoint_isolation.py tests/test_postgres_isolation.py -v
```

## Regression Tests

See `prevention-scanner/tests/test_incident_006.py`:

```python
def test_checkpoint_thread_id_scopes_per_user():
    """Same thread_id for different users must produce different checkpoint keys."""
    uid_a = CheckpointIdentity.checkpoint_thread_id("user_a", "career_goal", "conv_1")
    uid_b = CheckpointIdentity.checkpoint_thread_id("user_b", "career_goal", "conv_1")
    assert uid_a != uid_b

def test_checkpoint_thread_id_is_deterministic():
    """Same inputs must always produce the same checkpoint key."""
    result1 = CheckpointIdentity.checkpoint_thread_id("user_a", "career_goal", "conv_1")
    result2 = CheckpointIdentity.checkpoint_thread_id("user_a", "career_goal", "conv_1")
    assert result1 == result2

def test_checkpoint_thread_id_no_pii():
    """The hash must not contain the raw user_id."""
    uid = CheckpointIdentity.checkpoint_thread_id("user_a@example.com", "career_goal", "conv_1")
    assert "user_a" not in uid
    assert "example.com" not in uid

def test_cross_user_isolation():
    """User B must not see User A's checkpoint state."""
    store_a = save_checkpoint(user_id="user_a", thread_id="shared")
    state_b = load_checkpoint(user_id="user_b", thread_id="shared")
    assert state_b is None or state_b != store_a
```

## Prevention Checklist

- [ ] Checkpoint thread_id is owner-scoped (hash of user_id + graph_id + thread_id).
- [ ] The framework's `checkpoint_ns` is NOT relied upon for tenant isolation (it is discarded by LangGraph).
- [ ] Audit events are emitted for every checkpoint save and load.
- [ ] Cross-user isolation is tested with colliding thread_ids.
- [ ] Checkpoint rows are retained for 7 days and then purged (no PII accumulation).
- [ ] CI runs `test_postgres_isolation.py` against a real PostgreSQL checkpoint store.

## Prompt For Copilot

```
You are reviewing a LangGraph codebase for checkpoint tenant isolation. Check:

1. Does checkpoint isolation rely on `checkpoint_ns` (namespace) for tenant separation?
   If yes → "CRITICAL: checkpoint_ns is discarded by LangGraph framework - use
   owner-scoped thread_id hashing instead"

2. Is thread_id derived from raw user input without tenant scoping?
   If yes → "CRITICAL: thread_id collision allows cross-user state leakage"

3. Are checkpoint save/load events audited?
   If not → "WARNING: No checkpoint audit events - cannot detect state leaks"

4. Is the user_id included in the checkpoint key via a deterministic hash?
   If not → "CRITICAL: No owner hash in checkpoint key"

Output format:
CRITICAL: <issue> in <file>
WARNING: <issue> in <file>
OK: <check>
```

## Related Incidents

- [INC-002 - Context Explosion](./INC-002-context-explosion.md): Contaminated checkpoints carry stale large context.
- [INC-005 - Tool Replay](./INC-005-tool-replay.md): Wrong checkpoint → wrong journal → replayed tools.

## Scanner Rule

This incident maps to scanner rules **R-04: Missing Checkpoint Versioning** and **R-13: State Divergence Risk**.

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->

  → configurable["checkpoint_ns"] = "user_b"    # SET BUT IGNORED
  → configurable["thread_id"] = "conv_abc"      # SAME AS USER A
  → checkpointer.load(thread_id="conv_abc", ns="user_b")
  → LangGraph: ns = "" (discards "user_b")
  → Returns USER A's checkpoints!  # STATE LEAK
```

### File locations

| Component | File | Function |
|---|---|---|
| Checkpoint identity | `app/graphs/checkpoint_identity.py` | `checkpoint_namespace()` |
| Runner | `app/graphs/runner.py` | `invoke_graph()` |
| API | `app/api/v1/graphs.py` | `invoke()` |
| Checkpointer | `app/checkpoint/store.py` | `load()`, `save()` |
