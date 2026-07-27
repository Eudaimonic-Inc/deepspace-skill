_Load this reference when: syncing code between machines or collaborators without GitHub (`deepspace push`/`pull`/`clone`, or plain `git push space`), saving or restoring work-in-progress durably (`deepspace checkpoint`), viewing deploy history or undoing a bad deploy (`deepspace releases`, `deepspace rollback`), a deploy fails with `stale_base`, a teammate/agent needs the code for an app they collaborate on, several agents work the same app in parallel (`deepspace workspace`, `deepspace activity`), or work transfers between agents/sessions (`deepspace handoff`, `deepspace validate`)._

# Version control: cloud repo, checkpoints, releases

Every DeepSpace app has a **cloud repo** on the platform, keyed by its immutable `DEEPSPACE_APP_ID` — and it is a **real git remote**. The first time a VC command that touches the repo runs (`push`/`pull`/`clone`/`checkpoint save`/`restore`, or `deploy`) it configures a remote named `space` plus git credentials, and from then on **plain `git push space main`, `git fetch space`, `git log space/main`** work in any terminal, no wrapper needed (as long as the CLI is still installed in the repo — the credential helper invokes it). The CLI wrappers add what bare git can't: app resolution, `--json` output for agents, durable work-in-progress snapshots, and deploy history with rollback. No GitHub account or remote is required for any of it.

## Sync: push / pull / clone

```bash
npx deepspace push                  # current branch → cloud repo (fast-forward only)
npx deepspace push --force          # after an intentional history rewrite
npx deepspace pull                  # fetch + fast-forward the current branch
npx deepspace clone <app|name> [dir]  # materialize an app's repo on a new machine
git push space <branch>             # after any wrapper has run once, bare git works too
```

- Every VC command takes `--json` (parse, don't scrape). `-a <app id or subdomain name>` targets the app explicitly on every VC verb except `clone` (its app is the positional argument) and `deploy` (it deploys the checkout you run it in). Pure-server verbs work from ANYWHERE with `-a` — `activity`, `releases`, `rollback`, `workspace list` (and `drop` with an explicit ws_ id), `handoff list`/`show`/`drop`/`delete`, `checkpoint list`/`show`/`delete`; verbs that touch a working tree (`push`/`pull`, `checkpoint save`/`restore`, `handoff save`/`take`, `workspace new`/`attach`/`sync`/`status`/`land`, `validate`) run inside a checkout.
- `push` is fast-forward-only, like git. A `non_fast_forward` or `ref_conflict` result means someone pushed since you last synced: run `npx deepspace pull`, integrate, push again.
- `pull` never touches a dirty worktree. If histories diverged it fetches the branch's history and tells you the exact local merge to run (`git merge refs/remotes/space/<branch>`, prefixed with a `git checkout <branch>` when you pulled a branch that isn't checked out) — then push the result. Exit code 2 = fetched but action still required — `--json` then carries `ok:false, actionRequired:true` plus `next` (the exact command to run). `workspace land` reports merge conflicts with the same exit-2/JSON shape.
- `clone` is the collaborator on-ramp: the clone is deploy-ready because `wrangler.toml` (with the app id) is part of the history, and the `space` remote comes pre-configured. Follow with `npm install`, `npx deepspace secrets pull`, `npx deepspace dev`. Collaborators must be added by the owner first (`references/collaborators.md`).
- First `push` works before the first deploy — it registers the app id to your account, exactly like a first deploy would.
- Limits worth knowing: git ≥ 2.29, sha1 repos only (git's default), and no shallow clones (`--depth` is rejected). A `--filter` partial clone isn't supported either, but it degrades gracefully — git warns (`filtering not recognized by server, ignoring`) and falls back to a normal full clone. A single push is capped at ~32 MB of packed history, and any single object over ~20 MB is rejected server-side with a clear message that names the cap (e.g. `remote unpack failed: … per-object limit is 20971520` / `[remote rejected] (object exceeds the 20 MiB limit — remove it or use Git LFS)`). Find the offender with `git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' | sort -k2 -n | tail`, then `.gitignore` it (or use Git LFS / drop it from history).
- Cloning a very large repo (lots of big-object history across many branches) in one shot can exceed the server's per-request assembly budget; when it does you get a clear `fatal: this repository is too large to clone in a single request. Fetch one branch at a time…`. Work around it with a single-branch clone — `git clone --single-branch --branch <branch> <url>` — then `git fetch` the other branches as needed. (Background compaction consolidates overlapping packs over time, which helps a many-branch repo; a single branch with very large blobs is better served by Git LFS.)

## Checkpoints: durable work-in-progress

A checkpoint snapshots the **entire working tree — uncommitted and untracked files included** (`.gitignore`d files excluded; `.dev.vars*` never captured, gitignored or not) — parents it on HEAD, and stores it in the cloud repo with structured context. Nothing touches your branch, index, or worktree. It survives the sandbox/session dying and restores on any machine with access to the app.

```bash
npx deepspace checkpoint save -m "auth flow half-wired; RBAC tests failing" \
  [--context-file notes.json]      # attach structured state (task, unresolved issues, next action)
npx deepspace checkpoint list      # list/show/delete work from anywhere with -a <app>
npx deepspace checkpoint show chk_…      # full context JSON
npx deepspace checkpoint restore chk_…   # new branch checkpoint/<ulid> at the snapshot (needs clean tree)
npx deepspace checkpoint restore chk_… --as-branch <name>     # pick the branch name yourself
npx deepspace checkpoint restore chk_… --in-place [--force]   # overwrite the worktree instead;
                                                              # --force DISCARDS current uncommitted changes.
                                                              # Files created AFTER the snapshot survive as
                                                              # untracked leftovers (--json flags this with
                                                              # untrackedLeftoversPossible: true — git status after)
npx deepspace checkpoint delete chk_…    # deletable by its creator, the app owner, or an admin (a collaborator can delete their own, not another's)
```

Use them **before risky passes** (big refactors, redesigns, destructive migrations) and **at handoff points** — `-m` plus `--context-file` is how the next session/agent learns the objective, what's done, and what's broken without re-deriving it. A clean tree checkpoints as HEAD itself (cheap pointer, still restorable anywhere). Under the hood a checkpoint is a hidden ref (`refs/deepspace/checkpoints/…`) pushed over the normal git protocol — it never appears on your branches.

## Workspaces: parallel agents on one app

A workspace is a **registered line of work**: a ref in the app's cloud repo plus a local worktree, visible to every collaborator with its task, who's on it, and which files it touches. When more than one agent (or a human plus agents) works on one app at the same time, each takes a workspace instead of committing to the shared branch — work proceeds in parallel, publishes continuously without touching trunk, and merges ("lands") when done.

The loop:

```bash
npx deepspace workspace new -t "add auth pages"   # → prints the worktree dir to work in
cd <printed dir>                                  # branch ws/<ulid> is checked out there
# … edit, git add, git commit as usual (commit early and often) …
npx deepspace validate                            # run + record the project check
npx deepspace workspace sync                      # publish — repeat after each commit
npx deepspace workspace land                      # merge into trunk when the task is done
```

**Two views of truth:** wrapper commands (`workspace …`, `sync`, `pull`, `deploy`, `activity`) always consult the **live cloud repo**; plain `git` sees only your last `git fetch space`. So `workspace new` branches from the live trunk tip (its `baseOid` may be ahead of a checkout you haven't pulled), `behindTrunk` grows while you work, and those counts include lands' merge commits (a `git log a..b` range can look one short). None of that is an error — fetch when you want plain git to catch up.

- `new -t "<task>"` (required) creates the server record + a worktree at `.deepspace/ws/<ulid>` (the id's lowercased ULID) (kept out of `git status` automatically — a custom in-repo `--dir` is excluded likewise; `--base <rev>` to branch from something other than the cloud trunk tip). `--json` returns `{workspaceId, branch, dir, baseOid, ref}` — **`cd` into `dir` and do all work there** (the worktree is a DIFFERENT absolute path from your main checkout — every file you create or edit must live under the printed `dir`, whichever tool or absolute path you use). The default worktree lives *inside* the app dir, so Node tooling resolves the app's `node_modules` by walking up — **no `npm install` needed** (the worktree has no `node_modules/.bin` of its own; invoke tools via `npx` or scripts, not direct `.bin` paths); only `attach` into a fresh clone (or a `--dir` outside the repo) needs its own install.
- `sync` publishes your **commits** (uncommitted edits aren't synced — commit first; for a durable snapshot of *uncommitted* work use `npx deepspace checkpoint save`, which auto-links to the workspace) by force-pushing your own workspace ref — no one else's work is ever touched — and records which paths you changed. Run it from the worktree (the workspace is inferred from the `ws/<ulid>` branch). Sync after each meaningful commit: it's how other agents see your progress, how your work survives the sandbox dying, and how **overlap warnings** surface. An overlap in the output means another active workspace touched the same paths — check `deepspace activity`/`workspace list`, coordinate, and expect merge attention at land time. The signal is only as fresh as everyone's sync cadence: an agent that edits but hasn't synced is **invisible** until they do. `sync` also warns when **trunk already contains changes** to files you touch (`trunkOverlap` in `--json`) — that's landed work you haven't merged yet; merge trunk into your line before landing.
- `list [--all]` shows every workspace: `↑` commits ahead of base, `↓` trunk commits since base. It works from **anywhere** with `-a <app>` — a fresh sandbox can survey in-flight work before it clones. Note `↓` measures *base staleness*, so it stays nonzero even after you merge trunk into your line; `status` (from the worktree) gives exact local truth — whether your HEAD is synced, dirty files, which **trunk** changes overlap yours, and linked checkpoints.
- `land [--into <branch>]` merges the workspace into trunk — fast-forward when possible, else a real merge commit (both histories retained) — pushes trunk, and marks the workspace `landed`. Anyone with app access may land any workspace (finishing a crashed agent's work is legitimate). On a merge **conflict** it exits **2** (`--json`: `ok:false, actionRequired:true, conflict:true`, plus `next: "deepspace workspace land"` — the single safe re-entry command to run *after* you've resolved), leaving the conflict in your worktree and your original work safe on the workspace ref: resolve the conflicted files, `git add` them, `git commit`, then run `workspace land` again. (`next` is deliberately just the re-entry command, not a chained `git add`/`git commit` — running that before genuinely resolving would stage the conflict markers and commit them, and a resumed land pushes straight to trunk; bare `land` refuses on the still-dirty worktree until you've actually resolved.) **Validate the merged tree before re-landing** (`npx deepspace validate`): a merge with no conflict markers can still be semantically wrong — git happily keeps both sides' incompatible test blocks side by side. When trunk has moved — overlap or not — merging `refs/remotes/space/<trunk>` into your line, validating, then landing is **always correct** (you validate the exact tree trunk becomes — and if trunk moves again before your land, land stops on the moved tip and you simply repeat); landing directly is also safe when `trunkOverlap` is empty, and land merges for you.
- `drop [<id>]` abandons a workspace (its creator, the owner, or an admin). Deleting an active workspace's ref through raw git is refused — drop is the intended path.
- **After `land` or `drop`, remove the finished worktree** — from the main checkout: `git worktree remove <dir>` (then `git branch -D ws/<…>`; if git refuses over leftover untracked build artifacts, add `--force`). Landed/dropped worktrees otherwise pile up in every collaborator's `git worktree list`; both commands print (and `--json`-return) the path when you ran them from inside it.
- `attach <id> [dir]` resumes a workspace elsewhere: inside a checkout of the app it adds a worktree; in a fresh sandbox (empty dir, pass `-a <app>`) it clones just that workspace.
- Limits: at most **32 active** workspaces per app — land or drop finished ones. Landed workspace refs are retained 14 days, dropped ones 7 (the records persist).

Parallel etiquette, derived from the above: check `workspace list` + `activity` before starting (don't duplicate work), prefer tasks that touch different files, commit+sync often, land promptly, and while others have active workspaces let trunk move via lands rather than direct pushes to it.

## Handoffs: transferring work between agents

A handoff packages **unfinished work plus the context to continue it**, for exactly one other agent/session to claim. It snapshots the entire tree (checkpoint mechanics — uncommitted and untracked files included) together with structured context.

```bash
npx deepspace handoff save -m "wire RBAC into admin routes" --next "make role check pass in admin.test.ts" \
  [--context-file ctx.json]   # {objective, nextAction, state?, unresolved?: []} — -m/--next fill objective/nextAction
npx deepspace handoff list [--all]   # open handoffs (--all includes taken)
npx deepspace handoff show hnd_…     # full context
npx deepspace handoff take hnd_…     # CLAIM it (exclusive) + restore onto branch handoff/<ulid>
npx deepspace handoff take hnd_… --no-restore   # claim + print context without touching the worktree
npx deepspace handoff drop hnd_…     # give an UNFINISHED claim back (re-opens it)
npx deepspace handoff delete hnd_…   # done (or obsolete): delete it outright
```

- `save` requires an objective and a next action (the taker reads `nextAction` first — make it concrete, and machine-independent: the taker's checkout has different absolute paths). `filesChanged` is computed automatically, and the latest `validate` result for the snapshot is auto-linked. The snapshot captures the whole tree, **uncommitted and untracked files included**.
- `take` **claims first**: a second taker is refused and told who holds it; re-running `take` as the same user replays safely (crash-safe). Restoring needs a clean worktree — commit or checkpoint your own work first (the claim itself already succeeded and survives). The restore **materializes the snapshot as a real commit** on the `handoff/<ulid>` branch, even if it was captured uncommitted. Until you `take`, the snapshot commit exists only server-side — probing its oid with plain git fails (`fatal: bad object` or `Not a valid object name`, depending on the command) — that's expected, not corruption. The snapshot also carries the creator's **untracked** scratch files — review `git status`/the diff before landing and drop planning notes that shouldn't ship. If the handoff's linked workspace already **landed** or was **dropped**, `take` and `show` warn and `--json` carries `staleLinkedWorkspace` — the work may already be on the main line; check trunk before building on the snapshot.
- Finishing a handoff that's linked to a workspace: complete the work on the `handoff/<ulid>` branch, then land it *as that workspace* — `npx deepspace workspace land -w <ws_id>` from the branch publishes your HEAD to the workspace's ref and merges it into trunk, closing the workspace as `landed`. Then `handoff delete hnd_…`.
- `delete` is the completion path: the current **taker** may delete a taken handoff directly (no release dance — `drop` would momentarily re-advertise finished work as open). When it's open, its creator/the owner can delete it as obsolete.
- Save a handoff whenever work stops mid-task — session ending, blocked on something, or splitting a task between agents.

## Validate: recorded check results

```bash
npx deepspace validate                       # runs the package.json "validate" script
npx deepspace validate -c "npx vitest run"   # or any explicit command
```

Runs the project's check and **records the result against the current commit** in the cloud repo — command, pass/fail, duration, output tail — visible in `activity` and auto-linked into handoffs. Exit 0 = the check passed; exit 1 = it ran and failed. In `--json`, key on **`passed`** (`ok` only means the result was recorded); on failure it includes `outputTail` so the cause is diagnosable from the one document. It runs on the **working tree**: with uncommitted changes the record carries `dirty:true` — it describes the tree, not the commit alone. Advisory by design (`land`/`deploy` don't gate on it); skip it for docs-only changes, record it for anything with code. This is how one agent knows another's synced tip was actually validated, without re-running the suite or trusting prose.

## Activity: the app's coordination feed

```bash
npx deepspace activity [--limit <n>]        # newest events
npx deepspace activity --since <cursor>     # only what's new (cursor comes from the previous page)
npx deepspace activity --follow             # keep polling until Ctrl-C
```

One append-only feed per app: workspaces created/synced/landed/dropped, branch pushes, checkpoints and handoffs saved/taken, validations, releases — each with actor and time. It's how parallel agents stay aware without a chat channel: check it before starting work and when an overlap warning appears. Reading the feed: a **land** appears as a trunk `push` immediately followed by `workspace.landed` with the same oid — that pair is a land, not someone pushing directly to trunk (the human output labels it `(land of ws_…)`). `--json` emits **exactly one page per invocation** — parse the document, then call again with `--since <cursor>` while `hasMore` is true (`--follow --json` instead streams one JSON line per page).

## Releases: deploy history and rollback

Every `deploy` records an immutable **release**: actor, timestamp, the commit (or checkpoint) it was built from, and the exact built bundle — retained server-side.

```bash
npx deepspace releases [--json]     # history: #seq, id, deploy|rollback, actor, source commit
npx deepspace rollback              # restore the previous release
npx deepspace rollback rel_…        # restore a specific one
```

- `rollback` re-ships the stored bundle — **no rebuild, no checkout needed**, and secrets are untouched. The rollback itself becomes the newest release (history is append-only), so a rollback can be rolled back. With only ONE release there is no previous release to restore — undoing a bad first release is `deepspace undeploy` (owner-only).
- If rollback refuses with `do_class_deletion`: the target release declares fewer Durable Object classes than what's live, and proceeding would **delete those classes' stored data**. Only pass `--allow-do-deletion` (owner or admin only — a collaborator is refused) after confirming with the user that the data loss is acceptable.

### What deploy does automatically

A release ships **the tree of the checkout you deploy from** — other agents' unlanded workspace commits are never part of it (their work joins a release only by landing on the deployed branch). Your OWN uncommitted changes DO ship: deploy auto-checkpoints the dirty tree it builds.

- **Auto-push:** `deploy` pushes the current branch to the cloud repo first so the release's source is recoverable, retrying a transient rate-limit (HTTP 429) so an unlucky hiccup can't silently drop the sync. It stays best-effort with one exception: when the push fails because your branch is strictly **behind** the cloud repo (every local commit is already landed), the deploy **stops** with `code: "behind_trunk"` — shipping it would take already-landed work off the live app. `npx deepspace pull`, then redeploy (`--ignore-stale` ships the older tree deliberately). Any other push failure (a genuine divergence needing `--force`, a committed secret file, or 429s past the retries) only warns and the deploy proceeds with the source **not** recoverable. `deploy --json` includes the recorded `releaseId` and reports this as a `recoverable` boolean — an agent that sees `recoverable:false` should `npx deepspace push` (integrating first if the push was rejected) and redeploy before treating the release as safe. `--no-push` skips the sync entirely, but the stale-base guard below still applies — and it needs the deployed commit to exist in the cloud repo, so `deploy --no-push` of a never-pushed commit on an app with prior releases will 409 until you push (or say `--ignore-stale`, accepting that the release's source commit won't be recoverable from the platform).
- **Auto-checkpoint on dirty trees:** deploying with uncommitted changes creates a checkpoint of the exact deployed tree and records it as the release's source — what's live is always recoverable even if you never committed.
- **Stale-base guard:** `deploy` verifies the code you're shipping *contains* the live release's commit (an ancestry check against the cloud repo). It fails with `stale_base` — naming who released what and when — in two cases: your history doesn't include the live release (someone released since you last synced), or the commit you're deploying was never synced at all (e.g. the auto-push hit a conflict). Either way: `npx deepspace pull`, integrate, redeploy. `--ignore-stale` overrides when clobbering is intended — two agents on one app should sync before deploying rather than lean on the override. In the rare case the repo store itself was unreachable, the deploy proceeds unchecked and `deploy --json` carries `staleBaseGuard: "skipped"` — treat that deploy as unverified against the live release.

## Recovery playbook

| Situation | Do |
|---|---|
| Bad deploy live in prod | `npx deepspace rollback` — instant, no rebuild |
| Session/sandbox died mid-task | `npx deepspace checkpoint list` → `restore` the latest |
| Teammate/agent needs the code | owner adds them (`collaborators add`), they `npx deepspace clone <app>` |
| Push rejected (`non_fast_forward`/`ref_conflict`) | `npx deepspace pull`, integrate (merge if diverged), push again |
| Deploy rejected (`stale_base`) | same as above, then redeploy — on a non-trunk branch `pull` only updates that branch, so integrate trunk explicitly: `git fetch space && git merge refs/remotes/space/main` |
| Deploy refused (`behind_trunk`) | `npx deepspace pull`, then redeploy — your checkout was behind the already-landed main line |
| `deploy --json` returned `recoverable:false` | `npx deepspace push` (integrate first if it was rejected), then redeploy |
| About to do something risky | `npx deepspace checkpoint save -m "before <risky thing>"` |
| Need history/diff/blame of the cloud repo | plain git against the `space` remote — **always `git fetch space` first**: the tracking ref only reflects your last fetch — don't rely on wrapper commands to keep it fresh (`git log space/main` after a fetch) |
| Working alongside other agents on one app | `npx deepspace workspace new -t "…"` → work in the printed dir → `sync` often → `land` |
| `workspace land` exited 2 (merge conflict) | resolve the conflicted files in the worktree, `git add` + `git commit`, `validate` the merged tree (clean text merge ≠ working code), run `land` again |
| Stopping mid-task; someone else will continue | `npx deepspace handoff save -m "…" --next "…"` — the taker runs `handoff take hnd_…` |
| A collaborator left (or a sandbox died) with work in flight | their **synced** workspace: anyone attaches and lands it; their **taken** handoff: `npx deepspace handoff drop hnd_…` reopens it for another taker. Un-synced local work is gone — sync/checkpoint often |
| Finished a handoff you took | land it (via `workspace land -w <ws_id>` when linked), then `npx deepspace handoff delete hnd_…` |
| What are the other agents doing right now? | `npx deepspace activity` / `npx deepspace workspace list` (both work anywhere with `-a <app>`) |
