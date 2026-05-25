# gitree — multi-site WordPress testing

This file is agent context for gitree workspaces that use `switch.*` locations to test
plugins across multiple WordPress installs. Read `AGENTS.md` and `.gitree-context/AGENTS.md`
at the workspace root for project-specific site topology.

---

## How `gitree switch -l` maps to WordPress sites

Each `switch.*` key in `.gitree` points to a `WP_PLUGIN_DIR` for a distinct WordPress
install. Use `-l <location>` to target plugins on that site:

```bash
gitree switch [-l <location>] [<project>] [<branch>]

gitree switch events-manager gutenberg-v1          # default site
gitree switch -l wpml events-manager gutenberg-v1  # wpml site
gitree switch -l wpml events-manager               # back to main on wpml site
```

### What happens physically

```
plugins-wpml/events-manager          ← symlink in the site's WP_PLUGIN_DIR
  → <workspace>/events-manager/main  ← gitree updates THIS symlink
    → events-manager/.worktrees/gutenberg-v1/  ← the actual worktree
```

The symlink gitree manages is the one inside `WP_PLUGIN_DIR`, pointing at the worktree.
No plugin files are copied — a switch is just a symlink retarget.

---

## Verifying what version is running

```bash
# Confirm WP_PLUGIN_DIR and loaded plugin version on a specific domain:
wp --url=<domain> eval '
  echo WP_PLUGIN_DIR . "\n";
  $d = get_plugin_data( WP_PLUGIN_DIR . "/<plugin>/<plugin>.php" );
  echo $d["Version"] . "\n";
'

# List all active plugins on a domain:
wp --url=<domain> plugin list

# Check worktrees and current switch state:
gitree list
```

---

## Custom plugin directories

When a site uses a custom `WP_PLUGIN_DIR` (e.g. `plugins-wpml/`), entries in that dir
fall into two categories:

| Kind | Target | Notes |
|---|---|---|
| **Gitree override** | `→ <workspace>/<project>/main` | Managed by `gitree switch -l`; retargeted on each switch |
| **Fallback symlink** | `→ ../plugins/<name>` | Legacy; points back to default plugin dir |

Plugins with no entry in the custom dir are loaded transparently from the default
`plugins/` dir by the WP Gitree mu-plugin — no symlinks needed.

---

## Before and after a switch

```bash
# Confirm current state
gitree list

# Switch
gitree switch -l <location> <project> <branch>

# Verify (opcache.revalidate_path=1 means no server restart needed)
wp --url=<domain> eval 'echo get_plugin_data(WP_PLUGIN_DIR."/<plugin>/<plugin>.php")["Version"]."\n";'
```
