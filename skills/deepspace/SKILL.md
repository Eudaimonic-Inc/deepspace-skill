---
name: deepspace
description: >
  Use when building or maintaining real-time collaborative apps with the
  DeepSpace SDK on Cloudflare Workers; when code imports `deepspace`,
  `deepspace/worker`, or uses `RecordRoom`; when running `npx deepspace`; or
  when the task involves app.space deploys, live sync, presence, collaborative
  editing, RBAC, messaging, payments, Durable Objects, or DeepSpace source; or
  when creating or migrating native DeepSpace documentation with
  `documentation.json`, Markdown/MDX, the documentation feature, or an
  explicitly attached documentation domain.
---

# DeepSpace

DeepSpace is one package for real-time collaborative apps on Cloudflare
Workers. It provides auth, RBAC, synchronized records, messaging, integrations,
payments, and deploys to `<name>.app.space`.

This skill is the bootstrap: how to start, how to operate, and how to consult
the documentation. The documentation at <https://docs.deep.space> is the
authority for everything else — it is written to be opinionated, so when it
recommends an approach, take that as the default rather than one option among
many. Do not rely on memorized DeepSpace APIs: verify against the docs or the
installed package's `.d.ts` files, both of which track the released SDK.

## Consulting the documentation

Use whichever surface fits the moment:

- **MCP (preferred):** `https://docs.deep.space/mcp` exposes
  `documentation_search` and `documentation_read`. Register it when your
  harness supports MCP servers, e.g.
  `claude mcp add --transport http deepspace-docs https://docs.deep.space/mcp`.
- **Raw Markdown:** append `.md` to any page URL
  (`https://docs.deep.space/guides/authentication.md`); fetch
  <https://docs.deep.space/llms.txt> for the full index with one-line
  summaries.
- **In the app:** exact type signatures live in
  `node_modules/deepspace/dist/*.d.ts` (index, schema, worker, server,
  testing, documentation, documentation-react) —
  these are authoritative when any doc lags the installed version.

Consult the docs BEFORE building in an area, not after something breaks: read
the matching guide when adding a feature (auth model, payments, jobs, cron,
AI chat, custom bindings…), the concepts pages when touching `worker.ts`,
schemas, or sync behavior, and the CLI reference before scripting commands.

## Documentation router

| When working on | Read |
| --- | --- |
| First app, project layout | `/get-started/quickstart`, `/get-started/project-structure` |
| Worker, Durable Objects, scopes, WebSockets | `/concepts/architecture`, `/concepts/realtime-sync` |
| Collections, permissions, visibility | `/concepts/data-model`, `/concepts/permissions`, `/guides/data-storage` |
| Sign-in models, gated pages | `/guides/authentication` |
| Messaging, presence, collaborative editing, canvas | `/guides/messaging`, `/guides/presence-and-cursors`, `/guides/collaborative-editing`, `/guides/canvas` |
| AI chat, one-shot LLM calls | `/guides/ai-chat` |
| Money flows | `/guides/payments` |
| Privileged writes bypassing RBAC | `/guides/server-actions` |
| Scheduled + background work | `/guides/scheduled-jobs`, `/guides/background-jobs` |
| External APIs (`integration.post`) | `/guides/external-apis`, plus the live catalog: `npx deepspace integrations list` / `integrations info <integration>/<endpoint>` |
| Google OAuth endpoints (`google/*`) | `/guides/google-oauth` |
| Real-time audio/video rooms | `/guides/livekit` |
| File uploads | `/guides/file-uploads` |
| Custom Cloudflare resources, metering | `/guides/custom-bindings` |
| Deploys, environments | `/concepts/deployment` |
| Secrets | `/guides/secrets` |
| Releases, rollback, deploy refusals | `/guides/releases-and-rollback` |
| App ids, renames, undeploy, transfers | `/guides/app-identity` |
| Source authority, GitHub vs DeepSpace source | `/guides/source-control` |
| Workspaces, push/pull, activity | `/guides/workspaces` |
| Custom domains | `/guides/custom-domains` |
| Teammates on one app | `/guides/collaborators` |
| Local dev server, worktrees, desktop preview | `/guides/dev-workflow` |
| Building a whole app end to end | `/guides/building-an-app` |
| Landing pages, visual design, product polish | `/design/overview` and the design section |
| Tests | `/guides/testing` |
| Native documentation sites | `/guides/documentation` |
| Exact exports and signatures | `/sdk-reference/overview` and its per-module pages |
| CLI output, exit codes, the `action` contract | `/cli-reference/overview` |
| Any CLI command | `/cli-reference/commands` |

Paths are relative to `https://docs.deep.space`; append `.md` for raw
Markdown.

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
Transfer with `deepspace app source`, never by maintaining two sources of
truth. Before any source, workspace, push, pull, clone, or source-transfer
operation, read the docs on source control and releases.

## Rules that prevent expensive mistakes

- Treat records as envelopes: fields are under `record.data`; `put(id, patch)`
  merges a partial value server-side.
- Disable write controls until `useMutations().ready`. Use a confirmed mutation
  when navigation, access changes, or a success message depends on acceptance.
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
- Run apps on a supported Node line — the installation guide
  (`/get-started/installation`) is the authority on which lines those are.
