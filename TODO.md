# gitree — TODO

## Bugs / polish

- [ ] `gitree drop` — clean up dangling symlinks in switch directories (currently leaves stale symlinks pointing at removed project)
- [ ] `gitree restore` — doesn't restore switch symlinks, only the bare repos; should replay switch state from a stored snapshot or prompt
- [ ] `gitree list` — full paths are noisy; consider showing relative paths or just project/branch names

## Features (quick)

- [ ] `gitree remove` — offer to delete the branch from remote after removing the local worktree
- [ ] Tab completion — complete project names and branch names (currently only subcommands complete)
- [ ] `gitree switch` — print current switch state for all locations when run with no args

## npm packaging

- [ ] Publish `@pixelitemedia/gitree` to npm registry
- [ ] Verify `postinstall` (`gitree install --shell-only`) behaves correctly on headless/agent machines (no rc file to write)
- [ ] Add `--no-shell` flag for headless install (skip rc patching)
- [ ] Test install on shared hosting environment

## Chores

- [ ] Tag v1.1.0 after npm + polish pass
- [ ] Update Homebrew formula SHA after new tag
