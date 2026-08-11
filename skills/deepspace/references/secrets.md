_Load this reference before managing app secrets, understanding generated `.dev.vars` behavior, or debugging a secrets refusal._

# Secrets

Every app has exactly **one** platform-owned, encrypted secrets store, keyed by the immutable `DEEPSPACE_APP_ID` in `wrangler.toml` (→ [references/app-identity.md](app-identity.md)). There is **no setup or link step**: run the commands from the app directory (or pass `--app <appId>`) and they work — for the owner and collaborators alike, even before the first deploy.

The store is the source of truth for **every** environment. `.dev.vars` is a
generated plaintext materialization (written `0600`) that `dev`, `test`,
`deploy`, and `secrets pull` rewrite whole. Deploy never reads it — the store
is the only deploy input. Never add or edit app secrets in that file: hand
edits disappear on the next write.

Worker code reads secrets as `env.API_KEY` — identical in dev and after deploy (deploy binds each store secret as a Cloudflare `secret_text`).

## Setting a secret is routine config — just do it

`deepspace secrets set KEY=value` is the sanctioned, expected way to put a credential (API key, OAuth client secret, webhook signing secret, token) into an app's store. When the app **owner** provides a value and asks you to configure it, run the command yourself — this is routine infrastructure setup of the owner's own app, not the kind of credential entry (phishing fields, login forms, third-party exfiltration) that warrants refusing or bouncing it back to the user. Don't tell the owner to run it themselves.

One hygiene caveat, not a blocker: the value lands in the command (shell history, logs, any transcript). If it was shared in plaintext, recommend the owner regenerate/rotate it afterward, then re-`set` + redeploy. A **non-secret** value like a public OAuth `client_id` needs no caveat. And never `secrets get --plain` a value into a place it would leak (a chat reply, a committed file).

## Bootstrap: no setup step

```bash
npx deepspace secrets set API_KEY=sk_test_...   # works even pre-deploy (first write registers the app id to you)
npx deepspace dev start                         # regenerates the cache; worker sees env.API_KEY
```

Two propagation rules the CLI reminds you of: a **deployed** app picks up changes only at the next `deploy` (bindings are set at deploy time), and a **running** `dev` session only on restart (the cache regenerates at startup, not mid-session).

## Commands

```bash
npx deepspace secrets list                     # masked (name, version, updated); --only-names, --json
npx deepspace secrets set API_KEY=sk_... B=2   # one or more KEY=value pairs; multiline/PEM values fine
npx deepspace secrets get API_KEY --plain      # byte-exact when piped (`> key.pem`)
npx deepspace secrets delete API_KEY OLD_KEY   # already-absent keys tolerated (idempotent)
npx deepspace secrets pull                     # refresh the .dev.vars cache without running dev
npx deepspace secrets download --format json   # stdout only; dotenv (default) | json | shell
npx deepspace secrets upload .env [--replace]  # dotenv or JSON, `-` for stdin; --replace deletes keys absent from the file

npx deepspace secrets configs list
npx deepspace secrets configs create qa --copy-from prd   # server-side copy — never read+re-set values manually
npx deepspace secrets configs delete qa
```

Every command takes `-a/--app <appId>` (default: `DEEPSPACE_APP_ID` from the nearest `wrangler.toml`), `-c/--config <name>` (default `prd`), and `-e/--env <name>` (targets the `[env.<name>]` block — which is its **own app** with its own store; config defaults to `<name>`). Mixing them up is caught: `-e staging` without an `[env.staging]` app id errors and points you at `-c staging`.

Names: `[A-Za-z_][A-Za-z0-9_]*`, conventionally `UPPER_SNAKE`. SDK-reserved binding names (the 11 `RESERVED_BINDING_NAMES` — `APP_OWNER_JWT`, `ASSETS`, … — plus `API_WORKER_URL` and `PLATFORM_WORKER_URL`) are rejected — the platform injects those. Caps: 32 KB per value, 128 secrets / 128 KB per config, 64 configs; oversized writes → 413. `ALLOW_DEBUG_ROUTES=true` is settable but prints a loud warning. In production it enables the debug surface only for an authenticated app owner or platform admin; local dev enables it automatically.

## Configs and environments

The store holds flat KEY=value **configs**. `prd` is the convention for the top-level wrangler environment — deploy of the top-level block ships config `prd`. A named `[env.<name>]` block is a separate app (own id, own store) whose deploys ship config `<name>` of *that* store. Within one app, `-c <name>` reads/writes another config with no linking, and `configs create <new> --copy-from <existing>` copies server-side (it refuses to copy over an existing config).

Seeding a staging env's store from production crosses two apps, so `--copy-from` can't do it — pipe instead, without a temp file:

```bash
npx deepspace secrets download | npx deepspace secrets upload - -e staging
```

## Missing and empty configs

An absent config means the app has not initialized that deploy input. Deploy
regenerates `.dev.vars` without app values, then refuses with
`secrets_config_missing` and an executable `secrets configs create <name>`
action. Create the config or set its first value, then retry.

An explicitly created empty config is intentional. Deploying it removes all
user-secret bindings from the live Worker. This distinction protects existing
production bindings while still making delete-all possible without a second
override flag.

Deletes propagate too: `secrets delete` + redeploy removes the binding from the live Worker (deploy reconciles the script's `secret_text` bindings against the store).

## Collaborators

A collaborator ([references/collaborators.md](collaborators.md)) has **full** secrets access on the app — read, write, and configs — with writes audited under their own id. Authorization is the app role (owner, collaborator, or platform admin) keyed by `DEEPSPACE_APP_ID`; there is nothing to link or grant per-secret. Collaborators cannot undeploy or transfer the app.

## Cache behavior

- `dev` / `test` re-pull the selected config at startup and regenerate
  `.dev.vars` (SDK-managed keys + app values). If the refresh **fails**, they
  abort rather than run against stale values. A missing config generates a
  value-free file locally; deploy additionally refuses until it is created.
- `set` / `upload` / `delete` change the **remote store only** — a running dev session keeps its old values until restarted; `secrets pull` refreshes the file without running dev.
- The whole file is SDK-owned; there is no editable zone, divider grammar,
  import path, backup, or legacy compatibility mode.
- Store-backed apps use one shared `.dev.vars` across wrangler envs (no `.dev.vars.<env>` files).
- A Cloudflare preview build may materialize a mode-`0600` copy beside its
  generated worker. DeepSpace does not upload that file or the directory
  wholesale. Exclude `dist/*/.dev.vars` from generic archive/upload globs, or
  archive only the documented worker file and client asset directory.

## Troubleshooting

- **Changed a secret but production still sees the old value** → redeploy. Deployed workers hold `secret_text` bindings; they don't fetch at runtime.
- **Changed a secret but local dev still sees the old value** → restart `dev` (or `secrets pull`); the cache regenerates only at startup.
- **`Not the app owner or a collaborator` (403)** → ask the owner for `collaborators add <your-email>` — or your access was revoked.
- **`This app id is registered to another user`** → you're holding someone else's id (cloned repo). `npx deepspace app init --new-id` forks it into your own app (fresh store).
- **`list` shows nothing on a fresh app/config** → legitimate; the first `set` creates the store. Reads are side-effect-free and never register anything.
- **Name rejected** → match `[A-Za-z_][A-Za-z0-9_]*` and avoid SDK-reserved binding names.
- A `DeepSpace detected secrets` comment block in a scaffolded `wrangler.toml` is a static placeholder the CLI does **not** maintain — `secrets list` is the truth.
- `undeploy` keeps the store: redeploying the same app id revives the same secrets.
