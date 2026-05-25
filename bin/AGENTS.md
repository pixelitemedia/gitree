# gitree — multi-environment testing with switch locations

Agent context for gitree workspaces that use `switch.*` locations to target
different runtime environments. Read `.gitree-context/AGENTS.md` at the workspace
root for the project-specific environment topology.

---

## How `gitree switch -l` maps to environments

Each `switch.*` key in `.gitree` points to a directory in the runtime where a symlink
from `<location>/<project>` will be managed. Use `-l <location>` to target a specific
environment:

```bash
gitree switch [-l <location>] [<project>] [<branch>]

gitree switch <project> <branch>          # default environment
gitree switch -l <loc> <project> <branch> # named environment
gitree switch -l <loc> <project>          # back to main on that environment
```

### What happens physically

```
<switch-dir>/<project>          ← symlink in the runtime's target directory
  → <workspace>/<project>/main  ← gitree updates THIS symlink
    → <project>/.worktrees/<branch>/  ← the actual worktree
```

The symlink gitree manages is the one inside the switch directory, pointing at the
active worktree. Nothing is copied — a switch is a symlink retarget.

---

## Verifying the active version

```bash
# Check worktrees and current switch state across all projects:
gitree list

# Inspect a specific switch symlink:
readlink <switch-path>/<project>
```

---

## Before and after a switch

```bash
# Confirm current state
gitree list

# Switch
gitree switch -l <location> <project> <branch>

# Verify the runtime is now serving the new worktree
readlink <switch-path>/<project>
```
