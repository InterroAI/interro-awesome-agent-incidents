# Awesome Agent Incidents

> A curated knowledge base of real production failures in AI agent systems - and how we fixed them.

Each incident follows a strict template so you can reproduce, validate, and prevent the same class of failure.

## Incident Index

| ID | Title | Severity |
|---|---|---|
| [INC-001](./INC-001-goal-loop.md) | Goal Tool Infinite Loop | Critical |
| [INC-002](./INC-002-context-explosion.md) | Context Explosion | High |
| [INC-003](./INC-003-empty-bedrock-response.md) | Empty Bedrock Response | High |
| [INC-004](./INC-004-toolresult-ordering.md) | ToolResult Ordering Failure | High |
| [INC-005](./INC-005-tool-replay.md) | Tool Replay | Critical |
| [INC-006](./INC-006-checkpoint-isolation.md) | Checkpoint Tenant Isolation | Critical |
| [INC-007](./INC-007-mcp-capability-drift.md) | MCP Capability Drift | Medium |
| [INC-008](./INC-008-prompt-injection.md) | Prompt Injection Through Tool Outputs | Critical |
| [INC-009](./INC-009-retry-storm.md) | Retry Storm | High |
| [INC-010](./INC-010-state-divergence.md) | State Divergence | High |

## Template

Every incident document contains:

1. **Symptoms** - What went wrong, user-visible
2. **Evidence** - Logs, trace IDs, metrics
3. **Root Cause** - Technical explanation
4. **Code Path** - Where in the code the failure occurs
5. **Fix** - The actual code change
6. **Validation** - How to verify the fix works
7. **Regression Tests** - Automated tests that fail before, pass after
8. **Prevention Checklist** - Guardrails for your own code
9. **Prompt For Copilot** - AI-assisted prevention prompt
10. **Related Incidents** - Cross-references

## Running Regression Tests

```bash
pip install -e ../../prevention-scanner
prevention-scanner test-incident INC-001  # runs the test case for this incident
```
