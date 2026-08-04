---
name: deepspace
description: >
  Use when building or maintaining real-time collaborative apps with the
  DeepSpace SDK on Cloudflare Workers; when code imports `deepspace`,
  `deepspace/worker`, or uses `RecordRoom`; when running `npx deepspace`; or
  when the task involves app.space deploys, live sync, presence, collaborative
  editing, RBAC, messaging, payments, Durable Objects, or DeepSpace source.
---

# DeepSpace

DeepSpace is one package for real-time collaborative apps on Cloudflare
Workers. It provides auth, RBAC, synchronized records, messaging, integrations,
payments, and deploys to `<name>.app.space`.

## Operating sequence

1. **Authenticate before running app commands.**

   ```bash
   npx deepspace auth whoami --json
   npx deepspace auth login # only when signed out
   ```

   Login opens browser OAuth and polls for up to ten minutes. Leave it in the
   foreground and let the user finish it; never request or handle a password.

2. **Scaffold instead of assembling the runtime by hand.**

   ```bash
   npm create deepspace@latest <app-name>
   cd <app-name>
   npx deepspace dev start
   ```

3. **Inspect catalogs before hand-building a feature.** Names alone are not a
   sufficient fit check.

   ```bash
   npx deepspace add --list
   npx deepspace add --info <feature>
   npx deepspace integrations list
   npx deepspace integrations info <integration>/<endpoint>
   ```

4. **Extend the scaffold.** Keep schemas in `src/schemas.ts` and
   `src/schemas/`, routes in `src/pages/`, app providers in
   `src/pages/(app)/_layout.tsx`, and Durable Object wiring in `worker.ts`.
   `src/constants.ts` exposes the display `APP_NAME`, immutable `APP_ID`, and
   primary `SCOPE_ID = app:${APP_ID}`.

5. **Test runtime changes, then deploy.**

   ```bash
   npx deepspace test run
   npx deepspace deploy
   ```

   Multi-user behavior needs a two-user test. Use a distinct port for parallel
   apps or worktrees. Never kill a sibling session's server.

## Source authority

Every app has exactly one Git authority. DeepSpace source is the packaged,
commit-first workflow; GitHub source preserves the traditional manual workflow,
including deploying local dirty or unpushed bytes without Git operations.
Transfer with `deepspace app source`, never by maintaining two sources of truth.
Read `references/source-control.md` before any source, workspace, push, pull,
clone, or source-transfer operation.

## Rules that prevent expensive mistakes

- Treat records as envelopes: fields are under `record.data`; `put(id, patch)`
  merges a partial value server-side.
- Data and auth hooks require the `(app)/` provider boundary. Top-level pages
  are static and must not call them.
- Keep the scaffold's required `users` schema. Extend it; do not rename it.
- App secrets belong in `deepspace secrets`, never hand-edited `.dev.vars`,
  shell environment prefixes, logs, commits, or screenshots.
- Caller identity comes only from a verified JWT. Never send identity in a
  WebSocket URL or client-controlled internal headers.
- The local `ToastProvider` and UI primitives come from `src/components/ui`,
  not from the SDK.
- Treat scaffold themes and the starter home as placeholders. Give shipped
  apps their own design.
- Consumer apps require Node `>=22.15.0`; the SDK repository's development
  toolchain uses Node 24. Check the relevant `package.json` instead of guessing.

## Reference router

Before editing a surface, read every matching reference below. Each reference's
opening “Load…” gate is authoritative.

- **Workflow and coordination:** `workflow.md`, `coordination.md`.
- **CLI, source, and lifecycle:** `cli.md`, `source-control.md`,
  `version-control.md`, `releases.md`, `github.md`, `deploy.md`, `secrets.md`,
  `app-identity.md`, `legacy-migration.md`, `collaborators.md`, `domain.md`.
- **Data and runtime:** `sdk-reference.md`, `schemas.md`, `auth.md`,
  `architecture.md`, `server-actions.md`, `bindings.md`.
- **Features:** `ai-chat.md`, `cron.md`, `jobs.md`, `integrations.md`,
  `payments.md`.
- **Product quality:** `uiux.md`, `landing-design.md`, `testing.md`,
  `preview.md`.

Paths above are relative to this skill's `references/` directory. Integration
schemas are in skill-relative `assets/integrations/`. Installed package `.d.ts`
files are authoritative when a reference intentionally omits a low-use export.
