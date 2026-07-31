_Load this reference when you commit, push, or pull an app's code; when you start a parallel line of work (`workspace`); when a deploy is refused with `dirty_worktree` / `behind_trunk` / `stale_base`; when you need to undo a deploy (`releases` / `rollback`); when you need to know what other agents did (`activity` / `status`); or when the user asks about GitHub, remotes, or "where does the code live". For deploy mechanics and secrets, see `references/deploy.md`; for the CLI catalog, `references/cli.md`._

# Version control: cloud repo, workspaces, releases

**The mental model, one sentence:** every DeepSpace app has a real git repo hosted on the platform — you work on a **workspace**, land it to **trunk**, and every deploy is a **release** you can roll back to.

Consequences worth internalizing before you touch anything:

- **It is a real git repo,** not a sync gimmick. The platform speaks the Git smart-HTTP protocol; `deepspace push` / `pull` / `clone` are thin wrappers over `git push` / `git fetch` / `git clone` against a remote named `space`. Plain `git push space main` works.
- **It is the default VCS for a DeepSpace app, and it is additive.** No GitHub required — there's no account to create and no remote to add, because the wrappers install the `space` remote and its credential helper for you. It also doesn't *displace* GitHub: `space` is one more remote on the same repo, so a GitHub `origin` coexists with it perfectly (→ "Works alongside GitHub" below). Don't `git init` a second history and don't set GitHub up unprompted — but when the user wants GitHub, wire it up; nothing about DeepSpace VC is in the way.
- **Nothing is private to your sandbox.** Workspaces, pushes, and releases are visible to every collaborator and every other agent on the app. That's the point: coordination lives on the server, not in your context window.
- **The platform holds the source that a release shipped.** That's why deploy is commit-first (below) — a release with no commit behind it is a release nobody can recover.

## Session: log in once, then stop thinking about it

| Fact | Consequence for you |
|---|---|
| Login is a **one-time human act** — browser OAuth, polls up to 10 min | Pause and let the user finish it at the keyboard. **Never** wrap `deepspace auth login` in `timeout` / `sleep N && kill` — that aborts the OAuth poll and leaves no session. |
| Sessions live **30 days with daily sliding renewal** | Any agent activity within a month keeps the session alive indefinitely. Re-login is the rare exception, not a per-session ritual. |
| CLI JWTs are short-lived (15 min) and **refresh automatically** | Don't manage tokens. Don't stat `~/.deepspace/session` or `~/.deepspace/token` — implementation details. |
| A 401 mid-run is **healed once, silently** (re-exchange + retry; mutations are idempotency-keyed) | A transient auth blip is not a reason to re-login or to abandon the operation. |

`deepspace auth whoami` is the login-state probe (`--json` for agents). A `not_authenticated` refusal is the only signal that means "log in again".

## `deepspace status` — the re-orientation command

```bash
npx deepspace status          # one screenful: session, app, deps, branch/workspace, live release, unseen activity
npx deepspace status --json
```

Run it **first** after any context loss, at the start of a session, or whenever you're unsure where you stand — instead of reconstructing state from five reads and memory. It reports state, not story; interpretation is your job. Network sections degrade per line to `(unreachable)`, so it always reports the local facts it has, and it exits 0 whenever the report was produced (it is informational, never a gate).

Its final line is a derived `Next:` — a first-match state machine over the workflow (`not logged in → auth login`; `no app → app create` / `clone`; `dirty → commit`; `unsynced → workspace sync`; `ahead of base → workspace land`; `trunk differs → pull`). Treat it as a strong default, not an order.

## Workspaces — the one unit of in-progress work

A workspace is **one hidden ref** (`refs/deepspace/ws/<id>`) in the app's cloud repo plus a metadata row. It is what a bare `git worktree` can't be: durable (survives the sandbox dying), discoverable (every collaborator sees it), resumable from any machine, and coordinated. Inside a workspace worktree the local branch is `ws/<ulid>` — every workspace command infers the workspace from that branch, so there is no state file to lose.

```bash
npx deepspace workspace new -t "wire RBAC into billing"   # --base <rev> --dir <d>; -t/--task is REQUIRED
npx deepspace workspace attach ws_… [dir]                 # resume on another machine or a fresh sandbox
npx deepspace workspace sync                              # publish: push the branch + report overlapping workspaces
npx deepspace workspace list [--all] [--limit N]          # --all includes landed/dropped
npx deepspace workspace status                            # THIS workspace: base staleness, overlaps, sync state
npx deepspace workspace land [--into <branch>] [--validate]
npx deepspace workspace drop [ws_…]
```

Every one of these takes `-a/--app` and `--json`; `sync` / `status` / `land` take `-w/--workspace` to name the workspace explicitly instead of inferring it.

**Lifecycle rules that actually bite:**

| Rule | What it means |
|---|---|
| **WIP belongs in commits on the workspace branch.** | Commits are cheap, durable once synced, and **squash at land** — so don't hoard uncommitted work to keep history tidy. This is also the escape hatch for a `dirty_worktree` deploy refusal. |
| **`land` merges into trunk and deletes the workspace ref in the same action.** | The merge commit keeps the whole line reachable through history — nothing is lost. The workspace is gone afterward; a new line of work means a new workspace. |
| **`land` / `drop` clean up by default.** | The local worktree and branch are removed. `--keep-worktree` opts out. (`--cleanup` is accepted and is now a no-op default; `--no-cleanup` also opts out.) |
| **Workspace refs are fast-forward-only.** | A push that would drop another checkout's work is refused (`non_fast_forward`, exit 2) rather than force-moved. Integrate the tip, then retry — see the playbook below. |
| **`node_modules` is auto-provisioned** into a fresh worktree (symlinked from the main checkout; junction on Windows; reported as `provisioned` in `--json`). | Don't run a manual install in a new workspace. If `package.json` diverges *in* the workspace, a plain `npm install` there replaces the link with a real dir. |
| **Overlap warnings are computed client-side.** | `sync` / `status` / `list` fetch peer workspace tips and diff paths locally, so a report can never go stale against the refs. Outside a clone of the app's repo, `list` simply shows no overlap markers. Overlaps are a **warning**, not a block — read them and coordinate before you land. |
| **`land --validate`** runs the project's `validate` script (package.json) against the merged tree and aborts the land on failure (exit 2). | A pure local gate — nothing is recorded server-side. Use it when landing risky work; it is not a substitute for `deepspace test`. |
| **A `ws/<id>` branch is never plain-pushed.** | `deepspace push` and deploy's auto-push both skip it — publishing goes through `workspace sync`, which writes the hidden ref plus the metadata that makes overlaps, staleness, and activity work. |

## Deploy is commit-first

`deepspace deploy` records the commit it ships. Therefore:

- **A dirty worktree is refused before the build** — `dirty_worktree`, exit 2. Commit the changes (to a workspace branch if this is WIP — they squash at land), then redeploy.
- **`--no-push` is the escape hatch,** and it costs you lineage: the release records no source, so the platform cannot recover what shipped. Use it only when the user knowingly wants an off-the-record deploy.
- **Deploy auto-pushes the branch by default** (`--push` is the default; `--no-push` disables). It refuses a checkout that is strictly behind the cloud trunk (`behind_trunk`, exit 2) or whose base is stale relative to a newer release (`stale_base`) — `--ignore-stale` overrides the stale-base guard.
- There is **no dirty-tree snapshot subsystem**. If you remember hidden `chk_` refs or `deepspace checkpoint`, that surface is gone.

## Releases and rollback

```bash
npx deepspace releases [--limit N] [--json]     # immutable, append-only deploy history
npx deepspace rollback [rel_…]                  # default: the previous release
npx deepspace rollback rel_… --allow-do-deletion
```

- The ledger is **append-only**. A rollback does not rewind history — it **mints a new release** that re-ships a prior release's retained bundle, with no rebuild.
- **DO-deletion guard:** if the target release declares fewer Durable Object classes than the release currently live, rollback refuses. Proceeding requires `--allow-do-deletion`, and **the removed classes' stored data is deleted**. Never pass that flag on the user's behalf without saying what will be destroyed and getting a yes.
- `rollback` is the correct answer to "the deploy broke prod" — faster and safer than rebuilding an older tree.

## Activity — the coordination feed

```bash
npx deepspace activity                 # exactly-once: continues from the stored per-app cursor
npx deepspace activity --all           # ignore the cursor, replay from the start
npx deepspace activity --follow        # keep polling (Ctrl-C to stop)
npx deepspace activity --since <cursor> --limit N --json
```

This is how you answer "did trunk move / did a workspace land / did someone deploy?" without cloning anything. Events are pushes, workspace lifecycle, and releases — merged, oldest first, delivered **exactly once per cursor chain** (a per-app cursor persists between runs; `--all` replays).

**The feed is facts, not narrative.** It does not tell you what to do, and it does not summarize. You own the interpretation — including the one honest ambiguity: a land reaches the feed as a trunk push followed by `workspace.landed`, so an undifferentiated push can read as someone pushing directly to trunk. Correlate within the page before you conclude a teammate bypassed the workflow.

## push / pull / clone, and plain-git interop

```bash
npx deepspace clone <app|app_id> [dir]     # real git clone; the remote is named `space`
npx deepspace push [-b <branch>] [--force] [--allow-committed-secrets]
npx deepspace pull [-b <branch>]           # fetch + fast-forward
```

- **Remote setup is implicit.** Every wrapper (`clone` / `push` / `pull` / `workspace` / `deploy`) adds or repairs the `space` remote and installs a repo-scoped credential-helper config. After any wrapper has run once, **plain `git push space main` / `git fetch space` work in any terminal** — that's the point of speaking real git. `deepspace git-credential` is that helper; never invoke it directly.
- **`pull` exits 2 when a local step remains** — it fetched successfully, but the branch diverged, the worktree is dirty, or the branch is checked out in another worktree. The message names the exact `git` command to run; nothing is broken.
- **`push --force` is guarded,** not raw: it refuses whenever the remote tip is a commit your branch doesn't contain, so no work is silently dropped. A first force from a fresh clone or pull is refused until you re-integrate that tip. Don't reach for it to escape a `non_fast_forward` — integrate instead.
- **Committed secrets are pre-checked.** `push` refuses a branch tracking `.dev.vars*`, `.env*`, `.npmrc`, `.envrc`, or `.mcp.json`. `--allow-committed-secrets` skips only the CLI's check — the platform also refuses recognized secret files, and its check is case-sensitive, so an oddly-cased name (`.ENV`) can still slip through. Fix the branch; don't pass the flag.
- **`DEEPSPACE_DEPLOY_URL` selects the platform host** the repo talks to (staging vs production) and is baked into the injected credential config. If it is set in the environment, the repo, releases, and activity you see belong to *that* host — a mismatch between what you set for one command and not another is a common source of "my workspace vanished". Keep it consistent for the whole session or leave it unset.

## Works alongside GitHub

DeepSpace VC is **not** a GitHub replacement, and the two are not a choice. One local repo, one history, two remotes:

| Remote | Points at | Synced by | Owns |
|---|---|---|---|
| `space` | The app's cloud repo on the DeepSpace platform | `deepspace push` / `pull` / `clone` (or plain `git push space main`) | Deploy lineage, releases/rollback, workspaces, activity |
| `origin` | GitHub (or GitLab, or anything else) — if the user wants one | `git push origin main` / `git pull origin main`, as always | PRs, code review, Actions, whatever the team already uses |

Adding GitHub to an app that already has a cloud repo is the ordinary git move — `git remote add origin git@github.com:org/app.git && git push -u origin main`. Nothing to migrate: the commits are the same objects, so both remotes end up holding the same history. Going the other way (a repo that started on GitHub) needs nothing at all — the first `deepspace push` or `deploy` adds `space` and uploads the history.

**The one rule: for deploys, trunk truth is the cloud repo.** Deploy's `behind_trunk` / `stale_base` guards, the release ledger, `activity`, and `rollback` all read `space` — a commit that exists only on GitHub is invisible to them, and a release can't record source the platform doesn't hold. So keep the two trunks in step: push to both after landing (`deepspace push && git push origin main`), or make one a scheduled mirror of the other. Drift is not corruption — it just means the live app and the GitHub view disagree about what "latest" is.

Two smaller things worth knowing:

- **Workspace refs live only on `space`.** They're `refs/deepspace/ws/*`, not `refs/heads/*`, so a plain `git push origin` never carries them and coordination (overlap warnings, base staleness, activity) only works through `space`. That's usually what you want: in-flight agent work stays off GitHub, and GitHub sees the merge commit after `workspace land`. If the team wants PR review of a line of work, push the `ws/<id>` branch to `origin` as a normal branch and open the PR there — the workspace keeps coordinating on `space` either way.
- **Releases have no GitHub equivalent and need none.** Release bundles are retained platform-side; `rollback` re-ships one without a rebuild. GitHub tags are a fine parallel habit, not a substitute.

If the user asks for CI deploys from GitHub Actions, that's a supported thing to build — the runner needs a DeepSpace session (`deepspace auth login --email <e> --password-stdin`, or `DEEPSPACE_EMAIL` / `DEEPSPACE_PASSWORD` in the environment; never the plaintext `--password` flag, which leaks through argv). Don't set it up unprompted.

## The output contract and exit codes

Every command follows the same shape, because agents read text:

- **Human output is the interface and is complete.** `--json` mirrors it (`code`, `next: string[]`); it is never the only place a fact lives.
- **A refusal's final line carries its machine slug in brackets:** `…the push was refused rather than drop that work. [non_fast_forward]`. Key on the slug, not on prose.
- **The last line names the follow-up:** `Next: deepspace pull` — copy-paste runnable, at most two commands, and absent (never filler) when the operation is terminal.

| Exit | Meaning | What to do |
|---|---|---|
| `0` | Done. | Continue. |
| `1` | Failed — retrying as-is won't help. | Read the slug; fix the cause. |
| `2` | **Succeeded, but a local step remains** (`actionRequired: true` in `--json`). | Not an error. Run the `Next:` command, then re-run the original. |

Unknown commands exit 1 with a did-you-mean hint (`--json`: `code: unknown_command`). `--help` is always read-only and exits 0.

## Recovery playbook

| Symptom (slug) | What it means | Command |
|---|---|---|
| `non_fast_forward` | Another checkout advanced this workspace's line; publishing yours would drop their work. | `git pull space refs/deepspace/ws/<id>` (or re-attach in a fresh dir), resolve, re-run `workspace sync`. Amended/rebased your own commits? Same refusal — merge your old tip back in; WIP squashes at land. |
| `behind_trunk` | Your branch is strictly behind the cloud trunk; deploying would take already-landed work off the live app. | `npx deepspace pull`, then redeploy. (`--ignore-stale` ships the older tree anyway — only on request.) |
| `stale_base` | A newer release landed while you worked. | `npx deepspace pull`, integrate, redeploy — or `npx deepspace deploy --ignore-stale`. |
| `dirty_worktree` | Deploy is commit-first and the tree has uncommitted changes (also: `pull` fetched but can't fast-forward over a dirty tree). | `git add -A && git commit -m "…"` (to the workspace branch if WIP), then re-run. Last resort: `deploy --no-push` — records no lineage. |
| `diverged` | `pull` fetched, but the local branch and the cloud repo diverged. | Run the `git merge` / `git rebase` command the message prints (no network needed), then `npx deepspace push`. |
| `workspace_not_active` | The workspace already landed or was dropped. | `npx deepspace workspace new -t "…"` — start a new line of work. |
| `not_authenticated` | No session, or the session expired (30d idle). | `npx deepspace auth login` — foreground, no `timeout` wrapper, then `npx deepspace auth whoami`. |
| `renamed` | You used a pre-0.6 top-level command name. | The message names the new path — run that. See below. |

Other slugs you may meet: `merge_conflict` / `merge_failed` (land hit a real conflict), `validation_failed` (`land --validate` gate), `no_releases` / `no_previous_release` (nothing to roll back to), `branch_exists`, `dir_exists`, `not_in_app_repo`, `not_in_workspace`, `force_unverified` (the guarded `push --force`).

## Old command names are hard tombstones

The CLI moved from 31 top-level commands to 16 (session verbs under `auth`, app-lifecycle verbs under `app`, and `kill` / `screenshot` / `test-accounts` / `invoke` folded into `dev` / `test` / `integrations`). Old names are **not aliases**: they exit 1 with `[renamed]` and name the new path (`--json`: `code: renamed`, plus `next`).

```
$ npx deepspace login
`deepspace login` moved: use `deepspace auth login`. [renamed]
```

So a stale command is a loud, one-line fix — never a silent success. If your memory of the CLI predates this, re-read `references/cli.md` rather than guessing. Removed outright (not renamed): `deepspace checkpoint …`, `deepspace handoff …`, and `deepspace validate` — the workspace is now the only work primitive, and validation survives only as `workspace land --validate`.
