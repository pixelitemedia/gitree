# gitree — Claude Operating Guide

## Project overview

gitree is a bash CLI for managing git repos as bare-repo-with-worktrees. It keeps a `main/` worktree locked as the default working surface and creates additional worktrees on demand under `.worktrees/`. A `.gitree` JSON file at the workspace root defines named switch paths and a project manifest for workspace restore.

Built for Marcus's WordPress plugin workflow: multiple plugins, multiple WP installs (default/wpml/wpwc), each switchable independently via symlinks from `WP_PLUGIN_DIR`.

## Tech stack

- **Language**: Bash (self-contained, no runtime deps beyond bash + python3 + git)
- **python3**: used for JSON `.gitree` config parsing only
- **Distribution**: npm (`@pixelitemedia/gitree`) + manual curl

## Canonical source of truth

`~/Code/gitree/main/bin/gitree` — this file IS the tool. Edits here are live immediately because `~/Code/gitree/main/bin` is on `$PATH`. Do not copy the script anywhere; do not maintain a separate workspace copy.

## Key architectural decisions

- **Bare repo layout**: `<project>/.bare/` is the git store; `<project>/.git` is a one-line pointer; `<project>/main/` is the default worktree; additional worktrees live at `<project>/.worktrees/<branch-dirname>/`
- **Branch → dirname**: slashes become `--` (`feature/foo` → `feature--foo`). Always accept real branch names; translate internally.
- **`$wp_plugin_paths` fix**: `wp-content/mu-plugins/symlink-plugin-paths.php` pre-registers symlink→realpath mappings so `plugins_url()` works correctly with `~/Code/` symlink targets.
- **HEAD drift**: bare repo HEAD can drift when worktree ops move it off the default branch. `gitree repair-head` fixes this by querying `git remote show origin`.
- **JSON config**: `.gitree` is JSON with `switch` (named locations) and `projects` (manifest). Legacy key=value format still supported transparently.
- **Shell functions can't cd**: `gitree wt` and `gitree goto` print a path; the shell function (`wt()` / `gitree-goto()`) does the actual `cd`. The subprocess cannot navigate the parent shell.

## File layout

```
bin/gitree          ← the entire CLI (single bash script)
bin/AGENTS.md       ← full workspace agent guide; shipped with the tool; symlinked
                      from other workspaces (e.g. wordpress-plugins/AGENTS.md)
package.json        ← npm package metadata + postinstall hook
CLAUDE.md           ← this file (Claude operating guide for this project)
CONTEXT-SPEC.md     ← spec for .gitree-context/ branch-context scaffolding feature
```

Project working files (README, TODO, ROADMAP) live at `../.gitree-context/gitree/`
following the mono-repo convention — they are local workspace state, not in the repo.

## How to work in this project

- All changes go directly to `bin/gitree`. No build step.
- Test by running `gitree <command>` in `~/Code/wordpress-plugins/` (Marcus's live workspace).
- When adding a command, add it to: the `case` dispatch table, the `_usage()` function, and README.md.
- Keep python3 usage minimal — JSON parsing only. Everything else must be pure bash.
- Shell function names: `wt()` and `gitree-goto()` (not `goto` — namespace clash risk).

## Key constraints

- Do not introduce non-bash/python3/git dependencies.
- Do not copy the script to other locations — `$PATH` is the install mechanism.
- `gitree install --shell-only` writes shell functions to `~/.bashrc` / `~/.zshrc`. Don't modify other rc files.
- `gitree switch` manages symlinks; `gitree restore` clones missing repos. Neither should destructively overwrite existing work.
- `gitree drop` must prompt for confirmation before `rm -rf`.

## Session files

- **README.md** — Public-facing docs: commands, config format, layout, install instructions, achievements log
- **TODO.md** — Immediate bugs, polish items, next tasks
- **ROADMAP.md** — Larger features, future phases, speculative ideas
- **CLAUDE.gitree.md** — Instructions for Claude when working *inside* a gitree workspace (not this project)
- **CONTEXT-SPEC.md** — Detailed spec for the `.gitree-context/` branch-context scaffolding feature

## Useful commands

```bash
# Run the live tool (bin/ is on $PATH)
gitree list
gitree switch -l wpml events-manager some-branch
gitree repair-head

# Release flow — push a tag and GitHub Actions publishes to npm automatically
git tag v1.2.0 && git push origin v1.2.0
```
