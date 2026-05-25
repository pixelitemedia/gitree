# gitree — bare-repo-with-worktrees workflow

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
gitree switch events-manager               # default switch → main/
gitree switch events-manager gutenberg-v1  # default switch → .worktrees/gutenberg-v1/

# Named location:
gitree switch -l wpml events-manager gutenberg-v1
gitree switch -l wpml events-manager       # back to main on that location

gitree wt events-manager-pro feature/x    # get/create worktree, print path
```

From inside a project dir (auto-detected, no project name needed):
```bash
cd events-manager/main
gitree switch gutenberg-v1
gitree switch -l wpml gutenberg-v1
gitree wt feature/new-thing
gitree audit
```

Branch names with slashes map to dir names with `--` (`codex/foo` → `.worktrees/codex--foo/`). Always pass the real branch name — gitree handles the translation.

## Shell function

`gitree install` adds this to your rc for one-word navigation:
```bash
wt() { local d; d=$(gitree wt "$@") && cd "$d"; }
```
`wt feature/foo` from anywhere inside a project creates the worktree and drops you in it.

## Removing worktrees

```bash
git -C <project>/.bare worktree remove .worktrees/<dirname>
```
Confirm nobody's working in it first.

## Multi-session worktree safety

Other Claude sessions, Codex, or the user may be working in different worktrees of the same project simultaneously. To avoid clobbering their WIP:

- **Stay in the worktree you were opened in.** Don't `cd` into a sibling worktree without explicit instruction.
- **Don't `git checkout <branch>` inside an existing worktree.** To test a different branch, add a new worktree at `.worktrees/<branch>/` instead. Switching branches inside `main/` may overwrite another session's uncommitted changes.
- **Before creating a worktree, run `git worktree list`** to see what's already checked out. Git will refuse to add a worktree for a branch already checked out elsewhere — that's a signal another session may be using it. Don't force-add without explicit confirmation.
- **If you must `git checkout` inside an existing worktree**, warn first and wait for confirmation.
- **Don't `git worktree remove` a worktree** without confirming nobody's working in it.
