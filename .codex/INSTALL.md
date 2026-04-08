# Installing Superpowers for Codex

Enable superpowers skills in Codex via native skill discovery. Just clone and symlink.

## Prerequisites

- Git

## Installation

1. **Clone the superpowers repository:**
   ```bash
   git clone https://github.com/Nhanddtse61874/happypowerprocess.git ~/.codex/happypowerprocess
   ```

2. **Create the skills symlink:**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/happypowerprocess/skills ~/.agents/skills/happypowerprocess
   ```

   **Windows (PowerShell):**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\happypowerprocess" "$env:USERPROFILE\.codex\happypowerprocess\skills"
   ```

3. **Restart Codex** (quit and relaunch the CLI) to discover the skills.

## Migrating from old bootstrap

If you installed superpowers before native skill discovery, you need to:

1. **Update the repo:**
   ```bash
   cd ~/.codex/happypowerprocess && git pull
   ```

2. **Create the skills symlink** (step 2 above) — this is the new discovery mechanism.

3. **Remove the old bootstrap block** from `~/.codex/AGENTS.md` — any block referencing `happypowerprocess bootstrap` is no longer needed.

4. **Restart Codex.**

## Verify

```bash
ls -la ~/.agents/skills/happypowerprocess
```

You should see a symlink (or junction on Windows) pointing to your superpowers skills directory.

## Updating

```bash
cd ~/.codex/happypowerprocess && git pull
```

Skills update instantly through the symlink.

## Uninstalling

```bash
rm ~/.agents/skills/happypowerprocess
```

Optionally delete the clone: `rm -rf ~/.codex/happypowerprocess`.
