_Load for deploy mechanics, generated `.dev.vars`, or named Wrangler environments. Git lineage/rollback use `releases.md`; secret commands and limits use `secrets.md`; identity/renames use `app-identity.md`._

# Deploy and environments

## Ship

```bash
npx deepspace deploy                     # <wrangler.name>.app.space
npx deepspace deploy --env staging       # [env.staging]
```

The target name comes from `wrangler.toml`; deploy has no name override. It must be a canonical lowercase label. `DEEPSPACE_APP_ID` is the immutable identity—commit it—while `name` is the URL lease.

On a repo without an id, first deploy mints one and writes it to
`wrangler.toml`; commit that identity. A name already held by another app is
refused. Changing an existing app's name is explicit (`--rename`).
Collaborators deploy against the owner's app and billing but cannot undeploy
it.

Before the first live deploy, clear the UI checklist in `uiux.md`. If subscription/product catalogs exist, deploy also syncs them; read `payments.md` before changing those files. Custom Cloudflare binding and reserved-route behavior lives in `bindings.md`.

DeepSpace-source deploy is commit-first. GitHub-source deploy preserves the
manual workflow and ships local bytes without Git operations. Read
`source-control.md` for that boundary and `releases.md` for lineage, stale
guards, and rollback retention.

## `.dev.vars` is generated

`dev start`, `test run`, `deploy`, and `secrets pull` rewrite `.dev.vars` whole
from SDK connection/auth keys plus the selected remote secret config.
`APP_IDENTITY_TOKEN` appears once the app is registered; deploy registers it,
and an earlier secrets write may register it before the first deploy. Until
then, local payments, files, and screenshot APIs lack app-origin
authentication.

The remote store is the source of truth. Deploy never reads `.dev.vars`; it reconciles Worker secret bindings from the store. Use `npx deepspace secrets …` for values and configs.

- Never print/read `.dev.vars` values, include them in artifacts, or assert on them.
- Never edit secrets into the file or pass them through unrelated command environment/argv; use `secrets set` or `secrets upload`.
- Never commit the file. Leave it ignored even if Git reports it untracked.
- An absent config is not the same as an explicitly empty config: deploy
  refuses the former and gives an executable `secrets configs create` action.

Cloudflare's build may transiently copy `.dev.vars` beside the generated server
worker. The scaffold deletes that output copy before the build completes;
deploy uploads the worker file and separate client assets. The root mode-`0600`
`.dev.vars` remains the only local materialization.

## Named environments

Each `[env.<name>]` is a separate app: distinct canonical `name`, `DEEPSPACE_APP_ID`, Durable Objects, and secret config. Initialize explicitly with `npx deepspace app init --env <name>` or let its first deploy mint the id.

Wrangler named environments do not inherit `vars`, Durable Object bindings/migrations, assets, or KV/R2/D1 declarations. Repeat the required blocks under `[env.<name>]`; otherwise the deployed Worker may boot without them.

The browser bundle must carry the selected environment's app id, and the CLI
handles this itself: `deploy`, `dev start`, and `test run` (its browser
suites) inject `VITE_APP_ID` with the id of the app actually selected (with
`--env`, the env's own id). New scaffolds consume it automatically; an app whose
`src/constants.ts` still hardcodes the literal only needs that one line
changed to:

```ts
export const APP_ID = (import.meta.env?.VITE_APP_ID as string | undefined) ?? '<scaffolded literal>'
```

Do not parse `wrangler.toml` in `vite.config.ts` or read
`process.env.CLOUDFLARE_ENV` — the CLI deletes that variable when `--env` is
active, so recipes built on it silently ship the production app id (browser
on production rooms, staging server actions on staging rooms). Gate temporary
staging-only routes on an explicit staging signal, and remove the environment
with `npx deepspace app undeploy --env staging` when finished.
