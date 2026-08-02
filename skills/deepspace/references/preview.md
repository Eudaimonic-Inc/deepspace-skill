_Load when the desktop preview serves stale code or when developing inside `.claude/worktrees/<name>`._

# Desktop preview

DeepSpace keeps `.claude/launch.json` aligned with the dev server. A normal app entry runs the current command tree:

```json
{
  "name": "<app>",
  "runtimeExecutable": "npx",
  "runtimeArgs": ["deepspace", "dev", "start", "--port", "5173"],
  "port": 5173
}
```

The file is machine-local. Keep `.claude/launch.json` and `.claude/worktrees` ignored; do not commit absolute worktree paths.

## Worktrees

The desktop preview reads the primary checkout's launch file, not a nested worktree's. From the worktree, run once:

```bash
npx deepspace dev start
```

The CLI adds a `wt-<name>` entry to the primary launch file with the worktree `cwd` and a stable port in 5180–5199, prints the entry name, and removes entries for deleted worktrees. Start preview with that printed name. `test run` and `dev kill` resolve the same port; pass `--port` to override it.

If preview still looks stale:

1. Check the server `cwd` and port with the preview listing.
2. Add a distinctive source string and request that source through the dev server; absence proves the wrong checkout is running.
3. Stop the mismatched server and start the printed `wt-<name>` entry. Do not kill an unrelated process on the same port.
