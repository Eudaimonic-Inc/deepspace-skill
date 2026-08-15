# Information-loss audit for the thin-out (2026-08-14)

> **RESOLUTION (2026-08-14).** The absorption this audit demanded shipped in
> deepdotspace-site#71: 16 new pages (including the design section and
> google-oauth/livekit), the documented contradictions corrected, and the
> version stamps removed. The body below is the historical evidence record,
> not an open work queue. The merge gate for THIS PR is now: #71 merged +
> deployed.

Six parallel reviewers audited every file deleted by this PR against the live
documentation (each page fetched as `.md`), adjudicating ambiguous cases
against the SDK source. Deleted content was read from `git show HEAD~1:...`
(commit `c686cf8` is the last full-content state). This file is the unit-level
ledger; the PR comment carries the executive summary and the revised merge
gate. Result across 30 reference files: 0 fully covered; 17 partial
(Reviewers 1–3); 2 missing with the prior docs mapping wrong (google-oauth,
livekit); and Reviewer 4's 9 lifecycle files effectively missing (no docs
home); design tree absent from docs entirely; integration YAMLs partially
lost. The docs self-declare CLI 0.12.0 against an SDK at 0.19.5,
and in ~10 places the deleted references were silently correcting stale docs.

---

## Reviewer 1 — data/runtime (schemas, auth, architecture, server-actions, bindings, sdk-reference)

### schemas.md — PARTIAL
- `deepspace/schema` entry point + decision rule ("runtime-neutral entry for every schema imported by browser/shared code; reserve `deepspace/worker` for worker-only helpers"). The entry exists in `packages/deepspace/package.json` exports, but no docs page mentions it — docs teach `deepspace/worker` imports everywhere. Should carry: `/sdk-reference/worker/schemas` or `/sdk-reference/overview`.
- `settingsSchema` is scaffold-starter only — "no SDK feature depends on it — customize the columns freely, or remove it entirely." Docs register it in examples but never say it's removable (usersSchema's don't-replace rule IS covered). Should carry: `/concepts/data-model`.
- `writableFields` restricts create as well as update — "that role may only supply the listed columns on create or update." Docs comment says only "can be updated by this role"; source (`server/handlers/records.ts`) runs `checkFieldPermissions` on both paths. Should carry: `/sdk-reference/worker/schemas`.
- JSON auto-serialization applies inside server actions — `tools.get`/`tools.query` return already-parsed JSON columns (client-side behavior is documented; the server-action side isn't). Should carry: `/guides/server-actions`.
- CLI up-front schema lint — `deepspace dev start`/`deploy` also run the lint and print "Schema lint: N warnings in src/schemas.ts" in the terminal (docs describe only worker-console `[schema-lint]` at boot; `cli/lib/schema-lint.ts` exists). Should carry: `/sdk-reference/worker/schemas` or CLI reference.
- ownerField/userBound lint nuance — when every ordinary client role has `create: false`, the lint deliberately does not warn (server writes may legitimately assign another user); "do not add `userBound` mechanically in that shape." Docs give only cause/fix. Should carry: `/sdk-reference/worker/schemas` lint section.
- `useConversations().createChannel(name)` defaults to `Visibility: 'public'`/`Type: 'public'` (whole directory sees it); override via `createDM`/`createGroupDM` or direct create with `Visibility: 'private'` + `ParticipantIds`; app-scoped `useChannels().create` uses lowercase `{ name, type }`. The ParticipantIds pattern is documented in `/concepts/permissions`, but this default-public gotcha is not (useConversations is wholly undocumented). Should carry: `/sdk-reference/client/messaging`.
- Not counted (docs deliberately correct the reference): "channel `type` gates access at schema level" (docs: type is informational, `read: true`); `PermissionContext.isTeamMember` internals (docs' `team_members`-collection mechanism matches current source).

### auth.md — PARTIAL
- Server-side gate recipe for fully-private apps — scaffold `realtime-routes.ts` `wsRoute` deliberately accepts tokenless WS connections; to go private, replace the optional-token block with `401` on missing/invalid token ("that single edit gates every route using the helper"), with the explicit warning "do not treat `<AuthGate>` as server authorization." Docs document the three token states but give no hardening recipe or warning. Should carry: `/guides/authentication` (fully-gated) or `/concepts/architecture`.
- users-directory privacy contract — anonymous sockets receive no `useUsers()` directory; authenticated results follow the app's `users` read policy then a public-identity projection for non-admins (admins get full fields — which is also why `useUserLookup().getEmail` works only for admins); fresh scaffold `member.read: 'own'` means a member's directory is self-only; plain `useQuery('users')` bypasses the projection and returns every schema-allowed field — "never set `read: true` unless every current and future users column is intentionally member-visible." Entirely absent. Should carry: `/sdk-reference/client/records` (useUsers) + `/concepts/permissions`.
- `useAuthStatus()` and `useAuthProfileReady({ requireUser })` — auth-only readiness safe outside `RecordProvider`; auth+profile readiness for profile-backed UI (skeleton while `isSignedIn && userLoading`). Both exist in `src/client/status.ts`; absent from `/sdk-reference/client/auth`.
- `RecordProvider onWriteError` — the only surface where server-rejected optimistic writes surface; scaffold wires it to toasts (`permission` → warning, else error); retrofit guidance for older apps. Prop exists in source (`client/storage/context.tsx`); missing from the records-reference prop table.
- Blank-page tell — `RecordProvider` without `allowAnonymous` renders nothing for signed-out visitors; on localhost the SDK shows a signed-out diagnostic box, production stays blank. Debugging recipe absent.
- `data-testid="app-root"` is the canonical "shell mounted" test hook — don't rename (still in template `_app.tsx` with that exact comment; `/guides/testing` names other testids but not this one).
- AuthBoot contract — local helper, not the SDK's `<AuthGate>`; waits for `useAuthStatus().isLoaded` then mounts the data layer for signed-in AND signed-out users. Project-structure names it in the mount chain only.

### architecture.md — PARTIAL
- Route reservation (`run_worker_first`) — baseline `["/api/*","/ws/*","/internal/*","/v1/*","/_deepspace/*",…]` is platform-reserved (apps can't override); `/v1/*` exists so OpenAI-compatible routes beat the SPA fallback and `/_deepspace/*` is the billing-hook proxy — don't strip either; apps may append their own dynamic prefixes and `deploy` merges the extras. Zero docs coverage (confirmed present in template `wrangler.toml`). Should carry: `/concepts/architecture` or `/concepts/deployment`.
- Scope ids key to the immutable app id, not the name — template ships `SCOPE_ID = app:${APP_ID}` with `__APP_ID__` replaced by the app id; worker keys the DO via `env.DEEPSPACE_APP_ID`; same for `dir:<APP_ID>` (platform maps it to the permanent resource id for identity-migrated apps). Docs say `app:<APP_NAME>` / `dir:<appHandle>` — stale relative to the SDK; the deleted file carried the accurate fact.
- `workspace:default` is the only workspace instance — the `workspace:` namespace is reserved but not multi-instance.
- Post-rename subdomain reservation — "the old subdomain stays reserved for you for 30 days"; docs' deployment page instead says renaming "orphans the old one." (30-day claim not confirmed in platform source — lost-but-unverified.)
- Anonymous guests are session-local, not accounts — do not persist or promote the `anon-<uuid>` identity into an account. Docs describe the id assignment only.
- "No user-scope DOs" design rule — user-scoped data lives in app DOs with RBAC filtering.
- Covered: DO classes/routes, docs-aware `/ws/yjs` role resolution + don't-replace warning, cross-app proxy (binding + `platformWorkerFetch` + error 1042 + dev URL fallback), full security model (identity stripping, three states, `X-App-Action`/`X-User-Id`/`X-Billing-User-Id`), app-name canonical form + rename-follows-id, proxy helpers (incl. Set-Cookie rationale, actionable throw), dev-start guidance, schemas-baked/no-mux rules.

### server-actions.md — PARTIAL
- `tools.registerUser(opts)` — seeds/refreshes a `users` row (CLI-triggered actions where the caller may have no row); bypasses SYSTEM_MANAGED column stripping so name/email/imageUrl are written. Exists in source (`server/utils/action-types.ts`, `record-room.ts`); the docs `ActionTools` interface omits it entirely. Should carry: `/sdk-reference/worker/server-actions`.
- Not counted (docs deliberately correct it): the "wire carries `record` but the type says `{ recordId }` — cast if needed" note; docs now state only `{ recordId }` returns and prescribe re-fetch. Everything else — context shape, `callerJwt` forwarding + hygiene, billing routing incl. unlisted-defaults-to-developer, owner gate, upsert-by-id, `tools.query` sees everything, testing — is covered, often more precisely.

### bindings.md — PARTIAL (near-covered)
- `runMigrations` concurrency caution — run "in a controlled initialization path (avoid concurrent callers)"; docs suggest "inside fetch handler or at module init" with no concurrency warning. Should carry: `/sdk-reference/worker/bindings`.
- Meta-table rationale — a meta-table is used because D1's SQLite authorizer rejects `PRAGMA user_version` writes with `SQLITE_AUTH` (minor why-fact).
- Everything else covered: declaration matrix + companions, auto-provisioning table + `app-<appName>-` prefixing, registry + adopt-on-conflict, 11 reserved names + DO-class blind spot, secret-collision rule, metering formulas incl. the storedCount undercount warning, `COST_RATES` rollup formula, undeploy semantics (non-empty R2 skip), all gotchas.

### sdk-reference.md — PARTIAL
- Entry-point map understated — `/sdk-reference/overview` says the package has "three entry points"; `deepspace/schema` and `deepspace/server` are real exports (package.json) — `deepspace/server` appears only incidentally on bindings/payments pages, `deepspace/schema` nowhere; their `.d.ts` paths (`dist/schema.d.ts`, `dist/server.d.ts`) are missing from the overview's table.
- Undocumented existing client APIs (all verified in current source): `useConversations` (+ `createChannel`/`createDM`/`createGroupDM`), `useCommunities`, `usePosts` (messaging ref says "use the directory hooks" without naming them); `useVoiceAgent` (managed OpenAI Realtime WebRTC session — and the "no LiveKit room hook exists" caveat); `useAsyncResource` / `usePagedResource`; `useAuthStatus` / `useAuthProfileReady`.
- `RecordProvider.onWriteError` and `useMutations().ready` — the ready gate (`RecordRoomNotReadyError`, code `not_ready`; "disable write controls until `ready`") and the write-error surface are absent from the records-reference signatures.
- useUsers directory projection / `getEmail` admin-only (same loss as auth.md item 2).
- Environment helpers — `detectEnvironment`, `getEnvironmentConfig`, `getApiUrl`, `getPlatformWorkerUrl`, `getAuthUrl`, `isLocalDev`, `isProduction`, `resetEnvironmentCache`, `ENV`: no docs page mentions any of them.
- Custom WS-client surface — `MSG`, `ClientMessage`, `ServerMessage`, `clientBuild`, `dispatch`, `encode`: unmentioned.
- `applyAiToolDefaults` — worker/ai page covers the rest of the AI surface (`BUILT_IN_TOOLS`, `ToolSchema`, compaction, history trust boundary) but not this export.
- Covered: rooms (incl. `BaseRoom`, `DEFAULT_DO_MANIFEST` two-entry caveat, `onHydrateState`, `enqueueJob`, LiveKit/no-media-DO note), realtime (`connected` vs `synced`, `canWrite` no-op semantics incl. viewport exemption), files (`list()` async, `R2FileInfo` shape), integrations (`issues`, PlatformProvider/useInbox/usePlatformWS), payments (server helpers + error classes), theming, worker auth/cron/proxy-helpers, schema constants, testing fixture exports, `captureScreenshot`.

Cross-cutting: guide/concept-level content survived well; the systematic losses are (a) whole API surfaces that never got a docs home, (b) security/privacy decision guidance, (c) two places where docs are stale against source (`app:<APP_NAME>` vs `app:${APP_ID}`; `writableFields` update-only phrasing).

---

## Reviewer 2 — features (ai-chat, cron, jobs, integrations + nested, payments, native-docs)

Overall: 0 fully covered, 6 PARTIAL, 2 MISSING. Two files actively contradicted by the docs.

### ai-chat.md — PARTIAL
(/guides/ai-chat + /sdk-reference/worker/ai cover the architecture, schemas/RBAC rationale, endpoints, billing, compaction, wire helpers, limitations)
- LOST — copilot-template rule: copilot-template apps already ship ChatPanel (`src/components/chat/ChatPanel.tsx` + shell ChatDock); running `add ai-chat` there installs a second divergent ChatPanel; the feature is for starter-template apps only. Belongs in /guides/ai-chat.
- LOST — ChatPanel embedding contract: `chatId={null}` auto-create, `onChatCreated`, `disabled` to suppress duplicate auto-create; overlay reconciliation (pending turns removed as persisted recordIds arrive; stop preserves partial text; chat-switch aborts+clears; Retry re-sends last content).
- LOST — testing recipe: `api.spec.ts` pattern, `X-Asst-Id` matches `/^asst-/`, one test per turn-shape (text/tool/multi-step/abort), "assert behavior not parser fidelity — SDK already unit-tests `decodeAiStreamChunk`", 401/404 auth assertions.
- LOST — `registerAiChatRoutes(app, resolveAuth)` registration API; 32K-model compaction tuning (~40,000 chars; docs stop at 128K→120,000); `fail-tool-input` fires with no preceding upsert; exact `parts` item shape; consecutive-user-message dedup step; residual failure mode (assistant write can still fail after user write succeeds — guide's "zero orphan rows" overstates); `buildSystemPrompt` guardrail content (required-marker `!`, "confirm before destructive", "RBAC-denied → tell user, don't retry").
- CONFLICT (docs likely newer/corrected): reference's `ALLOWED_MODELS`/`DEFAULT_MODELS` 2-file sync + model IDs superseded by SDK catalog (`resolveDeepSpaceAgentModel`); installer described as 5 files under `(app)/(protected)/` with auto `npm install`, docs say 3 files, no auto-install. One side is stale — needs reconciling, not restoring.

### cron.md — PARTIAL
(/guides/scheduled-jobs + /sdk-reference/worker/cron cover tasks, ctx shapes, wiring, monitor, trigger-not-wait)
- LOST + CONFLICT — scope rule: reference scopes room ids by immutable `app:${env.DEEPSPACE_APP_ID}` ("names are mutable URL labels"); every docs page uses `app:${env.APP_NAME}` as "the scaffold convention".
- LOST — migration note: stale `cron.json` / `handleCron` / `/internal/cron` are the pre-CronRoom pattern; delete and rewrite. `cron.json` appears in no docs page.
- LOST — alarm-path testing exception: `trigger` bypasses the alarm; verifying the alarm itself needs `intervalMinutes: 1` and a ~130s budget.
- LOST — installer behaviors: customized `src/cron.ts` → exact manual-integration refusal (not overwrite); page placed at `src/pages/(app)/cron-log.tsx` so providers mount; feature ships no Playwright spec.
- CONFLICT — scaffold `/ws/cron/:roomId` role: reference says it resolves the caller's current app role (viewers/anon read-only); docs say "role: 'member' for any signed-in connection".

### jobs.md — PARTIAL, with a critical security contradiction
(/guides/background-jobs + /sdk-reference/client/realtime cover nearly everything else, incl. cancel/10s-timeout semantics)
- CONTRADICTED — write authorization: reference (latest state): `enqueue`/`cancel`/`retry` require a verified member/admin write role, rechecked on every mutation. Docs guide AND realtime reference state the opposite: "The JobRoom DO does not enforce a role… any signed-in (or anonymous) connection can enqueue", advising client-side-only gating. If the reference reflects shipped code, two docs pages teach a stale security model.
- LOST — public-producer recipe: for a deliberately public enqueue path, expose one app-owned HTTP action that validates a named job type + bounded payload, rate-limits, then calls `enqueueJob`; paid jobs stay owner-only.
- LOST (minor) — crash-recovery outcome: rescued `running` rows are retried if attempts remain, else marked failed; late cross-isolate return value is discarded; `useJobs` auto-reconnect. Same `APP_NAME` vs `DEEPSPACE_APP_ID` room-id discrepancy as cron.

### integrations.md — PARTIAL
(/guides/external-apis + /sdk-reference/client/integrations cover discovery, envelope/issues, billing table, empty-success gotcha; `--yes` cost-confirm is in /cli-reference/commands)
- LOST — "the caller cannot redirect billing with a header" (billing section should carry it).
- LOST — "app-rate-limit every UI that can trigger a paid call" (docs say auth-gate only).
- LOST — test discipline: "never flip billing modes just to make a test pass"; "mock only the external integration boundary; keep app-internal hooks and routes real".
- LOST — UI-state guidance: loading/error-with-local-retry/empty/success; `useAsyncResource` (one-shot) / bounded `usePagedResource` (feeds); failed resource must not reload the page. Neither hook appears anywhere in the docs.

### integrations/google-oauth.md — MISSING (and the docs teach the opposite contract)
No Google OAuth page exists (probes 404).
- The mapped pages document `requiresOAuth` as `{ success: false, error: 'requiresOAuth', connectUrl }`; the reference says the api-worker returns HTTP 200 `{ success: true, data: { requiresOAuth: true, provider, scopes[], authUrl } }`, warns "check `result.data?.requiresOAuth`, never `success === false`", `data ?? result` unwrap, and that legacy error-shapes no longer apply. Opposite envelope, different field name — per the reference, the docs shape is the stale one.
- Also lost: `'user'` billing is non-negotiable for `google/*` + rationale (developer billing operates on the owner's Gmail/Drive/Calendar for every visitor); no separate auth-url endpoint — POST the real endpoint whose scope you need, one code path serves connect-button and data-load; each endpoint requests only its own scope; incremental consent with scope-unioning; per-feature gating rule + the composite-gate deadlock anti-pattern; one response covers all three failures (no tokens / 403 scope / 401 revoked); per-scope status flags `{connected, gmailSend, gmailRead, gmail, calendar, drive, contacts}` with broader-implies-narrower (docs claim `{connected, scopes[]}` — conflict); the three `page.route` mock recipes (connected state, nested-envelope recovery, disconnect) and "real round-trips are deploy-and-manual only; paper-trail the gap".

### integrations/livekit.md — MISSING
No LiveKit page (404s; only "no SDK DO class, use `livekit/*`" in rooms ref and a name-drop in external-apis). Lost in full:
- No `useMediaRoom` hook; SDK = room lifecycle proxy only, client WebRTC (`livekit-client` / `@livekit/components-react`) is a self-installed peer dep.
- Five-endpoint table with per-endpoint billing: `generate-token` free (ttl 60–86400s, default 3600; room auto-creates; no caps), `create-room` reserves worst case `maxParticipants × durationMinutes × $0.0005`, settles down on `settle-room` with refund; abandoned rooms capped at reservation by a cron; self-hosted (non-`*.livekit.cloud`) reservation voided; `settle-room` idempotent + creator-only, skip it and the full reservation bills; `delete-room` does not settle billing and has no creator check (any authenticated caller can delete any room — app-layer gating mandatory); `list-rooms` free Twirp shape.
- Ad-hoc/free vs billable flow decision rule; auth-gate the token-minting page (leaked token = room access until TTL); host-only teardown via `createdBy`; schemas pointer `assets/integrations/livekit.yaml`.

### payments.md — PARTIAL (best covered)
(guide + client/payments carry the catalog, hooks, gating rules, cancel/refund constraints, webhook-lag rule)
- LOST — `taxCode` default (`txcd_10000000`, digital services) and the rule that a taxCode applies to the whole plan (use separate plans when monthly vs annual tax treatment differs).
- LOST — dropping a `products.ts` row deactivates the product; existing purchases stay valid.
- LOST — `owned` is true only for a non-refunded matching purchase (docs omit the refund qualifier).
- Softened — "Do not install Stripe libraries" imperative appears only descriptively; `not_app_owner` 403 error code unnamed.
- Docs-internal conflicts observed while checking: trialDays max 365 (guide) vs max 90 (client/payments type); `currentPeriodEnd` Unix ms (guide) vs unix seconds (client/payments type).

### native-docs.md — PARTIAL, heavy losses
(/guides/documentation covers wiring, basic config, /docs + domains serving, search/assistant/MCP existence, reserved paths)
- LOST — install path: `npx deepspace add --info documentation` / `add documentation` appears nowhere (guide shows only manual wiring; /cli-reference/commands never mentions the feature); "there is no `deepspace docs` command group — don't invent commands".
- LOST — config surface: `documentation.json` also owns links, redirects, assistant/MCP access + billing mode, contextual actions, SEO, and OpenAPI inputs; "inspect the installed `DocumentationConfig` types"; docs show only name/colors/logo/domains/navigation.
- LOST — Mintlify migration rule: do not preserve `docs.json` or Mintlify aliases as a second authority.
- LOST — authoring rules: optional root `documentation.tsx` (omit for default reader, add only to wrap/replace); Markdown-inert-default vs MDX-only-when-trusted-React-useful; no plugin registry; never edit generated `public/_documentation*`; media in app-owned source.
- CONFLICT — navigation: reference says explicit navigation must contain every public page exactly once unless hidden, and missing entries/broken links/redirect loops/duplicate routes are defects (don't bypass validation); docs say unlisted pages "still build and are reachable by URL".
- LOST — machine surfaces enumeration (per-page `.md`, `llms.txt`, sitemap, robots, OpenAPI pages, contextual actions) + "consume the generated manifest; don't infer enablement from output directories".
- LOST — assistant governance: never copy the shared agent runner/model catalog/provider policy into app code; keep access + billing explicit; never silently enable an owner-billed public assistant (docs cover only the 20-call cap / read-only tools).
- LOST — verification: the 5-point pre-deploy checklist (routes+/docs release, nav/search/not-found, machine-surface base URLs, assistant/MCP exposes no mutation tools, console/keyboard/mobile/a11y) and "don't use production as a routine docs test target".

Cross-cutting: three places the live docs contradict the deleted references rather than merely omitting them — JobRoom write-role enforcement, the Google `requiresOAuth` envelope, and the `DEEPSPACE_APP_ID`-vs-`APP_NAME` room-scoping convention (plus the smaller cron-role and ai-chat-installer discrepancies). Each needs a which-side-is-current determination against the SDK source.

---

## Reviewer 3 — lifecycle (cli, deploy, domain, collaborators, testing)

### cli.md → /cli-reference/overview, /cli-reference/commands — PARTIAL
- `app update` lost entirely (commands page, at 0.12.0, has no such command): run `npx deepspace@latest app update` before installing a newer SDK; it pins DeepSpace and — only when the app already declares `ai` — the compatible AI SDK version; never adds `ai`; run the returned install action instead of editing versions; read every `manualInstructions` entry (JSON) / warning (human) — app-owned Vite and users-schema files are never rewritten automatically and may get an exact `docs/migrations/` retrofit to apply. → /cli-reference/commands
- Exit-code contract lost: 0 = completed; 1 = failure/refusal where retrying unchanged won't help; 2 = safe partial progress/stop with `actionRequired: true`, which may be `ok:true` or `ok:false` after a safe partial mutation. Docs mention exit 2 only ad hoc. → /cli-reference/overview
- JSON `action` contract lost: `{"action":{"cwd":...,"argv":[...]}}` appears only when exactly one deterministic follow-up exists (rendered `Next:` in human output); execute `argv` directly in `cwd`, never joined into a shell string; omitted for terminal results, status reports, consent, destructive overrides, and input-dependent choices — absence does not mean nothing remains. → /cli-reference/overview
- Node support matrix lost and contradicted: consumers support `>=22.15.0 <23`, `>=24 <25`, `>=26 <27`; docs' installation page says "Node.js 20 or later" (20 is unsupported; 23/25 excluded). SDK-repo pin (Node 24; pnpm engine 11.16–11.x with packageManager 11.18, use the bundled pnpm) also unrecorded. → /get-started/installation (+ repo docs for the pnpm note)
- Streamed-JSON details lost: `activity --follow --json` is NDJSON (docs say this only for `logs`); `dev start` / `test run` / `test screenshot` inherit child output so their final `--json` envelope follows the stream; `logs --json` may emit a metadata frame when output is truncated. → /cli-reference/commands
- Minor: unknown command paths are matched by lexical distance over the full tree including subcommands and always return a concrete suggestion, never silent root help.

### deploy.md → /concepts/deployment — PARTIAL, with active contradictions
- Secrets model lost and contradicted: the remote secrets store is the source of truth — deploy never reads `.dev.vars`; it reconciles Worker secret bindings from the store; values change via `secrets set`/`upload`, never by editing the file. /concepts/deployment teaches the opposite ("applies your user secrets from `.dev.vars`", "re-uploaded each deploy", "to rotate a secret, edit the file and redeploy"), as does /concepts/architecture — while /cli-reference/commands' `deploy --allow-missing-secrets` already presumes the store model. Needs reconciliation on /concepts/deployment.
- `.dev.vars` generation semantics lost: `dev start`, `test run`, `deploy`, and `secrets pull` rewrite the file whole from SDK keys plus the selected remote secret config (docs claim below-divider keys are preserved verbatim — same conflict).
- `APP_IDENTITY_TOKEN` lifecycle lost: appears once the app is registered (deploy registers it; an earlier secrets write can register it pre-first-deploy); until then local payments, files, and screenshot APIs lack app-origin authentication. → /concepts/deployment
- Absent-vs-empty config rule lost: deploy refuses an absent secrets config (distinct from an explicitly empty one) and returns an executable `secrets configs create` action. → /concepts/deployment or /cli-reference/commands#deploy
- First-deploy id minting lost: a repo without an id gets `DEEPSPACE_APP_ID` minted into `wrangler.toml` by its first deploy — commit that identity (docs cover only explicit `app init`). → /concepts/deployment
- Named environments lost: each `[env.<name>]` is a separate app — distinct canonical name, app id, Durable Objects, and secrets config; init via `app init --env` or first deploy; remove with `app undeploy --env`. /concepts/deployment has no environments section. → /concepts/deployment
- Wrangler env non-inheritance trap lost: `[env.<name>]` does not inherit `vars`, DO bindings/migrations, assets, or KV/R2/D1 declarations — repeat the blocks or the deployed Worker boots without them. Nowhere in docs. → /concepts/deployment
- Browser env-id injection recipe lost: staging builds must inject that env's app id (`VITE_APP_ID` defined in vite.config from wrangler.toml via `process.env.CLOUDFLARE_ENV`), else the browser connects to production rooms while staging server actions write staging rooms; gate staging-only routes on an explicit staging signal. Nowhere in docs. → /concepts/deployment
- Minor: Cloudflare's build transiently copies `.dev.vars` beside the generated server worker; the scaffold deletes that copy before the build completes; the root mode-0600 file is the only local materialization.
- Covered elsewhere (not lost): catalog sync on deploy (/guides/payments), collaborator on-behalf deploy / no-undeploy (/guides/collaborators).

### domain.md → /guides/custom-domains — PARTIAL (near-covered, one contradiction)
- Attach semantics lost and contradicted: `attach` re-points the registration to the resolved immutable app id, so renames do not require reattachment. The docs page states the opposite twice. Needs reconciliation on /guides/custom-domains.
- `--app` accepts the immutable app id or current live name; scripts should prefer the id since names are URL leases. Docs examples use names only. → /guides/custom-domains
- `buy --json` returns the Checkout session without polling and supplies the `domain status` follow-up action. → /guides/custom-domains
- Minor: `search --limit N` flag; guidance not to automate `buy` in CI/tests nor use a real owned domain for routine tests.

### collaborators.md → /guides/collaborators — PARTIAL (docs are richer in most areas)
- Platform-admin scope lost: admins can deploy, manage secrets, and undeploy via platform override, but cannot manage collaborators, offer ownership transfer, or change source authority; the override never makes a collaborator an admin. Docs carry only "undeploy: owner (or platform admin) only". → /guides/collaborators
- Pre-first-deploy naming nuance lost: a DeepSpace-source app has no leased name before its first deploy, so collaborators must clone by app id (`deepspace clone <app-id>`); the name resolves only after deploy. → /guides/collaborators
- Minor: `transfer status` participation limited to owner/named recipient.
- Everything else covered; docs add detail the reference lacked (7-day expiry, `invites`/`accept` CLI path, failure-code table, rotate-secrets-after-removal).

### testing.md → /guides/testing, /sdk-reference/testing — PARTIAL (near-covered)
- Mocking carve-out lost: docs state "no mocking" flatly; the reference permits mocking exactly the paid or user-OAuth external integration boundaries when a real call would charge, mutate provider state, or require unavailable credits — app-internal hooks/services stay real. → /guides/testing
- Account-pool deletion discipline lost: delete only accounts created for the current run; never clear the shared pool indiscriminately. → /guides/testing
- `(app)` route-tier auth matrix lost: `src/pages/(app)/*` = signed-out dynamic content with no auth overlay, `src/pages/(app)/(protected)/*` = the gated tier, and `src/pages/*` must additionally preserve the static landing contract; docs document only a two-tier layout. → /guides/testing
- Minor: test-account credentials live in a mode-0600 local store.

Cross-cutting: three places where live docs actively contradict the newer reference semantics (deploy secrets store, domain attach-by-id, installation Node floor) — corrections to make, not just gaps.

---

## Reviewer 4 — no-home files (secrets, releases, app-identity, source-control, version-control, github, coordination, preview, workflow)

Global finding: the docs site is pinned at CLI 0.12.0 (stated on /cli-reference/commands.md) while the repo is at v0.19.5. Overlap is (1) genuine command-surface coverage and (2) active contradictions from the pre-store secrets model:
- /concepts/deployment.md "Secrets and .dev.vars": divider model, "preserved verbatim", "edit the file and redeploy", deploy "applies your user secrets from `.dev.vars`" — all contradicted by the store model.
- /cli-reference/commands.md `dev start`: "Keys you add by hand are preserved" — contradicts "hand edits disappear on the next write."
- /cli-reference/commands.md `deploy --allow-missing-secrets`: described in terms of "hand-edited `.dev.vars` secrets" — stale model.
- /concepts/deployment.md undeploy section: silent on identity/secret-store/cloud-repo retention and quota-checked revival.

No docs page exists for workspaces/version-control semantics, source-authority transfer, activity NDJSON, desktop preview/worktrees, or build workflow. Secrets/releases/app-identity have partial homes (command tables) plus the stale content above.

### secrets.md — absorption checklist
- One platform-owned encrypted store per app, keyed by immutable `DEEPSPACE_APP_ID`; no setup/link step; works from app dir or `--app`, for owner and collaborators, even pre-deploy.
- Store is source of truth for every environment; `.dev.vars` is a generated plaintext materialization written mode 0600; `dev`/`test`/`deploy`/`secrets pull` rewrite it whole.
- Deploy never reads `.dev.vars`; the store is the only deploy input; hand edits disappear on the next write.
- Worker code reads `env.KEY` identically in dev and deployed; deploy binds each store secret as a Cloudflare `secret_text`.
- Agent policy: `secrets set KEY=value` is the sanctioned way to configure an owner-provided credential — run it directly, don't bounce it back to the user.
- Hygiene: the value lands in shell history/logs/transcripts; if shared in plaintext, recommend rotate → re-`set` → redeploy; non-secret values need no caveat.
- Never `secrets get --plain` into a leaky destination (chat reply, committed file).
- First write registers the app id to you — pre-deploy `set` works with no bootstrap step.
- Propagation: deployed app picks up changes only at next `deploy`; running `dev` only on restart.
- Command details: `list` masked (`--only-names`, `--json`); `set` multi-pair, multiline/PEM fine; `get --plain` byte-exact when piped; `delete` idempotent; `download` stdout-only, formats dotenv/json/shell; `upload` dotenv or JSON, `-` for stdin, `--replace` deletes keys absent from the file.
- `configs create <new> --copy-from <existing>` copies server-side and refuses to copy over an existing config.
- Flags: `-a/--app` (default from nearest wrangler.toml), `-c/--config` (default `prd`), `-e/--env` targets the `[env.<name>]` app with its own store (config defaults to `<name>`); `-e staging` without an `[env.staging]` app id errors and points at `-c staging`.
- Names: `[A-Za-z_][A-Za-z0-9_]*`; the 11 `RESERVED_BINDING_NAMES` plus `API_WORKER_URL` and `PLATFORM_WORKER_URL` are rejected.
- Caps: 32 KB per value, 128 secrets / 128 KB per config, 64 configs; oversized writes → HTTP 413.
- `ALLOW_DEBUG_ROUTES=true` settable with a loud warning; production enables the debug surface only for authenticated owner/platform admin; local dev auto-enables.
- Config semantics: flat KEY=value configs; `prd` is the top-level convention; an `[env.<name>]` deploy ships config `<name>` of that env-app's own store.
- Cross-app seeding: `secrets download | secrets upload - -e staging` — pipe, no temp file.
- Missing config: deploy refuses with `secrets_config_missing` plus an executable `secrets configs create` action.
- An explicitly created empty config is intentional: deploying it removes all user-secret bindings (delete-all without a second override flag); the missing/empty distinction protects production bindings.
- Deletes propagate: `secrets delete` + redeploy removes the binding — deploy reconciles bindings against the store.
- Cache: `dev`/`test` re-pull at startup; on refresh failure they abort rather than run stale.
- `set`/`upload`/`delete` change the remote store only — running dev keeps old values until restart; `secrets pull` refreshes without running dev.
- The whole `.dev.vars` is SDK-owned: no editable zone, divider grammar, import path, backup, or legacy mode; one shared file across wrangler envs.
- A Cloudflare build may transiently materialize `.dev.vars` beside its generated worker; the scaffold deletes that copy.
- Troubleshooting: prod sees old value → redeploy; local sees old value → restart dev or `secrets pull`; 403 → owner runs `collaborators add`; "registered to another user" → `app init --new-id`; empty `list` on fresh config is legitimate; wrangler.toml "DeepSpace detected secrets" comment is a static placeholder; `undeploy` keeps the store and same-id redeploy revives it.

### releases.md — absorption checklist
- Every deploy records an append-only release fact.
- DeepSpace source is commit-first: deploy publishes the attached clean branch, records its commit, enforces workspace/ancestry/stale-base guards.
- GitHub source ships the local working tree with no Git operations; dirty/unpushed bytes are valid; `commitOid: null` with repository/source-revision metadata retained.
- `--no-push` is a one-release legacy escape for an unclaimed or DeepSpace-source app; never bypasses GitHub source authority.
- Commands: `releases [--limit N]` (default 20); `rollback [rel_…]` (defaults to previous); `rollback rel_… --allow-do-deletion`.
- Rollback re-ships a retained bundle without rebuilding or changing Git; every rollback appends another release fact.
- `releases` JSON marks `rollbackAvailable`; storage pressure can evict an old bundle while retaining the ledger row.
- `no_bundle` refusal = bundle evicted — choose another retained release.
- DO-class removal refusal; `--allow-do-deletion` can permanently delete stored data — explain the exact loss and get explicit user approval first.
- Refusal taxonomy routing: `dirty_worktree`/`behind_trunk`/`stale_base`/`no_bundle` belong here; build/secret mechanics belong with deploy docs.

### app-identity.md — absorption checklist
- Immutable id `app_` + 26 chars, minted at creation, in `wrangler.toml` under `[vars] DEEPSPACE_APP_ID`.
- The id is the durable public identity: data, secrets, collaborators, billing, custom domains all address through it.
- Identity-migrated apps: backend maps the app id to a permanent private `resourceId`.
- `name` is only a lease on `<name>.app.space`.
- Commit `wrangler.toml` — the id is not a secret.
- Id sources: scaffold mints; first deploy mints and writes wrangler.toml (commit it); `app init` stamps explicitly.
- `app init --new-id` forks a cloned repo: same code, separate data and secrets; does not change `name` or reserve a URL.
- Each `[env.<name>]` block is its own app with its own id.
- Rename: change `name` and deploy; CLI confirms (or `--rename`); data/secrets/collaborators/domains follow the id.
- Old name reserved for you for 30 days, then frees up.
- `app list`: id, URL, deploy state (`--json`).
- `app undeploy` positional resolves id or live subdomain from anywhere; positional overrides `--env`; falls back to nearest wrangler.toml.
- Undeploy removes Worker + routes, best-effort deletes recorded auto-provisioned resources; id, registry identity, collaborators, secret store, cloud repo remain — later deploy revives.
- Active apps count against the tier cap; revival is quota-checked.
- Transfer: two-step handshake, `transfer offer <email>` (7-day; `--replace` swaps), `status`, `cancel`; recipient `transfer accept --app app_…`.
- On acceptance the app moves as-is; only owner and billing change.
- Tell the recipient the app id out of band — no in-product notification.
- Roles: only owner offers; named recipient may inspect/decline/accept; either party cancels; collaborators cannot initiate transfer or undeploy; platform admins may undeploy but not transfer.

### source-control.md — absorption checklist
- Inspect authority with `app source --json` before deciding anything from remote names.
- Every app has exactly one authoritative Git repository.
- DeepSpace source: packaged default; deploy publishes automatically; `space` remote, clone/push/pull, workspaces, activity, releases, rollback work together.
- GitHub source: manual ownership; ordinary deploy ships the local working tree with no Git read/write/verification.
- Both remotes / retained objects on the inactive provider ≠ two sources of truth — server policy enables only the selected provider's writes.
- Releases/activity remain platform facts under either source.
- GitHub source: dirty/unpushed bytes valid; no DeepSpace commit id in the release; rollback uses the retained bundle; do not add Git checks to this path.
- Select/transfer: `app source github|deepspace [--remote <name>]`; use the command — never rewrite remotes or registry fields by hand.
- Transfer inventories branches/tags and deploy lineage, copies or asks the operator to publish missing objects, verifies exact object ids, flips authority once.
- Failed copy or stale source revision leaves the old provider authoritative; follow the structured action; never bypass with `--no-push`.
- DeepSpace→GitHub is manual (create/select remote, publish requested refs, rerun); GitHub→DeepSpace is packaged.
- Transfers preserve branch/tag-reachable history; do not move identity, data, secrets, collaborators, URLs.
- Not transferred: LFS objects, submodules, notes, replace refs, host metadata.
- Only the owner changes authority; collaborators and platform admins inspect only.
- Active DeepSpace workspaces must be landed or dropped before moving to GitHub.

### version-control.md — absorption checklist
- Platform remote named `space`; wrappers install/repair it and configure a credential helper (global for durable CLI, worktree-private for transient).
- Plain `git fetch/push` works from a configured checkout; new clones via `deepspace clone`.
- GitHub-source apps intentionally refuse these writes and workspaces.
- Commands: `clone <app-or-id> [dir]`, `push [-b]`, `pull [-b]` — guarded operations with stable refusal codes.
- A workspace is a durable server ref plus task metadata, materialized as a `ws/<id>` branch and worktree; one per line of work.
- Surface: `workspace new -t`, `attach ws_… [dir]`, `sync`, `status`, `list [--all]`, `land [--into <branch>] [--validate]`, `drop [ws_…]`.
- Commit WIP, then `workspace sync` for durability and collaborator visibility.
- Run `sync`/`land` from the workspace checkout; `-w` selects identity only.
- `land` merges preserving workspace commits, then deletes the server ref.
- Land's cleanup removes only DeepSpace-marked checkouts; `--keep-worktree` opts out.
- Overlap reports are advisory, not locks.
- Managed workspace defaults anchor under the primary checkout; per-checkout `node_modules`, never symlinked.
- Never plain-push `ws/*` to `space`; publish via `workspace sync`.
- Deploying from a workspace requires its exact HEAD synced first.
- `workspace drop` refuses to delete unpublished commits; CAS-protected deletion; follow its action or `--keep-worktree`; never force cleanup.
- `pull` fetches + fast-forwards; dirtiness/divergence/another-worktree stops it safely.
- `push --force` still refuses to discard a remote commit the local branch lacks; integrate instead.
- Push preflight rejects commonly committed secret files; fix the branch, don't normalize bypass flags.
- Size ceilings: 20 MiB/object, 32 MiB compressed history, `push_too_large`, untracking-doesn't-fix (docs carry these).
- Keep `DEEPSPACE_DEPLOY_URL` consistent per session — it selects the owning platform.
- Refusal contract: `non_fast_forward`, `behind_trunk`, `diverged`, `dirty_worktree`, `workspace_unsynced` — execute `action.cwd`+`action.argv` when present; no action = judgment required; never invent a command.

### github.md — absorption checklist
- GitHub-mode invariants: ordinary Git pushes; deploy ships the working tree without touching Git; DeepSpace Git writes/workspaces refuse; releases/activity remain readable platform facts.
- Do not create or replace `origin` unprompted; when asked, ordinary Git commands.
- Workspace-review recipe: `git push -u origin HEAD:refs/heads/<review-branch>` — publish exactly the checked-out commit.
- Never `git push --mirror` — acts on every ref including refs that don't belong on GitHub.
- Claim/transfer authority only via `app source github [--remote]`; follow its structured actions.
- For DeepSpace-source apps, GitHub may hold a review branch but is not the deploy source; never maintain two trunks.
- CI deploys: inspect `auth login --help` / `deploy --help` for current flags; credentials in the CI secret manager; never in argv, repo files, or logs.

### coordination.md — absorption checklist
- `status`/`status --json` report present facts (session, app, install, branch/workspace, sync relation, live release); no synthetic next step.
- Activity is stateless; the caller owns the cursor.
- Forms: `activity` (after cursor 0), `--since <cursor>` (strictly after), `--follow` (tail from now), combined.
- Persist the last returned cursor for continuity.
- `activity --follow --json` NDJSON: one `ready` frame, then `activity` frames; recoverable polling failures emit `transport` frames without advancing the cursor.
- Events are facts, not instructions.

### preview.md — absorption checklist
- One supported local runtime: `npx deepspace dev start`; deliberately no second `vite preview` script.
- Cloudflare build may emit `dist/<worker>/.dev.vars`; scaffold removes it post-build; deploy deletes it pre-collection; root file is the single local cache.
- App-owned Vite configs not rewritten: run `app update` and apply the returned build-preview cleanup before relying on a generic `vite build` archive.
- Linked checkouts detected via Git metadata, not directory names.
- Without `--port`/`$DEEPSPACE_PORT`: each linked worktree gets a deterministic port in 5180–6179 from its canonical path; primary keeps 5173; `dev start`/`test run`/`dev kill` resolve the same port.
- Claude desktop preview reads `.claude/launch.json`; normal entry shape documented; machine-local, gitignored, no absolute worktree paths committed.
- Adapter reads the owning checkout's launch file; `<owner>/.claude/worktrees/<name>` counts only when Git reports both as registered checkouts.
- From a worktree, one `dev start` adds a `wt-<name>` entry with exact cwd + resolved port, prints the name, prunes only stale `wt-*` entries under that owner.
- Stale-preview recipe: check server cwd/port; add a distinctive source string and request it (absence proves wrong checkout); stop the mismatched server, start the printed entry; never kill an unrelated process on the port.

### workflow.md — absorption checklist (end-to-end build methodology)
- Scope: whole-app builds; skip for single features/bugfixes.
- Ordering law: research and de-risking before building; verification before "done"; phases loop but never skip forward.
- Research: never build "like X" from memory — reverse-engineer the reference end to end; drive it headlessly and watch network traffic; get the secret sauce or state the black box; ask for access rather than creating accounts on someone else's product.
- Reference corpus in a stable folder; diff every later phase against it; findings in a small docs wiki with sources; inferences marked as hypotheses.
- Check platform catalogs first; classify capabilities SDK / catalog / wire-yourself; wire-yourself = prime de-risking targets.
- From scratch: study 2–3 real products; ungrounded invented features get cut.
- Spec to the zero-questions test; a feature left as a noun is a hole (input, source of truth, edit path, empty/loading/failure states).
- Spec the whole researched product; staging to MVP is the user's call; realism gate (who clicks it and why); say what you cut.
- Ambiguous product decisions surfaced in one batch with recommendations; if user away, record + adopt recommendations + keep moving; decisions file; no relitigating without new evidence.
- One design source of truth, chosen up front (design-tool path / reference-screenshots path / self-design path with their respective rules; parity covers structure, never identity).
- A vetted design is copied, not "improved"; motion/live behavior is designed deliberately.
- De-risk load-bearing bets with timeboxed spikes using real calls (`integrations invoke`, budget-bounded); failed bet with no alternative = stop and ask.
- Plan top-down until nothing load-bearing is vague; fresh-agent depth test; plan never heavier than the code.
- Build order: shared foundation first through one writer; parallelize only under exclusive file ownership, inlined conventions, pinned absolute cwd, coordinator-routed cross-stream changes, spiked shared recipes; reading work parallelizes freely.
- One isolated checkout per line of work; DeepSpace source → workspace, GitHub source → branch + worktree.
- Verification: green gates are a false green; the six gates (type-check+tests; visual inspection of changed surfaces; side-by-side diff vs design source; live smoke of the core loop as a fresh account; multi-user via multiple real sessions; failure-state exercises).
- Report with evidence; hard line between built-and-verified / built-but-unverified / not built; budget verify-fix cycles; close on real pass/fail.
- Checks measure intent, not proxies; taste calls get side-by-sides for the user.
- Review periodically; independent fresh-context review mandatory for money/auth/fan-out/hard-to-reverse; adjudicate findings yourself; loop until a pass finds nothing.
- Money paths: hardest review, invariants pinned by tests (server-resolved amounts, per-request entitlements, fail closed); never hand-roll Stripe.
- Completeness walk against spec decides "done"; then whole-system design review + re-run gates.
- Every cut/deferral communicated with a reason; a missing named resource means ask, never swap.
- Land before deploy (DeepSpace-source trunk work); GitHub-source may ship dirty bytes; every deploy appends a release; rollback only while the bundle is retained.
- Deploy policy pre-launch: autonomous on green gates + live smoke; rehearse risky changes on `--env staging`; hand the user the live URL + what-to-test-first.
- Long builds: state on disk — task list, state/decisions file, lessons file; on resume `status` + activity from cursor + re-read state files.
- Commits are the undo: commit before risky passes; publish so work survives the machine; planning docs out of the repo.
- Decide vs ask: solo default decide-and-log; ask only for irreversible/outward-facing.
- Know when to stop: repeated failed fixes → return with observations, not another lap.
- Reporting style: result first, point-wise, dead honest about verified vs not.
- The failure table (13 failure → rule pairs, from done-on-green-gates through two-streams-same-files).

---

## Reviewer 5 — design (uiux, landing-design + tree)

(a) NO-DOCS-COVERAGE CLAIM: CONFIRMED. llms.txt has no design section; 9 plausible URLs 404. The deleted guidance is recorded nowhere public.

(b) Absorption checklist — [DS] = DeepSpace-specific; unmarked = generic craft.

uiux.md (213 lines): scaffold-is-placeholder principle [DS]; static index vs dynamic `(app)/home` + move-home recipe [DS]; 5-step home-page decision procedure with 5 named home skeletons + declaration comment [taxonomy generic, wiring DS]; named AI tells for home pages; theme-creation path (`[data-theme]`, THEMES registry, `--radius`, nav rebuild preserving test ids) [DS]; Tailwind v4 `@theme` shadow caveat (generic); DeepSpaceThemeProvider scoping [DS]; `data-ui-theme` + color-scheme mechanics [DS]; no-emoji-in-chrome rule + 3 allowed contexts; primitives-vs-browser-defaults mapping [principle generic, kit DS]; toast + confirmed-write feedback discipline [DS]; Base UI gotchas (render-not-asChild, SelectValue, nested-dialog forceRender, Tabs data-active, animate-in/out); prop-shape reference for Button/ConfirmModal/EmptyState/AuthOverlay/useToast [DS]; interaction-polish checklist; smoke-spec assertions + two-half grep gate [DS].

landing-design.md (137 lines): the 5-step workflow (Direction → Style Tile → one archetype → composition → grep gate); placement decision static vs `add landing` [DS]; `_layout.tsx` nav-hiding patch [DS]; Design Direction block convention (6-prompt brief + 6-token Style Tile as source comment); sentence test; read exactly ONE archetype; composition budget (1 nav / 1 hero / 1–2 features / 0–1 social proof / 1 CTA / 1 footer / 0–N motion); image workflow (negative-text clause, useR2Files persistence [DS], React mockups never AI images); the 14 hard rules; MotionConfig reducedMotion + manual gates; BMP-marks-only emoji rule.

design-direction.md (79): six prompts with bad/good pairs; mechanically-checkable-behaviors rule; worked sentence-test example; cohesion rule (match app body font + palette; finalize app theme first [DS-leaning]); unstuck procedure.

style-tile.md (160): 60-30-10, hue psychology, saturation-as-meaning, WCAG floors, one-sentence pick rule; palette-equals-app-tokens constraint [DS]; two-fonts-max + pairing tables + hard don'ts; light-vs-dark decision + match app default [DS-edged]; 9-archetype art-direction table; 6-personality motion table; voice behavior menu + six reference voices; derived choices (radius scale, icon source, 8px grid, density, imagery, surface treatment).

anti-ai-checklist.md (68): 14-row violation→fix table; known scaffold violations note [DS]; canonical grep gate (hex/rgb/named-color, fractional-opacity, continuous-animation, pictograph-emoji PCRE, placeholder/cliché copy, TODO, illegal-example-import) [technique generic, scoping DS]; eyeball checklist + root-cause rule (grep clean but page fails → the Direction is the bug).

inspiration-gallery.md (45): pick by emotional adjacency never category; five-archetype mapping table; reading order inside an example; five distilled design lessons; why-only-five rationale; examples-are-read-only doctrine.

pattern-library.md (33) + hero (362) + nav (485) + features (231) + cta (106) + footer (114) + social-proof (127) + scroll-motion (139): per-section budgets; scaffold integration notes (clean primitives list; GlassCard/PlaceholderImage/BrowserMockup/SectionHeading carry gate violations; CTA targets [DS]); 5 hero patterns (split-screen mockup, atmospheric image + scrim, bento, typographic poster, live terminal incl. reduced-motion setTimeout lesson); 6 nav patterns (dual-state floating pill recommended default, sticky bar, corner brand, hamburger overlay, editorial anchor index, mega menu) + direction→choice counsel; 5 feature patterns breaking the 3-identical-cards tell; 3 closers; 3 footers; 4 social-proof patterns + real-proof-only cardinal rule + ungated useCountUp warning [DS]; 4 scroll-motion patterns, all manually reduced-motion gated.

examples/01–05.tsx (~1,475 lines): five complete worked landing pages, each a filled Direction block translated line-by-line into code; per-example distilled lessons (texture as handmade signal; restraint as identity; 4.5s slowness serving calm; coordinated wobble as intentional play; animate-once bento hierarchy); the only worked sentence-test demonstrations.

Placement: ~70% generic craft → standalone docs design section; ~30% [DS] units → scaffold-aware guides (theming, project structure, landing feature).

---

## Reviewer 6 — integration YAMLs vs live CLI catalog

Verdict: partial — deletion rationale holds (CLI is authority, fresher and billing-richer; YAMLs demonstrably stale), but the CLI does not carry: per-endpoint output schemas, per-endpoint one-line descriptions, any OAuth marker. OAuth gap fully closed by docs; output schemas and descriptions genuinely lost.

Sample (google/gmail-send): YAML input schema (to/subject/content/html/threadId) vs CLI superset (+cc/bcc/inReplyTo/references); YAML output schema (requiresOAuth/authUrl/provider/scopes/messageId/threadId/recipient/subject/sentAt) vs CLI absent; YAML no billing vs CLI `$0.01 per request`; YAML `oauth_required: true` vs CLI absent; YAML per-endpoint description vs CLI absent. Same pattern across all six endpoints tested.

Staleness evidence: anthropic default model outdated in YAML; live google has endpoints the YAML lacks (gmail-modify/archive/star/mark-read/unread); live calendar adds `sendUpdates`, email/send adds `headers`; live catalog covers integrations absent from YAMLs (alphavantage, coinbase, per-sport splits); the YAML README itself declared the CLI authoritative.

Docs coverage: /guides/external-apis (catalog workflow, envelope, billing modes, OAuth section) and /sdk-reference/client/integrations (OAuth endpoints + requiresOAuth shape — which supersedes and contradicts the YAML's envelope). Neither page has per-endpoint output schemas or descriptions.

Discrepancy: external-apis.md claims `info` prints "the input schema (Zod), output schema, and an example body" and `list` returns "one-line descriptions" — both empirically false today. Either the docs overstate the CLI or the server catalog doesn't ship those fields; the deleted YAMLs were the only place output schemas and descriptions existed.
