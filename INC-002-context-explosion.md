# INC-002 - Context Explosion

| Field | Detail |
|---|---|
| **Incident** | #002 |
| **Title** | Context Explosion |
| **Tagline** | The 261k Token Incident |
| **Severity** | High |
| **Date Documented** | 2026-09-14 |
| **Provider** | OpenAI Responses API / LangGraph |
| **Tags** | context explosion, token growth, replay, checkpointing |

---

## Symptoms

- Conversation context grew to **261,834 tokens** - far exceeding model limits.
- The request eventually **failed outright** with a context-overflow error.
- No response was produced; the agent appeared to hang during long-running operations.
- Cost tracking showed runaway token consumption for a single request.

## Evidence

```
[2026-09-10T09:14:22Z] context_size: 128,456 tokens
[2026-09-10T09:14:23Z] context_size: 189,771 tokens
[2026-09-10T09:14:24Z] context_size: 261,834 tokens
[2026-09-10T09:14:24Z] ERROR: context_length_exceeded - request_id=abc-123
```

LangGraph checkpoint data shows function-call results accumulating in message history:

```
Checkpoint #012: messages.count=4, tokens=42,100
Checkpoint #025: messages.count=18, tokens=156,300
Checkpoint #038: messages.count=47, tokens=261,834
```

## Root Cause

**Function-call replay accumulation**: LangGraph's checkpointing mechanism stores every tool result as a message in the conversation state. When a tool result is large (e.g., a 50k-token web scrape, search results, file contents), and the tool is called multiple times across turns, the message history grows without bounds.

This was compounded by:

1. **Missing replay controls** - the framework replayed *all* prior tool results on every resume.
2. **No message growth limits** - no cap on message count or token budget per checkpoint.
3. **Large tool outputs not compressed** - raw file contents and search results were injected directly into context without RTK compression.

## Code Path

```
graph.invoke(input)
  → search_node(results)    # returns 80k tokens of search results
  → read_node(file1)        # returns 40k tokens of file content
  → read_node(file2)        # returns 35k tokens of file content

## Fix

### 1. Replay Controls - Selective Tool Result Replay

```python
class ReplayControlledCheckpointer:
    """Only replay tool results that are marked as 'replayable'."""

    MAX_TOOL_RESULTS = 25  # cap to prevent accumulation

    def save(self, checkpoint):
        messages = checkpoint["messages"]
        tool_results = [m for m in messages if m.type == "tool"]
        if len(tool_results) > self.MAX_TOOL_RESULTS:
            to_remove = set(tool_results[:-self.MAX_TOOL_RESULTS])
            messages = [m for m in messages if m not in to_remove]
            checkpoint["messages"] = messages
        return super().save(checkpoint)

    def load(self, thread_id, checkpoint_ns=""):
        checkpoint = super().load(thread_id, checkpoint_ns)
        checkpoint = self._prune_replayable(checkpoint)
        return checkpoint
```

### 2. Message Growth Limits

```python
MAX_MESSAGES_PER_TURN = 100
MAX_CONTEXT_TOKENS = 65_536
COMPRESS_AFTER_TOKENS = 32_768

def enforce_context_budget(state):
    """Run after each node to prevent context explosion."""
    messages = state["messages"]
    total_tokens = sum(count_tokens(m.content) for m in messages)

    if total_tokens > MAX_CONTEXT_TOKENS:
        raise ContextBudgetExceeded(
            f"Context would exceed {MAX_CONTEXT_TOKENS} tokens. "
            f"Current: {total_tokens}."
        )

    if len(messages) > MAX_MESSAGES_PER_TURN:
        state["messages"] = messages[-MAX_MESSAGES_PER_TURN:]

    if total_tokens > COMPRESS_AFTER_TOKENS:
        state["messages"] = compress_old_messages(messages)

    return state
```

### 3. RTK Compression for Tool Observations

```python
from interro_gateway.utils.compression import rtk_compress

def compress_observation(text: str, max_tokens: int = 5000) -> str:
See `prevention-scanner/tests/test_incident_002.py`:

```python
def test_context_does_not_exceed_budget():
    """A multi-tool conversation must stay under MAX_CONTEXT_TOKENS."""
    graph = build_agent_graph(max_context_tokens=65536, max_messages=100)
    state = run_complex_query(graph, query="research DeepSeek and compare with GPT-4")
    total_tokens = sum(count_tokens(m.content) for m in state["messages"])
    assert total_tokens <= 65536

def test_tool_results_pruned_beyond_limit():
    """Checkpointer must prune old tool results beyond MAX_TOOL_RESULTS."""
    store = ReplayControlledCheckpointer(max_tool_results=10)
    checkpoint = make_checkpoint_with_50_tool_results()
    saved = store.save(checkpoint)
    tool_results = [m for m in saved["messages"] if m.type == "tool"]
    assert len(tool_results) <= 10

def test_compression_preserves_identifiers():
    long_text = "https://example.com/doc?id=abc123 " + "text " * 10000
    compressed = compress_observation(long_text)
    assert "https://example.com/doc?id=abc123" in compressed
    assert count_tokens(compressed) <= 5000
```

## Prevention Checklist

- [ ] `MAX_CONTEXT_TOKENS` is set on every graph (never default to infinity).
- [ ] `MAX_MESSAGES_PER_TURN` is enforced as a hard cap.
- [ ] Tool results are compressed via RTK before entering context.
- [ ] Checkpointer prunes old tool results beyond `MAX_TOOL_RESULTS`.
- [ ] Context budget is checked after every node execution.
- [ ] CI runs a test that simulates 50+ tool calls and asserts context stays bounded.

## Prompt For Copilot

```
You are reviewing a LangGraph codebase. Find any configuration that sets a graph's
max_tokens, context window, or message limit.

For each graph found:
1. If `max_tokens` or `context_length` is unset or defaults to None/inf, flag:
   "MISSING: max_tokens not set on graph <name> - context explosion risk"
2. If the checkpointer does not prune old tool results, flag:
   "MISSING: Replay controls - tool results accumulate on checkpoint restore"
3. If large observations are injected without compression, flag:
   "MISSING: No RTK compression on <tool_name> outputs"

Output format:
HIGH: <issue> in <file>
OK: <file> - max_tokens=<value>, max_messages=<value>
```

## Related Incidents

- [INC-009 - Retry Storm](./INC-009-retry-storm.md): Large context → retries → larger context.
- [INC-001 - Goal Loop](./INC-001-goal-loop.md): Each loop iteration adds tool results to context.

## Scanner Rule

This incident maps to scanner rules **R-03: Missing Replay Protection**, **R-08: Missing Event Sourcing**, and **R-09: Missing Reconciliation**.

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->

    """Deterministic RTK-style compression - preserves identifiers and URLs."""
    return rtk_compress(text, max_tokens=max_tokens)

# In search_node:
results = await web_search(query)
results = [compress_observation(r.text) for r in results]
```

## Validation

### Manual validation

1. **Token budget**: Set `MAX_CONTEXT_TOKENS=65536` and run a complex multi-tool query. Confirm it does not exceed the budget.
2. **Message cap**: Set `MAX_MESSAGES_PER_TURN=50` and verify the conversation truncates gracefully.
3. **Compression**: Verify that compressed tool results retain source URLs and IDs.

### Automated validation

```bash
python -m pytest tests/test_context_explosion_prevention.py -v
```

## Regression Tests

  → plan_node               # all 155k tokens now in context
  → execute_node(results2)  # another 80k tokens of results
  → reflect_node            # 235k tokens in context
  → checkpoint.save()       # 261k token snapshot persisted
  → context_length_exceeded → FAIL
```

### File locations (production codebase)

| Component | File | Function |
|---|---|---|
| Checkpoint persistence | `app/checkpoint/store.py` | `save()`, `load()` |
| Tool execution | `app/nodes/search.py` | `search_node()` |
| Tool execution | `app/nodes/read.py` | `read_node()` |
| Compression | `app/utils/compression.py` | `compress_observation()` |
| Config | `app/config.py` | `MAX_CONTEXT_TOKENS`, `MAX_MESSAGES_PER_TURN` |
