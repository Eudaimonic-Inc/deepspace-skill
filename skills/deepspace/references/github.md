_Load only when the user asks for GitHub, another hosted remote, pull requests, or CI._

# GitHub source and review

An app may use GitHub as its one authoritative repository. In that mode:

- push branches and tags with ordinary Git;
- DeepSpace deploy verifies the configured repository and exact remote commit,
  but never pushes it;
- DeepSpace Git writes and workspaces refuse with a source-authority error; and
- releases and activity remain readable platform facts, not a second source
  repository.

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

Never use `git push --mirror`; it acts on every ref, including refs that do not
belong on GitHub. Use `npx deepspace app source github [--remote <name>]` to
claim or transfer authority, and follow its exact structured actions. Read
`source-control.md` before changing authority.

For DeepSpace-source apps, GitHub may still hold a review branch, but it is not
the deploy source. Do not describe both remotes as authoritative or maintain
two trunks as a product invariant.

For CI deploys, inspect current authentication and deploy flags with `npx deepspace auth login --help` and `npx deepspace deploy --help`. Store credentials in the CI secret manager; never place passwords or tokens in argv, repository files, or workflow logs.
