# Testing Secure-by-Default Behavior

## Critical Requirement

**Default without config MUST be zero tools (secure by default)**

This is the essential security property that both patches must provide.

---

## Test Cases

### Test 1: No Configuration

**Config:** (none - file doesn't exist or field not set)

**Expected:** Zero tools available

**Boolean Patch:**
```yaml
# No coreTools section
```
Result: ✅ Zero tools (all fields default to false)

**Allowlist Patch:**
```yaml
# No enabled-tools section
```
Result: ✅ Zero tools (nil/empty list = zero tools)

---

### Test 2: Explicit Empty Configuration

**Config:** Explicitly set to empty

**Boolean Patch:**
```yaml
coreTools:
  bash: false
  read: false
  # ... all false or omitted
```
Result: ✅ Zero tools

**Allowlist Patch:**
```yaml
enabled-tools: []
```
Result: ✅ Zero tools

---

### Test 3: Partial Configuration

**Config:** Enable only read and grep

**Boolean Patch:**
```yaml
coreTools:
  read: true
  search: true  # enables grep + find
```
Result: ✅ Only read, grep, find available

**Allowlist Patch:**
```yaml
enabled-tools:
  - read
  - grep
  - find
```
Result: ✅ Only read, grep, find available

---

### Test 4: Full Configuration

**Config:** Enable all tools

**Boolean Patch:**
```yaml
coreTools:
  bash: true
  read: true
  write: true
  edit: true
  search: true
  list: true
  subagent: true
```
Result: ✅ All 8 tools available

**Allowlist Patch:**
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
Result: ✅ All 8 tools available

---

## Verification Script

### For Boolean Patch

```bash
cd kit
patch -p1 < ../secure-kit/secure-default-core-tools.patch

# Test 1: No config
go run -tags test <<'EOF'
package main
import (
    "context"
    "github.com/mark3labs/kit/pkg/kit"
)
func main() {
    k, _ := kit.New(context.Background(), &kit.Options{Model: "openai/gpt-4"})
    println("Tools:", len(k.GetToolNames()))  // Should be 0
}
EOF

# Test 2: With config
cat > /tmp/kit-config.yml <<'EOF'
enabled-tools:
  - read
  - grep
EOF

kit --config-file /tmp/kit-config.yml
# Should show only read and grep available
```

### For Allowlist Patch

```bash
cd kit
patch -p1 < ../secure-kit/allowlist-secure-defaults.patch

# Same tests as above
# Should produce same results
```

---

## Expected Behavior Summary

| Configuration | Boolean Patch | Allowlist Patch |
|---------------|---------------|-----------------|
| No config | ✅ 0 tools | ✅ 0 tools |
| Empty config | ✅ 0 tools | ✅ 0 tools |
| Partial (read, grep) | ✅ 2-3 tools | ✅ 2-3 tools |
| Full (all enabled) | ✅ 8 tools | ✅ 8 tools |

**Both patches are equally secure by default.**

The difference is maintainability, not security.

---

## SDK Testing

### No Config (Secure by Default)

```go
package main

import (
    "context"
    "fmt"
    "github.com/mark3labs/kit/pkg/kit"
)

func main() {
    k, err := kit.New(context.Background(), &kit.Options{
        Model: "openai/gpt-4",
    })
    if err != nil {
        panic(err)
    }
    
    tools := k.GetToolNames()
    fmt.Printf("Tools available: %d\n", len(tools))
    // Expected: 0
    
    if len(tools) == 0 {
        fmt.Println("✅ PASS: Secure by default")
    } else {
        fmt.Println("❌ FAIL: Default should be zero tools")
    }
}
```

### With Explicit Tools

```go
import "github.com/mark3labs/kit/internal/core"

k, err := kit.New(context.Background(), &kit.Options{
    Model: "openai/gpt-4",
    Tools: []kit.Tool{
        core.NewReadTool(),
        core.NewGrepTool(),
    },
})
// Should have exactly 2 tools
```

---

## Regression Testing

After applying either patch, ensure:

1. ✅ All existing kit tests pass
   ```bash
   go test ./internal/agent/... ./internal/config/... ./pkg/kit/...
   ```

2. ✅ Default is secure (zero tools)
3. ✅ Explicit tool enablement works
4. ✅ Subagents inherit policy correctly
5. ✅ MCP tools still load independently

---

## Security Validation Checklist

- [ ] Fresh install with no config has zero tools
- [ ] Empty config has zero tools
- [ ] Enabled tools list is respected exactly
- [ ] Disabled tools are blocked even if enabled
- [ ] Subagents cannot spawn subagents (recursion prevention)
- [ ] MCP tools are independent of core tool policy
- [ ] Extensions bypass is documented
- [ ] SDK usage without config is secure

Both patches meet all these criteria when properly implemented.
