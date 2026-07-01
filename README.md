# Claude Science (operon) — WSL2 Setup

> **Unofficial, community-maintained setup scripts.** This repo is not published by Anthropic. It automates the local setup of [Claude Science](https://claude.com/product/claude-science) (the `operon` daemon) on Windows via WSL2. It does **not** contain or redistribute the `operon` binary itself — download that directly from the official page linked above.

## What you need to do first

1. **Download the Linux installer** from the official page: **https://claude.com/product/claude-science**
   (Do not download it from anywhere else, and don't pick a macOS/Windows build — this setup is for WSL2, so you need the **Linux** download. It's a single file named `operon`, no extension. Some platform labels on that page have been inconsistent, so verify the link actually points to a Linux build before downloading.)
2. **Confirm your environment:**
   - From **Windows** (PowerShell or Command Prompt), before opening a Linux shell:
     ```powershell
     wsl --list --verbose
     ```
     This lists each installed distro and its version:
     ```
       NAME      STATE           VERSION
     * Ubuntu    Running         2
     ```
     The `VERSION` column must say `2`. If it says `1`, run `wsl --set-version <DistroName> 2`. If `wsl` isn't recognized at all, run `wsl --install` and reboot.
     To open that distro's Linux shell, just type `wsl` (or open it from the Start menu).
   - Once inside the Linux shell, double-check from there too:
     - `uname -r` — output should contain `WSL2`
     - `lsb_release -rs` — Ubuntu 20.04 or newer
3. **Install build dependencies** — these require your `sudo` password, so run them yourself in a WSL2 terminal:
   ```bash
   sudo apt update
   sudo apt install -y meson ninja-build libcap-dev git pkg-config socat
   ```

## Then, with Claude Code

Open a Claude Code session in your WSL2 terminal and run:

```
/install-claude-science
```

This walks through the remaining steps automatically: installing the `operon` binary to your PATH, building bubblewrap ≥0.8.0 from source if your Ubuntu version ships an older one (it will pause and ask you to run the `sudo ninja ... install` step yourself), starting the daemon, and printing a login URL to open in your browser.

See [`commands/install-claude-science.md`](commands/install-claude-science.md) for the full instructions if you want to read or run them manually instead.

## Optional: one-click launcher (Windows)

`/install-claude-science` is everything you need — once it's done, `operon serve --no-browser --detached` and `operon url` in a WSL2 terminal are all it takes to start it up again later.

If you'd rather not open a terminal each time, just ask Claude Code (in the same session, after setup finishes) to generate a `.bat` launcher for you. It's not a file shipped in this repo — Claude Code generates it on the spot, tailored to your actual WSL distro name and a verified path to the `operon` binary, which avoids the "wrong default distro" and "PATH not resolving in a non-interactive shell" problems a generic pre-made script would run into. See the "Optional: generate a Windows one-click launcher" section in [`commands/install-claude-science.md`](commands/install-claude-science.md) for exactly what it does.

## Security notes

- `operon` bundles its own agent/code-execution runtime (it spawns Python and executes code, not just a passive viewer). The sandbox — bubblewrap ≥0.8.0 plus `socat` — is what contains that runtime's filesystem and network access.
- **Never run it with `--dangerously-no-sandbox`.** That flag grants the daemon full read/write access to your home directory and unrestricted network access (equivalent to `--dangerously-skip-permissions` in Claude Code). If you hit a sandbox error, fix the underlying dependency (see Troubleshooting in the command file) instead of disabling the sandbox.
- Only download `operon` from the official Anthropic page. This repo intentionally does not host or mirror the binary.

## Troubleshooting

See the Troubleshooting section in [`commands/install-claude-science.md`](commands/install-claude-science.md) — it covers the bubblewrap version error, the missing-`socat` error, PATH issues, and a couple of Windows batch-script encoding bugs we hit and fixed during development.

## License

The setup scripts and documentation in this repository are licensed under the [MIT License](LICENSE). This does **not** cover the `operon` binary itself, which is not distributed here — download it from the official page above and refer to its own licensing terms.
