_Load this reference when: syncing code between machines or collaborators without GitHub (`deepspace push`/`pull`/`clone`, or plain `git push space`), saving or restoring work-in-progress durably (`deepspace checkpoint`), viewing deploy history or undoing a bad deploy (`deepspace releases`, `deepspace rollback`), a deploy fails with `stale_base`, or a teammate/agent needs the code for an app they collaborate on._

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

- Every VC command takes `--json` (parse, don't scrape). `push`/`pull`/`checkpoint`/`releases`/`rollback` take `-a <app id or subdomain name>` to target an app explicitly; `clone`'s app is its positional argument.
- `push` is fast-forward-only, like git. A `non_fast_forward` or `ref_conflict` result means someone pushed since you last synced: run `npx deepspace pull`, integrate, push again.
- `pull` never touches a dirty worktree. If histories diverged it fetches the branch's history and tells you to run `git merge refs/remotes/space/<branch>` — that merge is fully local, then push the result. Exit code 2 = fetched but action still required.
- `clone` is the collaborator on-ramp: the clone is deploy-ready because `wrangler.toml` (with the app id) is part of the history, and the `space` remote comes pre-configured. Follow with `npm install`, `npx deepspace secrets pull`, `npx deepspace dev`. Collaborators must be added by the owner first (`references/collaborators.md`).
- First `push` works before the first deploy — it registers the app id to your account, exactly like a first deploy would.
- Limits worth knowing: git ≥ 2.29, sha1 repos only (git's default), no shallow or partial clones (`--depth`/`--filter` are rejected), a single push is capped at ~32 MB of packed history, and any single object over ~20 MB is rejected. The rejection is server-side (`ng … unpacker error`) and does NOT name the file — find the offender with `git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' | sort -k2 -n | tail`, then `.gitignore` it (or use Git LFS / drop it from history).

## Checkpoints: durable work-in-progress

A checkpoint snapshots the **entire working tree — uncommitted and untracked files included** (`.gitignore`d files excluded; `.dev.vars*` never captured, gitignored or not) — parents it on HEAD, and stores it in the cloud repo with structured context. Nothing touches your branch, index, or worktree. It survives the sandbox/session dying and restores on any machine with access to the app.

```bash
npx deepspace checkpoint save -m "auth flow half-wired; RBAC tests failing" \
  [--context-file notes.json]      # attach structured state (task, unresolved issues, next action)
npx deepspace checkpoint list
npx deepspace checkpoint show chk_…      # full context JSON
npx deepspace checkpoint restore chk_…   # new branch checkpoint/<ulid> at the snapshot (needs clean tree)
npx deepspace checkpoint restore chk_… --as-branch <name>     # pick the branch name yourself
npx deepspace checkpoint restore chk_… --in-place [--force]   # overwrite the worktree instead;
                                                              # --force DISCARDS current uncommitted changes
npx deepspace checkpoint delete chk_…
```

Use them **before risky passes** (big refactors, redesigns, destructive migrations) and **at handoff points** — `-m` plus `--context-file` is how the next session/agent learns the objective, what's done, and what's broken without re-deriving it. A clean tree checkpoints as HEAD itself (cheap pointer, still restorable anywhere). Under the hood a checkpoint is a hidden ref (`refs/deepspace/checkpoints/…`) pushed over the normal git protocol — it never appears on your branches.

## Releases: deploy history and rollback

Every `deploy` records an immutable **release**: actor, timestamp, the commit (or checkpoint) it was built from, and the exact built bundle — retained server-side.

```bash
npx deepspace releases [--json]     # history: #seq, id, deploy|rollback, actor, source commit
npx deepspace rollback              # restore the previous release
npx deepspace rollback rel_…        # restore a specific one
```

- `rollback` re-ships the stored bundle — **no rebuild, no checkout needed**, and secrets are untouched. The rollback itself becomes the newest release (history is append-only), so a rollback can be rolled back.
- If rollback refuses with `do_class_deletion`: the target release declares fewer Durable Object classes than what's live, and proceeding would **delete those classes' stored data**. Only pass `--allow-do-deletion` (owner or admin only — a collaborator is refused) after confirming with the user that the data loss is acceptable.

### What deploy does automatically

- **Auto-push:** `deploy` pushes the current branch to the cloud repo first, so every release's source is recoverable. `--no-push` skips the sync (a push hiccup itself only warns), but the stale-base guard below still applies — and it needs the deployed commit to exist in the cloud repo, so `deploy --no-push` of a never-pushed commit on an app with prior releases will 409 until you push (or say `--ignore-stale`, accepting that the release's source commit won't be recoverable from the platform).
- **Auto-checkpoint on dirty trees:** deploying with uncommitted changes creates a checkpoint of the exact deployed tree and records it as the release's source — what's live is always recoverable even if you never committed.
- **Stale-base guard:** `deploy` verifies the code you're shipping *contains* the live release's commit (an ancestry check against the cloud repo). It fails with `stale_base` — naming who released what and when — in two cases: your history doesn't include the live release (someone released since you last synced), or the commit you're deploying was never synced at all (e.g. the auto-push hit a conflict). Either way: `npx deepspace pull`, integrate, redeploy. `--ignore-stale` overrides when clobbering is intended — two agents on one app should sync before deploying rather than lean on the override.

## Recovery playbook

| Situation | Do |
|---|---|
| Bad deploy live in prod | `npx deepspace rollback` — instant, no rebuild |
| Session/sandbox died mid-task | `npx deepspace checkpoint list` → `restore` the latest |
| Teammate/agent needs the code | owner adds them (`collaborators add`), they `npx deepspace clone <app>` |
| Push rejected (`non_fast_forward`/`ref_conflict`) | `npx deepspace pull`, integrate (merge if diverged), push again |
| Deploy rejected (`stale_base`) | same as above, then redeploy |
| About to do something risky | `npx deepspace checkpoint save -m "before <risky thing>"` |
| Need history/diff/blame of the cloud repo | plain git against the `space` remote (`git fetch space`, `git log space/main`) |
