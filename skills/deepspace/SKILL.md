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
many.

## One source authority

Every app has exactly one Git authority — DeepSpace source (packaged,
commit-first) or GitHub source (manual; deploys ship the local working tree,
dirty bytes included). Never maintain two sources of truth: transfer with
`deepspace app source`, and read the source-control and workspaces docs
before any source, push, pull, or clone operation.

## How to read the documentation

Consult the docs BEFORE building in an area, not after something breaks. The
reading procedure:

1. **Fetch the index once per task area:** <https://docs.deep.space/llms.txt>
   — generated from the docs on every deploy, so it is never stale — lists
   every page with a one-line summary.
2. **Pick 1–2 pages from it and fetch each as Markdown** by appending `.md`
   to the page URL (`https://docs.deep.space/guides/authentication.md`).
3. **For point lookups, prefer MCP:** `https://docs.deep.space/mcp` exposes
   `documentation_search` and `documentation_read`. Register it when your
   harness supports MCP servers, e.g.
   `claude mcp add --transport http deepspace-docs https://docs.deep.space/mcp`.
4. **For exact type signatures, read the installed package** —
   `ls node_modules/deepspace/dist/*.d.ts` — authoritative when any doc lags
   the installed version.

The docs' shape, so you pick pages fast: `/get-started/*` is setup and
project layout; `/concepts/*` is the runtime model — read it before touching
`worker.ts`, schemas, or sync behavior; `/guides/*` is one feature or
lifecycle area per page; `/design/*` is visual design; `/sdk-reference/*` is
exact exports per module; `/cli-reference/*` is the CLI, including its exit
codes and machine-executable `action` contract. The catalogs are live CLI
calls, not pages: `npx deepspace integrations list` / `integrations info
<integration>/<endpoint>` and `npx deepspace add --list`.

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
