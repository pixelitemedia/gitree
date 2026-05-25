# gitree

Bare-repo worktree manager — create, navigate, and switch worktrees with one command.

Works with any bare-repo-with-worktrees Git layout. Keeps your `main` worktree locked so branch switching is always through an explicit worktree, never by accident.

## Install

**Homebrew (macOS / Linux):**
```sh
brew tap pixelitemedia/tap
brew install pixelitemedia/tap/gitree
```

**npm:**
```sh
npm install -g @pixelitemedia/gitree
```

**Manual:**
```sh
curl -fsSL https://raw.githubusercontent.com/pixelitemedia/gitree/main/bin/gitree \
  -o /usr/local/bin/gitree && chmod +x /usr/local/bin/gitree
```

After any install, add shell functions for one-word navigation:
```sh
gitree install --shell-only
```

## Shell functions

```bash
wt()          { local d; d=$(gitree wt   "$@") && cd "$d"; }
gitree-goto() { local d; d=$(gitree goto "$@") && cd "$d"; }
```

Added automatically by `gitree install`. Then `wt feature/foo` creates the worktree and drops you in it. `gitree-goto events-manager main` navigates to a project's worktree from anywhere.

## Commands

### Setup
```sh
gitree init                           # init current directory as workspace root
gitree add [<repo>] [<name>|.]        # add a repo as a bare worktree project
gitree restore                        # clone all projects listed in .gitree manifest
gitree convert multi|mono             # convert between single- and multi-repo layouts
```

### Worktrees
```sh
gitree new [<project>] <branch>       # create branch + worktree in one step
gitree wt [<project>] <branch>        # get or create a worktree; print its path
gitree goto [<project>] <branch>      # alias for wt (use via gitree-goto shell fn)
gitree remove [<project>] <branch>    # remove a worktree safely (checks for dirty state)
gitree drop <project>                 # remove an entire project (prompts for confirmation)
gitree audit [<project>]              # show branches with no worktree yet
```

### Runtime
```sh
gitree list [<project>]               # show worktrees; scoped to one project if specified
gitree switch [-l <loc>] [<project>] [<branch>]  # point switch symlink at a worktree
gitree pull [<project>|*]             # fetch + fast-forward main worktree
```

### Safety & maintenance
```sh
gitree repair-head [<project>|*]          # fix bare repo HEAD drift to match remote default
gitree lock [<project>|*] [<worktree>]    # lock against branch switching (default: main)
gitree unlock [<project>|*] [<worktree>]  # remove the lock
```

### Tools
```sh
gitree install [--shell-only] [<dir>]  # install gitree globally + shell functions
gitree completion <bash|zsh>           # print shell completion script
gitree --version
```

## Config — `.gitree`

Place a `.gitree` JSON file at the workspace root:

```json
{
  "switch": {
    "default": "~/path/to/symlink/dir",
    "staging": "~/path/to/staging/symlink/dir"
  },
  "projects": {
    "my-plugin": "https://github.com/org/my-plugin.git",
    "another-plugin": "https://github.com/org/another-plugin.git"
  }
}
```

- **`switch`**: named symlink directories. `gitree switch` manages a symlink in the target dir pointing to the active worktree. Use `-l <name>` to target a named location.
- **`projects`**: full project manifest. `gitree restore` uses this to clone missing repos on a new machine.

Legacy key=value format (`SWITCH_PATH=...`) is still supported for single-location setups.

`gitree add` and `gitree drop` keep the manifest in sync automatically.

### Multiple switch locations

```sh
gitree switch events-manager               # default location → main/
gitree switch events-manager feature/foo   # default location → .worktrees/feature--foo/
gitree switch -l staging events-manager    # staging location → main/
```

Project name is auto-detected when run from inside a project directory.

## Layout

gitree works with the bare-repo-with-worktrees pattern:

```
<project>/
├── .bare/              ← bare repo: refs, objects, worktree metadata
├── .git                ← 1-line file: "gitdir: ./.bare"
├── main/               ← default working surface (main branch)
└── .worktrees/
    └── <branch>/       ← additional worktrees, created on demand
```

Branch names with slashes map to dir names with `--` (`feature/foo` → `.worktrees/feature--foo/`). Always pass the real branch name — gitree handles the translation.

## Shell completion

```sh
# zsh
eval "$(gitree completion zsh)"

# bash
eval "$(gitree completion bash)"
```

Add to your `.zshrc` / `.bashrc` for persistent tab completion.

## HEAD drift

Worktree operations can move the bare repo's HEAD off the default branch. This causes `git pull`, `git log`, and similar to target the wrong branch. Run `gitree repair-head` to query the remote for the authoritative default branch and reset bare HEAD accordingly.

```sh
gitree repair-head           # fix all projects
gitree repair-head my-plugin # fix one project
```

## WordPress / symlinked plugin note

When plugins are served via symlinks from `~/Code/` into `wp-content/plugins/`, PHP's `__FILE__` resolves the symlink target. WordPress's `plugins_url()` requires the real path to share a prefix with `WP_PLUGIN_DIR` — paths under `~/Code/` don't satisfy this.

Fix: add `wp-content/mu-plugins/symlink-plugin-paths.php` to pre-register every symlinked plugin dir in `$wp_plugin_paths`. Asset URLs then resolve correctly regardless of which worktree is active.

## Lock behaviour

`gitree lock` installs a `post-checkout` hook in the bare repo. If someone tries to `git checkout <branch>` inside a locked worktree, the hook:

1. Reverts HEAD and the working tree to the previous branch
2. Deletes the switched-to branch if it was freshly created (reflog count ≤ 1)
3. Prints a message: `Use: gitree wt <branch> to work on another branch`

Locking is per-worktree. Lock `main` to protect your default surface while freely creating worktrees for feature branches.

## Achievements

- v1.0.0 — initial release: bare-repo layout, worktree create/remove/lock, switch, pull, audit, install, shell completion
- JSON `.gitree` config with named switch locations (`switch.*`) and project manifest (`projects`)
- `gitree switch -l <location>` — multi-location switching (e.g. default / wpml / wpwc)
- `gitree restore` — clone all manifest projects on a new machine (idempotent)
- `gitree drop` — remove a project and update manifest
- `gitree repair-head` — fix bare repo HEAD drift by querying remote default branch
- `gitree goto` / `gitree-goto()` shell fn — navigate to any project worktree from anywhere
- `gitree list [project]` — scoped listing, auto-detected from cwd
- HEAD drift detection in `gitree list` output with repair hint
- Homebrew tap: `pixelitemedia/tap`
- npm package: `@pixelitemedia/gitree`

## License

MIT
