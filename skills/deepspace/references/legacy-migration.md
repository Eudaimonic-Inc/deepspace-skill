_Load when an app uses a name-shaped id (not `app_…`), a pre-id config has no
`DEEPSPACE_APP_ID`, or identity migration needs recovery._

# Migrate a legacy GitHub app identity

Legacy apps remain GitHub-source apps. The migration changes their public app
id to an immutable `app_…` id while Worker, Durable Objects, files, secrets,
releases, routes, collaborators, and billing stay at one private physical
resource id. Never mint a replacement app or copy those stores manually.

Require a CLI whose `npx deepspace app migrate --help` exists. Work only from
the authoritative GitHub checkout. Pause deploy, rollback, undeploy, source
changes, and ownership changes during the short prepare/commit sequence; live
app traffic continues.

## Run the resumable workflow

```bash
npx deepspace app migrate --dry-run [--env <name>] [--remote <name>]
npx deepspace app migrate [--env <name>] [--remote <name>]
```

Dry-run must report `ready:true`, the exact registry rows to re-key, and every
physical-store category retained at the same `resourceId`. It reserves nothing
and changes neither registry, Git, nor files. Under `--json`, inspect
`inventory.ready`, `inventory.blockers`, `inventory.rekey`, and
`inventory.retainedPhysicalStores`; do not guess readiness from human text.

The mutating command intentionally stops between safe stages. Execute only the
returned `action.cwd` + `action.argv`:

1. prepare one server journal and write the reserved id to `wrangler.toml`;
2. manually commit and push that deterministic file change to GitHub;
3. rerun so the server verifies exact remote `HEAD` and atomically re-keys the
   registry rows; and
4. run the returned deploy action to verify the existing physical Worker under
   the new public id.

Rerun after interruption; the server journal reuses the same destination id.
Do not edit a second state file or let DeepSpace push GitHub.

A pre-id config is eligible only when the selected Wrangler environment has
both Worker `name` and `APP_NAME`, and they are identical. Missing or mismatched
declarations are ambiguous; fix the committed config deliberately before
retrying.

## Recovery before canonical deploy

```bash
npx deepspace app migrate --cancel    # prepared journal
npx deepspace app migrate --rollback  # committed registry cutover
```

Both are GitHub-first, two-stage operations: the command restores the legacy id
and stops for a manual commit/push; only a rerun after exact remote verification
cancels or rolls back server state. Once any canonical deploy attempt has been
recorded, simple rollback refuses because Cloudflare may already have mutated
the Worker. Use the separately audited reverse-deploy/support workflow; never
force a registry re-key around ambiguous live state.

After successful verification, compare route, collaborators, secrets, release
history, files/records, billing attribution, and HTTP serving against the
dry-run/canary inventory. Keep compatibility code until external owners have
been notified and the remaining legacy population is measured.

This reference overrides the generic shorthand that resources “key to app id”:
for migrated legacy apps, the public id changes while physical resources remain
keyed by the private `resourceId`.
