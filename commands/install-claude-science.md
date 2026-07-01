---
description: Install and start Claude Science (operon) locally on WSL2 — checks WSL2/Ubuntu, installs the operon binary, upgrades bubblewrap/socat if needed, starts the daemon, and prints a login URL. Only run when the user explicitly asks to install or start Claude Science.
---

# Install Claude Science (operon) on WSL2

Follow these steps in order. Several steps require an interactive `sudo` password, which you (Claude Code) cannot supply — when you reach one, stop and ask the user to run that specific command themselves in their terminal, then wait for confirmation before continuing.

## What this installs

[Claude Science](https://claude.com/product/claude-science) runs Claude locally via a lightweight daemon called `operon`. It bundles its own agent/code-execution runtime (it spawns Python and executes code, not just a passive viewer) — the sandbox (bubblewrap + socat) is what contains that runtime's filesystem/network access. Never start it with `--dangerously-no-sandbox`; that flag disables the sandbox entirely (full home-directory read/write + unrestricted network).

## Step 0 — Requirements check

Since you (Claude Code) are running inside the WSL Linux environment already, check directly here:

1. **WSL2** — run `uname -r` and confirm the output contains `WSL2`. If it says `WSL` without the 2, stop. Tell the user to check their distro's version from Windows PowerShell/Command Prompt with `wsl --list --verbose` (the `VERSION` column must say `2`), and upgrade it with `wsl --set-version <DistroName> 2` if it says `1`.
2. **Ubuntu** — run `lsb_release -rs`. Any version from 20.04 onward works.

## Step 1 — Locate the operon binary

The user should have already downloaded it from **https://claude.com/product/claude-science**. Ask where they saved it.

## Step 2 — Install operon

```bash
mkdir -p ~/.local/bin
cp <path-to-operon-file> ~/.local/bin/operon
chmod +x ~/.local/bin/operon
which operon
```

If `which operon` returns nothing, add to `~/.bashrc` (or `~/.zshrc`):
```bash
export PATH="$HOME/.local/bin:$PATH"
```
then `source ~/.bashrc`.

## Step 3 — Check bubblewrap version

The sandbox requires bubblewrap (`bwrap`) **≥ 0.8.0**:
```bash
bwrap --version
```
- **0.8.0 or higher** → skip to Step 5.
- **Lower than 0.8.0** (or missing) → continue with Step 4. Ubuntu 22.04 and earlier ship 0.6.x via apt, so this is usually needed.

## Step 4 — Build bubblewrap 0.8.0 from source

**Ask the user to run this themselves** (interactive sudo password):
```bash
sudo apt update
sudo apt install -y meson ninja-build libcap-dev git pkg-config
```

Once confirmed, you can run the build yourself (no sudo needed for this part):
```bash
cd /tmp
git clone --depth 1 --branch v0.8.0 https://github.com/containers/bubblewrap.git
cd bubblewrap
meson setup build
ninja -C build
```

**Ask the user to run this themselves** (interactive sudo password):
```bash
sudo ninja -C build install
```

Once confirmed, verify:
```bash
bwrap --version
# Should print: bubblewrap 0.8.0
```

## Step 4b — Install socat (sandbox networking)

The sandbox also needs `socat`. If the daemon fails to start with:
```
[claude-science] fatal: Error: socat is required for Linux sandbox networking but was not found in PATH.
```
**ask the user to run this themselves** (interactive sudo password):
```bash
sudo apt install -y socat
```

## Step 5 — Start the daemon

```bash
operon serve --no-browser --detached
```
`--no-browser` is required under WSL (the daemon can't open a Windows browser directly). `--detached` runs it in the background. Safe to re-run even if it's already running.

## Step 6 — Get the login link

```bash
operon url
```
Prints a single-use login URL (valid ~3 minutes) plus a second descriptive line. Tell the user to open the URL in their **Windows browser**. If it expires, just run `operon url` again.

## Optional: generate a Windows one-click launcher

Setup is complete after Step 6 — nothing further is required. Only once Step 6 has actually succeeded (the daemon is running and `operon url` returned a working login URL confirmed executable via bash), ask the user a single question with exactly these three options:

1. **Yes — generate a `.bat` file I can just double-click from Windows**
2. **No need**
3. **I have questions about it first**

If they pick option 3, answer their questions, then ask again. If option 2, stop here — setup is complete. If option 1, proceed:

1. Get the exact running distro name from the environment: `echo $WSL_DISTRO_NAME`. Use this exact name in the generated script's `wsl -d <name> ...` calls — do not omit `-d`, since a bare `wsl` targets whichever distro is set as default, which may not be this one if the user has more than one installed.
2. Use the literal path `~/.local/bin/operon` in the generated script rather than bare `operon`. This avoids relying on `operon` being resolvable via PATH inside a non-interactive login shell (`bash -lc`), which isn't guaranteed depending on where the user's `~/.bashrc` puts the PATH export from Step 2.
3. Ask where to save the file (e.g. their Desktop or Downloads — a Windows path like `/mnt/c/Users/<name>/Desktop/start-claude-science.bat`).
4. Write the file with this content, substituting the real distro name:

   ```bat
   @echo off
   setlocal

   set "OUTFILE=%TEMP%\operon_url.txt"
   set "DISTRO=<WSL_DISTRO_NAME>"

   echo Starting Claude Science (operon)...
   wsl -d %DISTRO% -e bash -lc "~/.local/bin/operon serve --no-browser --detached"

   timeout /t 2 /nobreak >nul

   wsl -d %DISTRO% -e bash -lc "~/.local/bin/operon url 2>&1 | grep -oE 'http://[^[:space:]]+' | head -n1" > "%OUTFILE%"

   set "OPERON_URL="
   for /f "usebackq delims=" %%A in ("%OUTFILE%") do set "OPERON_URL=%%A"

   if "%OPERON_URL%"=="" (
       echo Could not get a login URL. Raw output was:
       wsl -d %DISTRO% -e bash -lc "~/.local/bin/operon url"
       pause
       exit /b 1
   )

   echo Opening %OPERON_URL%
   start "" "%OPERON_URL%"

   del "%OUTFILE%" >nul 2>&1
   exit /b 0
   ```

5. Before telling the user it's ready, verify it actually resolves: run `wsl -d <name> -e bash -lc "~/.local/bin/operon --version"` (or similar) and confirm it succeeds, so the generated file isn't handed over untested.

This script is generated locally on the user's machine, not distributed as a repo file — it isn't guaranteed to be correct for setups this repo hasn't been tested on, so treat it as a best-effort convenience, not a supported artifact.

## Useful commands

| Command | What it does |
|---|---|
| `operon status` | Is the daemon running? |
| `operon logs --tail` | Watch live logs |
| `operon stop` | Stop the daemon |
| `operon url` | Get a fresh login link |
| `operon update` | Self-update to the latest build |
| `operon serve --help` | All available flags |

## Troubleshooting

**"Sandbox unavailable: bwrap too old"** — bubblewrap is below 0.8.0. Repeat Step 4.

**`socat is required for Linux sandbox networking`** — run `sudo apt install -y socat` (Step 4b). This is a separate dependency from bubblewrap and can surface even after bubblewrap is upgraded.

**`operon url` gives a link but the browser says "connection refused"** — the daemon isn't running. Run `operon serve --no-browser --detached` first.

**`which operon` returns nothing after install** — `~/.local/bin` isn't on PATH. Add the export line from Step 2 and reload the shell.

**Port 8000 already in use** — start on a different port: `operon serve --no-browser --detached --port 8080`.

**Windows one-click launcher (`scripts/start-claude-science.bat`) says "Could not get a login URL" or shows `??` / a truncated URL** — this happens if the script parses `operon url`'s raw two-line output on the Windows side. The second line contains a UTF-8 em dash that gets corrupted crossing the `wsl.exe` → Windows redirection boundary, which also scrambles the adjacent URL text. The script in this repo already avoids this by extracting only the URL inside bash with `grep -oE 'http://[^[:space:]]+' | head -n1` before anything reaches Windows — make sure you're using the current version of the script, not an older copy.

**"RUNNING WITHOUT THE SECURITY SANDBOX" banner in the logs** — something started operon with `--dangerously-no-sandbox`. This grants full home-directory read/write and unrestricted network access. Check `~/.claude-science/logs/spawn.log` for when/why, and fix the underlying bubblewrap/socat dependency instead of using this flag.
