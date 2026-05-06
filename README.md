# Secure KIT Patch

This directory contains a forward-portable patch that changes KIT to a secure-by-default posture for core tools.

## Files

- `0001-secure-default-core-tools.mbox.patch`: `git format-patch` (mbox) patch.

## What This Patch Does

- Disables bash core tool by default.
- Disables filesystem core tools by default (`read`, `write`, `edit`, `grep`, `find`, `ls`).
- Adds granular config controls under `coreTools`:
  - `coreTools.bash.enabled`
  - `coreTools.bash.allowedCommands`
  - `coreTools.bash.blockedCommands`
  - `coreTools.filesystem.read`
  - `coreTools.filesystem.write`
  - `coreTools.filesystem.edit`
  - `coreTools.filesystem.search`
  - `coreTools.filesystem.list`
  - `coreTools.filesystem.allowedPaths`

## Apply To KIT

From your KIT repository root:

```bash
git am --3way ~/git/secure-kit/0001-secure-default-core-tools.mbox.patch
```

Or with an explicit repo path:

```bash
git -C ~/git/kit am --3way ~/git/secure-kit/0001-secure-default-core-tools.mbox.patch
```

## If Applying To Newer Versions Fails

Abort `git am` and use 3-way apply with rejects:

```bash
git am --abort
git apply --3way --reject ~/git/secure-kit/0001-secure-default-core-tools.mbox.patch
```

Then resolve `*.rej`, run tests, and commit manually.

## Verify After Apply

```bash
go test ./internal/core ./internal/config ./internal/agent
```

## Example Config

```yaml
coreTools:
  bash:
    enabled: true
    allowedCommands: ["go", "git", "ls", "cat"]
    blockedCommands: ["rm", "sudo"]
  filesystem:
    read: true
    write: false
    edit: false
    search: true
    list: true
    allowedPaths:
      - "./"
      - "./docs"
```
