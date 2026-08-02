_Load only when the user asks for GitHub, another hosted remote, pull requests, or CI._

# GitHub alongside DeepSpace

DeepSpace and GitHub share one local history with separate remotes:

- `space` is the app's DeepSpace repository and source of deploy lineage, workspaces, releases, and activity.
- `origin` can be GitHub, GitLab, or another host for review and CI.

Do not create or replace `origin` unprompted. When requested, use ordinary Git:

```bash
git remote add origin git@github.com:<org>/<repo>.git
git push -u origin main
```

For workspace review, publish exactly the checked-out commit under an explicit review branch:

```bash
git remote get-url origin
git push -u origin HEAD:refs/heads/<review-branch>
```

Never use `git push --mirror`; it acts on every ref, including refs that do not belong on GitHub.

Keep both trunks synchronized when both are authoritative to the team. A commit present only on `origin` is invisible to DeepSpace deploy guards; a commit present only on `space` is absent from GitHub review.

Workspace coordination refs live on `space`. Continue using `workspace sync` for DeepSpace coordination even when its current HEAD also has a GitHub review branch.

For CI deploys, inspect current authentication and deploy flags with `npx deepspace auth login --help` and `npx deepspace deploy --help`. Store credentials in the CI secret manager; never place passwords or tokens in argv, repository files, or workflow logs.
