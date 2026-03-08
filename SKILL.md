---
name: wsl-interop
description: WSL (Windows Subsystem for Linux) path conversion, file access, and interoperability patterns for Claude Code running inside WSL. Use this skill whenever working in a WSL environment and dealing with file paths (converting between Windows and WSL formats), opening files in Windows applications from WSL, SSH agent forwarding (1Password, GPG) via Windows-side executables, clipboard operations, filesystem quirks (sed -i failures on mounted filesystems, CRLF line endings, /mnt/ permission issues), or any Windows-Linux interop task. Trigger when providing file path references the user might want to click or open in Windows, when receiving Windows-formatted paths (C:\, \\wsl$\, \\wsl.localhost\) that need WSL conversion, when the user pastes a screenshot or image with a Windows temp path, or when troubleshooting WSL-specific issues like network access, Docker Desktop integration, or interop executable usage (.exe from WSL). Even if the user doesn't mention "WSL" explicitly, trigger whenever the environment is WSL and the task involves cross-boundary file access or tool usage.
---

# WSL Interop Skill

Patterns and conventions for Claude Code running inside WSL (Windows Subsystem for Linux). This skill handles the friction points at the boundary between the Linux environment where Claude runs and the Windows environment where the user works.

## Core Principle

Claude Code runs in WSL (Linux), but the user interacts through a Windows terminal and uses Windows applications. Every file reference, path, and tool invocation needs to account for this boundary.

## Path Conversion

### The Tool: `wslpath`

`wslpath` is the canonical path conversion utility, available in all modern WSL installations.

| Flag | Direction | Example |
|------|-----------|---------|
| `-w` | WSL → Windows | `/home/user/file.md` → `\\wsl.localhost\Ubuntu\home\user\file.md` |
| `-u` | Windows → WSL | `C:\Users\user\file.md` → `/mnt/c/Users/user/file.md` |
| `-m` | WSL → Windows (forward slash) | `/home/user/file.md` → `//wsl.localhost/Ubuntu/home/user/file.md` |

### When You Receive Windows Paths

The user may paste or reference Windows-formatted paths from screenshots, file dialogs, or Explorer. Convert them before using:

- `C:\Users\me\Downloads\file.pdf` → `/mnt/c/Users/me/Downloads/file.pdf`
- `\\wsl$\Ubuntu\home\...` → strip the UNC prefix to get the native WSL path
- `\\wsl.localhost\Ubuntu\home\...` → same treatment

Use `wslpath -u` for programmatic conversion. For simple `/mnt/c/` mappings, direct string substitution is fine.

### When You Reference Files (Clickable Paths)

When referencing a file the user might want to open, view, or navigate to, **provide a clickable `file:` URI** so they can ctrl+click to open it.

**Note:** Claude Code's terminal does NOT render markdown link syntax (`[text](url)`) — it strips the brackets and shows the raw URL. So always output the bare URI, not a markdown link.

**Format — WSL-native files** (`/home/...`, `/tmp/...`):
Use a `file:` URI with **4 slashes** after `file:` and **forward slashes** for the path:

```
Created the output file:
file:////wsl.localhost/Ubuntu/home/user/project/output.pdf
```

The general pattern is:
```
file:////wsl.localhost/Ubuntu<wsl-absolute-path-with-forward-slashes>
```

**Format — Windows filesystem files** (`/mnt/c/...`, `/mnt/d/...`):
No clickable file URI works for Windows drive paths. Convert to the native drive letter and present as plain text:

```
The file is at C:\Users\me\Documents\report.pdf
```

Conversion: strip `/mnt/c/` → `C:\`, `/mnt/d/` → `D:\`, and replace `/` with `\`.

**When to do this:**
- Files the user asked you to create or modify
- Documents, images, or artifacts the user will want to open
- Log files or reports the user should review
- Any path the user might want to click to navigate to

**When NOT to do this:**
- Internal code references during development (source files you're editing back and forth)
- Paths in code, configs, or scripts (those must stay as WSL paths)
- Inline `file_path:line_number` references (standard Claude Code format for code navigation)

### When You Reference URLs and Hostnames

**Always use full URLs with a scheme** — never bare hostnames. The terminal auto-links `https://` URLs but bare hostnames are not clickable.

```
# BAD — not clickable:
Check the dashboard at monitor.example.com

# GOOD — clickable:
Check the dashboard at https://monitor.example.com
```

For internal services, always prefix with the appropriate scheme:
```
Service dashboard at https://monitor.example.com
Router admin at https://192.168.1.1
Non-HTTPS service at http://192.168.1.50:3001
```

This applies to all hostnames, IPs with ports, and service references mentioned in prose output.

### Quick Reference for Manual Conversion

| WSL Path | Windows Path |
|----------|-------------|
| `/mnt/c/...` | `C:\...` |
| `/mnt/d/...` | `D:\...` |
| `/home/<user>/...` | `\\wsl.localhost\<Distro>\home\<user>\...` |
| `/tmp/...` | `\\wsl.localhost\<Distro>\tmp\...` |

## Opening Files in Windows Applications

To open a file from WSL in the default Windows application:

```bash
# Open in default app
explorer.exe "$(wslpath -w /path/to/file)"

# Or use cmd.exe
cmd.exe /c start "" "$(wslpath -w /path/to/file)"

# Open a directory in Explorer
explorer.exe "$(wslpath -w /path/to/directory)"

# Open a URL in the default browser
cmd.exe /c start "" "https://example.com"
```

Specific applications (when the user asks to open in a particular app):
```bash
# VS Code (has native WSL support)
code /path/to/file

# Notepad
notepad.exe "$(wslpath -w /path/to/file)"
```

## SSH Agent Interop

When the SSH agent runs on the Windows side (common with 1Password, GPG4Win, or Windows OpenSSH), use the Windows SSH executable:

```bash
# Use ssh.exe instead of ssh
ssh.exe user@host

# SCP also needs the .exe suffix
scp.exe file user@host:/path

# Git operations that need SSH auth
GIT_SSH_COMMAND="ssh.exe" git push
```

The reason: WSL's native `ssh` cannot access the Windows SSH agent socket. Using `ssh.exe` invokes Windows OpenSSH which connects to the Windows-side agent (1Password, Pageant, etc.).

**Detection:** If `SSH_AUTH_SOCK` is empty/unset in WSL but `ssh.exe` works, the agent is on the Windows side.

## Clipboard Interop

```bash
# Copy to Windows clipboard
echo "text" | clip.exe
cat file.txt | clip.exe

# Paste from Windows clipboard
powershell.exe -command "Get-Clipboard"

# Copy an image/binary to clipboard (PowerShell)
powershell.exe -command "Set-Clipboard -Path '$(wslpath -w /path/to/image.png)'"
```

## Filesystem Quirks

### `sed -i` on Mounted Filesystems

`sed -i` (in-place edit) may fail on certain filesystem mounts (9P/Plan 9 mounts, some network shares, and the HA host filesystem). The failure is silent or produces permission errors.

**Workaround:**
```bash
cat /path/to/file | sed 's/old/new/' > /tmp/edited && cat /tmp/edited > /path/to/file
```

Or use the Edit tool instead of sed, which handles this transparently.

### Line Endings (CRLF vs LF)

Files created on Windows use `\r\n` (CRLF). Linux tools expect `\n` (LF). This causes issues with:
- Shell scripts (bash fails with `\r` in shebangs)
- Git diffs showing `^M` characters
- Config files parsed by Linux tools

**Fix:**
```bash
# Convert CRLF to LF
dos2unix file.sh
# Or with sed
sed -i 's/\r$//' file.sh
```

**Prevent:** Configure Git to handle this:
```bash
git config --global core.autocrlf input
```

### Permissions on /mnt/ Drives

Windows drives mounted at `/mnt/c/`, `/mnt/d/` etc. don't fully support Linux permissions. `chmod` appears to work but has no effect by default.

To enable proper permissions (if needed), add to `/etc/wsl.conf`:
```ini
[automount]
options = "metadata"
```

Then restart WSL. Usually not needed — most workflows work fine without this.

## Network Access

### Accessing Windows Services from WSL

Windows services (localhost:3000, etc.) are accessible from WSL via:
- `localhost` (WSL2 with mirrored networking, or WSL1)
- The Windows host IP: `$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')` (WSL2 NAT mode)

### Accessing WSL Services from Windows

Services running in WSL are accessible from Windows via `localhost` in most configurations. If not:
```bash
# Check WSL IP
ip addr show eth0 | grep "inet " | awk '{print $2}' | cut -d/ -f1
```

## Docker Desktop Integration

Docker Desktop for Windows can share its daemon with WSL. If Docker commands fail in WSL:
1. Open Docker Desktop → Settings → Resources → WSL Integration
2. Enable integration for your distro
3. No need to install Docker separately in WSL

## Troubleshooting Interop

### `.exe` Commands Not Found
Windows executables must include the `.exe` suffix in WSL. If `explorer.exe` or `cmd.exe` isn't found:
```bash
# Check if interop is enabled
cat /proc/sys/fs/binfmt_misc/WSLInterop

# Check PATH includes Windows paths
echo $PATH | tr ':' '\n' | grep -i windows
```

### Slow File Access on /mnt/
File operations on `/mnt/c/` are significantly slower than on the native WSL filesystem (`/home/...`). For performance-sensitive work (git repos, node_modules, build artifacts), keep files in the WSL filesystem.
