_Load this reference when: syncing app code without GitHub (`deepspace push`/`pull`/`clone`, or plain `git push space`), saving or restoring work-in-progress (`deepspace checkpoint`), viewing or undoing a deploy (`deepspace releases`/`rollback`, or a `stale_base` failure), sharing an app with a collaborator, running several agents on one app at once (`deepspace workspace`/`activity`), or transferring work between agents (`deepspace handoff`)._

# Version control: cloud repo, checkpoints, releases

Every DeepSpace app has a **cloud repo** on the platform — no GitHub, and no Git knowledge required. Agents drive it with `deepspace` verbs: `push`/`pull`/`clone` to sync, `checkpoint` to save work-in-progress, `releases`/`rollback` to view and undo deploys. Every verb takes `--json`. Under the hood it's a real git remote named `space`, so after any wrapper has run once, plain `git push space main` / `git fetch space` work in any terminal too — but you never have to touch git directly.

Read-only verbs (`releases`, `activity`, `workspace list`, `checkpoint`/`handoff` `list`/`show`) run from anywhere with `-a <app>`; working-tree verbs (`push`/`pull`, `workspace sync`/`land`, `checkpoint save`) run inside a checkout.

## Sync: push / pull / clone

```bash
npx deepspace push                     # your branch → cloud repo (add --force after an intentional history rewrite)
npx deepspace pull                     # fetch + fast-forward; on diverged histories, stops with the exact merge command
npx deepspace clone <app|name> [dir]   # get an app's repo on a new machine
```

- `-a <app>` targets an app explicitly; the default is the app directory you're in.
- `push` is fast-forward-only. If it's rejected because someone pushed first, run `deepspace pull`, then push again.
- `clone` is the collaborator on-ramp — deploy-ready (the app id is already in history, and `clone` wires up the `space` remote). Follow with `npm install`, `npx deepspace secrets pull`, `npx deepspace dev`. The owner adds collaborators first (`references/collaborators.md`).
- An object over the ~20 MB cap is rejected server-side, and the CLI **names the file** to `.gitignore` (or move to Git LFS) and re-commit. A very large multi-branch repo may need cloning one branch at a time (`git clone --single-branch --branch <b> <url>`). Requires git ≥ 2.29; shallow/partial clones aren't supported.

## Checkpoints: durable work-in-progress

```bash
npx deepspace checkpoint save -m "auth half-wired; RBAC tests failing" [--context-file notes.json]
npx deepspace checkpoint list | show chk_… | restore chk_… | delete chk_…
```

A checkpoint snapshots your **whole working tree** — uncommitted and untracked files included (`.gitignore`d files are excluded, so don't rely on it to preserve a gitignored local artifact like a dev database; secrets such as `.env*`/`.dev.vars*`/`.mcp.json` are stripped either way, but templates like `.env.example` are kept) — into the cloud repo, parented on HEAD. It survives the sandbox/session dying, restores on any machine with app access, and touches nothing on your branch, index, or worktree.

- `restore chk_…` checks the snapshot out on a fresh `checkpoint/…` branch (the CLI prints its exact name); `--in-place` overwrites the current worktree instead (needs `--force` once HEAD has moved past the snapshot).
- Use one **before risky passes** (big refactors, migrations) and **before pausing a session** — `-m` plus `--context-file` is how the next session resumes without re-deriving state.
- To hand work to a **different** agent, use `handoff` (it adds an exclusive claim), not a checkpoint. A secret committed in history blocks the snapshot — untrack it and retry.

## Releases: deploy history & rollback

```bash
npx deepspace releases          # history: seq, id, deploy|rollback, actor, source commit
npx deepspace rollback          # undo the last deploy — re-ships the previous release instantly
npx deepspace rollback rel_…    # roll back to a specific release
```

Every `deploy` records an immutable **release**: actor, time, the source commit, and the exact built bundle.

- `rollback` re-ships the stored bundle — **no rebuild, secrets untouched** — and is itself a new release, so it's reversible. With only one release ever, undo it with `deepspace undeploy` (owner only). A rollback that would delete a Durable Object class's stored data is refused unless you pass `--allow-do-deletion` (owner/admin).
- **`deploy` handles the git for you:** it auto-pushes your branch so the source stays recoverable, auto-checkpoints an uncommitted tree (what's live is always restorable), and **refuses rather than clobber a newer release** — `stale_base`/`behind_trunk` both mean `deepspace pull`, then redeploy. If the source didn't sync, `deploy --json` reports `recoverable:false`; run `deepspace push`, then redeploy.

## Parallel agents: workspaces & handoffs

Skip this section for solo work. When several agents (or a human plus agents) work one app at once, each takes a **workspace** — a parallel line of work that publishes continuously and **lands** (merges) into trunk when done — instead of committing to the shared branch. A **handoff** transfers unfinished work, with an exclusive claim, to one other agent.

### Workspaces

```bash
npx deepspace workspace new -t "add auth pages"     # prints a worktree dir — do ALL work there
npx deepspace workspace sync                         # publish your commits (repeat as you go)
npx deepspace workspace land --validate --cleanup    # merge trunk in, validate, land to trunk, remove the worktree
npx deepspace workspace list | status | attach <id> | drop
```

- `cd` into the printed dir — it's a separate path from your main checkout, so every file you touch must live under it. No `npm install`: it resolves the app's `node_modules` by walking up; invoke tools with `npx`.
- `sync` publishes **committed** work only (commit first, or `checkpoint save` for a WIP snapshot) and warns on **overlap** (another active workspace touched your files — coordinate via `activity`) and **trunkOverlap** (landed work you haven't merged). If another checkout of the *same* workspace advanced it, `sync`/`land` refuse (`diverged`, exit 2) rather than lose that work — `git merge` it in, then retry.
- `land` merges the latest trunk in **automatically**, then pushes; on a conflict it stops (exit 2) with your original line safe on the workspace ref — resolve, commit, re-run. `--validate` gates on the merged tree; `--cleanup` removes the finished worktree.
- Ordinary `push`/`pull` refuse on a `ws/` branch (exit 2, `workspace_branch`) — use `workspace sync`/`status` inside a workspace.
- Check `workspace list` + `activity` before starting (don't duplicate work); commit + sync often; land promptly. Limit: 32 active workspaces per app.

### Handoffs

```bash
npx deepspace handoff save -m "wire RBAC into admin routes" --next "make admin.test.ts pass" [--context-file ctx.json]
npx deepspace handoff take hnd_…       # claim it (exclusive) + restore the work onto a branch
npx deepspace handoff complete hnd_…   # finish it: land the work into trunk + close the handoff, in one step
npx deepspace handoff list | show hnd_… | drop hnd_… | delete hnd_…
```

- A handoff packages unfinished work plus the context to continue it, for exactly one other agent to `take` (a second taker is refused and told who holds it). `save` needs an objective (`-m`) and a concrete, machine-independent next action (`--next`).
- `take` claims first (crash-safe), then restores the snapshot onto a fresh `handoff/…` branch (its exact name is printed) — review the diff before continuing (it carries the author's untracked scratch files too; drop notes that shouldn't ship).
- `complete` finishes a workspace-linked handoff — run it **from that handoff branch, with your work committed**: it lands the work into trunk and deletes the record. (For a handoff not linked to a workspace, merge to trunk yourself, then `handoff delete`.)
- Never `checkpoint restore` a handoff — that materializes the snapshot **without** the exclusive claim. Each verb accepts only its own id kind.

### Validate & activity

```bash
npx deepspace validate            # run the project's check; record pass/fail against the commit
npx deepspace activity [--follow]  # the app's shared event feed — who did what, when
```

- `validate` records a result (command, pass/fail, output tail) visible in `activity` and auto-linked into handoffs — advisory; `land`/`deploy` don't gate on it. In `--json`, key on `passed`.
- `activity` is how parallel agents stay aware without a chat channel — check it before starting and when an overlap warning appears. `--json` returns one page; page with `--since <cursor>` while `hasMore` is true.

## Recovery playbook

| Situation | Do |
|---|---|
| Bad deploy live in prod | `npx deepspace rollback` — instant, no rebuild |
| Session/sandbox died mid-task | `npx deepspace checkpoint list` → `restore` the latest |
| Push rejected (someone pushed first) | `npx deepspace pull`, integrate, push again |
| Deploy refused (`stale_base`/`behind_trunk`) | `npx deepspace pull`, then redeploy |
| `deploy --json` said `recoverable:false` | `npx deepspace push` (integrate first if rejected), then redeploy |
| About to do something risky | `npx deepspace checkpoint save -m "before <risky thing>"` |
| A push named an oversized file | remove it from history (`git rm --cached <path>` + `.gitignore`) or use Git LFS, then re-commit |
| Teammate/agent needs the code | owner runs `collaborators add`; they `npx deepspace clone <app>` |
| Need cloud-repo history/diff/blame | plain git on the `space` remote — **`git fetch space` first** (the tracking ref is only as fresh as your last fetch) |
| Working alongside other agents | `npx deepspace workspace new -t "…"` → work in the printed dir → `sync` often → `land --validate --cleanup` |
| `workspace land` exited 2 (conflict) | resolve the files, `git add` + commit, then re-run `land` (validate the merged tree first) |
| Someone else will continue your work | `npx deepspace handoff save -m "…" --next "…"` — they `handoff take`, then `handoff complete` when done |
| What are the other agents doing? | `npx deepspace activity` / `workspace list` (both work anywhere with `-a`) |

**Errors are machine-readable.** Every refusal returns `--json {ok:false, code, error}`; when you must act it adds `next` (the exact command to run) and exits **2** (vs 1 for a plain failure). Every input error — a blank or malformed `--app`, a bad id, an invalid branch name — is emitted BEFORE any login or network, so it's deterministic regardless of auth state. Branch on `code`, never the prose.
