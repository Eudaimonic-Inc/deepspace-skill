_Load this reference when: syncing code between machines or collaborators without GitHub (`deepspace push`/`pull`/`clone`, or plain `git push space`), saving or restoring work-in-progress durably (`deepspace checkpoint`), viewing deploy history or undoing a bad deploy (`deepspace releases`, `deepspace rollback`), a deploy fails with `stale_base`, a teammate/agent needs the code for an app they collaborate on, several agents work the same app in parallel (`deepspace workspace`, `deepspace activity`), or work transfers between agents/sessions (`deepspace handoff`, `deepspace validate`)._

# Version control: cloud repo, checkpoints, releases

Every DeepSpace app has a **cloud repo** on the platform, keyed by its immutable `DEEPSPACE_APP_ID` — and it's a **real git remote**, no GitHub required. The first VC command that touches it (`push`/`pull`/`clone`/`checkpoint`/`deploy`) configures a remote named `space` plus git credentials; from then on plain **`git push space main`, `git fetch space`, `git log space/main`** work in any terminal (as long as the CLI stays installed in the repo — the credential helper invokes it). The wrappers add what bare git can't: app resolution, `--json` for agents, durable work-in-progress snapshots, and deploy history with rollback.

## Sync: push / pull / clone

```bash
npx deepspace push                    # current branch → cloud repo (fast-forward only)
npx deepspace push --force            # after an intentional history rewrite
npx deepspace pull                    # fetch + fast-forward the current branch
npx deepspace clone <app|name> [dir]  # materialize an app's repo on a new machine
git push space <branch>               # bare git works once any wrapper has run
```

- Every verb takes `--json` (parse, don't scrape). `-a <app id|subdomain>` targets the app on every VC verb except `clone` (app is positional) and `deploy` (deploys the checkout you're in). **Pure-server verbs run from ANYWHERE with `-a`** — `activity`, `releases`, `rollback`, `workspace list` (+ `drop <ws_id>`), `handoff list`/`show`/`drop`/`delete`, `checkpoint list`/`show`/`delete`. **Working-tree verbs run inside a checkout** — `push`/`pull`, `checkpoint save`/`restore`, `handoff save`/`take`, `workspace new`/`attach`/`sync`/`status`/`land`, `validate`.
- `push` is fast-forward-only. `non_fast_forward`/`ref_conflict` = someone pushed since you synced: `npx deepspace pull`, integrate, push again. The first `push` registers the app id to your account, exactly like a first deploy.
- `pull` never touches a dirty worktree. On diverged histories it fetches and tells you the exact local merge (`git merge refs/remotes/space/<branch>`, prefixed with `git checkout <branch>` when that branch isn't checked out), then push the result. **Exit code 2 = fetched but action still required** — `--json` then carries `ok:false, actionRequired:true, next` (the exact command). `workspace land` reports merge conflicts with the same exit-2 shape.
- `clone` is the collaborator on-ramp: deploy-ready because `wrangler.toml` (with the app id) is in history and the `space` remote is pre-configured. Follow with `npm install`, `npx deepspace secrets pull`, `npx deepspace dev`. The owner must add collaborators first (`references/collaborators.md`).
- Limits: git ≥ 2.29, sha1 repos only, no shallow clones (`--depth` rejected; `--filter` degrades to a full clone with a warning). One push caps at ~32 MB of packed history, and any single object over ~20 MB is rejected server-side with a message naming the cap (e.g. `object exceeds the 20 MiB limit`). Find the offender with `git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' | sort -k2 -n | tail`, then `.gitignore` it (or use Git LFS). Cloning a very large multi-branch repo can exceed the assembly budget (`fatal: … too large to clone in a single request`) — clone one branch (`git clone --single-branch --branch <b> <url>`), then `git fetch` the rest.

## Checkpoints: durable work-in-progress

A checkpoint snapshots the **entire working tree — uncommitted and untracked files included** (`.gitignore`d files excluded; **secrets never captured, gitignored or not: `.dev.vars*`, `.env`/`.env.*`, `.envrc`, `.npmrc`, `.mcp.json`**), parents it on HEAD, and stores it in the cloud repo with structured context. Nothing touches your branch, index, or worktree. It survives the sandbox/session dying and restores on any machine with app access.

```bash
npx deepspace checkpoint save -m "auth half-wired; RBAC tests failing" \
  [--context-file notes.json]            # attach {task, unresolved, next action}
npx deepspace checkpoint list            # list/show/delete from anywhere with -a
npx deepspace checkpoint show chk_…      # full context JSON
npx deepspace checkpoint restore chk_…   # → new branch checkpoint/<ulid> at the snapshot (needs clean tree; --as-branch <name> to name it)
npx deepspace checkpoint restore chk_… --in-place [--force]  # overwrite the worktree in place;
  # refuses without --force once HEAD has moved past the snapshot — restore then reverts/removes tracked
  # files NEWER than the snapshot AND discards uncommitted changes (worktree only; the index stays at HEAD,
  # so `git restore -- <path>` brings a reverted file back). Files created after the snapshot survive as
  # untracked leftovers (--json: untrackedLeftoversPossible — check `git status` after).
npx deepspace checkpoint delete chk_…    # its creator, the app owner, or an admin (collaborators: only their own)
```

Use them **before risky passes** (big refactors, destructive migrations) and **at handoff points** — `-m` plus `--context-file` is how the next session/agent learns the objective, what's done, and what's broken without re-deriving it. A clean tree checkpoints as HEAD itself (cheap pointer, still restorable anywhere). Under the hood it's a hidden ref (`refs/deepspace/checkpoints/…`), never on your branches.

## Workspaces: parallel agents on one app

A workspace is a **registered line of work** — a ref in the cloud repo plus a local worktree, visible to every collaborator with its task, owner, and touched files. When more than one agent (or a human plus agents) works an app at once, each takes a workspace instead of committing to the shared branch: work runs in parallel, publishes continuously without touching trunk, and **lands** (merges) when done.

```bash
npx deepspace workspace new -t "add auth pages"   # → prints the worktree dir; branch ws/<ulid> is checked out there
cd <printed dir>                                  # do ALL work here
# … edit, git add, git commit early and often …
npx deepspace validate                            # run + record the project check
npx deepspace workspace sync                      # publish — repeat after each commit
npx deepspace workspace land --cleanup            # merge into trunk, then remove this worktree + branch
```

- **`--cleanup`** on `land`/`drop` removes this clone's finished worktree + branch in one step (best-effort — it never unwinds a successful land/drop; without it, landed/dropped worktrees pile up in every collaborator's `git worktree list`). Run from inside the workspace's own worktree it also gets you out: it prints the main-checkout path (`cd` there — your old dir is gone) and `--json` returns a `cleanup:{worktreeRemoved, branchDeleted, mainDir?, error?}` object.
- **Two views of truth:** wrappers (`workspace …`, `sync`, `pull`, `deploy`, `activity`) always read the **live cloud repo**; plain `git` sees only your last `git fetch space`. So `new` branches from the live trunk tip, `behindTrunk` grows while you work, and those counts include lands' merge commits (a `git log a..b` range can look one short) — none of it an error; `git fetch space` to let plain git catch up.
- `new -t "<task>"` (required) creates the server record + a worktree at `.deepspace/ws/<ulid>` (kept out of `git status`; `--base <rev>` / `--dir` override the defaults). **`cd` into the printed `dir` and do all work there** — it's a DIFFERENT absolute path from your main checkout, so every file you create or edit must live under it. The default dir sits *inside* the app dir, so Node resolves the app's `node_modules` by walking up — **no `npm install`** (invoke tools via `npx`, not `.bin` paths); only `attach` into a fresh clone (or a `--dir` outside the repo) needs its own install.
- `sync` publishes your **commits** (uncommitted edits aren't synced — commit first, or `checkpoint save` for a durable WIP snapshot that auto-links to the workspace). It force-pushes only your own workspace ref, records the paths you changed, and surfaces **overlap warnings** (another active workspace touched the same paths — coordinate via `activity`/`workspace list`, expect merge attention at land time) and **`trunkOverlap`** (landed work you haven't merged — merge trunk into your line before landing). The signal is only as fresh as everyone's sync cadence — an agent who edits but hasn't synced is invisible. Inferred from the `ws/<ulid>` branch; `-w <id>` targets a workspace explicitly but must run from a checkout that **holds the workspace (its `ws/<ulid>` branch) or builds on its tip** — a fresh unrelated checkout is refused.
- `list [--all]` shows every workspace (`↑` commits ahead of base, `↓` trunk commits since base = *staleness*, so it stays nonzero even after you merge trunk); works from **anywhere** with `-a`. `status` (from the worktree) gives exact local truth: HEAD synced?, dirty files, which trunk changes overlap yours, linked checkpoints.
- `land [--into <branch>]` merges into trunk — fast-forward or a real merge commit (both histories retained) — pushes, and marks the workspace `landed`. Anyone with app access may land any workspace. On a merge **conflict** it exits **2** (`--json`: `ok:false, actionRequired:true, conflict:true, next:"deepspace workspace land"`), leaving the conflict in your worktree and your original line safe on the workspace ref: resolve the files, `git add`, `git commit`, **`validate` the merged tree** (a marker-free merge can still be semantically wrong), then run `land` again. (`next` is deliberately just the re-entry command, not a chained `git add`/`git commit` — running that before resolving would stage and commit the markers, and a resumed land pushes straight to trunk; bare `land` refuses on the still-dirty worktree until you've truly resolved.) When trunk has moved, merging `refs/remotes/space/<trunk>`, validating, then landing is **always correct** (land stops on the moved tip if it moves again — just repeat); landing directly is also safe when `trunkOverlap` is empty (land merges for you).
- `drop [<id>] [--cleanup]` abandons a workspace (its creator, the owner, or an admin). Deleting an active workspace's ref through raw git is refused — drop is the intended path.
- `attach <id> [dir]` resumes a workspace elsewhere: inside a checkout of the app it adds a worktree; in a fresh sandbox (empty dir, pass `-a <app>`) it clones just that workspace.
- Limits: at most **32 active** workspaces per app — land or drop finished ones. Landed refs are retained 14 days, dropped ones 7 (the records persist).
- Etiquette: check `workspace list` + `activity` before starting (don't duplicate work), prefer tasks that touch different files, commit+sync often, land promptly, and while others are active let trunk move via lands rather than direct pushes.

## Handoffs: transferring work between agents

A handoff packages **unfinished work plus the context to continue it**, for exactly one other agent/session to claim. It snapshots the whole tree (checkpoint mechanics — uncommitted and untracked files included) together with structured context.

```bash
npx deepspace handoff save -m "wire RBAC into admin routes" --next "make admin.test.ts role check pass" \
  [--context-file ctx.json]        # {objective, nextAction, state?, unresolved?[]} — -m/--next fill objective/nextAction
npx deepspace handoff list [--all]   # open handoffs (--all includes taken)
npx deepspace handoff show hnd_…     # full context
npx deepspace handoff take hnd_…     # CLAIM (exclusive) + restore onto branch handoff/<ulid> (--no-restore: claim only, worktree untouched)
npx deepspace handoff drop hnd_…     # give an UNFINISHED claim back (re-opens it)
npx deepspace handoff delete hnd_…   # done or obsolete: delete outright
```

- `save` requires an objective and a **concrete, machine-independent** next action (the taker's checkout has different absolute paths). `filesChanged` and the latest `validate` result for the snapshot are auto-linked.
- `take` **claims first**: a second taker is refused and told who holds it; re-running as the same user replays safely (crash-safe). Restore needs a clean worktree — commit or checkpoint your own work first (the claim already succeeded). It **materializes the snapshot as a real commit** on `handoff/<ulid>`, even if captured uncommitted; until you take, that commit exists only server-side (probing its oid with plain git fails — expected, not corruption). Review `git status`/the diff before landing — the snapshot carries the creator's **untracked** scratch files; drop planning notes that shouldn't ship. If the linked workspace already **landed/dropped**, `take`/`show` warn (`--json: staleLinkedWorkspace`) — check trunk before building on the snapshot.
- **Finishing a handoff linked to a workspace:** complete the work on `handoff/<ulid>`, then land it *as that workspace* — `npx deepspace workspace land -w <ws_id>` **from that branch** publishes your HEAD to the workspace's ref and merges it into trunk, closing the workspace as `landed` (the branch builds on the workspace tip, which is exactly what `-w` requires). Then `handoff delete hnd_…`.
- `delete` is the completion path: the current **taker** deletes a taken handoff directly (no drop/re-advertise dance). When open, its creator/the owner deletes it as obsolete.
- **The `chk_`/`hnd_` boundary is enforced.** Checkpoints and handoffs share one snapshot store, but each verb accepts only its own id kind — a wrong-kind id (`checkpoint restore hnd_…`, `handoff show chk_…`) is refused with `code:"wrong_kind"`. This is a claim-safety property: **never materialize a handoff with `checkpoint restore` — that restores the snapshot without the exclusive claim.** To work a handoff, `handoff take` it (which claims first).

## Validate: recorded check results

```bash
npx deepspace validate                       # runs the package.json "validate" script
npx deepspace validate -c "npx vitest run"   # or any explicit command
```

Runs the project's check and **records the result against the current commit** (command, pass/fail, duration, output tail) — visible in `activity`, auto-linked into handoffs. Exit 0 = passed, exit 1 = ran and failed. In `--json` key on **`passed`** (`ok` only means the result was recorded); failures include `outputTail`. It runs on the **working tree** — with uncommitted changes the record carries `dirty:true`. Advisory by design (`land`/`deploy` don't gate on it); skip for docs-only changes, record it for anything with code. It's how one agent knows another's synced tip was actually validated, without re-running the suite.

## Activity: the app's coordination feed

```bash
npx deepspace activity [--limit <n>]        # newest events
npx deepspace activity --since <cursor>     # only what's new (cursor from the previous page)
npx deepspace activity --follow             # keep polling until Ctrl-C
```

One append-only feed per app — workspaces created/synced/landed/dropped, branch pushes, checkpoints and handoffs, validations, releases — each with actor and time. It's how parallel agents stay aware without a chat channel: check it before starting and when an overlap warning appears. A **land** reads as a trunk `push` immediately followed by `workspace.landed` with the same oid (human output labels it `(land of ws_…)`), not a direct push. `--json` emits **exactly one page per invocation** — parse it, then call again with `--since <cursor>` while `hasMore` is true (`--follow --json` streams one line per page).

## Releases: deploy history and rollback

Every `deploy` records an immutable **release**: actor, timestamp, the source commit (or checkpoint), and the exact built bundle — retained server-side.

```bash
npx deepspace releases [--json]     # history: #seq, id, deploy|rollback, actor, source commit
npx deepspace rollback              # restore the previous release
npx deepspace rollback rel_…        # restore a specific one
```

- `rollback` re-ships the stored bundle — **no rebuild, no checkout, secrets untouched** — and itself becomes the newest release (so a rollback can be rolled back). With only ONE release there is no previous to restore — undo a bad first release with `deepspace undeploy` (owner-only).
- Rollback refused with `do_class_deletion`: the target release declares fewer Durable Object classes than what's live, and proceeding would **delete those classes' stored data**. Pass `--allow-do-deletion` (owner/admin only) only after confirming the data loss is acceptable.

**What deploy does automatically** — a release ships **the tree of the checkout you deploy from** (other agents' unlanded workspace commits are never included; your OWN uncommitted changes DO ship — deploy auto-checkpoints the dirty tree it builds):

- **Auto-push** the current branch first so the source is recoverable (retrying a 429). One hard stop: if the push fails because your branch is strictly **behind** the cloud repo (every local commit already landed), deploy **stops** with `code:"behind_trunk"` — `npx deepspace pull`, then redeploy (`--ignore-stale` ships the older tree deliberately). Any other push failure only warns and the deploy proceeds with the source **not** recoverable — `deploy --json` reports `recoverable:false`; then `npx deepspace push` (integrate first if rejected) and redeploy. `--no-push` skips the sync but still needs the deployed commit in the cloud repo (a never-pushed commit 409s until you push, or `--ignore-stale`).
- **Auto-checkpoint on dirty trees:** deploying with uncommitted changes checkpoints the exact deployed tree as the release's source — what's live is always recoverable even if you never committed.
- **Stale-base guard:** deploy verifies the code you're shipping *contains* the live release's commit (an ancestry check). It fails with `stale_base` — naming who released what, when — when your history doesn't include the live release, or the deployed commit was never synced. Either way: `npx deepspace pull`, integrate, redeploy (`--ignore-stale` overrides when clobbering is intended). If the repo store was unreachable the deploy proceeds unchecked and `--json` carries `staleBaseGuard:"skipped"` — treat that deploy as unverified.

## Recovery playbook

| Situation | Do |
|---|---|
| Bad deploy live in prod | `npx deepspace rollback` — instant, no rebuild |
| Session/sandbox died mid-task | `npx deepspace checkpoint list` → `restore` the latest |
| Teammate/agent needs the code | owner adds them (`collaborators add`), they `npx deepspace clone <app>` |
| Push rejected (`non_fast_forward`/`ref_conflict`) | `npx deepspace pull`, integrate (merge if diverged), push again |
| Deploy rejected (`stale_base`) | `pull`, integrate, redeploy — on a non-trunk branch integrate trunk explicitly: `git fetch space && git merge refs/remotes/space/main` |
| Deploy refused (`behind_trunk`) | `npx deepspace pull`, then redeploy — your checkout was behind the already-landed main line |
| `deploy --json` returned `recoverable:false` | `npx deepspace push` (integrate first if rejected), then redeploy |
| About to do something risky | `npx deepspace checkpoint save -m "before <risky thing>"` |
| Need history/diff/blame of the cloud repo | plain git on the `space` remote — **always `git fetch space` first** (the tracking ref is only as fresh as your last fetch) |
| Working alongside other agents on one app | `npx deepspace workspace new -t "…"` → work in the printed dir → `sync` often → `land --cleanup` |
| `workspace land` exited 2 (merge conflict) | resolve the conflicted files, `git add` + `git commit`, `validate` the merged tree (clean text merge ≠ working code), run `land` again |
| Stopping mid-task; someone else will continue | `npx deepspace handoff save -m "…" --next "…"` — the taker runs `handoff take hnd_…` |
| A collaborator left / sandbox died with work in flight | their **synced** workspace: anyone attaches and lands it; their **taken** handoff: `handoff drop hnd_…` reopens it. Un-synced local work is gone — sync/checkpoint often |
| Finished a handoff you took | land it (`workspace land -w <ws_id>` when linked), then `npx deepspace handoff delete hnd_…` |
| What are the other agents doing right now? | `npx deepspace activity` / `npx deepspace workspace list` (both work anywhere with `-a`) |
