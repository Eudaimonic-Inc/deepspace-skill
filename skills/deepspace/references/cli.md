_Load this reference for the login contract, the full CLI command catalog, and the `test` command. For deploy mechanics, the `.dev.vars` contract, and secret handling, see `references/deploy.md`; for the cloud repo, workspaces, releases/rollback, and the exit-code/output contract, `references/version-control.md`. The core SKILL covers the happy-path commands; the full contracts behind them live here._

# CLI catalog, login, and test

## The command tree is 16 top-level commands — old names are tombstones

Session verbs live under `deepspace auth`, app-lifecycle verbs under `deepspace app`, and `kill` / `screenshot` / `test-accounts` / `invoke` fold into `dev` / `test` / `integrations`. Pre-0.6 top-level names are **hard tombstones, not aliases**: they exit 1 with `[renamed]` and name the new path (`--json`: `code: renamed` + `next`), so a stale command is a loud one-line fix, never a silent success.

| You may remember | Run instead |
|---|---|
| `login` · `logout` · `whoami` | `auth login` · `auth logout` · `auth whoami` |
| `create` · `init` · `apps` · `undeploy` | `app create` · `app init` · `app list` · `app undeploy` |
| `transfer` · `collaborators` · `domain` · `library` · `usage` | `app transfer` · `app collaborators` · `app domain` · `app library` · `app usage` |
| `kill` | `dev kill` |
| `screenshot` · `test-accounts` | `test screenshot` · `test accounts` |
| `invoke` | `integrations invoke` |

Removed outright (not renamed): `checkpoint`, `handoff`, `validate` → `references/version-control.md`.

The 16: `status` · `activity` · `push` · `pull` · `clone` · `workspace` · `releases` · `rollback` · `deploy` · `dev` · `test` · `add` · `secrets` · `integrations` · `auth` · `app` (plus `feedback`, and `git-credential` as plumbing you never invoke). The first eight are version control → `references/version-control.md`.

## Login contract (`npx deepspace auth login`)

DeepSpace runs against a hosted platform. Every `npx deepspace` command except scaffolding (`npm create deepspace`) and the no-auth catalog probes (`integrations list`/`info`, `add --list`/`--info`) needs a logged-in account.

**`npx deepspace auth whoami`** is the canonical login-state probe (add `--json` from agents). It refreshes the JWT in the same call path `dev` / `test` / `deploy` use — if `whoami` succeeds, those will too. On failure: stderr ``Not logged in. Run `deepspace auth login` first.`` (`code: not_authenticated`), exit 1. Don't stat `~/.deepspace/session` — that's a CLI implementation detail.

Four hard rules:

1. **Pause and tell the user.** Login opens a browser tab (GitHub/Google OAuth) on their machine and polls up to 10 minutes. They need to be at the keyboard. There is no agent-runnable bypass — never ask the user for their password.
2. **Run interactive login without an artificial time bound.** **Do not** wrap in `timeout N`, `sleep N && kill`, or any cutoff — those terminate OAuth before completion and leave no session. Run in foreground or a true background process.
3. **After login completes, verify with `npx deepspace auth whoami`** before retrying `dev` / `test` / `deploy`. Re-running them while login is still polling produces the same error — that's expected order, not a bug.
4. **Login state is shared across all apps on the machine, and lasts.** Sessions live **30 days with daily sliding renewal**, so any activity within a month keeps them alive indefinitely; the CLI's JWTs are short-lived and refresh themselves (a one-off 401 is silently re-exchanged and retried). One `deepspace auth login` covers `dev`, `test accounts`, and `deploy` for any app — re-login only when `whoami` reports signed-out or expired. **Never copy `.dev.vars` from a sibling app** — `APP_OWNER_JWT` is minted per app; borrowing causes silent auth mismatches.

## Full command catalog

The happy-path commands (login, `npm create deepspace`, `add`, `dev`, `test`, `deploy`) are in the SKILL. The rest, by lifecycle stage:

```bash
# --- Local dev ---
npx deepspace dev                  # Vite + worker in-process; HMR on localhost:5173, --strictPort fails loudly on clash
npx deepspace dev --port 5180      # parallel apps
# (no local/prod toggle: dev/test/deploy always target the production platform workers)
npx deepspace dev kill             # kill YOUR leaked listener on 5173 + its workerd children (never a sibling session's)
npx deepspace dev kill --port 5180
npx deepspace dev kill --all       # sweep every workerd/wrangler/vite on the box

# --- Test accounts (one-time per machine; pool shared across all apps, hard cap of 10) ---
npx deepspace test accounts list
npx deepspace test accounts create --email <…@deepspace.test> --password <p> --name <n>

# --- Tests (→ references/testing.md for the real depth) ---
npx deepspace test                 # default suite (smoke + api); auto-installs Playwright + chromium on first run
npx deepspace test e2e             # all Playwright specs
npx deepspace test unit            # vitest
npx deepspace test screenshot http://localhost:5173/ out.png [--full-page --wait-for-timeout 500]

# --- App secrets (one store per app, keyed by DEEPSPACE_APP_ID; source of truth for all envs; never hand-edit .dev.vars) → references/secrets.md ---
npx deepspace secrets set API_KEY=...    # no setup step; works pre-deploy
npx deepspace secrets list
npx deepspace secrets pull               # refresh the .dev.vars cache
npx deepspace secrets upload .env        # dotenv/JSON; the legacy-.dev.vars migration path
npx deepspace secrets get KEY --plain    # print one value (byte-exact when redirected)
npx deepspace secrets download --format dotenv|json|shell      # a whole config to stdout
npx deepspace secrets configs list       # the app's configs (one per env)
npx deepspace secrets configs create staging --copy-from prd   # server-side copy — don't re-set values by hand
npx deepspace secrets configs delete staging                   # config + all its secrets
# every secrets command: -a/--app <appId>, -c/--config <name>, -e/--env <name>

# --- App identity & lifecycle (id = the app; name = the URL) → references/app-identity.md ---
npx deepspace app list                       # all your apps: id, URL, deploy state (--json)
npx deepspace app init                       # stamp DEEPSPACE_APP_ID into an existing repo
npx deepspace app init --new-id              # fork a cloned repo into your OWN app (fresh data + secrets)
npx deepspace app undeploy [--env <name>]    # off the network; the id survives, redeploy revives
npx deepspace app undeploy <app-id-or-name>  # positional (registry-resolved) works from anywhere — no wrangler.toml needed
npx deepspace app transfer offer <email>     # 7-day ownership handshake; accept --app <appId> | cancel | status
npx deepspace app usage                      # credit balance, quota headroom, per-integration spend

# --- Collaborators (owner-only management; collaborators can deploy + manage secrets, not undeploy/transfer) → references/collaborators.md ---
npx deepspace app collaborators list                         # also prints a PENDING INVITES section
npx deepspace app collaborators add teammate@example.com     # existing user → added now; non-user → emailed invite (billed to owner), joins on sign-in; re-add while live → already_invited
npx deepspace app collaborators cancel teammate@example.com  # rescind a pending (un-accepted) invite
npx deepspace app collaborators remove teammate@example.com

# --- Integrations discovery (NO AUTH for list/info; invoke is billed) → references/integrations.md ---
npx deepspace integrations list
npx deepspace integrations info openai/chat-completion
npx deepspace integrations invoke openai/chat-completion --body '{...}'      # AUTH REQUIRED — actually calls, billed to caller
npx deepspace integrations invoke openai/chat-completion --body-file request.json

# --- Custom domain (→ references/domain.md) ---
npx deepspace app domain search <query>
npx deepspace app domain buy <domain>
npx deepspace app domain list
npx deepspace app domain attach <domain> --app <name>

# --- Publish to the DeepSpace community library ---
npx deepspace app library publish [--name "<title>"] [--description "<short>"] [--category <cat>]
npx deepspace app library unpublish <handle>

# --- Version control: the app's cloud git repo (→ references/version-control.md) ---
npx deepspace status                     # re-orient: session, app, workspace, live release, unseen activity
npx deepspace activity                   # coordination feed (exactly-once cursor; --all replays, --follow polls)
npx deepspace push / pull                # sync commits with the `space` remote
npx deepspace clone <app|app_id> [dir]   # materialize the repo (remote is named `space`)
npx deepspace workspace new -t "…"       # durable parallel line of work; sync | status | list | land | drop
npx deepspace releases                   # append-only deploy history
npx deepspace rollback [rel_…]           # re-ship a prior release's bundle (no rebuild)

# --- Features & feedback ---
npx deepspace add <feature> --install    # also runs your package manager for the feature's new deps
npx deepspace feedback "<title>" -t bug|feature|other -m "<details>" [--yes --json]  # file it with DeepSpace
```

Scaffolding also has a CLI home: `npx deepspace app create <name>` runs `create-deepspace` and forwards every flag.

The scaffolder (`npm create deepspace@latest <app-name>`) is non-interactive by default (agent-friendly): omitting `<app-name>` prints usage and exits 1 instead of prompting. Pass `--interactive` / `-i` for the wizard; probe with `--help` / `--version` (plain stdout, no ANSI) before scripting. It scaffolds into a fresh, near-empty, or current dir ("near-empty" = only boilerplate like `.git`, `*.md`, `.dev.vars`); anything else triggers `Directory <name> already exists` and it bails. After scaffold, dependencies install in a detached background process; every subsequent `npx deepspace` command waits on it (gates on `node_modules/deepspace`), so you never need a manual `npm install`.

## Test (`npx deepspace test`)

Tests are the primary way to verify code changes. The scaffolded specs (`smoke.spec.ts` / `api.spec.ts` / `collab.spec.ts`) are starting points — extend them per the Step 8 checklist in `references/testing.md`. The full extension table, debug-from-failures rule, route coverage, multi-user patterns, and the `'deepspace/testing'` fixture all live there.

**One rule that bites first-time runs:** run tests only after a runtime-affecting code change (`src/`, `worker.ts`, etc.). Skip them for conversation, planning, reading, or pure-doc edits — don't run as a ritual. Tests always use dev workers and need provisioned test accounts (see catalog above).

## Output contract (every command)

Human output is the interface and is complete; `--json` mirrors it. A refusal's final line carries its machine slug in brackets (`… [dirty_worktree]`), and the last line is `Next: <copy-paste runnable command>`. Exit codes are load-bearing: **0** done, **1** failed (retrying as-is won't help), **2** succeeded but a local step remains (`actionRequired: true` — run the `Next:` command, then re-run). Unknown commands exit 1 with a did-you-mean hint; `--help` is read-only and always exits 0. Full slug list and recovery playbook → `references/version-control.md`.

**Deploy mechanics, the `.dev.vars` contract, and secret handling → `references/deploy.md`. The cloud repo, workspaces, releases/rollback, and activity → `references/version-control.md`.**
