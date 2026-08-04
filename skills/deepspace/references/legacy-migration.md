_Load this reference when running `deepspace app migrate`, upgrading an app
across a breaking DeepSpace contract, or handling a pre-app-id/name-shaped app._

# App migrations

`deepspace app migrate` is the permanent, versioned app-upgrade workflow. Do
not search for or invent a version-specific migration script.

```bash
npx deepspace app migrate --dry-run
npx deepspace app migrate
```

The installed SDK owns an ordered registry of structural, idempotent steps.
The command applies one safe boundary at a time and returns at most one exact
`commit`, `push`, or `deepspace deploy` action. Execute that action verbatim,
then rerun `npx deepspace app migrate` until it reports no pending migrations.
Use `--json` for a machine consumer; agents should still surface the human
meaning of each pause.

Source-only migrations update the checked-in `deepspace.migrations.json`
manifest in the same commit as their source changes. Normal deploys carry those
ids in the retained bundle and release metadata. The next run returns
`deepspace deploy` while an id is absent from the live release and reports
`up_to_date` once all ids are live. Do not edit the manifest by hand.

## Safety rules

- Start from the app's authoritative Git checkout and a clean branch.
- Run `--dry-run` first. It changes no files or registry rows.
- Let the command edit recognized source/config shapes. If it names a file and
  line it cannot transform safely, inspect that site; never use a broad search
  and replace.
- Keep physical `APP_NAME` uses for existing room ids, Durable Objects, and R2
  keys unless a migration explicitly owns a data move. Canonical platform
  authentication uses `DEEPSPACE_APP_ID`.
- GitHub-source apps remain manual: the command never writes GitHub. Commit and
  push only through the returned actions. DeepSpace-source apps use their
  normal packaged deploy/source flow.
- Do not deploy, release-rollback, undeploy, transfer, or change source
  authority concurrently while a server-backed migration is prepared.

## Legacy identity step

A name-shaped/pre-id GitHub app is one registered migration step. The command:

1. inventories the exact registry rows to re-key and physical stores to retain;
2. repairs recognized old runtime identity wiring (`x-app-name` → canonical
   `x-app-id`) without changing `APP_NAME` data namespaces;
3. reserves a strict `app_<ULID>`, writes it to `wrangler.toml`, and pauses for
   the returned commit/push action;
4. commits the registry cutover transaction; and
5. returns one canonical deploy action that updates the existing physical app.

Before the registry cutover commits, `--cancel` reverses a prepared identity
migration after restoring and publishing the legacy GitHub configuration.
After commit, recovery is forward-only: rerun the migration/deploy loop until
the canonical release manifest is live and the migration verifies.

Verify the app's real data surfaces after deploy—not only `/`: authenticated
records, files, subscriptions/charges when present, collaborators, routes, and
release/source lineage. For retained files, probe representative existing keys
and confirm their physical `apps/<resourceId>/...` paths did not change.
