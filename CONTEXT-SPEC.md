# Branch Context — gitree feature spec

## Overview

gitree maintains a `.gitree-context/` directory at the workspace root alongside `.gitree`.
When a worktree is created, gitree scaffolds a per-branch context directory inside
`.gitree-context/` and symlinks it into the worktree as `.branch-context`.

These files are never committed to git — they are workspace coordination state, not code.
No `.gitattributes` guard, no merge driver, no GitHub interaction of any kind.

`.gitree` + `.gitree-context/` together form the complete portable state of a workspace:
rsync both to another machine, run `gitree restore`, and the full setup is reproduced.

---

## Directory structure

```
<workspace>/
├── .gitree                              ← repo manifest + switch paths
├── .gitree-context/                     ← branch context for all projects
│   ├── <project>/
│   │   └── <branch-dirname>/
│   │       ├── README.md
│   │       ├── AGENTS.md
│   │       ├── TODO.md
│   │       └── ROADMAP.md
│   └── .archived/                       ← context preserved after worktree removal
│       └── <project>/
│           └── <branch-dirname>/
└── <project>/
    ├── .bare/
    ├── main/                            ← no .branch-context on main
    └── .worktrees/
        └── <branch>/
            └── .branch-context  →  ../../../.gitree-context/<project>/<branch-dirname>/
```

Branch dirnames follow the existing convention: `feature/foo` → `feature--foo`.

---

## Context files

Four files are scaffolded when a branch context directory is created.
`README.md` and `AGENTS.md` get template content; `TODO.md` gets a stub; `ROADMAP.md` is
created empty.

**`README.md`** — human-readable branch description:
```markdown
# <branch>

<!-- What is this branch for? -->
```

**`AGENTS.md`** — agent instructions for this branch:
```markdown
<!-- branch: <branch> | created: <YYYY-MM-DD> -->
<!-- See README.md for branch overview -->
```

**`TODO.md`** — task list:
```markdown
# TODO — <branch>

- [ ] 
```

**`ROADMAP.md`** — empty (populated as needed)

Only write a file if it does not already exist (idempotent scaffolding).

---

## `.gitignore` setup (per repo, one-time)

During `gitree add` / `_setup_bare_repo`, append to `main/.gitignore`:

```
# gitree branch context — local only, never committed
.branch-context
```

Stage and commit:
```bash
git -C <pdir>/main add .gitignore
git -C <pdir>/main commit -m "chore: ignore .branch-context (gitree)"
```

Skip if `.branch-context` is already present in `.gitignore`.

---

## Symlink paths

gitree computes a relative symlink from the worktree to the context directory.
Depth varies by layout:

| Layout | Worktree path | Symlink target |
|---|---|---|
| multi-repo | `<ws>/<project>/.worktrees/<branch>/` | `../../../.gitree-context/<project>/<dirname>/` |
| multi-repo `main/` | `<ws>/<project>/main/` | *(no symlink — main has no branch context)* |
| mono-repo | `<ws>/.worktrees/<branch>/` | `../../.gitree-context/<project>/<dirname>/` |

For mono-repo, `<project>` is `basename "$GITREE_WORKSPACE"`.

Use `python3 -c "import os; print(os.path.relpath('$context_dir', '$wt_path'))"` to
compute the relative path rather than hardcoding depth.

---

## CLI changes

### `_setup_bare_repo` (called by `gitree add`)

After the main worktree is created:

1. Append `.branch-context` to `main/.gitignore`, commit it (skip if already present)
2. `mkdir -p "$GITREE_WORKSPACE/.gitree-context/$project_name"`

Output line: `✓ Branch context ready (.gitree-context/<project>/)`

---

### New internal function: `_scaffold_branch_context`

Called by `cmd_new` after the worktree is created. Also callable directly for retrofitting.

```bash
_scaffold_branch_context() {
    local project="$1" branch="$2" wt_path="$3"
    local dirname; dirname=$(_branch_to_dirname "$branch")
    local context_dir="$GITREE_WORKSPACE/.gitree-context/$project/$dirname"
    local link="$wt_path/.branch-context"

    mkdir -p "$context_dir"

    local today; today=$(date +%F)

    [[ -f "$context_dir/README.md" ]] || \
        printf '# %s\n\n<!-- What is this branch for? -->\n' \
            "$branch" > "$context_dir/README.md"

    [[ -f "$context_dir/AGENTS.md" ]] || \
        printf '<!-- branch: %s | created: %s -->\n<!-- See README.md for branch overview -->\n' \
            "$branch" "$today" > "$context_dir/AGENTS.md"

    [[ -f "$context_dir/TODO.md" ]] || \
        printf '# TODO — %s\n\n- [ ] \n' "$branch" > "$context_dir/TODO.md"

    [[ -f "$context_dir/ROADMAP.md" ]] || touch "$context_dir/ROADMAP.md"

    local rel
    rel=$(python3 -c "import os; print(os.path.relpath('$context_dir', '$wt_path'))")
    ln -sfn "$rel" "$link"

    printf '✓ Branch context scaffolded (.branch-context/)\n' >&2
}
```

---

### `cmd_new`

After `_resolve_worktree` succeeds, call:

```bash
_scaffold_branch_context "$GITREE_PLUGIN_NAME" "$branch" "$path"
```

---

### `cmd_wt`

After `_resolve_worktree` succeeds, wire the symlink if:
- the worktree is not `main`, AND
- a context directory exists (e.g. synced from another machine), AND
- the symlink does not already exist

```bash
local dirname; dirname=$(_branch_to_dirname "$branch")
local context_dir="$GITREE_WORKSPACE/.gitree-context/$GITREE_PLUGIN_NAME/$dirname"
local link="$path/.branch-context"

if [[ "$branch" != "main" && -d "$context_dir" && ! -L "$link" ]]; then
    local rel
    rel=$(python3 -c "import os; print(os.path.relpath('$context_dir', '$path'))")
    ln -sfn "$rel" "$link"
fi
```

---

### `cmd_remove`

After the worktree is removed, handle the context directory:

```bash
local context_dir="$GITREE_WORKSPACE/.gitree-context/$GITREE_PLUGIN_NAME/$dirname"
if [[ -d "$context_dir" ]]; then
    local yn=""
    read -r -p "Archive branch context? [Y/n]: " yn || true
    case "${yn,,}" in
        n|no)
            rm -rf "$context_dir"
            printf '✓ Branch context deleted\n' >&2
            ;;
        *)
            local archive="$GITREE_WORKSPACE/.gitree-context/.archived/$GITREE_PLUGIN_NAME/$dirname"
            mkdir -p "$(dirname "$archive")"
            mv "$context_dir" "$archive"
            printf '✓ Branch context archived to .gitree-context/.archived/%s/%s\n' \
                "$GITREE_PLUGIN_NAME" "$dirname" >&2
            ;;
    esac
fi
```

---

### `cmd_restore`

After each project is cloned, wire symlinks for any context directories that exist but
have no worktree symlink yet (common after rsync from another machine):

```bash
local ctx_root="$GITREE_WORKSPACE/.gitree-context/$name"
if [[ -d "$ctx_root" ]]; then
    for ctx_dir in "$ctx_root"/*/; do
        [[ -d "$ctx_dir" ]] || continue
        local dirname; dirname=$(basename "$ctx_dir")
        local wt_path="$workspace/$name/.worktrees/$dirname"
        local link="$wt_path/.branch-context"
        if [[ -d "$wt_path" && ! -L "$link" ]]; then
            local rel
            rel=$(python3 -c "import os; print(os.path.relpath('$ctx_dir', '$wt_path'))")
            ln -sfn "$rel" "$link"
        fi
    done
fi
```

---

### New subcommand: `gitree branch-context init [<project>|*]`

Retrofits existing repos that predate this feature:

1. Adds `.branch-context` to `main/.gitignore` + commits (skips if already present)
2. Creates `.gitree-context/<project>/`
3. Wires symlinks for any existing non-main worktrees

`*` processes all projects in the workspace, same pattern as `gitree lock *`.

---

## Sync between machines

```bash
# Push workspace state to another machine
rsync -av \
    /path/to/workspace/.gitree \
    /path/to/workspace/.gitree-context/ \
    user@server:/path/to/workspace/

# On the server — first time setup
gitree restore        # clones all repos from manifest
                      # symlinks wired automatically on next gitree wt/new
```

To exclude archived context from the sync:
```bash
rsync -av --exclude='.archived/' /path/to/workspace/.gitree-context/ ...
```

---

## `CLAUDE.gitree.md` additions

Add a `## Branch context` section:

```markdown
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
```
