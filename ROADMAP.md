# gitree — ROADMAP

## Near-term

- **`.gitree-context/` branch context scaffolding** (spec in CONTEXT-SPEC.md) — when a worktree is created, scaffold `.branch-context/` with `README.md`, `AGENTS.md`, `TODO.md`, `ROADMAP.md` so agents have per-branch context immediately. Symlinked from `.gitree-context/<project>/<branch>/` so context is workspace-global, not inside the repo.
- **Remote agent / shared hosting install** — `npm install -g @pixelitemedia/gitree` on a remote server; agents use `gitree wt <plugin> <branch>` to get isolated worktrees for testing. The `--no-shell` flag and headless-safe install path are prerequisites.
- **wp-gitree WordPress plugin** — WP plugin that wraps gitree concepts: knows about the active worktree, handles `plugins_url()` mapping, exposes gitree workspace state to WP admin. Companion to `symlink-plugin-paths.php` mu-plugin.

## Mid-term

- **`gitree switch` — atomic swap** — create new worktree before dropping old symlink so there's zero downtime on a live dev server during a switch.
- **`gitree status`** — single-command overview: current switch state for all locations, dirty worktrees, HEAD drift warnings, locked worktrees.
- **`gitree sync`** — pull all projects in the manifest in parallel (`git fetch` + fast-forward `main/`).
- **Saved switch snapshots** — `gitree switch --save <name>` to save the current switch state for all locations; `gitree switch --restore <name>` to replay it. Enables named "configurations" (e.g. `production`, `wpml-sprint`, `stripe-integration`).

## Someday / Maybe

- **TUI** (`gitree tui`) — interactive worktree browser with arrow-key navigation, switch preview, dirty-state indicators.
- **Multi-machine sync** — rsync `.gitree` + `.gitree-context/` to another machine, `gitree restore` reproduces the full workspace.
- **`gitree prune`** — find and remove worktrees whose branches have been merged/deleted on remote.
- **GitHub Actions integration** — `gitree wt` in CI to test multiple feature branches in isolated environments within the same runner.
- **Shell completion for branch names** — query bare repo refs to complete branch names in `wt`, `switch`, `remove`, etc.
