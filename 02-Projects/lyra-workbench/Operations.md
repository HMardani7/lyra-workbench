# Lyra Workbench — Operations

## Sync model

```text
Obsidian on Hammad's PC
  ↔ GitHub lyra-workbench
  ↔ /home/hermes/workbench on VPS
  ↔ Lyra/Hermes via OBSIDIAN_VAULT_PATH
```

## Normal agent workflow

```bash
cd /home/hermes/workbench
git pull --ff-only
# read/search/write targeted files
git status --short
git add -A
git commit -m "clear message"
git push
```

## Token discipline

Default read set for project work:

1. `01-Knowledge/Context/Preferences.md`
2. `01-Knowledge/Context/Environment.md`
3. `01-Knowledge/Context/Active Work.md`
4. `01-Knowledge/Context/Vault Operating Rules.md`
5. relevant project overview/index files only

Use search before reading long reports.

## Obsidian Git

On PC, Hammad uses the Obsidian Git plugin. In the current plugin version, the manual command is `Git: Commit-and-sync` rather than `Create backup`.
