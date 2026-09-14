# INC-004 - ToolResult Ordering Failure

| Field | Detail |
|---|---|
| **Incident** | #004 |
| **Title** | ToolResult Ordering Failure |
| **Tagline** | Bedrock toolResult IDs do not match toolUse IDs |
| **Severity** | High |
| **Date Documented** | 2026-09-14 |
| **Provider** | AWS Bedrock Converse |
| **Tags** | bedrock, tool results, ordering, validation |

---

## Symptoms

- Bedrock Converse API rejects requests with `InvalidParameterException`:
  ```
  toolResult IDs do not match toolUse IDs in the same message
  ```
- Tool execution succeeds (the function ran), but the results cannot be
  returned to the model for the next turn.
- The agent gets stuck - it issued a tool call but never sees the result.

## Evidence

```
Request to Bedrock Converse:
  messages: [
    { role: "assistant", content: [{ toolUse: { toolUseId: "tooluse_abc" } }] },
    { role: "user", content: [{ toolResult: { toolUseId: "tooluse_xyz" } }] }  # MISMATCH
  ]

Bedrock Response:
  InvalidParameterException: toolResult.toolUseId 'tooluse_xyz' does not match
  any toolUseId in the preceding assistant message. Expected 'tooluse_abc'.
```

## Root Cause

**Strict ordering requirements**: Bedrock Converse requires that `toolResult` blocks in a user message are **strictly ordered** and **ID-matched** to the `toolUse` blocks in the preceding assistant message.

When LangGraph checkpoints and replays tool results, the order can get shuffled - especially when:

1. Multiple parallel tool calls are made simultaneously.
2. Checkpointing persists tool results in a different order than they were issued.
3. The `toolUseId` values are auto-generated (or regenerated on replay) and don't match.
4. Intermediate middleware or compression layers reorder message blocks.

Unlike OpenAI's API (which is more lenient), **Bedrock enforces strict positional + ID matching**.

## Code Path

```
agent invokes multiple tools in parallel

## Fix

### 1. ToolResult Normalization

```python
class BedrockToolResultNormalizer:
    """Ensures toolResult blocks are strictly ordered and ID-matched."""

    @staticmethod
    def normalize(assistant_content: list[dict],
                  tool_results: dict[str, Any]) -> list[dict]:
        tool_use_order = [
            block["toolUse"]["toolUseId"]
            for block in assistant_content
            if "toolUse" in block
        ]

        result_blocks = []
        for tool_use_id in tool_use_order:
            if tool_use_id not in tool_results:
                result_blocks.append({"toolResult": {
                    "toolUseId": tool_use_id,
                    "content": [{"text": f"Error: result for {tool_use_id} not found"}],
                    "status": "ERROR"
                }})
            else:
                result_blocks.append({"toolResult": {
                    "toolUseId": tool_use_id,
                    "content": [{"text": str(tool_results[tool_use_id])}],
                    "status": "SUCCESS"
                }})

        return result_blocks
```

### 2. Validation Diagnostics

```python
def validate_tool_ordering(messages: list[dict]) -> list[str]:
    violations = []
    for i in range(1, len(messages)):
        if messages[i]["role"] != "user":
            continue
        tool_use_ids, result_ids = _extract_ids(messages, i)
        if result_ids != tool_use_ids:
            violations.append(f"ToolResult ordering mismatch at message {i}")
    return violations
```

## Validation

```python
def test_toolresult_normalization_preserves_order():
    assistant_content = [
        {"toolUse": {"toolUseId": "t1"}},
        {"toolUse": {"toolUseId": "t2"}},
        {"toolUse": {"toolUseId": "t3"}},
    ]
    tool_results = {"t3": "res3", "t1": "res1", "t2": "res2"}
    normalized = BedrockToolResultNormalizer.normalize(assistant_content, tool_results)
    result_ids = [b["toolResult"]["toolUseId"] for b in normalized]
    assert result_ids == ["t1", "t2", "t3"]

def test_validation_catches_ordering_mismatch():
    messages = [
        {"role": "assistant", "content": [
            {"toolUse": {"toolUseId": "a"}},
            {"toolUse": {"toolUseId": "b"}},
        ]},
        {"role": "user", "content": [
            {"toolResult": {"toolUseId": "b"}},
            {"toolResult": {"toolUseId": "a"}},
        ]},
    ]
    violations = validate_tool_ordering(messages)
    assert len(violations) == 1

def test_missing_result_emits_error_block():
    assistant_content = [{"toolUse": {"toolUseId": "t1"}}]
    tool_results = {}
    normalized = BedrockToolResultNormalizer.normalize(assistant_content, tool_results)
    assert normalized[0]["toolResult"]["status"] == "ERROR"
```

## Prevention Checklist

- [ ] Bedrock provider normalizes tool results to match toolUse ordering.
- [ ] `validate_tool_ordering()` runs on every multi-tool response.
- [ ] Missing tool results are converted to error blocks, not silently dropped.
- [ ] Checkpoint restore preserves tool result ordering.
- [ ] CI asserts ordering is preserved across checkpoint save/restore cycles.

## Prompt For Copilot

```
Review the Bedrock message assembler. Does it preserve the ordering of
toolUse IDs from the assistant message when building toolResult blocks?
If results are stored in a dict and iterated without matching the assistant's
toolUse order → flag: "CRITICAL: ToolResult ordering can drift - use normalize()"
If validate_tool_ordering is missing → flag: "WARNING: No ToolResult ID validation"
```

## Related Incidents

- [INC-003 - Empty Bedrock Response](./INC-003-empty-bedrock-response.md): Both are Bedrock Converse protocol failures.
- [INC-005 - Tool Replay](./INC-005-tool-replay.md): Replay can reorder tool results.

## Scanner Rule

Maps to scanner rule **R-14: Bedrock Schema Drift Risk**.

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->


### Manual validation

1. **Parallel tools**: Invoke 3 tools in parallel, return results out of order. Confirm normalization reorders correctly.
2. **Missing result**: Omit one tool result. Confirm an error block is emitted.
3. **Reordered checkpoint**: Restore from shuffled checkpoint. Confirm validation catches the mismatch.

### Automated validation

```bash
python -m pytest tests/test_toolresult_ordering.py -v
```

## Regression Tests

  → tool_call_1: toolUseId=tooluse_abc
  → tool_call_2: toolUseId=tooluse_def
  → results come back in order: [result_2, result_1]  (reordered by async completion)
  → checkpoint restore reorders to: [result_1, result_2]  (different from original!)
  → assemble_converse_message(content=[toolResult_1, toolResult_2])
  → Bedrock rejects: IDs don't match the original toolUse ordering
```

### File locations

| Component | File | Function |
|---|---|---|
| Message assembly | `app/providers/bedrock.py` | `_build_converse_messages()` |
| Tool result handling | `app/tool_executor.py` | `execute_tool()` |
| Checkpoint restore | `app/checkpoint/store.py` | `restore_messages()` |
| Validation | `app/providers/bedrock.py` | `_validate_tool_ordering()` |
