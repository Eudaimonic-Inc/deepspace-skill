_Load this reference when adding teammates to an app, accepting a collaborator invite, deploying an app you don't own, or debugging a 403 on deploy/secrets as a non-owner._

# App collaborators

Collaborators are DeepSpace users the app **owner** authorizes to work on the app. Ownership never moves: the deployed worker keeps the owner's identity, billing, and `APP_OWNER_JWT`. Authorization keys to the app's immutable `DEEPSPACE_APP_ID` (→ [references/app-identity.md](app-identity.md)) — there is no per-resource grant or link step.

> **Not the same thing:** record-level collaborators (`collaboratorsField` in a schema, the `'shared'` read rule) control who sees *rows inside* an app → `references/schemas.md`. This file is about who can *ship* the app.

## Managing (owner only)

```bash
npx deepspace app collaborators list                         # COLLABORATORS + PENDING INVITES (with expiry); --json → {collaborators, pending}
npx deepspace app collaborators add teammate@example.com     # existing user → added now; non-user → emailed invite (see below)
npx deepspace app collaborators cancel teammate@example.com  # rescind a pending (un-accepted) invite
npx deepspace app collaborators remove teammate@example.com
```

Run from the app checkout (or pass `-a`/`--app <id or name>`). Test accounts (`…@deepspace.test`) can never be collaborators — grants to them fail closed. Collaborators get owner-equivalent deploy and secrets access, so only add people you trust.

**`add` has two paths.** If the email already belongs to a DeepSpace user, they're added as a collaborator immediately. If it doesn't, the server creates a **pending invite**, emails the person, and bills the invite to the owner. The invite has an expiry date; re-running `add` while an invite is still live returns `already_invited` (no new email, no re-charge) — `cancel <email>` then re-`add` to reset. `list` shows live invites under a `PENDING INVITES ON <app>` section.

## Accepting an invite (invitee side)

Two equivalent paths; authority is the signed-in account's verified email matching the invited address — signing in elsewhere does not grant access:

```bash
npx deepspace app collaborators invites            # pending invites for my account
npx deepspace app collaborators accept <app-id>    # accept; follow-up action: deepspace clone <app-id>
```

or open the emailed `/join/<token>` page and accept there. The CLI path is the
one an agent uses — no browser, no token handling (tokens never travel through
`invites`/`accept`).

## What a collaborator can and can't do

| Action | Allowed? |
|---|---|
| `deploy` (incl. `--env`) | **Yes** — on-behalf; CLI prints `Deployed on behalf of owner <id>`; billing stays the owner's |
| `dev` / `test` | **Yes** |
| Secrets: `list` / `get` / `download` / `pull` | **Yes** — every config in the app's store |
| Secrets: `set` / `upload` / `delete`, `configs create` / `delete` | **Yes** — writes are audited under the collaborator's own id |
| `undeploy` | No — owner or platform admin only |
| `transfer offer` | No — owner only (→ `references/app-identity.md`) |
| `transfer status` / `cancel` / `accept` | Only when the collaborator is the named recipient; either party may cancel, and only the recipient may accept |
| `collaborators add` / `remove` / `cancel` | No — owner only |

Platform admins can deploy, manage secrets, and undeploy through their platform
override, but cannot manage collaborators, offer an ownership transfer, or
change source authority. A collaborator who is the named transfer recipient
participates as that recipient, not through the collaborator role. This does
not turn an ordinary app collaborator into an admin.

## Mechanics and traps

- **Access is the app role.** Every deploy/secrets request is authorized against the app id: owner, collaborator, or neither. A 403 `Not the app owner or a collaborator` means ask the owner for `collaborators add` — or your access was revoked.
- **On-behalf deploys keep the owner's identity.** A collaborator deploy ships code plus the store's secrets, tagged to the owner for billing; nothing about the app's ownership changes.
- **Getting started as a collaborator:** clone the repo (its `wrangler.toml` already carries `DEEPSPACE_APP_ID`), `npx deepspace auth login`, and `dev start` / `test run` / `deploy` / `secrets` just work — no linking. A DeepSpace-source app has no leased name before its first deploy, so get its id from `deepspace app list` and use `deepspace clone <app-id>` in that case; after deploy, its name also resolves. If you actually wanted your **own** copy of the app rather than to collaborate, run `npx deepspace app init --new-id` to fork it (fresh data, fresh secrets store).
