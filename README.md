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

After any install, add the `wt()` shell function for one-word navigation:
```sh
gitree install --shell-only
```
Or let `gitree install` (no flag) do the full setup including the binary copy.

## Shell function

```bash
wt() { local d; d=$(gitree wt "$@") && cd "$d"; }
```

Added automatically by `gitree install`. Then `wt feature/foo` creates the worktree and drops you in it.

## Commands

### Setup
```sh
gitree init                           # init current directory as workspace root
gitree add [<repo>] [<name>|.]        # add a repo as a bare worktree project
gitree convert multi|mono             # convert between single- and multi-repo layouts
```

### Worktrees
```sh
gitree new [<project>] <branch>       # create branch + worktree in one step
gitree wt [<project>] <branch>        # get or create a worktree; print its path
gitree remove [<project>] <branch>    # remove a worktree safely (checks for dirty state)
gitree audit [<project>]              # show branches with no worktree yet
```

### Runtime
```sh
gitree list                           # show all worktrees and load state (default)
gitree switch [<project>] [<branch>]  # point SWITCH_PATH symlink at a worktree
gitree pull [<project>|*]             # fetch + fast-forward main worktree
```

### Safety
```sh
gitree lock [<project>|*] [<worktree>]    # lock against branch switching (default: main)
gitree unlock [<project>|*] [<worktree>]  # remove the lock
```

### Tools
```sh
gitree install [--shell-only] [<dir>]  # install gitree globally + wt() shell function
gitree completion <bash|zsh>           # print shell completion script
gitree --version
```

## Config

Place a `.gitree` file at the workspace root:

```sh
SWITCH_PATH=/path/to/symlink/dir
```

`gitree switch` manages a symlink in that directory pointing to the active worktree. Useful for switching the active version of an app served from a symlinked directory (e.g., a WordPress plugin runtime).

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

## Lock behaviour

`gitree lock` installs a `post-checkout` hook in the bare repo. If someone tries to `git checkout <branch>` inside a locked worktree, the hook:

1. Reverts HEAD and the working tree to the previous branch
2. Deletes the switched-to branch if it was freshly created (reflog count ≤ 1)
3. Prints a message: `Use: gitree wt <branch> to work on another branch`

Locking is per-worktree. Lock `main` to protect your default surface while freely creating worktrees for feature branches.

## License

MIT
