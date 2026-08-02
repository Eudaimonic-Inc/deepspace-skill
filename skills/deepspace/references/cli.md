_Load for command discovery, authentication, output/exit semantics, logs, or test entry points. Domain-specific behavior belongs in its matching reference._

# CLI contract

## Discover, do not memorize

The command tree is intentionally static, but it evolves. Use help instead of a copied inventory:

```bash
npx deepspace --help
npx deepspace <group> --help
npx deepspace <group> <command> --help
```

Important nested entry points are `dev start|kill`, `test run|screenshot|accounts`, `auth login|logout|whoami`, `app …`, and `integrations list|info|invoke`. Unknown commands fail with a hint; use only paths present in help.

## Authentication

Let a network/account/app operation report `not_authenticated`; local inspection and help do not all require login. Then run:

```bash
npx deepspace auth login
npx deepspace auth whoami
```

Login opens browser OAuth and polls for completion. Keep it in the foreground, let the user complete it, and never wrap it in a timeout or ask for a password. Sessions are stored per auth plane and refresh automatically; do not inspect or copy session files.

## Output and exits

Human text is complete; `--json` is the machine mirror. Refusals carry a stable `code`.

| Exit | Meaning |
|---|---|
| `0` | Operation completed. |
| `1` | Failure/refusal; retrying unchanged will not help. |
| `2` | Safe partial progress or stop with `actionRequired: true`; local work or judgment remains. The operation may have succeeded (`ok:true`) or refused after a safe partial mutation (`ok:false`). |

When exactly one deterministic follow-up exists, JSON includes:

```json
{"action":{"cwd":"/absolute/app","argv":["deepspace","pull"]}}
```

Human output renders the same action as `Next:`. Run `argv` directly in `cwd`; do not join it into a shell string. Terminal results, status reports, consent, destructive overrides, and input-dependent choices omit the field. An exit-2 result may therefore require inspecting its facts rather than executing a supplied command; do not infer an action from absence.

One-shot `--json` writes one document, except commands that inherit child output. `logs --follow --json` and `activity --follow --json` are NDJSON streams; parse one frame per line.

## Local development and tests

```bash
npx deepspace dev start [--port N]
npx deepspace dev kill [--port N]
npx deepspace test run [smoke|api|e2e|unit|all|<file>] [--port N]
npx deepspace test accounts list
npx deepspace test screenshot <url> <output>
```

`dev start`, `test run`, and screenshot capture inherit child output, so under `--json` their final envelope follows the stream. Screenshot is a small visual-inspection helper, not a substitute for assertions. Testing patterns and account-pool rules live in `testing.md`.

Use `npx deepspace logs [--follow]` for deployed worker logs. Follow mode is a stream and may emit a metadata frame when output is truncated.
