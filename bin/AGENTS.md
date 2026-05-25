# gitree — bare-repo-with-worktrees workspace guide

gitree manages git repos as bare repos with linked worktrees. Read `.gitree` at the workspace root for the authoritative project list (`projects` key) and runtime switch paths (`switch.*` keys).

## Per-project layout

Every project in a gitree workspace follows this structure:

```
<project>/
├── .bare/              ← bare repo: refs, objects, worktree metadata
├── .git                ← 1-line file: "gitdir: ./.bare"
├── main/               ← default working surface (main branch checked out here)
└── .worktrees/         ← additional worktrees, created on demand
    └── <branch>/
```

## Where git commands work

- **Inside a worktree** (`main/`, `.worktrees/<branch>/`) — everything works normally: `git status`, `git add`, `git commit`, `git diff`, etc.
- **From the project container** (`<project>/`) — read-only ops only: `git log`, `git branch -a`, `git worktree list`, `git remote -v`. `git status` errors with "must be run in a work tree" — this is expected, not a bug.
- **Default working surface is `main/`** unless explicitly directed elsewhere.

## gitree commands

```bash
gitree list [<project>]                         # list worktrees; scoped to project if given or cwd
gitree audit [<project>]                        # show branches with no worktree yet
gitree wt [<project>] <branch>                  # get or create a worktree; print its path
gitree goto [<project>] <branch>                # alias for wt (shell fn gitree-goto() does the cd)
gitree switch [-l <loc>] [<project>] [<branch>] # point a switch symlink at a worktree
gitree repair-head [<project>|*]                # fix bare repo HEAD drift to remote default branch
gitree restore                                  # clone all projects in .gitree manifest
gitree drop <project>                           # remove entire project + update manifest (prompts)
gitree install                                  # install gitree globally + shell functions
```

## Shell functions (added by gitree install)

```bash
wt()          { local d; d=$(gitree wt   "$@") && cd "$d"; }
gitree-goto() { local d; d=$(gitree goto "$@") && cd "$d"; }
```

Use `wt <branch>` from inside a project, or `gitree-goto <project> <branch>` from anywhere.

### `gitree switch` — single vs multi-location

`.gitree` defines named switch locations (`switch.*` keys). Use `-l <name>` to target one:

```bash
# Default location (omit -l):
gitree switch <project>                    # default switch → main/
gitree switch <project> <branch>           # default switch → .worktrees/<branch>/

# Named location:
gitree switch -l <loc> <project> <branch>
gitree switch -l <loc> <project>           # back to main on that location
```

From inside a project dir (auto-detected, no project name needed):
```bash
cd <project>/main
gitree switch <branch>
gitree switch -l <loc> <branch>
gitree wt <branch>
gitree audit
```

Branch names with slashes map to dir names with `--` (`feature/foo` → `.worktrees/feature--foo/`). Always pass the real branch name — gitree handles the translation.

## Removing worktrees

```bash
git -C <project>/.bare worktree remove .worktrees/<dirname>
```
Confirm nobody's working in it first.

## Multi-session worktree safety

Other agents or the user may be working in different worktrees of the same project simultaneously. To avoid clobbering their WIP:

- **Stay in the worktree you were opened in.** Don't `cd` into a sibling worktree without explicit instruction.
- **Don't `git checkout <branch>` inside an existing worktree.** To work on a different branch, create a new worktree with `gitree wt <branch>` instead. Switching branches inside `main/` may overwrite another session's uncommitted changes.
- **Before creating a worktree, run `git worktree list`** to see what's already checked out. Git will refuse to add a worktree for a branch already checked out elsewhere — that's a signal another session may be using it. Don't force-add without explicit confirmation.
- **If you must `git checkout` inside an existing worktree**, warn first and wait for confirmation.
- **Don't `git worktree remove` a worktree** without confirming nobody's working in it.

## Switch locations — multi-environment testing

Each `switch.*` key in `.gitree` points to a directory where a symlink `<location>/<project>` is managed by gitree. Use `-l <location>` to target a specific environment:

### What happens physically

```
<switch-dir>/<project>               ← symlink in the runtime's target directory
  → <workspace>/<project>/main       ← gitree updates THIS symlink
    → <project>/.worktrees/<branch>/ ← the actual worktree
```

Nothing is copied — a switch is a symlink retarget. The runtime sees the new worktree immediately on the next request.

### Verifying the active version

```bash
gitree list                          # worktree state + switch state for all projects
readlink <switch-path>/<project>     # confirm a specific symlink target
```

## Branch context

`.gitree-context/` at the workspace root holds per-branch context for all projects.
It is never committed to any repo — it is local workspace state, synced between
machines via rsync alongside `.gitree`.

When in a non-main worktree, `.branch-context/` is a symlink to
`.gitree-context/<project>/<branch>/`. Files:

- `README.md` — what this branch is for (read first)
- `AGENTS.md` — agent instructions specific to this branch
- `TODO.md` — feature task list
- `ROADMAP.md` — milestone/phase breakdown (optional)

**When starting work in a non-main worktree:**
1. Read `.branch-context/README.md` for branch purpose and context
2. Read `.branch-context/AGENTS.md` for any specific agent instructions
3. If both only contain stub content, ask the user what the branch is for and populate them
4. Keep `TODO.md` current — check off completed items, add new ones as they arise

**Never populate `.branch-context/` in the `main/` worktree** — main has no branch context.
If `.branch-context/` appears in `main/`, the symlink is stale or misdirected — flag it.

**These files are agent-agnostic.** The workspace config (e.g. CLAUDE.md) is responsible
for directing agents to read `.branch-context/AGENTS.md`. The context directory itself
makes no assumptions about which agent is reading it.

### Context file roles and anti-drift rules

Each piece of information lives in exactly one place at the right level. Higher-level files define structure and relationships — they do not summarise or echo lower-level files. When something changes, update it in one place only.

#### AGENTS.md
Operational instructions for working in a project: conventions, tooling constraints, how-tos specific to this codebase. Not a task list, not a README summary.

- **Update when:** conventions change, new constraints are discovered, tooling changes.
- **Don't:** duplicate README context, list tasks, describe roadmap plans, repeat gitree basics already covered at the workspace level.

#### README.md
Persistent context about what a project is — architecture, purpose, relationships to other repos. Written for both humans and agents starting cold.

- **Update when:** architecture changes, scope changes, key relationships shift.
- **Don't:** include task-level detail, branch-specific work, or ephemeral state.

#### TODO.md
At `.gitree-context/TODO.md`: cross-repo organisation tasks or triage items with no branch home yet. Per-branch work lives in `.branch-context/TODO.md`.

- **Update when:** a genuine workspace-level task exists with no natural branch.
- **Don't:** copy branch TODOs here, add implementation detail, let it grow stale.

#### ROADMAP.md
At `.gitree-context/ROADMAP.md`: high-level direction relevant to multiple repos — phases, milestone groupings, strategic pivots. Multi-repo only; per-feature planning belongs in branches.

- **Update when:** multi-repo strategy changes.
- **Don't:** duplicate per-feature detail that belongs in a branch.

#### Where files live

| | Workspace root | `.gitree-context/` |
|---|---|---|
| **Entry point** | `AGENTS.md` (+ `CLAUDE.md` symlink) | — |
| **Workspace state** | `.gitree` | `AGENTS.md`, `README.md`, `TODO.md`, `ROADMAP.md` |
| **Per-branch** | — | `<project>/<branch>/` (multi-repo) or `<branch>/` (mono-repo) |

Only `AGENTS.md` (auto-regenerated by gitree) and `.gitree` live at the workspace root. Everything else portable lives under `.gitree-context/`, so the export model is just `rsync .gitree .gitree-context/` to another machine, then `gitree restore`.

**Mono-repo:** the workspace IS the project — no nested `<project>/` layer under `.gitree-context/`. Branches go directly at `.gitree-context/<branch>/`. The workspace-level README/TODO/ROADMAP ARE the project's README/TODO/ROADMAP.
