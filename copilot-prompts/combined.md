# Copilot Prevention Prompts

Below are AI-assisted prevention prompts. Feed these to Copilot (or any code
review agent) to catch the corresponding incident classes in your codebase.

## INC-001: Goal Tool Infinite Loop

```
You are reviewing a LangGraph agent codebase. Your task is to identify goal
tools (get_goal, set_goal, complete_goal, check_goal, plan_goal, etc.) that are
exposed globally to every LLM node.

For each goal tool found:
1. Check if it is scoped to a specific node that owns goal state.
2. If it is registered in the global tool list, flag it as "Goal tools exposed globally".
3. Suggest moving it to a ScopedToolRegistry.
4. Verify a ToolCallCircuitBreaker or max_turns is configured.
5. If max_turns is unset, flag as "Missing loop protection".

Output format:
CRITICAL: Goal tools exposed globally - <tool_names> found in <file>
WARNING: Missing loop protection - <graph_name> has no max_turns set
OK: <goal_tool> scoped to <node>
```

## INC-005: Tool Replay

```
Review the agent codebase for tool replay protection:
1. Does every tool execution get recorded in a journal with (request_id, tool_call_id)?
2. Is the journal checked before re-executing a tool on checkpoint restore?
3. Do non-idempotent tools (charge_user, send_email) use idempotency keys?

Output: CRITICAL/HIGH/WARNING/OK per file
```

## INC-006: Checkpoint Isolation

```
Review the LangGraph codebase for checkpoint tenant isolation:
1. Does checkpoint isolation rely on checkpoint_ns?
2. Is thread_id derived from raw user input without tenant scoping?
3. Are checkpoint save/load events audited?

Output: CRITICAL/WARNING/OK per file
```

## INC-007: MCP Capability Drift

```
Review an MCP server implementation:
1. Does the server validate capabilities at startup?
2. Does tools/list match actual runtime capabilities?
3. Are violations logged with structured diagnostics?

Output: MEDIUM/WARNING/OK per file
```

## INC-008: Prompt Injection

```
Review the tool execution pipeline for prompt injection risks:
1. Are tool outputs from untrusted sources sanitized before passing to the LLM?
2. Is there a trust boundary that marks untrusted output?
3. Are injection patterns detected and logged?

Output: CRITICAL/HIGH/WARNING/MEDIUM/OK per file
```

## INC-009: Retry Storm

```
Review the retry logic for retry storm risks:
1. Is there a BudgetGovernor that limits max_attempts, max_cost, max_duration?
2. Are non-idempotent tools retried?
3. Is there a circuit breaker?

Output: CRITICAL/HIGH/WARNING/OK per file
```

## INC-002: Context Explosion

```
Review a LangGraph codebase. Find any configuration that sets a graph's
max_tokens, context window, or message limit.
1. If max_tokens is unset or defaults to infinity, flag it.
2. If the checkpointer does not prune old tool results, flag it.
3. If large observations are injected without compression, flag it.

Output: HIGH/OK per file
```
