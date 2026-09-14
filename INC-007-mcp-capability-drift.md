# INC-007 - MCP Capability Drift

| Field | Detail |
|---|---|
| **Incident** | #007 |
| **Title** | MCP Capability Drift |
| **Tagline** | Tool advertised but unavailable |
| **Severity** | Medium |
| **Date Documented** | 2026-09-14 |
| **Provider** | MCP servers / LangGraph |
| **Tags** | mcp, capability drift, runtime validation |

---

## Symptoms

- A tool is **advertised** as available (registered in the MCP server registry) but is **unavailable at runtime**.
- The agent attempts to call a tool that doesn't exist - gets a `tool_not_found` error.
- Dynamic tool loading failed silently - the MCP server started but some tools didn't register.
- Capabilities listed in `tools/list` response don't match what the server actually supports.

## Evidence

```
MCP Server: filesystem-server v1.2.0
  Registry says: ["read_file", "write_file", "list_dir", "delete_file"]
  Runtime actually loaded: ["read_file", "write_file", "list_dir"]
  → "delete_file" in registry, NOT in runtime

Agent attempt:
  → tool_call: delete_file(path="/tmp/test")
  → error: Tool "delete_file" not found
```

## Root Cause

**Runtime capabilities differ from registry**: The MCP server's tool registry (static list) does not match the actual runtime capabilities (dynamic, based on environment, permissions, or initialization failures).

When an MCP server starts:
1. It advertises all tools in its manifest.
2. At runtime, some tools may fail to initialize (missing dependencies, permission errors, environment variables).
3. These failures were not detected - the tools appeared available but were dead.
4. The agent would attempt to use them, get a confusing `tool_not_found` error.

## Code Path

```
mcp_server.start()
  → load_tools_from_plugins()         # some fail silently

## Fix

### 1. Startup MCP Validation

```python
class MCPCapabilityValidator:
    """Validates MCP server capabilities at startup."""

    def __init__(self, mcp_client: MCPClient):
        self.client = mcp_client

    async def validate(self) -> list[CapabilityViolation]:
        """
        Compare the server's advertised capabilities (from tools/list)
        against the actual runtime capabilities (what we can actually call).
        """
        violations = []

        # Get advertised tools
        advertised = await self.client.list_tools()  # from tool_registry

        # Get actual runtime capabilities
        actual = await self.client.list_runtime_tools()

        advertised_names = {t["name"] for t in advertised}
        actual_names = {t["name"] for t in actual}

        # Check 1: advertised but not available at runtime
        phantom_tools = advertised_names - actual_names
        for name in phantom_tools:
            violations.append(CapabilityViolation(
                tool_name=name,
                type="phantom_tool",
                message=f"Tool '{name}' advertised but unavailable at runtime"
            ))

        # Check 2: available but not in the manifest
        ghost_tools = actual_names - advertised_names
        for name in ghost_tools:
            violations.append(CapabilityViolation(
                tool_name=name,
                type="ghost_tool",
                message=f"Tool '{name}' registered at runtime but missing from manifest"
            ))

        return violations
```

### 2. Validation Hook

```python
# In MCP server startup:
async def on_server_ready(server: MCPServer):
    validator = MCPCapabilityValidator(server.client)
    violations = await validator.validate()

    if violations:
        for v in violations:
            logger.error("mcp.capability_drift",
                         tool=v.tool_name,
                         type=v.type,
                         message=v.message)

        # Fail startup if critical violations
        critical = [v for v in violations if v.severity == "critical"]
        if critical:
            raise MCPServerStartupError(
                f"{len(critical)} capability violations detected. "
                f"Server refusing to start."
            )

        # Warn on non-critical
        logger.warning("mcp.capability_drift_warning",
                       count=len(violations))
```

## Validation

### Manual validation

1. **Startup check**: Start the MCP server with a deliberately missing tool. Confirm it refuses to start.
2. **Capability mismatch**: Add a tool to the manifest but don't implement it. Confirm startup validation catches it.
3. **Runtime verification**: Call `tools/list` and `tools/call` in the same session. Confirm they agree.

### Automated validation

```bash
python -m pytest tests/test_mcp_capability_validator.py -v
```

## Regression Tests

See `prevention-scanner/tests/test_incident_007.py`:

```python
def test_phantom_tools_detected():
    """Tools in manifest but not at runtime are flagged."""
    validator = MCPCapabilityValidator(mock_client_with_phantom_tools())
    violations = validator.validate()
    ghost = [v for v in violations if v.type == "phantom_tool"]
    assert len(ghost) >= 1

def test_ghost_tools_detected():
    """Tools at runtime but not in manifest are flagged."""
    validator = MCPCapabilityValidator(mock_client_with_ghost_tools())
    violations = validator.validate()
    ghost = [v for v in violations if v.type == "ghost_tool"]
    assert len(ghost) >= 1

def test_no_violations_when_aligned():
    """When manifest and runtime match, no violations."""
    validator = MCPCapabilityValidator(mock_client_aligned())
    violations = validator.validate()
    assert len(violations) == 0
```

## Prevention Checklist

- [ ] MCP servers validate capabilities at startup (manifest vs runtime).
- [ ] Servers that fail validation refuse to start (fail-closed).
- [ ] Non-critical violations are logged with structured diagnostics.
- [ ] `tools/list` and `tools/call` responses are cross-verified at runtime.
- [ ] CI runs capability validation on every MCP server change.
- [ ] A health check endpoint reflects capability alignment status.

## Prompt For Copilot

```
You are reviewing an MCP server implementation. Check:

1. Does the server validate its runtime tool capabilities against its manifest
   at startup?
   If not → "MEDIUM: Missing MCP capability validation - tools advertised but
   unavailable"

2. Does the server have a tools/list handler that matches actual available tools?
   If not → "WARNING: tools/list may not reflect runtime capabilities"

3. Are capability drift violations logged with structured diagnostics?
   If not → "WARNING: No structured logging for MCP capability drift"

4. Does the server fail-closed (refuse to start) when critical tools are missing?
   If not → "WARNING: MCP server starts with missing tools - fail-open"

Output format:
MEDIUM: <issue> in <file>
WARNING: <issue> in <file>
OK: <check>
```

## Related Incidents

- [INC-008 - Prompt Injection](./INC-008-prompt-injection.md): Ghost tools can be weaponized for injection.
- [INC-001 - Goal Loop](./INC-001-goal-loop.md): MCP tool drift causes tool-not-found errors that amplify loops.

## Scanner Rule

This incident maps to scanner rule **R-06: Missing MCP Validation**.

<!-- AI-FOOTPRINT: TOOL=Cline | DATE=2026-09-14 -->

  → register_all_in_manifest()        # registers ALL, including failed ones
  → tools/list returns: [read, write, list, delete]  # delete is a ghost
  → agent calls delete_file()
  → MCP error: tool_not_found
```

### File locations

| Component | File | Function |
|---|---|---|
| MCP server | `app/mcp/server.py` | `start()`, `list_tools()` |
| Tool loading | `app/mcp/tools/__init__.py` | `load_tools()` |
| Validation | `app/mcp/validator.py` | `validate_tools()` |
| Registry | `app/mcp/registry.json` | static tool definitions |
