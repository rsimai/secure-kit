# Secure Default Tool Configuration

This repository holds `secure-default-core-tools.patch` — a forward-portable
patch for [kit](https://github.com/mark3labs/kit) that disables bash, filesystem,
and subagent tools by default. Users opt in through their config file.

Apply to a kit source tree:

```bash
patch -p1 < secure-default-core-tools.patch
```

Verify after applying:

```bash
go test ./internal/agent/... ./internal/config/... ./internal/core/... ./pkg/kit/...
```

---

## Overview

By default, Kit does **not** register any bash or filesystem tools for the agent.
This means the model cannot execute shell commands or read/write files unless you
explicitly enable those capabilities in your configuration file.

This is a security-first default: a freshly installed Kit instance exposes no tools. 
Every tool capability is opt-in.

## System prompt

Kit previously shipped with a hardcoded default system prompt that described
a fixed set of tools as always-available (bash, read, write, edit, grep, find,
ls). This was misleading once tool access became configurable — the model would
believe it had tools that were not registered.

The default system prompt has been removed. Kit now starts with no system prompt
unless the user provides one via:

- The `--system-prompt` CLI flag (text or file path)
- The `system-prompt` key in `~/.config/kit/config.yml`
- The `Options.SystemPrompt` field in the SDK

When no system prompt is set, the model receives only runtime context (current
date/time, working directory, and any `AGENTS.md` / skills found on disk).
This is accurate regardless of which tools are enabled.

## Affected tools

| Tool       | What it does                              | Config key             |
|------------|-------------------------------------------|------------------------|
| `bash`     | Execute shell commands (incl. env, date)  | `coreTools.bash`       |
| `read`     | Read file contents                        | `coreTools.read`       |
| `write`    | Write/create files                        | `coreTools.write`      |
| `edit`     | Patch or replace file sections            | `coreTools.edit`       |
| `grep`     | Search file contents (regex)              | `coreTools.search`     |
| `find`     | Search for files by name/pattern          | `coreTools.search`     |
| `ls`       | List directory contents                   | `coreTools.list`       |
| `subagent` | Spawn a child agent for parallel tasks    | `coreTools.subagent`   |

`grep` and `find` share the `search` key because they serve the same purpose
(exploring the filesystem without modifying it).

## Enabling tools

Add a `coreTools` section to `~/.config/kit/config.yml`:

```yaml
coreTools:
  bash: true      # Allow executing shell commands
  read: true      # Allow reading files
  write: true     # Allow writing files
  edit: true      # Allow editing files (patch/replace)
  search: true    # Allow grep/find operations
  list: true      # Allow listing directory contents
```

Set only the keys you need. Omitted keys default to `false` (disabled).

### Common profiles

**Read-only exploration** — let the agent inspect the codebase but not change anything:
```yaml
coreTools:
  read: true
  search: true
  list: true
```

**Coding assistant** — read, write, edit, and run tests:
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

**Chat only** — no tool access at all (default, no config needed):
```yaml
# coreTools section omitted or all values false
```

## How it works

The `coreTools` config block is read in `internal/config/config.go` as
`CoreToolsConfig`. When Kit starts, `internal/agent/agent.go` calls
`coreToolsForPolicy()` which builds the tool list from the policy booleans.
Only tools whose corresponding field is `true` are registered with the agent.
A tool that is never registered cannot be invoked by the model.

## Limitations

The following capabilities are **not** currently covered by `coreTools` and
remain active regardless of these settings:

- **Extensions** — Yaegi-interpreted Go extensions loaded from
  `~/.config/kit/extensions/` can call `os/exec`, file I/O, and other Go
  standard library functions directly, bypassing tool policy.
- **MCP servers** — configured MCP servers may expose their own shell or
  filesystem tools. Restrict via the `mcpServers` config section or by not
  configuring MCP servers.
