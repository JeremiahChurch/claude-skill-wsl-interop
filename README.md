# wsl-interop

A [Claude Code](https://claude.ai/code) skill for seamless Windows Subsystem for Linux (WSL) interoperability.

## What It Does

When Claude Code runs inside WSL, there's constant friction at the Windows/Linux boundary — paths don't translate, SSH agents aren't accessible, clipboard doesn't bridge, `sed -i` silently fails. This skill teaches Claude to handle all of it automatically:

- **Clickable file links** — References WSL files as `file:////wsl.localhost/...` URIs in markdown links, so you can ctrl+click to open them directly from Claude Code's output (tested in Windows Terminal)
- **Clickable URLs** — Always outputs full `https://` URLs instead of bare hostnames, so service references are clickable
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

## Usage

The skill triggers automatically whenever Claude Code detects WSL-related scenarios:
- You paste a Windows path
- Claude references a file you might want to open
- SSH or clipboard operations cross the WSL boundary
- Filesystem operations hit WSL-specific quirks

No slash command needed — it's ambient.

## Requirements

- WSL (any version, WSL2 recommended)
- `wslpath` (included in all modern WSL installations)

## Updating Your Global CLAUDE.md

If you have WSL-specific instructions in `~/.claude/CLAUDE.md`, you can simplify them since the skill now handles those patterns. For example, you can replace detailed path conversion instructions with:

```markdown
## Environment
- Running inside WSL. The `wsl-interop` skill handles path conversion and Windows interop.
```

## License

MIT
