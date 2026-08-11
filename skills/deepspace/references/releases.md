_Load for deploy lineage, release history, rollback (including its
code-not-secrets boundary), or `dirty_worktree` / `behind_trunk` /
`stale_base` / `no_bundle` refusals. Build and secret mechanics remain in
`deploy.md` and `secrets.md`._

# Releases and rollback

Every deploy records an append-only release fact, but source behavior differs:

- **DeepSpace source** is commit-first. Deploy publishes the attached clean
  branch, records its commit, and enforces workspace/ancestry/stale-base guards.
- **GitHub source** ships the local working tree without Git operations. Dirty
  or unpushed bytes are valid; the release has `commitOid: null` and retains the
  configured repository/source revision as metadata.

`--no-push` is a one-release legacy escape for an unclaimed or DeepSpace-source
app. It never bypasses GitHub source authority and does not change providers.

```bash
npx deepspace releases [--limit N]
npx deepspace rollback [rel_…]                 # defaults to the previous release
npx deepspace rollback rel_… --allow-do-deletion
```

Rollback re-ships a retained bundle without rebuilding or changing Git. Every
rollback appends another release fact. `releases` marks `rollbackAvailable` in
JSON; storage pressure can evict an old bundle while retaining its ledger row.
`no_bundle` means choose another retained release. Rollback restores **code,
not secrets**: the rolled-back worker keeps the secret bindings already live
on the script from the **last deploy** — neither the values live when that
release originally shipped nor a fresh read of the store. A `secrets set` made
after the last deploy does not reach a rolled-back worker until the next
`deploy`, so a rollback across a secret rotation needs the live bindings
checked against the older code. Each release row also carries `environment` in `--json`
(`'production'` for all deploys today; the field exists ahead of environment
targets).

If rollback would remove Durable Object classes, the CLI refuses.
`--allow-do-deletion` can permanently delete those classes' stored data:
explain the exact loss and get explicit user approval before using it.
