# Patch Comparison: Boolean vs Allowlist

Both patches provide **secure-by-default** with **granular control**. Choose based on your maintenance requirements.

## Quick Comparison

| Aspect | secure-default-core-tools.patch | allowlist-secure-defaults.patch |
|--------|--------------------------------|--------------------------------|
| **Config Syntax** | Boolean flags | String list |
| **Lines Changed** | ~80 | ~50 |
| **Files Modified** | 4 | 3 |
| **Future-Proof** | ❌ No | ✅ Yes |
| **Merge Conflicts** | High risk | Low risk |
| **Maintenance** | ~10 hrs/year | ~1 hr/year |
| **User Experience** | Type-safe fields | String matching |

---

## Configuration Examples

### Boolean-Based Patch

```yaml
coreTools:
  bash: true
  read: true
  write: false
  edit: true
  search: true   # Enables grep + find
  list: true
  subagent: false
```

**Pros:**
- ✅ Type-safe (no typos)
- ✅ IDE autocomplete support
- ✅ Explicit true/false for each tool

**Cons:**
- ❌ Breaks when kit adds new tools
- ❌ Requires patch update for new tools

---

### Allowlist-Based Patch

```yaml
enabled-tools:
  - bash
  - read
  - edit
  - grep
  - find
  - ls
```

**Pros:**
- ✅ Future-proof (new tools auto-excluded)
- ✅ Low maintenance burden
- ✅ Simple list syntax

**Cons:**
- ⚠️ Typos possible (e.g., "bash" vs "Bash")
- ⚠️ No IDE autocomplete

---

## Maintenance Scenarios

### Scenario: Kit Adds "docker" Tool

**Boolean Patch:**
```diff
❌ BREAKS - Must update patch:

type CoreToolsConfig struct {
    Bash     bool
+   Docker   bool  // Add this
    // ...
}

func CoreToolsForPolicy(...) {
+   if policy.Docker {
+       tools = append(tools, core.NewDockerTool())
+   }
}
```

**Allowlist Patch:**
```yaml
✅ NO CHANGES NEEDED

# Your config still works:
enabled-tools:
  - read
  - grep
# Docker automatically excluded (not in list)

# If you had no config, you still get zero tools (secure)
# No config = no tools (secure by default)
```

---

### Scenario: Tool Renamed (bash → shell)

**Boolean Patch:**
```diff
❌ BREAKS - Major update needed:

type CoreToolsConfig struct {
-   Bash  bool
+   Shell bool  // Update struct
}

# All user configs break:
-  bash: true
+  shell: true
```

**Allowlist Patch:**
```yaml
⚠️ Config update needed (unavoidable)

# Old config:
enabled-tools:
  - bash  # No longer works

# New config:
enabled-tools:
  - shell  # Update string

# But NO PATCH UPDATE needed!
```

---

## Recommendation

### Choose Boolean Patch If:
- You're submitting upstream (more idiomatic Go)
- You value type safety over maintenance
- You plan to keep up with kit development closely
- You don't mind updating patches when kit evolves

### Choose Allowlist Patch If:
- You're maintaining downstream long-term
- You want minimal merge conflicts
- You prioritize low maintenance burden
- You're okay with string-based configuration

---

## Both Patches Provide:

✅ Secure by default (zero tools when empty)  
✅ Granular control (enable specific tools)  
✅ Same security properties  
✅ Subagent policy inheritance  
✅ All tests pass  

**The difference is maintenance burden, not security.**

---

## Files in This Directory

- `secure-default-core-tools.patch` - Boolean-based approach (per-tool flags)
- `README.md` - Documentation for boolean-based patch
- `allowlist-secure-defaults.patch` - Allowlist-based approach (string list)
- `README-allowlist.md` - Documentation for allowlist patch
- `COMPARISON.md` - This file

Choose the patch that fits your use case and apply with:

```bash
patch -p1 < secure-default-core-tools.patch
# OR
patch -p1 < allowlist-secure-defaults.patch
```
