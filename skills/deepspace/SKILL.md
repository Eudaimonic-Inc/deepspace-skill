---
name: deepspace
description: >
  Use when building real-time collaborative apps with the DeepSpace SDK on
  Cloudflare Workers, scaffolding a new DeepSpace app, or working in any
  project that imports from `deepspace` or `deepspace/worker`. Also use when
  the user mentions DeepSpace, app.space, RecordRoom, or asks to create a
  collaborative web app with real-time sync, auth, and deployment to
  Cloudflare Workers — even if they don't name DeepSpace explicitly.
---

# DeepSpace SDK

Build real-time collaborative apps on Cloudflare Workers. One npm package: SQLite-backed Durable Objects, RBAC, WebSocket subscriptions, Better Auth.

## Build a New App

### Step 1: Scaffold

```bash
# Published SDK (when available)
npx deepspace create <app-name>

# Local SDK (for development — replace path with your local SDK root)
<local-sdk-path>/packages/create-deepspace/dist/index.js <app-name> --local <local-sdk-path>
```

This generates: generouted file-based routing, `_app.tsx` providers, `nav.ts`, `worker.ts`, Cloudflare Vite plugin, and a working dev setup.

### Step 2: Define Schemas

Create `src/schemas.ts` with typed SQL columns. Every collection needs `name`, `columns`, `ownerField`, and `permissions`.

```typescript
import { USERS_COLUMNS, CHANNELS_SCHEMA, MESSAGES_SCHEMA } from 'deepspace/worker'

const itemsSchema = {
  name: 'items',
  columns: [
    { name: 'title', storage: 'text', interpretation: 'plain' },
    { name: 'status', storage: 'text', interpretation: { kind: 'select', options: ['draft', 'published'] } },
    { name: 'createdBy', storage: 'text', interpretation: 'plain' },
  ],
  ownerField: 'createdBy',
  permissions: {
    anonymous: { read: 'published', create: false, update: false, delete: false },
    viewer: { read: true, create: false, update: false, delete: false },
    member: { read: true, create: true, update: 'own', delete: 'own' },
    admin: { read: true, create: true, update: true, delete: true },
  },
  visibilityField: { field: 'status', value: 'published' },
}

export const schemas = [usersSchema, itemsSchema, CHANNELS_SCHEMA, MESSAGES_SCHEMA]
```

### Step 3: Wire Auth and Providers

In `src/pages/_app.tsx`, set up the provider stack:

```tsx
<DeepSpaceAuthProvider>
  <RecordProvider allowAnonymous>
    <RecordScope
      roomId={SCOPE_ID}
      schemas={schemas}
      appId={APP_NAME}
      sharedScopes={[{ roomId: 'workspace:default', schemas: workspaceSchemas }]}
    >
      <App />
    </RecordScope>
  </RecordProvider>
</DeepSpaceAuthProvider>
```

### Step 4: Auth Wiring Checklist

Every app MUST implement these auth flows — do not skip any:

- [ ] **Sign-in for unauthenticated users**: Render `<AuthOverlay />` when `!isSignedIn`. Gate the app behind auth or show a landing with a Sign In button. Use `useAuth().isSignedIn` (session-based, updates immediately) as the primary check — not `useUser()` alone.
- [ ] **Sign-out**: Add a sign-out option accessible from the user avatar/menu using `signOut()` from the SDK. Users must always be able to log out.
- [ ] **Auth state reactivity**: After `AuthOverlay` completes sign-in, `isSignedIn` flips `true` → show loading spinner → user profile loads → app renders. Do not require a page refresh.
- [ ] **AuthOverlay pattern**: Render without `onClose` and gate with `!isSignedIn` — the overlay auto-hides when signed in (returns `null`). This prevents users from dismissing it and getting stuck.

```tsx
// Recommended pattern
const { isSignedIn } = useAuth()
if (!isSignedIn) return <AuthOverlay />
```

### Step 5: Pick a Theme

Before building pages on an **initial build**, rewrite the `@theme` block in `src/styles.css` and update `<title>` / favicon in `index.html` so subsequent UI reflects the real brand instead of default dark-blue. If the user didn't specify a palette, pick one that fits the app's domain and tell them in one line. On initial builds, load `references/uiux.md` §2 for the palette picker and token list. On maintenance work against an already-themed app, skip this step.

### Step 6: Build Pages and Features

Pages go in `src/pages/` — generouted scans this directory for file-based routing.

```
src/pages/home.tsx    → /home
src/pages/items.tsx   → /items
src/pages/_app.tsx    → layout wrapper (providers + nav)
```

Features are reference implementations in `.deepspace/features/` (scaffolded into every app). To add one: read `.deepspace/features/<name>/FEATURE.md`, copy files to specified destinations, wire imports/routes/schemas.

Replace the scaffold home page, wire mutations to `useToast`, and use scaffolded UI primitives from `src/components/ui/` — never browser defaults. Load `references/uiux.md` on initial builds, when adding UI you haven't built in this session yet (confirmations, empty states, skeletons), or when the user says the app "feels generic". Skip it for small tweaks against UI that already exists and already uses the primitives.

Available features (check `.deepspace/features/` in the scaffolded app for the canonical list — names may evolve): `admin-page`, `ai-chat`, `canvas`, `cron`, `docs`, `file-manager`, `integration-test`, `items`, `kanban`, `landing`, `leaderboard`, `messaging`, `presence-test`, `sidebar`, `tasks`, `testing`, `topbar`, `tree`.

### Step 7: Run Locally

```bash
npx deepspace dev     # starts all workers + Vite with HMR on localhost:5173
```

### Step 8: Test-Driven Verification (run when code changes)

Tests are the primary way to verify and debug code changes. The scaffolded tests (`smoke.spec.ts`, `api.spec.ts`, `collab.spec.ts`) are **starting points** — extend them for every code change that affects runtime behavior.

**When to run tests**: only after a code change (added/edited files in `src/`, `worker.ts`, or similar). Skip tests for conversation, planning, reading files, or answering questions — don't run them as a ritual.

**Workflow for any code change that touches runtime behavior:**

1. **Customize or extend the relevant test file** to cover what you just built or modified:
   - **smoke.spec.ts** — update when adding a new page, route, nav item, or top-level UI (landing, gallery, dashboard, settings). Assert the page loads, expected content is visible, no console/page errors.
   - **api.spec.ts** — update when adding worker routes, integration calls, or endpoints that require auth. Assert status codes, response shape, auth gating, error cases.
   - **collab.spec.ts** — update when adding multi-user flows (shared records, messaging, permissions, presence, invites, real-time sync). Use `createTestUsers(browser, N)` and assert one user's action is visible/effective for another.
2. **Run the relevant tests** (`npx playwright test <file>`) with `npx deepspace dev` already running.
3. **Debug from failures, not from console logs.** If a test fails, read the assertion message, read the failing selector, then fix the code. Do not add `console.log` to diagnose — write a more specific assertion. Do not weaken or delete tests to make them green.
4. **Re-run after each follow-up change.** When the user asks for a tweak or new feature later, update the tests alongside the code change, then run them. Treat tests as a living contract — but only exercise them when the contract actually changes.

**When to reach for which test:**
- Single-user UI / CRUD / navigation → extend `smoke.spec.ts`.
- Worker route, integration call, RBAC on HTTP → extend `api.spec.ts`.
- Anything where user A's action should affect user B's view → extend `collab.spec.ts`.
- A bug you're trying to fix → write a failing test that reproduces it first, then fix the code.

Skipping tests after a code change is the #1 source of "I built it but it crashes on page load" handoffs.

### Step 9: Deploy

On an **initial build**, load `references/uiux.md` §5 and run the pre-deploy checklist (home replaced, theme updated, no browser-default primitives, mutations fire toasts). On **follow-up deploys** where those were already verified, skip straight to the commands below.

```bash
npx deepspace login  # opens browser — the ONE human step in the whole flow
npx deepspace deploy  # deploys to <app-name>.app.space
```

## Two Imports

```typescript
// Frontend (React)
import { RecordProvider, RecordScope, useQuery, useMutations, useAuth } from 'deepspace'

// Worker (Cloudflare Worker)
import { RecordRoom, verifyJwt, CHANNELS_SCHEMA } from 'deepspace/worker'
```

## Frontend Hooks

### Core
```typescript
const { records } = useQuery<Item>('items', { where: { status: 'published' }, orderBy: 'createdAt' })
const { create, put, remove } = useMutations<Item>('items')
const { user } = useUser()          // storage-level: id, name, email, role
const { isSignedIn } = useAuth()    // auth state (use this as primary check)
const { users } = useUsers()        // all room users
```

### Messaging (channel-based)
```typescript
const { channels, create } = useChannels()
const { messages, send, edit, remove } = useMessages(channelId)
const { getReactionsForMessage, toggle } = useReactions(channelId)
const { isMember, join, leave } = useChannelMembers(channelId)
const { markAsRead, getUnreadCount } = useReadReceipts()
```

### Directory (cross-app)
```typescript
const { conversations, createChannel, createDM } = useConversations()
const { communities, createCommunity, joinCommunity } = useCommunities()
const { posts, createPost } = usePosts()
```

### Other hooks and exports

The hooks shown above (Core, Messaging, Directory) are the data-and-identity primitives most apps use. For other SDK surfaces, read **`references/sdk-reference.md`** — it indexes every export grouped by domain, with usage snippets for the common non-obvious ones. Load it when building a feature that involves:

- **File uploads or image/video handling** → `useR2Files` (includes a local-dev caveat about `APP_IDENTITY_TOKEN` uploads)
- **Collaborative text editing** (docs, comments, notes) → `useYjsText` / `useYjsField`
- **Live cursors, typing indicators, "who's online"** → `usePresence`
- **Canvas / whiteboard features** → `useCanvas`
- **Video/audio rooms** → `useMediaRoom`
- **Theme customization** → `DeepSpaceThemeProvider`, `applyUIThemeTokens`
- **Environment-specific logic** → `isLocalDev()`, `getApiUrl()`
- **Any export not covered in this file** — `sdk-reference.md` is the canonical index.

For exact type signatures of any export, read `node_modules/deepspace/dist/index.d.ts` (frontend) or `node_modules/deepspace/dist/worker.d.ts` (worker). Do not guess hook names or argument shapes.

## Architecture

Each app has its own **RecordRoom** Durable Object with schemas baked in at deploy time.

```
App Worker (per-app)         Platform Worker (shared)
├── AppRecordRoom DO         ├── GlobalRecordRoom DO
├── /ws/:roomId              ├── /ws/:scopeId (conv, dir, workspace)
├── /api/auth/* → auth-worker├── /api/app-registry
└── Static assets (SPA)      └── /api/health
```

**Critical**: Global scopes (`workspace:*`, `dir:*`, `conv:*`) must proxy WebSocket connections through `PLATFORM_WORKER` service binding to the platform worker's shared DOs. If the worker routes ALL `/ws/:roomId` to local `RECORD_ROOMS`, each app gets its own isolated instance and cross-app data sharing breaks.

## Worker

```typescript
import { RecordRoom, verifyJwt, createScopedR2Handler } from 'deepspace/worker'
import { schemas } from './src/schemas.js'

export class AppRecordRoom extends RecordRoom {
  constructor(state: DurableObjectState, env: Env) {
    super(state, env, schemas, { ownerUserId: env.OWNER_USER_ID })
  }
}
```

## RBAC

Permissions are per-role, per-collection:
- `true` / `false` — allow/deny all
- `'own'` — only records where ownerField matches userId
- `'published'` — owner OR passes visibilityField check
- `'shared'` — owner OR collaborator OR published (uses `collaboratorsField` + `visibilityField`)
- `'team'` — owner OR collaborator OR team member

New authenticated users get `member` role. Anonymous connections get `anonymous` role.

### Data Visibility

When creating records scoped to specific users (e.g., conversations, private data):
- Set `Visibility: 'private'` — not `'public'`
- Populate `ParticipantIds` (or the relevant `collaboratorsField`) with all participant user IDs
- The SDK filters server-side in the DO — `canRead()` checks ownerField, collaboratorsField, and visibilityField before sending data over WebSocket
- Never rely on client-side filtering alone — data still syncs over WebSocket and is visible in dev tools

## Integrations

Call external APIs (OpenAI, weather, email, GitHub, Slack, Google, etc.) through the api-worker proxy:

```typescript
import { integration } from 'deepspace'
const result = await integration.post('openweathermap/geocoding', { q: city })
// Returns: { success: true, data: {...} } or { success: false, error: "..." }
```

**Do not guess endpoint names.** The format is `<integration-name>/<endpoint-name>` (two segments). Names like `geocode-city` or `weather-forecast` are not real — a wrong name will return a 404 at runtime.

**For the full list of available integrations and endpoints, read `references/integrations.md`.** It covers LLMs (OpenAI, Anthropic, Gemini), search (Exa, Firecrawl, SerpAPI), media (Freepik, ElevenLabs, CloudConvert), communication (Email, Slack, LiveKit), Google Workspace (Gmail, Drive, Calendar), social (GitHub, LinkedIn, YouTube, TikTok, Instagram), finance (Polymarket, stocks, crypto), sports, NASA, MTA, Wikipedia, and more. Always verify the endpoint exists in that reference before calling it.

## UI/UX Polish

The scaffold's home page, theme, and UI primitive choices are placeholders — shipping them as-is produces a generic-looking app. Load `references/uiux.md` **on initial builds** (home + theme + first pages) and **whenever reaching for a UI pattern not yet built in this session** (confirmations, empty states, skeletons, etc.) or when the user says the app "feels generic". Skip it for maintenance work against UI that already follows the primitives conventions.

## Testing

Every scaffolded app includes Playwright tests in `tests/` with helpers for auth, error tracking, and multi-user flows. Use these tests to verify your work — don't rely on manual testing or console logs to debug issues.

### Running Tests

```bash
npx playwright test              # run all tests
npx playwright test smoke.spec   # run a specific test file
```

Dev server must be running (`npx deepspace dev`) before running tests.

### Scaffolded Test Files

- `smoke.spec.ts` — app loads, navigation renders, sign-in button present, page title correct
- `api.spec.ts` — API endpoints return expected responses, auth required where expected
- `collab.spec.ts` — multi-user: two users connect, see each other, data syncs between them
- `tests/feature-tests/tests/<feature-id>.spec.ts` — per-feature merge-gate specs (e.g., `docs.spec.ts` for the docs feature). Auto-discovered and run by the e2e harness when the feature is installed. When you add a custom feature, drop a `<feature-id>.spec.ts` alongside the others with multi-user assertions.

Playwright runs with `retries: 1` locally (2 in CI) to absorb subprocess flakes; real bugs fail all retries. `trace: 'retain-on-failure'` preserves the full trace in `test-results/` for forensics.

### Test Helpers (`tests/helpers/`)

- `auth.ts` — `createTestUsers(browser, count)` creates N users with isolated browser contexts. Each user gets a unique email and signs up via `/api/auth/sign-up/email`.
- `global-setup.ts` — warms up the auth worker with retries before tests run.
- `errors.ts` — captures console errors and page errors during tests.

### Writing New Tests

**Single-user flows** (CRUD, navigation, UI state): use `signUp(page, email)` from the auth helper to create one authenticated session, then interact with the page.

**Multi-user flows** (real-time sync, sharing, permissions): use `createTestUsers(browser, 2)` to get isolated sessions, then act in one and assert in the other.

```typescript
// Multi-user pattern
const [userA, userB] = await createTestUsers(browser, 2)

// userA creates something
await userA.page.getByTestId('create-btn').click()
await userA.page.getByTestId('title-input').fill('My Item')
await userA.page.getByTestId('save-btn').click()

// userB should see it
await expect(userB.page.getByText('My Item')).toBeVisible()
```

### Proactive Test Authoring

Write and update tests **as you build**, not after. Every new page, feature, or user-visible change should trigger a corresponding test update in the same session — before saying "done":

- **New page / route / nav item** → extend `smoke.spec.ts`. Add a test that navigates to the page, asserts the expected headline/components are visible, and the page has no errors.
- **New CRUD feature** (items, posts, whatever) → extend `smoke.spec.ts` with a create/read/edit/delete happy path for a signed-in user.
- **New worker route or integration call** → extend `api.spec.ts`. Assert success responses, auth-required failures, and error shapes.
- **New multi-user behavior** (sharing, invites, messages, presence, permissions, shared scopes) → extend `collab.spec.ts`. Create two users, act in one, assert in the other.
- **RBAC changes or permission tweaks** → add tests in `collab.spec.ts` with users of different roles, asserting what each can and cannot see/do.
- **Bug fix** → write the failing test first (reproducing the bug), then fix the code until it passes. Leave the test in the suite.

When the user asks for a change in a follow-up message, update the tests in the same turn — don't let them drift. The test suite is a living contract.

### Self-Diagnosis with Tests

When something isn't working, do **not** start with console logs. Start with:
1. Write (or tighten) a test that expresses the expected behavior.
2. Run it. Read the failure message and the failing selector/assertion.
3. Fix the code until the test passes.
4. Leave the test in place — it now guards against regression.

Console logs are a last resort, not a first step. A failing test tells you more than a log ever will: what was expected, what was observed, where in the flow it diverged. If a test is flaky or passes locally but fails in CI, investigate the flake — do not mark it `.skip` or delete it.

## Gotchas

These are concrete issues discovered in real dev sessions. Read before building.

- **Page files MUST go in `src/pages/`** — generouted only scans this directory. Putting pages in `src/features/<name>/` or elsewhere results in 404s even if nav links exist.
- **`useAuth().isSignedIn` for auth gating, not `useUser()`** — `isSignedIn` is session-based and updates immediately after sign-in. `useUser()` loads async and causes a flash of "not signed in" state.
- **Safari + localhost cookies** — `__Secure-` cookies require HTTPS. Safari enforces this; Chrome doesn't on localhost. Auth will appear broken on Safari in local dev. Works fine once deployed.
- **Service bindings unavailable in local dev** — `c.env.API_WORKER.fetch()` doesn't exist locally. Use `API_WORKER_URL` env var as fallback: `env.API_WORKER?.fetch(req) ?? fetch(env.API_WORKER_URL + path)`.
- **Integration response format** — api-worker returns `{ success: true, data: [...] }` where `data` may be a flat array. Don't look for `result.data.forecast` or `result.data.list` — check `Array.isArray(result.data)`.
- **Cross-app workspace isolation** — Each app worker has its own DO namespace. `workspace:default` in app A is a different DO instance than `workspace:default` in app B. For shared data, route `workspace:*` WebSocket connections through `PLATFORM_WORKER`.
- **`createChannel()` defaults to `Visibility: 'public'`** — This means all users see all conversations. Override with `Visibility: 'private'` and set `ParticipantIds` for user-scoped data.
- **Schemas are columns only** — no `fields` property, no document-mode storage.
- **JWT provides user profile** — no separate `/api/users/me` call needed.
- **All tests use real services** — never mock internal hooks.
- **Scaffold has local UI primitives that shadow SDK names** — `src/components/ui/Toast.tsx` exports its own `ToastProvider` + `useToast`, and the scaffolded `_app.tsx` wraps the app in the **local** `ToastProvider`. If you import `useToast` from `deepspace`, you'll hit `useToast must be used within ToastProvider` at runtime because the contexts don't match. **Import UI primitives from `../components/ui` (or the equivalent local path), not from `deepspace`**, unless you've verified the scaffold uses the SDK version. The same shadowing can apply to other UI components — always check the scaffolded `_app.tsx` to see which provider is in the tree before picking an import source.

## Key Rules

- Schemas baked in at deploy time — no runtime schema loading
- Direct WebSocket per scope — no mux/gateway
- No user-scope DOs — user-scoped data lives in app DOs with RBAC filtering
- `pnpm dev` or `npx deepspace dev` for local dev — never run `wrangler dev` + `vite dev` separately
