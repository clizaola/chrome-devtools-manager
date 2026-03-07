# chrome-devtools-manager

A command-line tool to manage [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp) across multiple projects. It handles launching isolated Chrome instances, managing port assignments, detecting conflicts, and configuring projects automatically.

Built for developers who work on multiple projects simultaneously and need Chrome DevTools MCP running in each one without port collisions.

## The Problem

When using Chrome DevTools MCP with tools like [Claude Code](https://claude.com/claude-code), each project needs its own Chrome instance on a unique debugging port. With 2-3 projects this is manageable, but with 8-10+ projects it becomes a nightmare:

- Which port did I assign to which project?
- Is port 9225 already taken?
- How do I launch Chrome with the right flags every time?
- How do I add Chrome DevTools MCP to a new project quickly?

`chrome-devtools-manager` solves all of this with a single command.

## Features

- **Launch Chrome** with the correct debugging port for any project
- **Port registry** that tracks all assignments across your projects
- **Conflict detection** that warns you when two projects share a port
- **Auto-assign ports** when adding Chrome DevTools MCP to new projects
- **Configurable search paths** so it finds projects wherever you keep them
- **Creates/updates `.mcp.json`** files automatically with `jq`

## Requirements

- **Operating System:** macOS, Linux, or Windows via WSL2
- **Shell:** bash or zsh
- **Chrome/Chromium:** Google Chrome or Chromium installed
- **jq:** Required only for the `--add` command (see install instructions below)
- **Node.js/npx:** Required by Chrome DevTools MCP itself

### Installing jq

`jq` is only needed if you use `--add` to configure projects automatically. All other commands work without it.

```bash
# macOS
brew install jq

# Ubuntu / Debian
sudo apt install jq

# Fedora / RHEL
sudo dnf install jq
```

## Installation

### 1. Download the script

```bash
# Create the directory if it doesn't exist
mkdir -p ~/.local/bin

# Download the script
curl -o ~/.local/bin/chrome-devtools-manager \
  https://gist.githubusercontent.com/clizaola/GIST_ID/raw/chrome-devtools-manager.sh

# Make it executable
chmod +x ~/.local/bin/chrome-devtools-manager
```

### 2. Add `~/.local/bin` to your PATH (if not already)

Add this line to your `~/.zshrc` or `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Then reload your shell:

```bash
source ~/.zshrc   # or source ~/.bashrc
```

### 3. Verify installation

```bash
chrome-devtools-manager --help
```

## Usage

### Launch Chrome for the current project

```bash
cd ~/projects/my-app
chrome-devtools-manager
# => Launching Chrome for 'my-app' on port 9222...
```

This reads the port from the project's `.mcp.json` and launches an isolated Chrome instance with its own user profile. Each project gets completely separate cookies, sessions, and state.

### View port registry

```bash
chrome-devtools-manager --list
```

Output:

```
Chrome DevTools MCP Port Registry
===================================
PROJECT                        PORT     PATH
-------                        ----     ----
my-app                         9222     ~/projects/my-app
api-service                    9223     ~/projects/api-service
admin-panel                    9224     ~/Code/client/admin-panel

Last scanned: 2026-03-07 10:15:33
Next available port: 9225
```

### Scan for projects

Scans all configured search paths and rebuilds the registry:

```bash
chrome-devtools-manager --scan
```

Output:

```
Scanning for Chrome DevTools MCP configurations...
  ~/Herd/*
  ~/Herd/*/*
  ~/Code/*
  ~/Code/*/*

Chrome DevTools MCP Port Registry
===================================
PROJECT                        PORT     PATH
-------                        ----     ----
my-app                         9222     ~/projects/my-app
api-service                    9223     ~/projects/api-service

!! CONFLICT: my-app, another-app both use port 9222

Last scanned: 2026-03-07 10:20:00
Next available port: 9224

Registry saved to ~/.chrome-devtools-manager
```

Conflicts are flagged with `!! CONFLICT` so you can fix them immediately.

### Add Chrome DevTools MCP to a project

Navigate to your project folder and run:

```bash
cd ~/projects/new-project

# Auto-assign the next available port
chrome-devtools-manager --add

# Or specify a port manually
chrome-devtools-manager --add 9225
```

This does the following:

1. **If `.mcp.json` exists:** Adds the `chrome-devtools` entry to the existing `mcpServers` object, preserving all other MCP servers
2. **If `.mcp.json` doesn't exist:** Creates a new `.mcp.json` with the chrome-devtools configuration
3. **Checks for conflicts:** Refuses to assign a port that's already in use
4. **Auto-scans:** Updates the registry after adding

The generated `.mcp.json` entry looks like this:

```json
{
    "mcpServers": {
        "chrome-devtools": {
            "command": "npx",
            "args": [
                "-y",
                "chrome-devtools-mcp@latest",
                "--browserUrl",
                "http://127.0.0.1:9225"
            ]
        }
    }
}
```

### Add custom search paths

By default, the tool searches these directories for `.mcp.json` files:

```
~/Herd/*
~/Herd/*/*
~/Code/*
~/Code/*/*
```

Add your own:

```bash
# Single level deep
chrome-devtools-manager --path "~/Projects/*"

# Nested (e.g. org/project structure)
chrome-devtools-manager --path "~/Work/*/*"
```

Search paths are saved in the config file and used by `--scan`.

## Configuration File

The config and registry live in `~/.chrome-devtools-manager`. It's a plain text file with two sections:

```
# Chrome DevTools MCP Port Registry
# https://github.com/ChromeDevTools/chrome-devtools-mcp
#
# SEARCH PATHS (add your project directories here)
~/Herd/*
~/Herd/*/*
~/Code/*
~/Code/*/*
#
# REGISTRY (auto-generated by chrome-devtools-manager --scan)
# Last scanned: 2026-03-07 10:15:33
# PROJECT|PORT|PATH
my-app|9222|~/projects/my-app
api-service|9223|~/projects/api-service
```

- **SEARCH PATHS section:** You can edit this manually to add/remove search directories
- **REGISTRY section:** Auto-generated by `--scan`. Do not edit manually; it gets overwritten on every scan

## How It Works

### Port detection

The tool finds ports by scanning `.mcp.json` files for the pattern `127.0.0.1:<port>`. It does not parse JSON for this; it uses a simple `grep` pattern. This means it works regardless of how the JSON is formatted or indented.

### Chrome isolation

When launching Chrome, two flags ensure complete isolation:

- `--remote-debugging-port=<port>`: The debugging port that Chrome DevTools MCP connects to
- `--user-data-dir=/tmp/chrome-<project-name>`: A separate Chrome profile per project

This means each project gets its own:
- Cookies and sessions
- Browser history
- Extensions
- Local storage
- Cache

### Port auto-assignment

When using `--add` without specifying a port, the tool reads the registry, finds the highest port in use, and assigns the next one. The starting port is `9222` (the Chrome DevTools Protocol default).

## Platform Support

The script automatically detects your operating system and finds the correct Chrome/Chromium executable. No manual configuration needed.

### macOS

Works out of the box. Detects Chrome at:

```
/Applications/Google Chrome.app/Contents/MacOS/Google Chrome
```

### Linux

Automatically detects the first available executable in this order:

1. `google-chrome`
2. `google-chrome-stable`
3. `chromium-browser`
4. `chromium`

### Windows (via WSL2)

Works under WSL2 (Windows Subsystem for Linux). The script detects WSL2 automatically and launches the Windows Chrome installation at:

```
/mnt/c/Program Files/Google/Chrome/Application/chrome.exe
```

**Important:** WSL2 support has not been extensively tested. It should work, but if you run into issues please report them (see Feedback section below).

**Native Windows (PowerShell/CMD) is not supported.** The script requires a bash-compatible shell.

## Caveats and Known Limitations

1. **Chrome must not already be running without `--user-data-dir`:** If Chrome is already open normally, launching a new instance requires `--user-data-dir` to create a separate profile. The tool always uses `--user-data-dir`, so this is handled automatically.

2. **Temp profiles are cleared on reboot:** User profiles are stored in `/tmp/chrome-<project>`, which is cleared when you restart your machine. If you need persistent sessions (staying logged in, etc.), change the path in the script to a permanent location like `~/.chrome-devtools/<project>`.

3. **Port detection is grep-based:** The tool searches for `127.0.0.1:<port>` in `.mcp.json`. If your config uses `localhost` instead of `127.0.0.1`, the port won't be detected. Always use `127.0.0.1` in your MCP config.

4. **One port per `.mcp.json`:** The tool reads only the first `127.0.0.1:<port>` match in each `.mcp.json`. If you have multiple MCP servers using different ports in the same file, only the first will be tracked.

5. **Search paths use glob patterns:** Paths like `~/Code/*` only match one level deep. Use `~/Code/*/*` for nested directories. The tool does NOT recurse infinitely; you must specify the depth explicitly.

6. **`--add` requires `jq`:** The `--add` command uses `jq` to safely modify JSON files. All other commands (`--list`, `--scan`, launch) work without `jq`.

7. **Chrome path auto-detection:** The script detects Chrome automatically on macOS, Linux, and WSL2. If Chrome is installed in a non-standard location, you may need to update the `get_chrome_path()` function in the script.

8. **WSL2 support is untested:** The WSL2 detection (via `/proc/version`) and Windows Chrome path should work but have not been extensively tested. If you encounter issues, please report them.

## Quick Reference

| Command | Description |
|---------|-------------|
| `chrome-devtools-manager` | Launch Chrome for current project |
| `chrome-devtools-manager --list` | Show port registry |
| `chrome-devtools-manager --scan` | Rescan projects, rebuild registry |
| `chrome-devtools-manager --add` | Add chrome-devtools to current project (auto port) |
| `chrome-devtools-manager --add 9225` | Add with specific port |
| `chrome-devtools-manager --path "~/Dir/*"` | Add a search path |
| `chrome-devtools-manager --help` | Show help |

## Optional: Short alias

If `chrome-devtools-manager` is too long to type, add an alias to your `~/.zshrc` or `~/.bashrc`:

```bash
alias cdm='chrome-devtools-manager'
```

Then use `cdm`, `cdm --list`, `cdm --add`, etc.

## Feedback and Issues

- Report issues: [GitHub Issues](https://github.com/clizaola/issues)
- Email: support@cafali.com
- Chrome DevTools MCP: [github.com/ChromeDevTools/chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)

## License

MIT
