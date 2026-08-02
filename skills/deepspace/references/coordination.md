_Load when resuming after context loss or inspecting status/activity. Repository operations use `version-control.md`; output semantics use `cli.md`._

# Coordination

## Status

```bash
npx deepspace status
npx deepspace status --json
```

`status` reports present facts: session, app, install, branch/workspace, sync relation, and live release where reachable. It does not derive a workflow, consume activity, or print a synthetic next step. Interpret the facts, then run the relevant command.

## Activity

Activity is stateless; the caller owns the cursor.

```bash
npx deepspace activity                         # events after cursor 0
npx deepspace activity --since <cursor>        # resume strictly after cursor
npx deepspace activity --follow                # tail from now
npx deepspace activity --follow --since <cursor>
```

Persist the last returned cursor if continuity matters. Omitting `--since` in one-shot mode replays from the beginning; omitting it in follow mode starts at the current tail.

`activity --follow --json` emits NDJSON: one `ready` frame, then `activity` frames. Recoverable polling failures emit `transport` frames without advancing the cursor. Events are facts—pushes, workspace lifecycle, and releases—not instructions.
