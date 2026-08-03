_Load for deploy lineage, release history, rollback, or `dirty_worktree` / `behind_trunk` / `stale_base` / `no_bundle` refusals. Build and secret mechanics remain in `deploy.md`._

# Releases and rollback

Deploy is commit-first and records the commit in an append-only release row.
For DeepSpace source it publishes the normal branch automatically. For GitHub
source it never writes GitHub: the exact local HEAD must already be pushed to
the configured GitHub remote.

- A dirty checkout is refused before build. Commit the change, using a workspace for WIP.
- A DeepSpace workspace deploy requires its exact HEAD to have been published with `workspace sync`.
- `behind_trunk` and `stale_base` prevent silently replacing newer live work. Follow the supplied action or integrate deliberately; use `--ignore-stale` only when the user explicitly wants the older tree.
- `--no-push` is legacy compatibility, not normal source selection. Do not use
  it to bypass a GitHub or DeepSpace authority refusal.

```bash
npx deepspace releases [--limit N]
npx deepspace rollback [rel_…]                 # defaults to the previous release
npx deepspace rollback rel_… --allow-do-deletion
```

Every deploy and rollback appends a release fact; rollback never rewinds history. `releases` marks each row's rollback availability (`rollbackAvailable` in JSON). Storage pressure can evict older bundles while retaining their ledger rows, and a concurrent deploy may report `bundleRetained: false`. `no_bundle` means choose another available release; rebuilding is separate.

Rollback re-ships the retained bundle without rebuilding. If it would remove Durable Object classes, the CLI refuses. `--allow-do-deletion` can permanently delete those classes' stored data: explain the exact loss and get explicit user approval before using it.
