# wsl-interop

A [Claude Code](https://claude.ai/code) skill for seamless Windows Subsystem for Linux (WSL) interoperability.

## What It Does

When Claude Code runs inside WSL, there's constant friction at the Windows/Linux boundary — paths don't translate, SSH agents aren't accessible, clipboard doesn't bridge, `sed -i` silently fails. This skill teaches Claude to handle all of it automatically:

- **Clickable file links** — References WSL files as `file:////wsl.localhost/...` URIs, so you can ctrl+click to open them directly from Claude Code's output (tested in Windows Terminal)
- **Clickable URLs** — Always outputs full `https://` URLs instead of bare hostnames, so service references are clickable
- **Open files directly** — For files where clickable URIs don't work (Windows drive paths, Google Drive, network shares), offers to open them via `cmd.exe /c start` in their default Windows app. Safe by design: Claude Code's tool approval + Windows ShellExecute protections
- **Bidirectional path conversion** — Converts Windows paths (`C:\Users\...`) to WSL paths on input, and provides Windows-accessible paths when referencing files
- **SSH agent interop** — Uses `ssh.exe` instead of `ssh` when the SSH agent (1Password, GPG4Win) runs on the Windows side
- **Clipboard bridging** — `clip.exe` for copy, `powershell.exe Get-Clipboard` for paste
- **Filesystem quirk handling** — `sed -i` workarounds, CRLF/LF line ending fixes, `/mnt/` permission awareness
- **Opening files in Windows apps** — `explorer.exe`, `code`, `notepad.exe` patterns from WSL
- **Network access patterns** — Reaching Windows services from WSL and vice versa
- **Docker Desktop integration** — WSL2 backend configuration

## Installation

### Option 1: Claude Code Plugin (recommended)

```
/plugin install wsl-interop
```

### Option 2: Vercel Skills CLI

```bash
npx skills add JeremiahChurch/claude-skill-wsl-interop
```

### Option 3: Git Clone

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/JeremiahChurch/claude-skill-wsl-interop.git ~/.claude/skills/wsl-interop
```

### Option 4: Manual

Copy `SKILL.md` into `~/.claude/skills/wsl-interop/SKILL.md`.

## Important: Add a CLAUDE.md Directive

Claude Code's automatic skill triggering does **not** reliably fire for this skill. Tasks like "where's my file?" or "ssh isn't working" are seen as simple enough to handle without consulting a skill — so the description-based trigger never activates.

**You must add a directive to your global `~/.claude/CLAUDE.md`** to ensure the skill is loaded when it matters:

```markdown
## Environment
- Always running inside WSL. When your response will include file paths, URLs/hostnames, or SSH commands, invoke the `wsl-interop` skill and follow its output formatting rules. This applies every time — not just when the user mentions WSL or paths.
```

Without this, the skill will only activate if you explicitly invoke it or happen to mention WSL-specific keywords.

## Requirements

- WSL (any version, WSL2 recommended)
- `wslpath` (included in all modern WSL installations)

## License

MIT
