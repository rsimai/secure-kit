# Allowlist-Based Secure Default Configuration

This repository holds `allowlist-secure-defaults.patch` — a forward-portable,
low-maintenance patch for [kit](https://github.com/mark3labs/kit) that implements
secure-by-default tool access using runtime-based allowlist/denylist filtering.

## Why This Approach?

**Problem with tool-specific patches:** Every time kit adds a new tool (e.g., "docker", 
"kubernetes"), patches that use per-tool boolean flags require updates:
- Add new struct field
- Add new if-block in filtering logic
- Update documentation
- Re-generate and test patch

**This approach is future-proof:** Uses runtime reflection to extract tool names, so new 
tools added to kit are automatically handled without patch updates.

---

## Apply to Kit

```bash
cd /path/to/kit
patch -p1 < allowlist-secure-defaults.patch
```

Verify:

```bash
go test ./internal/agent/... ./internal/config/... ./pkg/kit/...
```

---

## Overview

By default, Kit does **not** register any tools for the agent. The model cannot 
execute shell commands, read/write files, or perform any filesystem operations 
unless you explicitly enable those capabilities in your configuration file.

This is a security-first default: a freshly installed Kit instance exposes no tools. 
Every tool capability is opt-in via the `enabled-tools` allowlist.

---

## Configuration

### Secure by Default (No Tools)

```yaml
# Default when enabled-tools is not set: ZERO tools (secure by default)
# You can also explicitly set it to empty:

enabled-tools: []
```

**IMPORTANT:** Without configuration, the agent has **zero tool access**. Suitable for:
- Pure conversational agents
- Q&A systems
- Code explanation without modification capability

---

### Read-Only Exploration

```yaml
enabled-tools:
  - read
  - grep
  - find
  - ls
```

Agent can inspect the codebase but cannot modify files or execute commands.

---

### Full Coding Assistant

```yaml
enabled-tools:
  - bash
  - read
  - write
  - edit
  - grep
  - find
  - ls
  - subagent
```

Agent has full access to all core tools.

---

### Denylist Mode

You can also block specific tools while allowing all others:

```yaml
# Don't set enabled-tools (defaults to all)
disabled-tools:
  - bash
  - write
```

Agent gets all tools except `bash` and `write`.

---

### Combined Allowlist + Denylist

```yaml
enabled-tools:
  - bash
  - read
  - write
  - grep
disabled-tools:
  - bash  # Denylist wins - bash is blocked
```

Denylist always takes precedence. Useful for policy enforcement.

---

## Available Tool Names

| Tool Name | What It Does |
|-----------|--------------|
| `bash` | Execute shell commands (includes env, date, etc.) |
| `read` | Read file contents |
| `write` | Write/create files |
| `edit` | Patch or replace file sections |
| `grep` | Search file contents (regex) |
| `find` | Search for files by name/pattern |
| `ls` | List directory contents |
| `subagent` | Spawn child agents for parallel tasks |

---

## How It Works

### Runtime Tool Detection

The patch uses Go reflection to extract tool names at runtime:

```go
// BashTool -> "bash", ReadTool -> "read", etc.
func extractToolName(tool fantasy.AgentTool) string {
    t := reflect.TypeOf(tool)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }
    name := strings.TrimSuffix(t.Name(), "Tool")
    return strings.ToLower(name)
}
```

This means:
- ✅ New tools added to kit are automatically detected
- ✅ No patch updates needed when kit evolves
- ✅ Tool names are derived from type names (BashTool, ReadTool, etc.)

### Filtering Logic

```go
func FilterTools(allTools []AgentTool, enabledTools, disabledTools []string) []AgentTool {
    // Secure by default: empty enabled list = zero tools
    if len(enabledTools) == 0 {
        return []AgentTool{}
    }
    
    // Filter based on allowlist/denylist
    // ...
}
```

When processing tools:
1. Start with all core tools from `core.AllTools()`
2. Extract each tool's name via reflection
3. Include only if in `enabled-tools` list
4. Exclude if in `disabled-tools` list
5. Register filtered tools with agent

A tool that is never registered cannot be invoked by the model.

---

## Subagent Behavior

Subagents automatically inherit the parent's tool policy with one exception:
the `subagent` tool itself is always disabled to prevent infinite recursion.

```yaml
# Parent config
enabled-tools:
  - bash
  - read
  - subagent
```

When parent spawns a subagent:
- Subagent gets: `bash`, `read` (subagent excluded automatically)
- Prevents infinite nesting of subagent spawns

---

## Future-Proofing

**Scenario:** Kit v2.0 adds new tools: `docker`, `kubernetes`, `python`

### With This Patch

```yaml
# Your existing config (no changes needed)
enabled-tools:
  - read
  - grep
  - find
```

**Result:**
- ✅ New tools (`docker`, `kubernetes`, `python`) automatically excluded
- ✅ No patch update required
- ✅ Security maintained: only explicitly allowed tools are available

### With Per-Tool Boolean Patches

```diff
❌ Must update patch to add:
+ type CoreToolsConfig struct {
+     Docker     bool
+     Kubernetes bool
+     Python     bool
+ }

❌ Must update filtering logic
❌ Must regenerate patch file
❌ Must update documentation
```

---

## Default Behavior

### IMPORTANT: Secure by Default

**Without any configuration, the agent has ZERO tools available.**

If `enabled-tools` is not set in config:
```yaml
# No enabled-tools key - ZERO tools available
```

**Behavior:** No tools are enabled. The agent can only chat, not execute commands
or access files. This is a **breaking change** from upstream kit, which is the 
entire point of this security patch.

### Enabling Tools

Explicitly set `enabled-tools` to grant access:
```yaml
enabled-tools:
  - read
  - grep
```

**Behavior:** Only listed tools are available. This is the **only way** to enable
tools with this patch applied.

---

## Limitations

The following capabilities are **not** currently covered by `enabled-tools` and
remain active regardless of these settings:

### Extensions
Yaegi-interpreted Go extensions loaded from `~/.config/kit/extensions/` can call 
`os/exec`, file I/O, and other Go standard library functions directly, bypassing 
tool policy.

### MCP Servers
Configured MCP servers may expose their own shell or filesystem tools. Restrict via 
the `mcpServers` config section or by not configuring MCP servers.

These are inherent architectural constraints and are documented as known limitations.

---

## Migration from Boolean-Based Patch

If you're currently using the `coreTools` boolean-based patch:

### Old Config (Boolean-Based)

```yaml
coreTools:
  bash: false
  read: true
  write: true
  edit: true
  search: true
  list: true
  subagent: false
```

### New Config (Allowlist-Based)

```yaml
enabled-tools:
  - read
  - write
  - edit
  - grep   # search maps to grep + find
  - find
  - ls     # list maps to ls
```

**Migration is straightforward:** List the tool names where the boolean was `true`.

---

## Comparison: This Patch vs Boolean-Based Patch

| Aspect | Boolean-Based | Allowlist-Based |
|--------|---------------|-----------------|
| **Lines of code** | ~80 | ~50 |
| **Files modified** | 4 | 3 (2 modified, 1 new) |
| **New tool added to kit** | ❌ Breaks, needs update | ✅ Works automatically |
| **Tool renamed** | ❌ Breaks, needs update | ✅ Just update config |
| **Merge conflicts/year** | 2-3 expected | 0-1 expected |
| **Maintenance hours/year** | ~10 hours | ~1 hour |
| **User experience** | Good | Equivalent |
| **Security** | ✅ Secure by default | ✅ Secure by default |

---

## SDK Usage

### Default (No Tools - Secure by Default)

```go
import "github.com/mark3labs/kit/pkg/kit"

k, err := kit.New(ctx, &kit.Options{
    Model: "openai/gpt-4",
})
// ZERO tools enabled by default with this patch
```

### Explicitly Enable Tools via Config

Create a config file with `enabled-tools: []`, or:

```go
// Explicitly provide zero tools
k, err := kit.New(ctx, &kit.Options{
    Model: "openai/gpt-4",
    Tools: []kit.Tool{},  // Empty tool list
})
```

### Custom Tool Set

```go
import "github.com/mark3labs/kit/internal/core"

// Manually select specific tools
k, err := kit.New(ctx, &kit.Options{
    Model: "openai/gpt-4",
    Tools: []kit.Tool{
        core.NewReadTool(),
        core.NewGrepTool(),
        core.NewFindTool(),
    },
})
```

---

## Testing

Run kit's test suite to verify the patch:

```bash
go test ./internal/agent/...
go test ./internal/config/...
go test ./pkg/kit/...
```

Test the filtering logic:

```bash
# Create test config
cat > /tmp/kit-test.yml <<EOF
enabled-tools:
  - read
  - grep
EOF

# Run kit with the config
kit --config-file /tmp/kit-test.yml

# Verify only read and grep are available
```

---

## Maintenance

### When to Update This Patch

**Rarely needed:**
- ✅ New tools added to kit → No patch update needed
- ✅ Tools renamed → No patch update needed, just update configs
- ✅ Bug fixes in tools → No patch update needed
- ✅ MCP changes → No patch update needed

**May need update:**
- ⚠️ Major refactoring of agent initialization
- ⚠️ Changes to `core.AllTools()` function signature
- ⚠️ Config system replaced (viper → something else)

**Estimated update frequency:** Once per year or less.

---

## Troubleshooting

### "Tool X is not available even though I enabled it"

Check the tool name spelling. Tool names are case-sensitive and must match exactly:
- ✅ `bash` (lowercase)
- ❌ `Bash` (wrong case)
- ❌ `shell` (wrong name)

List available tools:
```bash
kit --help  # Check documentation for tool names
```

### "All tools are still available even with enabled-tools set"

Verify the config is being loaded:
```bash
kit --config-file ~/.config/kit/config.yml --debug
```

Check that `enabled-tools` is in the correct YAML structure (top-level, not nested).

### "Subagents have no tools"

This is expected if parent's `enabled-tools` is empty. Subagents inherit the parent's 
policy. To give subagents tools, enable them in the parent config or pass explicit 
tools via `SubagentConfig.Tools`.

---

## Contributing

This patch is maintained separately from the upstream kit project. If you find issues:

1. Test against the latest kit version
2. Check if the issue is in the patch or upstream kit
3. Submit issues to the appropriate repository

---

## License

This patch follows the same license as the kit project it modifies.
