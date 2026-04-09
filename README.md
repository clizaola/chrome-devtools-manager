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
- **Removes `chrome-devtools`** from a project's `.mcp.json` when you no longer need it
- **Health check (`--doctor`)** that reports stale entries, drift, conflicts, and unregistered projects — without touching anything
- **Profile cleanup (`--clean`)** that wipes `/tmp/chrome-<project>` for the current project (or all of them with `--all`), with `--force` to also kill active Chrome instances

## Requirements

- **Operating System:** macOS, Linux, or Windows via WSL2
- **Shell:** bash or zsh
- **Chrome/Chromium:** Google Chrome or Chromium installed
- **jq:** Required for `--add` and `--remove` (JSON manipulation)
- **curl:** Optional but recommended. Enables post-launch verification (polling the Chrome DevTools Protocol endpoint to confirm Chrome is actually up) and the `[DEAD]` liveness checks in `--doctor`. The script still works without it — those checks are silently skipped.
- **Node.js / npx:** Required by Chrome DevTools MCP itself (not by this script directly, but by the MCP server it helps you run)

### Installing jq and curl

`jq` is needed for `--add` and `--remove`. `curl` ships with every major OS but if it's missing you may want to install it too.

```bash
# macOS (curl is pre-installed)
brew install jq

# Ubuntu / Debian
sudo apt install jq curl

# Fedora / RHEL
sudo dnf install jq curl
```

## Installation

The recommended install path is a git clone + symlink, so you can `git pull` for updates.

### 1. Clone the repository

```bash
git clone https://github.com/clizaola/chrome-devtools-manager.git ~/Code/chrome-devtools-manager
```

(You can clone it anywhere — `~/Code/chrome-devtools-manager` is just a convention.)

### 2. Symlink the script into your `PATH`

```bash
mkdir -p ~/.local/bin
ln -s ~/Code/chrome-devtools-manager/chrome-devtools-manager.sh ~/.local/bin/chrome-devtools-manager
```

Using a symlink means every `git pull` updates the command automatically — no reinstall step needed.

### 3. Add `~/.local/bin` to your `PATH` (if not already)

Add this line to your `~/.zshrc` or `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Then reload your shell:

```bash
source ~/.zshrc   # or source ~/.bashrc
```

### 4. Verify installation

```bash
chrome-devtools-manager --help
```

### Updating

```bash
cd ~/Code/chrome-devtools-manager
git pull
```

The symlink points at the script in the git working tree, so `git pull` is all you need — nothing to reinstall.

### Single-file alternative

If you don't want to clone a repo, you can grab just the script directly from GitHub:

```bash
mkdir -p ~/.local/bin
curl -o ~/.local/bin/chrome-devtools-manager \
  https://raw.githubusercontent.com/clizaola/chrome-devtools-manager/main/chrome-devtools-manager.sh
chmod +x ~/.local/bin/chrome-devtools-manager
```

Tradeoff: updating requires re-running the `curl` command instead of `git pull`.

## Usage

### Launch Chrome for the current project

```bash
cd ~/projects/my-app
chrome-devtools-manager
# => Launching Chrome for 'my-app' on port 9222...
```

This reads the port from the project's `.mcp.json` and launches an isolated Chrome instance with its own user profile. Each project gets completely separate cookies, sessions, and state.

The launcher also:

- **Refuses to launch** if a Chrome process is already running for this project's profile directory on a different port (Chrome's profile singleton would silently ignore the new port flag, so this would be a foot-gun). It tells you exactly which command to run to kill the stale instance.
- **Verifies Chrome actually bound the port** after launch by polling `http://127.0.0.1:<port>/json/version` for up to 5 seconds. If the endpoint doesn't respond, you get a warning instead of a silent failure.

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

### Remove Chrome DevTools MCP from a project

When you no longer want `chrome-devtools` in a project's `.mcp.json`, navigate to the project and run:

```bash
cd ~/projects/old-project
chrome-devtools-manager --remove
```

This removes only the `chrome-devtools` entry from `mcpServers`, preserving all other MCP servers. The registry is rescanned automatically so the project's port is freed for reuse. Requires `jq`.

### Run a health check (`--doctor`)

```bash
chrome-devtools-manager --doctor
```

A **non-destructive** diagnostic that inspects the current registry and your `.mcp.json` files and reports any issues it finds. It does not modify any files or the registry — you decide what to fix. Useful when something feels off (missing project, wrong port, conflict) and you want a quick health report before deciding what to run.

It reports:

- **`[STALE]`** — the project directory is gone, or `.mcp.json` no longer contains `chrome-devtools`
- **`[DRIFT]`** — the port in `.mcp.json` no longer matches the port in the registry (something external edited the file)
- **`[WARN]`** — a project's `chrome-devtools` entry uses a host other than `127.0.0.1` or `localhost`, so its port can't be detected
- **`[CONFLICT]`** — two or more projects share the same port
- **`[WRONG-PORT]`** — a Chrome instance is running with this project's profile directory but on a different port than `.mcp.json` wants. This happens when the configured port changes after Chrome was already launched; Chrome's profile singleton silently reuses the old session. The fix is to kill that Chrome process and relaunch.
- **`[DEAD]`** — informational. A registered project whose configured port is not currently listening. This is normal for projects you're not actively using. Not counted as an issue.
- **`[ORPHAN]`** — a project on disk has `chrome-devtools` in its `.mcp.json` but isn't in the registry (run `--scan` to pick it up)

Example output:

```
Chrome DevTools MCP Doctor
===========================

Checking registered projects...
  [CONFLICT] port 9229 used by: cafali,omniguard360

Scanning search paths for unregistered projects...
  [ORPHAN]   new-project at ~/Code/new-project (not in registry)

Found 2 issue(s).

Suggested actions:
  chrome-devtools-manager --scan                              Rebuild registry from disk (fixes STALE, DRIFT, ORPHAN)
  cd <project> && chrome-devtools-manager --remove            Drop a stale/unwanted entry
  cd <project> && chrome-devtools-manager --remove && chrome-devtools-manager --add   Reassign a conflicting port
```

### Clean up profile directories (`--clean`)

Chrome profiles for each project live in `/tmp/chrome-<project>/` and can grow very large over time — browser cache, service workers, IndexedDB, etc. A single project can easily hit multiple GB after heavy use. Temp profiles are cleared on reboot, but `--clean` lets you free disk space or reset a misbehaving browser state without rebooting.

**By default, `--clean` only touches the current project's profile.** Use `--all` if you want to clean every project at once.

```bash
# Clean ONLY the current project's profile (/tmp/chrome-<basename $PWD>)
cd ~/Code/myproject
chrome-devtools-manager --clean

# Clean the current project's profile AND kill it first if it's running
chrome-devtools-manager --clean --force

# Clean every /tmp/chrome-* profile. Skips any Chrome that is currently running.
chrome-devtools-manager --clean --all

# Nuclear: kill every running Chrome and wipe every profile
chrome-devtools-manager --clean --all --force
```

Example output (`--clean --all`):

```
Scanning /tmp/ for Chrome profile directories...

  [CLEAN]      cafali (246M)
  [CLEAN]      greenmedinfo (270M)
  [SKIP]       gumplayfactory (active on port 9234, 902M)
  [CLEAN]      homer (4.3G)
  [CLEAN]      kozmetik (5.3G)

Cleaned 4 profile(s), skipped 1.

To also kill active Chrome instances and clean their profiles, add --force.
```

**Safety defaults:**

- `--clean` without `--all` only touches `/tmp/chrome-<basename $PWD>`. Much smaller blast radius when you just want to reset one project.
- `--clean` without `--force` never kills a running Chrome. Active profiles are reported as `[SKIP]` and left alone.
- Use `--force` only when you're sure you want to terminate the Chrome window.

**Note:** MCP screenshots and snapshots are transmitted through the MCP protocol — they are **not** stored in these profile directories. The size you see is normal Chrome browser state (cache, cookies, local storage, etc.) from any browsing you did in the debugging windows. Cleaning only logs you out of sites in that profile; nothing MCP-related is lost.

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

The tool finds ports by scanning `.mcp.json` files for the pattern `127.0.0.1:<port>` or `localhost:<port>`. It does not parse JSON for this; it uses a simple `grep` pattern. This means it works regardless of how the JSON is formatted or indented.

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

3. **Port detection is grep-based:** The tool searches for `127.0.0.1:<port>` or `localhost:<port>` in `.mcp.json`. Other hostnames (e.g. `0.0.0.0`, custom domains) will not be detected.

4. **One port per `.mcp.json`:** The tool reads only the first matching host:port pair in each `.mcp.json`. If you have multiple MCP servers using different ports in the same file, only the first will be tracked.

5. **Search paths use glob patterns:** Paths like `~/Code/*` only match one level deep. Use `~/Code/*/*` for nested directories. The tool does NOT recurse infinitely; you must specify the depth explicitly.

6. **`--add` and `--remove` require `jq`:** These commands use `jq` to safely modify JSON files. All other commands (`--list`, `--scan`, `--doctor`, `--clean`, launch) work without `jq`.

7. **Chrome path auto-detection:** The script detects Chrome automatically on macOS, Linux, and WSL2. If Chrome is installed in a non-standard location, you may need to update the `get_chrome_path()` function in the script.

8. **WSL2 support is untested:** The WSL2 detection (via `/proc/version`) and Windows Chrome path should work but have not been extensively tested. If you encounter issues, please report them.

## Troubleshooting

### Start here: `--doctor`

Almost every problem can be diagnosed in one step:

```bash
chrome-devtools-manager --doctor
```

This non-destructive check will tell you which project has a stale entry, a port conflict, a wrong port, a dead listener, or an unregistered project. Each finding comes with a suggested fix command. If `--doctor` reports no issues, the tool itself is healthy and the problem is somewhere else.

### `No chrome-devtools port found in .mcp.json`

The launcher found a `.mcp.json` but no `127.0.0.1:<port>` or `localhost:<port>` pattern inside it. Either:

- `chrome-devtools` isn't in the file yet → run `chrome-devtools-manager --add`
- The URL uses a host the tool can't detect (e.g. `0.0.0.0`, a custom hostname) → edit `.mcp.json` to use `127.0.0.1` or `localhost`

### MCP client says `Failed to fetch browser webSocket URL from http://127.0.0.1:<port>/json/version: fetch failed`

The MCP client can't reach Chrome on the configured port. This means Chrome isn't actually listening there. Run:

```bash
chrome-devtools-manager --doctor
```

If doctor shows `[DEAD]` for that project, just launch Chrome:

```bash
cd /path/to/project
chrome-devtools-manager
```

If doctor shows `[WRONG-PORT]`, see the next section.

### `[WRONG-PORT]` or `Error: Chrome is already running for '<project>' on port X, but .mcp.json is configured for port Y`

**This is the most common gotcha, and the one that burned the tool's author into adding every feature on this page.**

**Root cause:** Chrome has a "profile singleton" behavior. If you launch a second Chrome process pointing at a `--user-data-dir` that is already in use by another Chrome process, Chrome **silently ignores the new `--remote-debugging-port` flag** and just opens a new window in the existing session. The old port stays bound; the new port never comes up.

This happens when:

- You reassigned a project's port (via `--remove` + `--add` or by editing `.mcp.json`) after Chrome was already launched for that project
- Another tool (Herd, IDE integration, etc.) rewrote `.mcp.json` and changed the port while Chrome was still running
- A crashed Chrome left its profile-lock state behind

**Fix:** kill the stale Chrome process for that specific profile, then relaunch. The launcher now detects this case and prints the exact command, but here it is manually:

```bash
pkill -f "user-data-dir=/tmp/chrome-<project-name>"
cd /path/to/project
chrome-devtools-manager
```

This only kills the Chrome instance holding the one specific profile directory — your other Chrome windows are untouched.

### `Chrome launched but port <X> is not responding after ~5s`

The launcher started Chrome but the CDP endpoint never came up. Possibilities:

1. Chrome crashed at startup (rare; try launching once manually)
2. Another process is holding the port (`lsof -iTCP:<port> -sTCP:LISTEN`)
3. You killed the launcher too fast, before Chrome had finished booting
4. A profile-singleton collision hid the port (run `--doctor`)

### `Opening in existing browser session.`

Chrome prints this itself — it means the profile singleton just redirected your launch into an already-running Chrome. The new `--remote-debugging-port` flag was ignored. Same fix as `[WRONG-PORT]` above.

### `/tmp/chrome-<project>` is huge

This is normal — Chrome caches web content aggressively (HTTP cache, service worker cache, IndexedDB). A single project with heavy browsing can easily grow to multiple GB. MCP screenshots and snapshots are **not** stored here; they go through the MCP protocol directly to the client.

Reclaim space:

```bash
# Clean just the current project
cd /path/to/project
chrome-devtools-manager --clean

# Or wipe everything
chrome-devtools-manager --clean --all
```

### `cafali,omniguard360 both use port 9229` (or similar conflict)

Two projects were assigned the same port. Pick one to move:

```bash
cd /path/to/project-to-move
chrome-devtools-manager --remove
chrome-devtools-manager --add          # auto-assigns next free port
```

### Stale entries in `--list` for projects you deleted

```bash
chrome-devtools-manager --scan
```

`--scan` rebuilds the registry from disk, so projects that no longer exist drop out automatically.

## Quick Reference

| Command | Description |
|---------|-------------|
| `chrome-devtools-manager` | Launch Chrome for current project |
| `chrome-devtools-manager --list` | Show port registry |
| `chrome-devtools-manager --scan` | Rescan projects, rebuild registry |
| `chrome-devtools-manager --add` | Add chrome-devtools to current project (auto port) |
| `chrome-devtools-manager --add 9225` | Add with specific port |
| `chrome-devtools-manager --remove` | Remove chrome-devtools from current project |
| `chrome-devtools-manager --doctor` | Non-destructive health check (stale, drift, conflicts, orphans) |
| `chrome-devtools-manager --clean` | Delete current project's `/tmp/chrome-<project>` profile |
| `chrome-devtools-manager --clean --force` | Current project: kill Chrome first if active, then delete |
| `chrome-devtools-manager --clean --all` | Delete every `/tmp/chrome-*` profile (skips active instances) |
| `chrome-devtools-manager --clean --all --force` | Kill every active Chrome and wipe every profile |
| `chrome-devtools-manager --path "~/Dir/*"` | Add a search path |
| `chrome-devtools-manager --help` | Show help |

## Optional: Short alias

If `chrome-devtools-manager` is too long to type, add an alias to your `~/.zshrc` or `~/.bashrc`:

```bash
alias cdm='chrome-devtools-manager'
```

Then use `cdm`, `cdm --list`, `cdm --add`, etc.

## Feedback and Issues

- Report issues: [GitHub Issues](https://github.com/clizaola/chrome-devtools-manager/issues)
- Email: support@cafali.com
- Chrome DevTools MCP: [github.com/ChromeDevTools/chrome-devtools-mcp](https://github.com/ChromeDevTools/chrome-devtools-mcp)

## License

MIT
