_Load when runtime behavior changes, a Playwright test is needed, or the shared test-account pool blocks a run._

# Testing

Run the narrowest relevant suite after runtime changes; skip test rituals for reading or documentation-only edits.

```bash
npx deepspace test run                    # smoke + api
npx deepspace test run smoke|api|e2e|unit|all
npx deepspace test run tests/foo.spec.ts
npx deepspace test run --port 5180        # match dev start --port 5180
npx deepspace test run -c dev             # pull the 'dev' secrets config for this run
npx deepspace test run --base-url https://myapp-staging.app.space
                                          # run the browser suites against a deployed URL
```

`test run` prepares local credentials and Playwright; the scaffold's Playwright config starts Vite when needed. Use plain `npx playwright test <spec>` only for fast iteration after setup. `test screenshot <url> <output>` is an optional visual-debug helper, never test coverage or a required sweep.

`--base-url` targets a deployed app instead of a local server: no local vite is
spawned and every test runs against that URL (mutually exclusive with `--port`;
not applicable to `unit`). Apps scaffolded before this flag hardcode
`localhost` in `tests/playwright.config.ts` — the CLI **refuses**
(`base_url_not_supported`) rather than silently testing localhost while
claiming the deployed URL; the fix is in the message: set
`baseURL: process.env.DEEPSPACE_BASE_URL ?? <local URL>` (in
`tests/helpers/global-setup.ts` too) and omit `webServer` when it is set.
`-c/--config` selects which secrets config the run pulls (default: the `--env`
name, or `prd`) — e.g. test API keys while production keeps live keys.

## Coverage by change

| Change | Required proof |
|---|---|
| Schema or CRUD | Signed-in create, read, edit, delete. |
| Route, page, nav, or top-level UI | Navigate and assert real content, not merely absence of errors. |
| Visibility/RBAC or realtime hook | Two isolated users: one acts, the other observes or is denied. |
| Worker route, server action, AI route, cron, integration call | API status, consumed response shape, and 401/403 negative path where applicable. |
| Bug fix | A regression test that fails before the fix. |

Keep app-internal hooks and services real. Mock only paid or user-OAuth external integration boundaries when a real call would charge, mutate provider state, or require unavailable credits. Never weaken an assertion or add retries merely to turn a failure green.

The scaffold starts with `smoke.spec.ts`, `api.spec.ts`, and `collab.spec.ts`; extend the closest contract unless a separately installed feature intentionally owns its own spec.

## Signed-in and multi-user tests

Use the published fixture; it reuses cached storage state and closes browser contexts automatically.

```ts
import { test, expect } from 'deepspace/testing'

test('A creates and B sees', async ({ users }) => {
  const [a, b] = await users(2)
  await a.page.goto('/items')
  await b.page.goto('/items')
  await a.page.getByTestId('create-item').click()
  await expect(b.page.getByText('Shared item')).toBeVisible()
})
```

`users(N)` picks existing accounts; use named accounts only when identity is part of the behavior. The pool is global per developer and capped at 10:

1. Run `npx deepspace test accounts list`.
2. Reuse existing accounts with `users(N)` when enough exist. If the scaffold's named `Collab A/B` lookup fails but the pool is large enough, change it to `users(2)`.
3. Create only the shortfall with `test accounts create`; credentials must use `@deepspace.test` and remain in the mode-0600 local store.
4. Delete only accounts created for the current run; never clear a shared pool indiscriminately.

Useful escape hatches from `deepspace/testing` are `loadAllTestAccounts`, `pickTestAccounts`, `findTestAccountByName`, `ensureStorageState`, and `newSignedInContext`.

## Data and route safety

Prefix test-created records with `__test-${Date.now()}__`, track their `recordId`s, and delete exactly those records in `finally`/cleanup hooks. Never wipe a collection or database shared with development.

For every reachable route, assert its actual data. Dynamic routes must create a fixture, navigate by its id, and prove that fixture rendered. Cover auth state by route location:

- `src/pages/*`: signed-out content, no auth overlay, and preserve the static landing contract when applicable.
- `src/pages/(app)/*`: signed-out dynamic content with no auth overlay.
- `src/pages/(app)/(protected)/*`: signed-out overlay with protected content absent; signed-in content with no overlay.
- Sign-out from a protected route: assert navigation to `redirectOnSignOut`, not a stranded overlay.

Use stable `data-testid` selectors for contract boundaries. For Yjs/ProseMirror, target the specific `.ProseMirror` surface and passively poll its `contenteditable` attribute; `toBeEditable()` can starve the WebSocket auth update.

```ts
const editor = page.locator('[data-testid="editor-content"] .ProseMirror')
await expect.poll(() => editor.getAttribute('contenteditable'), {
  timeout: 30_000,
  intervals: [500],
}).toBe('true')
```
