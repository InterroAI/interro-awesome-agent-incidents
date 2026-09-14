# Prevention Checklist - Master

This checklist compiles the prevention items from all 10 incidents into a single
reference. Review these before deploying any LangGraph agent to production.

## 1. Goal Tool Safety

- [ ] Goal/planning tools must NOT be in the global tool set exposed to LLM nodes.
- [ ] Every tool must be registered with an explicit `scope_nodes` list.
- [ ] A `ToolCallCircuitBreaker` must be active on every LLM invocation node.
- [ ] `max_turns` is set on the graph (never default to unlimited).
- [ ] CI validates that no goal tool is in the global registry.

*See: [INC-001](./INC-001-goal-loop.md)*

## 2. Context & Token Management

- [ ] `MAX_CONTEXT_TOKENS` is set on every graph.
- [ ] `MAX_MESSAGES_PER_TURN` is enforced as a hard cap.
- [ ] Tool results are compressed via RTK before entering context.
- [ ] Checkpointer prunes old tool results beyond `MAX_TOOL_RESULTS`.
- [ ] Context budget is checked after every node execution.

*See: [INC-002](./INC-002-context-explosion.md)*

## 3. Provider Response Handling

- [ ] Bedrock provider handles `text`, `reasoningContent`, `guardContent`, `toolUse`.
- [ ] `reasoningContent` is logged but never returned as user-visible text.
- [ ] `guardContent` replacement text IS returned to the user.
- [ ] Empty `text` blocks are preserved (`if result.text is not None:`).
- [ ] Diagnostics fire when `completion_tokens > 0` and `text == ""`.

*See: [INC-003](./INC-003-empty-bedrock-response.md)*

## 4. Tool Result Ordering

- [ ] Bedrock provider normalizes tool results to match toolUse ordering.
- [ ] `validate_tool_ordering()` runs on every multi-tool response.
- [ ] Missing tool results are converted to error blocks.
- [ ] Checkpoint restore preserves tool result ordering.

*See: [INC-004](./INC-004-toolresult-ordering.md)*

## 5. Tool Replay Protection

- [ ] Every tool execution is recorded in a durable `ToolExecutionJournal`.
- [ ] Journal keys on `(request_id, tool_call_id)`.
- [ ] Checkpoint restore checks the journal before re-executing tools.

*See: [INC-005](./INC-005-tool-replay.md)*

## 6. Checkpoint Tenant Isolation

- [ ] Checkpoint `thread_id` is owner-scoped (sha256 of user_id + graph_id).
- [ ] `checkpoint_ns` is NOT relied upon (LangGraph discards it).
- [ ] Audit events emitted for every checkpoint save/load.

*See: [INC-006](./INC-006-checkpoint-isolation.md)*

## 7. MCP Validation

- [ ] MCP servers validate capabilities at startup.
- [ ] Servers fail-closed when critical tools are missing.
- [ ] `tools/list` and `tools/call` responses are cross-verified.

*See: [INC-007](./INC-007-mcp-capability-drift.md)*

## 8. Prompt Injection Defense

- [ ] Untrusted tool outputs are sanitized before entering context.
- [ ] `read_file`, `web_search`, `fetch_url` outputs are marked untrusted.
- [ ] Injection patterns are detected and logged.

*See: [INC-008](./INC-008-prompt-injection.md)*

## 9. Retry Storm Prevention

- [ ] `BudgetGovernor` enforces max attempts, max cost, max duration.
- [ ] Non-idempotent tools require idempotency keys.
- [ ] Circuit breaker trips on repeated failures.

*See: [INC-009](./INC-009-retry-storm.md)*

## 10. State Divergence

- [ ] Single event log is the system of record.
- [ ] All stores (ledger, checkpoint, journal, audit) are projections.
- [ ] `ReconciliationEngine` runs periodically.

*See: [INC-010](./INC-010-state-divergence.md)*
