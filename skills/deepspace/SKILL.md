---
name: deepspace
description: >
  Use when building or maintaining real-time collaborative apps with the
  DeepSpace SDK on Cloudflare Workers — scaffolding new apps, adding
  features, debugging a `worker.ts` that imports from `deepspace` /
  `deepspace/worker` or uses `RecordRoom`, `__DO_MANIFEST__`, or `npx
  deepspace`. Also use when the user mentions DeepSpace or app.space, or
  asks for anything involving real-time sync, multiplayer state, live
  cursors / presence, whiteboards or canvases, collaborative text editing
  (Yjs), channel-based chat, per-role permissions (RBAC), Durable Object
  rooms, Stripe-backed subscriptions / paywalls / one-time products /
  tips / refunds, or one-package deploy to `.app.space` — even if they
  don't name DeepSpace explicitly.
---

DeepSpace builds real-time collaborative apps on Cloudflare Workers and deploys them to `<name>.app.space`. Start from its scaffold; do not hand-build the runtime shell.

## Start

```bash
npm create deepspace@latest <app-name>
cd <app-name>
npx deepspace dev start
```

Use `-- --template copilot` for the three-panel AI shell; the default `starter` is minimal. The directory name seeds the live subdomain label; `DEEPSPACE_APP_ID` is the app's durable identity.

Network/account operations report `not_authenticated` when login is needed:

```bash
npx deepspace auth login
npx deepspace auth whoami
```

Login opens browser OAuth. Keep it in the foreground and let the user finish it; never add a timeout or request a password. See `references/cli.md` for output and exit semantics.

## Discover before building

Inspect both catalogs before hand-building a feature:

```bash
npx deepspace add --list
npx deepspace add --info <feature>
npx deepspace integrations list
npx deepspace integrations info <provider>/<endpoint>
```

Install a matching block with `npx deepspace add <feature>`. Catalog names are summaries; inspect `--info` before ruling one out.

| Path | Owns |
|---|---|
| `src/schemas.ts`, `src/schemas/` | Collection schemas; keep the required `usersSchema`. |
| `src/pages/` | Generouted pages. `(app)/` mounts providers; `(app)/(protected)/` gates sign-in; extend `_app.tsx`. |
| `src/themes.ts`, `src/themes.css` | App-specific theme tokens; shipped themes are placeholders. |
| `src/constants.ts` | `APP_NAME`, immutable `APP_ID`, `SCOPE_ID`, role exports. |
| `worker.ts`, `src/server/` | DO manifest and ordered route assembly; action, HTTP, and realtime owners. |

Core records are envelopes (`recordId`, `data`, metadata); fields live under `.data`:

```ts
const { records } = useQuery<Item>('items', { where: { status: 'published' } })
const { createConfirmed, putConfirmed, removeConfirmed } = useMutations<Item>('items')
const { isSignedIn, isLoaded } = useAuth()
```

Read `references/sdk-reference.md` before guessing any other hook or export.

## Verify and ship

```bash
npx deepspace test run
npx deepspace deploy
npx deepspace deploy --env staging
npx deepspace dev kill
```

Run tests after runtime-affecting changes; shared behavior needs a two-user spec. Before a first deploy, replace placeholder home/theme/browser primitives per `references/uiux.md`.

Each app has exactly one authoritative Git repository: packaged DeepSpace source
or manually owned GitHub source. Inspect it with `npx deepspace app source
--json`; a configured remote alone is not authority. Use workspaces only for
DeepSpace source, and treat rollback as bundle-retention-dependent. Load the
narrow source, version-control, coordination, release, or GitHub reference
below rather than guessing from remote names.

Secrets belong in the encrypted app store (`npx deepspace secrets set KEY=value`), never hand-edited `.dev.vars`. `DEEPSPACE_APP_ID` is identity; Wrangler `name` is the URL. Collaborators can deploy and manage secrets but cannot undeploy or transfer.

## Reference routing

Before editing, load each matching reference and skip unrelated files.

| Reference | Load when |
|---|---|
| `references/workflow.md` | Starting a complete product, clone, or multi-feature build. |
| `references/cli.md` | Command discovery, login, output/actions/exits, logs, or test entry points. |
| `references/source-control.md` | Choosing, inspecting, or transferring GitHub versus DeepSpace source. |
| `references/version-control.md` | Clone/push/pull, workspaces, or repository refusals. |
| `references/coordination.md` | Status, activity, cursors, or resuming after context loss. |
| `references/releases.md` | Deploy lineage, releases, rollback, retention, or deploy Git refusals. |
| `references/github.md` | GitHub/remotes/PRs/Actions, only when requested. |
| `references/deploy.md` | Deploy mechanics, `.dev.vars`, secrets at deploy, or `--env`. |
| `references/secrets.md` | Secret store, configs, migrations, or generated cache. |
| `references/app-identity.md` | App ids, `app init`, forks, renames, list/undeploy/transfer. |
| `references/legacy-migration.md` | Name-shaped app ids, pre-id configs, or identity-migration recovery. |
| `references/collaborators.md` | Teammates, invitations, or collaborator 403s. |
| `references/sdk-reference.md` | Hooks/types beyond the three shown above. |
| `references/schemas.md` | Collections, permissions, or visibility bugs. |
| `references/auth.md` | Public/gated/mixed auth and `<AuthGate>`. |
| `references/architecture.md` | `worker.ts`, DOs, routing, shared scopes, identity stripping. |
| `references/server-actions.md` | Privileged writes bypassing caller RBAC. |
| `references/ai-chat.md` | Streamed record-aware chat and tools. |
| `references/cron.md` / `references/jobs.md` | Scheduled or durable background work. |
| `references/bindings.md` | Cloudflare bindings and autoprovisioning. |
| `references/integrations.md` | External APIs; it routes LiveKit and Google OAuth. |
| `references/payments.md` | Any money flow; never hand-roll Stripe. |
| `references/domain.md` | Custom-domain purchase, routing, or app targeting. |
| `references/uiux.md` | Theme, shell, primitives, or generic-design feedback. |
| `references/testing.md` | Specs, account pool, multi-user tests, or flaky failures. |
| `references/preview.md` | Desktop preview or stale worktree code. |
| `references/landing-design.md` | Marketing/landing/splash pages. |

## Common traps

- Scaffold UI and `slate`/`paper` themes are placeholders, not house style.
- Local UI primitives shadow SDK names; import them from `src/components/ui`, not `deepspace`.
- Pages outside `src/pages/` are not routed; data/auth hooks require the `(app)/` providers.
- Never trust port 5173 when sibling sessions exist. Use `dev start --port N` and match Playwright config; do not kill another session.
- Caller identity comes from the verified JWT, never WebSocket query params or forwarded `/api/*` headers.
- In any linked Git worktree, let `dev start`, `test run`, and `dev kill` share the CLI's per-checkout port. Claude desktop preview additionally uses the printed `wt-<name>` entry.
