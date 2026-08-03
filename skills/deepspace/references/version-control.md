_Load when cloning, pushing, pulling, or using a DeepSpace workspace. For status/activity use `coordination.md`; deploy/rollback use `releases.md`; GitHub remotes use `github.md`; output semantics use `cli.md`._

# Version control and workspaces

DeepSpace-source apps have a real platform Git repository. The platform remote
is named `space`; wrappers install or repair it and configure a helper for the
platform host (global for a durable CLI executable, repository-local for a
transient one). Afterward, ordinary commands such as `git fetch space` also
work. GitHub-source apps intentionally refuse these writes and workspaces; use
ordinary GitHub branches/worktrees instead. See `source-control.md` before
changing authority.

```bash
npx deepspace clone <app-or-id> [dir]
npx deepspace push [-b <branch>]
npx deepspace pull [-b <branch>]
```

`clone` creates a normal checkout. `push` and `pull` are guarded Git operations with stable refusal codes; they do not replace Git's history model.

## Workspaces

A workspace is a durable server ref plus task metadata, materialized locally as a `ws/<id>` branch and worktree. Use one for each independent line of work:

```bash
npx deepspace workspace new -t "wire RBAC into billing"
npx deepspace workspace attach ws_… [dir]
npx deepspace workspace sync
npx deepspace workspace status
npx deepspace workspace list [--all]
npx deepspace workspace land [--into <branch>] [--validate]
npx deepspace workspace drop [ws_…]
```

Rules:

- Commit WIP, then run `workspace sync` to make it durable and visible to collaborators.
- Run `workspace sync` and `workspace land` from the selected workspace checkout. `-w/--workspace` selects identity; it does not make another checkout safe.
- `land` makes an ordinary merge, preserving the workspace commits, then deletes the server workspace ref. Local worktree cleanup is the default; `--keep-worktree` opts out.
- Overlap reports compare live peer tips and are advisory. Read them and coordinate; they are not file ownership locks.
- Workspace worktrees share the primary checkout's dependency cache. Reinstall when their manifests diverge.
- Do not plain-push a `ws/*` branch to `space`; publish it with `workspace sync` so metadata and activity stay coherent.
- Deploying from a workspace requires its exact HEAD to be synced first. Follow the refusal's action rather than deploying an older workspace tip.

## Guardrails

- `pull` fetches and fast-forwards. If dirtiness, divergence, or another worktree prevents the local update, it stops safely and may return one local action.
- `push --force` still refuses to discard a remote commit the local branch does not contain. Integrate the remote tip instead of using force as conflict recovery.
- Push preflight rejects commonly committed secret files. Fix the branch; do not normalize bypass flags.
- Keep `DEEPSPACE_DEPLOY_URL` consistent for a session. It selects which platform owns the `space` remote, workspaces, releases, and activity.

For `non_fast_forward`, `behind_trunk`, `diverged`, `dirty_worktree`, or `workspace_unsynced`, inspect the stable code and execute `action.cwd` + `action.argv` when present. If no action is supplied, the recovery requires judgment; inspect Git state instead of inventing a command.
