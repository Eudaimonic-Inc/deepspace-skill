_Load when the desktop preview serves stale code or when developing inside any linked Git worktree._

# Desktop preview

DeepSpace detects linked checkouts from Git metadata, not agent-specific directory names. Without an explicit `--port` or `$DEEPSPACE_PORT`, every Codex, Claude, or ordinary linked worktree gets a deterministic port in 5180–6179 from its canonical checkout path. `dev start`, `test run`, and `dev kill` resolve the same port; the primary checkout keeps the normal 5173 default.

Claude desktop preview additionally reads `.claude/launch.json`. A normal app entry runs the current command tree:

```json
{
  "name": "<app>",
  "runtimeExecutable": "npx",
  "runtimeArgs": ["deepspace", "dev", "start", "--port", "5173"],
  "port": 5173
}
```

The file is machine-local. Keep `.claude/launch.json` and `.claude/worktrees` ignored; do not commit absolute worktree paths.

## Claude desktop adapter

The Claude desktop preview reads the owning checkout's launch file, not a nested worktree's. DeepSpace treats `<owner>/.claude/worktrees/<name>` as this adapter only when Git reports both the owner and child as registered checkouts of the same repository; a matching path string alone has no effect. From the worktree, run once:

```bash
npx deepspace dev start
```

The CLI adds a `wt-<name>` entry to the owning launch file with the exact worktree `cwd` and resolved port, prints the entry name, and prunes only stale `wt-*` entries whose absolute `cwd` is under that owner's `.claude/worktrees` directory. Start Claude preview with the printed name. Codex and ordinary Git worktrees need no worktree-specific adapter; a scaffold may still include the machine-local launch file for Claude interoperability.

If preview still looks stale:

1. Check the server `cwd` and port with the preview listing.
2. Add a distinctive source string and request that source through the dev server; absence proves the wrong checkout is running.
3. Stop the mismatched server and start the printed `wt-<name>` entry. Do not kill an unrelated process on the same port.
