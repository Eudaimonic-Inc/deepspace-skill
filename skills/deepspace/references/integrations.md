_Load for external APIs called through `integration.post`. For Google OAuth or LiveKit, also load only the matching nested reference._

# Integrations

## Discover the contract

Endpoint keys are exactly `<integration>/<endpoint>`. Inspect the live catalog before writing a call; `list` and `info` are public catalog reads and do not require login.

```bash
npx deepspace integrations list [--json]
npx deepspace integrations info <integration>/<endpoint> [--json]
```

When the CLI cannot reach the catalog, search `assets/integrations/index.yaml`, then load only the matching `assets/integrations/<integration>.yaml`. The bundled YAML can lag the live catalog; the CLI wins on disagreement.

```ts
import { integration } from 'deepspace'

const result = await integration.post('openai/chat-completion', body)
if (!result.success) throw new Error(result.error)
const data = result.data
```

Do not guess response nesting: inspect the endpoint output schema. Validation failures may include `issues`; success may still contain an empty array or zero-valued upstream result.

## Billing and access

`src/integrations.ts` selects billing per integration:

- `developer` (default): the app owner pays and the proxy permits anonymous callers. Auth-gate and app-rate-limit every UI that can trigger a paid call.
- `user`: the signed-in JWT subject pays; anonymous calls fail. Use this for per-user OAuth such as Google and when users knowingly fund their own calls.

The caller cannot redirect billing with a header. Load `integrations/google-oauth.md` for Google consent, scope step-up, `requiresOAuth`, and test doubles; load `integrations/livekit.md` for rooms.

## Invoke and test deliberately

Direct CLI invocation is authenticated and billed to the logged-in user:

```bash
npx deepspace integrations invoke <integration>/<endpoint> --body '{"key":"value"}'
npx deepspace integrations invoke <integration>/<endpoint> --body-file request.json
```

Use `--yes` only after inspecting `info` and accepting the stated cost. Keep real integration tests to one call per needed contract—never loops, parameter matrices, or retry-until-success polling. Do not make real `user`-billed calls with test accounts that lack credits, and never flip billing modes just to make a test pass. Mock only the external integration boundary; keep app-internal hooks and routes real.

For UI, render loading, error with local retry, empty, and success states. Use `useAsyncResource` for one-shot calls or a bounded `usePagedResource` for feeds; a failed resource should not reload the whole page.
