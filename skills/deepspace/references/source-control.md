_Load when choosing, inspecting, or transferring an app between GitHub and
DeepSpace source._

# One source authority, two first-class choices

Inspect the server-enforced authority before deciding from remote names:

```bash
npx deepspace app source --json
```

Every app has exactly one authoritative Git repository.

| Source | Experience |
|---|---|
| DeepSpace | Packaged default. Deploy publishes automatically; `space`, clone/push/pull, workspaces, activity, releases, and rollback work together. |
| GitHub | Manual ownership. The developer pushes with ordinary Git; ordinary deploy ships the local working tree and performs no Git read, write, or verification. |

The presence of both `origin` and `space`, or retained Git objects on an
inactive provider, does not create two sources of truth. Server policy enables
only the selected provider's source writes. Releases/activity remain platform
facts for either source.

For GitHub source, dirty and unpushed bytes are valid deploy inputs. The release
keeps the configured repository/source revision but has no DeepSpace commit id;
rollback uses the retained deployment bundle. Do not add Git checks to this
path—the developer selected GitHub to own synchronization manually.

## Select or transfer

```bash
npx deepspace app source github [--remote <name>]
npx deepspace app source deepspace [--remote <name>]
```

`--remote` selects the local GitHub remote to verify or import; inspect command
help when more than one GitHub remote exists.

Use the command rather than rewriting remotes or registry fields. It inventories
branches/tags and deploy lineage, copies or asks the operator to publish missing
objects, verifies exact object ids, and flips authority once. A failed copy or
stale source revision leaves the old provider authoritative. Follow its one
structured action when present; never bypass the refusal with `--no-push`.

DeepSpace → GitHub remains manual because the developer owns GitHub: create or
select the intended GitHub remote, publish the requested refs, then rerun.
GitHub → DeepSpace is the packaged path: the command imports verified refs and
then enables the `space` remote/workspace surface. Transfers preserve Git
history reachable through branches and tags; they do not move app identity,
data, secrets, collaborators, or URLs. Git LFS objects, submodule repositories,
notes, replace refs, and host-specific metadata are not part of this transfer.

Only the owner or a platform admin changes authority. Collaborators deploy
according to the owner's current source but cannot flip it. Active DeepSpace
workspaces must be landed or dropped before moving to GitHub.
