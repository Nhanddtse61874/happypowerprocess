# Installing Superpowers for Codex

Enable superpowers skills in Codex via native skill discovery. Just clone and symlink.

## Prerequisites

- Git

## Installation

1. **Clone the superpowers repository:**
   ```bash
   git clone https://github.com/Nhanddtse61874/firstworkflowagent.git ~/.codex/firstworkflowagent
   ```

2. **Create the skills symlink:**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/firstworkflowagent/skills ~/.agents/skills/firstworkflowagent
   ```

   **Windows (PowerShell):**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\firstworkflowagent" "$env:USERPROFILE\.codex\firstworkflowagent\skills"
   ```

3. **Restart Codex** (quit and relaunch the CLI) to discover the skills.

## Migrating from old bootstrap

If you installed superpowers before native skill discovery, you need to:

1. **Update the repo:**
   ```bash
   cd ~/.codex/firstworkflowagent && git pull
   ```

2. **Create the skills symlink** (step 2 above) — this is the new discovery mechanism.

3. **Remove the old bootstrap block** from `~/.codex/AGENTS.md` — any block referencing `firstworkflowagent bootstrap` is no longer needed.

4. **Restart Codex.**

## Verify

```bash
ls -la ~/.agents/skills/firstworkflowagent
```

You should see a symlink (or junction on Windows) pointing to your superpowers skills directory.

## Updating

```bash
cd ~/.codex/firstworkflowagent && git pull
```

Skills update instantly through the symlink.

## Uninstalling

```bash
rm ~/.agents/skills/firstworkflowagent
```

Optionally delete the clone: `rm -rf ~/.codex/firstworkflowagent`.
