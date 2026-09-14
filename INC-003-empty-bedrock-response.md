# INC-003 - Empty Bedrock Response

| Field | Detail |
|---|---|
| **Incident** | #003 |
| **Title** | Empty Bedrock Response |
| **Tagline** | output_tokens > 0 but text == "" |
| **Severity** | High |
| **Date Documented** | 2026-09-14 |
| **Provider** | AWS Bedrock Converse |
| **Tags** | bedrock, empty response, reasoning_content, guard_content, schema |

---

## Symptoms

- `output_tokens > 0` (the provider reported token generation).
- `text == ""` - the user-visible message was completely empty.
- The agent appeared to "stall" - no assistant turn was emitted to downstream consumers.
- Codex agents received a `response.completed` SSE event with no `response.output_text.delta` blocks.
- No error was raised - the failure was silent.

## Evidence

```
Request ID: c18b10e9-56e6-48e9-97db-12300fa7815f
output_tokens=1024
tool_call_count=0
text=""
stopReason: max_tokens
content blocks: [
  {"reasoningContent": {"text": "..."}},  # 1024 reasoning tokens, no visible text
]
```

## Root Cause

**Two compounding filters dropped every renderable byte:**

1. **Provider layer** (`bedrock.py`): The Converse output extraction only read truthy `block["text"]` values. Bedrock Converse content blocks can be:
   - `text` - normal text
   - `toolUse` - tool call
   - `reasoningContent` - model-internal chain-of-thought (e.g., DeepSeek-R1, MiniMax-M2)
   - `guardContent` - guardrail intervention replacement

   When a reasoning-only turn exhausted `maxTokens` (`stopReason: max_tokens`, `outputTokens: 1024`), the content blocks contained only `reasoningContent`, and the extraction loop produced `text = ""`.

2. **Responses layer** (`responses.py`): The message item was skipped entirely when `result.text` was falsy (`if result.text:` instead of `if result.text is not None:`). This meant the SSE stream emitted `output: []` - zero message items.

## Code Path

```
POST /v1/chat/completions or /v1/responses
  → Router.chat() or Router.responses()

## Fix

### 1. Provider Schema Validator (Handle all block types)

```python
def _extract_converse_text(content_blocks: list[dict]) -> tuple[str, dict]:
    text_parts = []
    diagnostics = {"block_types": [], "has_reasoning": False, "has_guard": False}

    for block in content_blocks:
        block_type = list(block.keys())[0] if block else "unknown"
        diagnostics["block_types"].append(block_type)

        if block_type == "text":
            text_parts.append(block["text"] or "")  # preserve empty
        elif block_type == "reasoningContent":
            diagnostics["has_reasoning"] = True
            preview = block["reasoningContent"].get("text", "")
            logger.info("reasoning_content_extracted", char_count=len(preview))
        elif block_type == "guardContent":
            diagnostics["has_guard"] = True
            guard_text = block["guardContent"].get("text", {}).get("text", "")
            if guard_text:
                text_parts.append(guard_text)
        elif block_type == "toolUse":
            pass  # not text
        else:
            logger.info("unknown_content_block", block_type=block_type)

    return "".join(text_parts), diagnostics
```

### 2. Responses Layer - Preserve Message Items on Empty Text

```python
# BEFORE (buggy):
if result.text:          # drops empty string
    emit_message_item(result)

# AFTER (fixed):
if result.text is not None:   # preserves empty string
    emit_message_item(result)

# New diagnostic
if completion_tokens > 0 and not result.text and not result.tool_calls:
    logger.warning("responses.empty_text_with_tokens",
                   completion_tokens=completion_tokens)
```

### 3. Diagnostics

Added three new diagnostic events:
- `bedrock.content_blocks_observed` - every block type + counts per request
- `bedrock.reasoning_content_extracted` - char count + truncated preview

```python
def test_reasoning_only_turn_returns_empty_text_with_diagnostics():
    blocks = [{"reasoningContent": {"text": "internal chain of thought"}}]
    text, diag = _extract_converse_text(blocks)
    assert text == ""
    assert diag["has_reasoning"] is True

def test_guard_content_returns_text():
    blocks = [{"guardContent": {"text": {"text": "I'm sorry..."}}}]
    text, diag = _extract_converse_text(blocks)
    assert text == "I'm sorry..."
    assert diag["has_guard"] is True

def test_empty_text_preserves_message_item():
    response = create_response(text="", completion_tokens=1024, tool_call_count=0)
    assert len(response["output"]) == 1
    assert response["output"][0]["content"] == ""

def test_empty_text_stream_emits_message_events():
    events = list(stream_response(text="", completion_tokens=512))
    event_types = [e["type"] for e in events]
    assert "response.output_item.added" in event_types
```

## Prevention Checklist

- [ ] Bedrock provider handles `text`, `reasoningContent`, `guardContent`, `toolUse`, and unknown blocks.
- [ ] `reasoningContent` is logged but never returned as user-visible text.
- [ ] `guardContent` replacement text IS returned to the user.
- [ ] Empty `text` blocks are preserved (`if result.text is not None:`).
- [ ] Diagnostics are emitted for every block type combination.
- [ ] A warning fires when `completion_tokens > 0` and `text == ""` and no tool calls.

## Prompt For Copilot

```
Review the Bedrock Converse provider for:
1. Does it only read block["text"] and skip reasoningContent/guardContent?
   If yes → "CRITICAL: Empty Bedrock response - content blocks not handled"
2. Does `if result.text:` skip empty text instead of `if result.text is not None:`?
   If yes → "CRITICAL: Message item dropped on empty text"
3. No diagnostic for empty-text-with-tokens?
   → "WARNING: No diagnostic for silent empty responses"
```

## Related Incidents

- [INC-004 - ToolResult Ordering](./INC-004-toolresult-ordering.md): Both are Bedrock Converse protocol failures.
- [INC-001 - Goal Loop](./INC-001-goal-loop.md): Empty responses cause retries, amplifying the loop.

## Scanner Rule

Maps to scanner rules **R-14: Bedrock Schema Drift Risk** and **R-15: Empty-Response Risk**.

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->

- `bedrock.empty_text_with_tokens` - the exact c18b10e9 failure signature

## Validation

### Manual validation

1. **Reasoning-only turn**: Confirm `text=""` is returned but diagnostics log the reasoning content.
2. **Guardrail turn**: Confirm `guardContent.text` is returned to the user.
3. **Empty-text response**: SSE stream emits `output_item.added` + `content_block.done` even when text is empty.

### Automated validation

```bash
python -m pytest tests/test_provider_quirks.py tests/test_api.py -v
```

## Regression Tests

See `prevention-scanner/tests/test_incident_003.py`:

  → Provider.bedrock.converse()
  → _extract_converse_text(content_blocks)     # BUG: skips reasoningContent
  → if result.text:                             # BUG: skips empty text
      emit_message_item()
  → response = {text: "", tool_calls: []}
  → SSE: response.created + response.completed (no output items)
```

### File locations

| Component | File | Function |
|---|---|---|
| Provider | `app/providers/bedrock.py` | `_converse_claude()`, `_extract_text()` |
| API layer | `app/api/v1/responses.py` | `create_response()` |
| API layer | `app/api/v1/chat.py` | `chat_completions()` |
