# INC-001 - Goal Tool Infinite Loop

| Field | Detail |
|---|---|
| **Incident** | #001 |
| **Title** | Goal Tool Infinite Loop |
| **Tagline** | How a simple 4+4 request triggered 105 tool calls |
| **Severity** | Critical |
| **Date Documented** | 2026-09-14 |
| **Provider** | OpenAI Responses API / LangGraph |
| **Tags** | goal tools, infinite loop, tool calling, LangGraph |

---

## Symptoms

- A goal-oriented tool (`get_goal`, `set_goal`, `check_goal`) was exposed to the agent as a **globally available** tool.
- The agent received a request with 4 goals (4+4 triggers), each requiring tool execution.
- Instead of terminating after achieving goals, the agent **repeatedly called `get_goal`** - 105 times in a single conversation.
- No user-visible response was ever produced. The request appeared "stuck."
- Token/usage counters showed escalating tool-call counts while the response stayed empty.

## Evidence

```
Tool call trace (excerpt):
  get_goal → (empty goal state) → plan → execute → get_goal → plan → execute → ...
  Total get_goal calls: 105
  Total tool rounds: 14
  Final response: (none - timeout)
```

Key log line:
```
[2026-09-09T14:22:18Z] tool_call: get_goal (attempt=105/120)
[2026-09-09T14:22:18Z] WARNING: max_turns reached, aborting
```

## Root Cause

**Goal tools were exposed globally to the LLM**, rather than being scoped to specific workflow nodes where they are the designated decision maker.

In LangGraph (and agent frameworks generally), the LLM is the orchestrator. When goal-setting tools are available to *every* node that calls the LLM, the model can enter a **goal-checking loop**:

1. Agent checks its goal state → finds it incomplete
2. Agent decides to "make progress" by... calling `get_goal` again
3. `get_goal` returns the same incomplete goal
4. Agent decides to check again...
5. Loop repeats until max_turns or token limits

The **root architectural flaw**: goal tools should only be callable by the node that owns goal state - never by the general-purpose LLM node.

## Code Path

```
graph.invoke({goals: [...]})         # 4+4 goal triggers
  → goal_check_node                 # calls get_goal ✅ (intended)
  → plan_node                       # LLM decides to "check goal" → get_goal ❌
  → execute_node                    # LLM decides to "re-check goal" → get_goal ❌
  → route                           # conditional edge back to plan_node (loop!)
  → plan_node → execute_node        # 105 repeated get_goal calls

## Fix

### 1. Goal-Tool Restrictions (Scope tools to owning nodes only)

```python
# BEFORE (buggy): all tools exposed to every LLM call
def get_all_tools():
    return [search_tool, read_tool, write_tool, get_goal, set_goal, complete_goal]

# AFTER (fixed): tools are scoped per-step
class ScopedToolRegistry:
    def __init__(self):
        self._scopes = {}

    def register(self, tool_name, fn, scope_nodes):
        """Scope a tool to only the nodes that need it."""
        self._scopes[tool_name] = {"fn": fn, "nodes": set(scope_nodes)}

    def get_tools_for_node(self, node_name):
        return [
            info["fn"]
            for name, info in self._scopes.items()
            if node_name in info["nodes"]
        ]

registry = ScopedToolRegistry()
registry.register("get_goal", get_goal_fn, scope_nodes=["goal_check_node"])
registry.register("set_goal", set_goal_fn, scope_nodes=["goal_check_node"])
```

### 2. Gating: Loop Detection

```python
# Add circuit breaker to prevent runaway loops
class ToolCallCircuitBreaker:
    def __init__(self, max_calls_per_tool: int = 10):
        self.max_calls_per_tool = max_calls_per_tool
        self._call_counts: dict[str, int] = defaultdict(int)

    def check(self, tool_name: str) -> bool:
        """Returns True if the tool call is allowed."""
        self._call_counts[tool_name] += 1
        if self._call_counts[tool_name] > self.max_calls_per_tool:
            raise CircuitBreakerOpen(
                f"Tool '{tool_name}' exceeded {self.max_calls_per_unit} calls. "
                f"Possible infinite loop."
            )
        return True
```

### 3. Gating: Max Turns Per Node Type

## Prevention Checklist

- [ ] Goal/planning tools must NOT be in the global tool set exposed to LLM nodes.
- [ ] Every tool must be registered with an explicit `scope_nodes` list.
- [ ] A `ToolCallCircuitBreaker` must be active on every LLM invocation node.
- [ ] `max_turns` is set on the graph (never default to unlimited).
- [ ] Each agent node logs cumulative tool-call counts per tool name.
- [ ] CI validates that no goal/tool is in the global registry via `validate_tool_scopes.py`.
- [ ] Production deployments ship with `TOOL_CALL_LIMIT` env var defaulting to 10.

## Prompt For Copilot

```
You are reviewing a LangGraph agent codebase. Your task is to identify goal tools
(get_goal, set_goal, complete_goal, check_goal, plan_goal, etc.) that are exposed
globally to every LLM node.

For each goal tool found:
1. Check if it is scoped to a specific node that owns goal state.
2. If it is registered in the global tool list, flag it as "Goal tools exposed globally".
3. Suggest moving it to a ScopedToolRegistry with scope_nodes=["<owning_node>"].
4. Verify a ToolCallCircuitBreaker or max_turns is configured on the graph.
5. If max_turns is unset, flag as "Missing loop protection".

Output format:
CRITICAL: Goal tools exposed globally - <tool_names> found in <file>
WARNING: Missing loop protection - <graph_name> has no max_turns set
OK: <goal_tool> scoped to <node>
```

## Related Incidents

- [INC-005 - Tool Replay](./INC-005-tool-replay.md): Goal tools that replay can amplify the loop.
- [INC-009 - Retry Storm](./INC-009-retry-storm.md): Goal-loop retries compound into storm behavior.
- [INC-010 - State Divergence](./INC-010-state-divergence.md): Loop produces divergent goal state across checkpoints.

## Scanner Rule

This incident maps to scanner rule **R-01: Goal Tools Exposed Globally** ([see rule spec](./preventai/rules/rule-01-goal-tools-global.md)).

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->


```python
# In the agent graph, enforce max turns on plan → execute cycles
PLAN_EXECUTE_MAX_ROUNDS = 5  # configurable per graph

def route_after_execute(state):
    rounds = state.get("plan_execute_rounds", 0)
    if rounds >= PLAN_EXECUTE_MAX_ROUNDS:
        return "finalize"
    return "plan"  # continue loop
```

## Validation

### Manual validation steps

1. **Scope tools**: Run `./scripts/validate_tool_scopes.py` - fails if any goal/tool is registered globally.
2. **Call counter**: Enable `TOOL_CALL_LIMIT=10` and run the same 4+4 request - confirm loop breaks at 10 calls.
3. **Trace inspection**: Use `langgraph-cli inspect <run_id>` to verify `get_goal` is only called from `goal_check_node`.

### Automated validation

```bash
python -m pytest tests/test_goal_loop_prevention.py -v
```

Tests included:
- `test_goal_tools_not_exposed_to_plan_node`
- `test_circuit_breaker_stops_at_max_calls`
- `test_4plus4_request_terminates_within_10_rounds`

## Regression Tests

See `prevention-scanner/tests/test_incident_001.py`:

```python
def test_goal_tools_scoped_to_owner_only():
    """get_goal must NOT be available in plan_node or execute_node."""
    scoped = build_scoped_registry()
    plan_tools = [t.name for t in scoped.get_tools_for_node("plan_node")]
    exec_tools = [t.name for t in scoped.get_tools_for_node("execute_node")]
    assert "get_goal" not in plan_tools
    assert "get_goal" not in exec_tools
    goal_tools = [t.name for t in scoped.get_tools_for_node("goal_check_node")]
    assert "get_goal" in goal_tools


def test_circuit_breaker_raises_after_max_calls():
    breaker = ToolCallCircuitBreaker(max_calls_per_tool=10)
    for _ in range(10):
        breaker.check("get_goal")
    with pytest.raises(CircuitBreakerOpen):
        breaker.check("get_goal")


def test_4plus4_request_terminates():
    """A 4+4 goal request should terminate within 10 plan-execute rounds."""
    state = {"goals": [{"id": f"g{i}", "subgoals": [f"s{i}"] * 4} for i in range(4)]}
    final_state = run_graph_with_limits(state, max_plan_execute_rounds=10)
    assert final_state["response"] is not None
    assert final_state["get_goal_call_count"] <= 15  # goal_check_node only
```

```

### File locations (production codebase)

| Component | File | Function |
|---|---|---|
| Tool registry | `app/tools/registry.py` | `register_tools(graph_id)` |
| Goal state | `app/tools/goal.py` | `get_goal`, `set_goal`, `complete_goal` |
| Node definitions | `app/graphs/agent_graph.py` | `plan_node()`, `execute_node()` |
| Routing | `app/graphs/agent_graph.py` | `route_after_plan()` |
