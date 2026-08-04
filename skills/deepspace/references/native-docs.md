_Load when creating, migrating, customizing, validating, or deploying a native DeepSpace documentation site; when editing `docs.json`, docs Markdown/MDX, or `docs.tsx`; or when debugging the docs subdomain, router, search, page actions, assistant, MCP, LLM artifacts, or OpenAPI pages._

# Native documentation

DeepSpace compiles documentation from the app repository and publishes it as a surface of the same app and immutable release. Keep `docs.json` as the sole configuration authority. Do not add a compatibility runtime, separate docs app, duplicated agent implementation, or second model/config catalog.

## Contents

- [Initialize and iterate](#initialize-and-iterate)
- [Author and customize](#author-and-customize)
- [Configure the public surface](#configure-the-public-surface)
- [Use the shipped reader and agent surfaces](#use-the-shipped-reader-and-agent-surfaces)
- [Generate API reference](#generate-api-reference)
- [Deploy and verify staging](#deploy-and-verify-staging)
- [Respect the current boundary](#respect-the-current-boundary)

## Initialize and iterate

New apps include starter docs. In an existing app:

```bash
npx deepspace docs init
npx deepspace docs validate
npx deepspace docs dev --port 4321
```

Use `npx deepspace docs build` for a deterministic production build. All four commands accept an optional app directory and `--json`; `build` also accepts `--out-dir`, and `dev` accepts `--port`. `init` is idempotent and preserves authored files. Treat validation errors—unsafe paths or markup, broken local links, redirect loops, duplicate routes, missing navigation entries—as product defects instead of bypassing them.

The normal app development plugin compiles docs into the ignored `public/_docs` namespace before Worker assets initialize and rebuilds on source changes. Do not maintain a parallel docs build pipeline.

## Author and customize

Put `.md` and `.mdx` under the configured `source` directory (default `docs`). File paths become routes; `index` represents its directory. Front matter supports `title`, `description`, `slug`, `hidden`, and `noindex`.

- Use `.md` for inert GFM content and the built-in `Note`, `Tip`, `Warning`, `Info`, `Card`, `Accordion`, `Steps`/`Step`, `Tabs`/`Tab`, and `CodeGroup` components.
- Use `.mdx` for trusted same-repository React imports. Components must server-render safely; place browser-only work in effects.
- Add ordinary local components and CSS directly. There is no registry or plugin layer.
- Keep locally referenced media within the docs source tree.

For site-wide customization, use the single optional root file:

```tsx
import { DefaultDocs, type DocsSiteProps } from 'deepspace/docs/react'
import './src/docs/docs.css'

export default function Docs(props: DocsSiteProps) {
  return <div className="product-docs"><DefaultDocs {...props} /></div>
}
```

Omit `docs.tsx` for the SDK template, wrap `DefaultDocs` to extend it, or render `props.children` in a custom shell. Preserve same-origin links so the SDK router can prefetch and navigate without page reloads. Test accessibility for controls and color tokens introduced by custom code.

## Configure the public surface

`docs.json` owns the source, output, branding, navigation, links, redirects, assistant, MCP, contextual actions, SEO, and OpenAPI inputs:

```json
{
  "$schema": "https://deep.space/docs.schema.json",
  "name": "Acme",
  "source": "docs",
  "output": "dist/_docs",
  "navigation": [{ "group": "Get started", "pages": ["index", "quickstart"] }],
  "assistant": { "access": "authenticated" },
  "mcp": { "access": "public" }
}
```

Use native keys for new sites. The loader normalizes a narrow set of high-value Mintlify-shaped source values for migration into the same graph, but no Mintlify code runs at runtime. Do not add aliases, fallback config files, or multiple config authorities.

Explicit navigation must contain every public page exactly once unless the page is hidden. Deploy injects the environment's canonical URL; do not hard-code a production `url` into staging.

## Use the shipped reader and agent surfaces

The SDK template owns the SPA reader, prefetched same-origin routing, search, keyboard navigation, responsive navigation, article table of contents, and contextual page actions. Configure `contextual.actions` with unique values from `copy`, `view`, `assistant`, `chatgpt`, `claude`, `mcp`, `add-mcp`, `cursor`, and `vscode`. These expose the generated page Markdown or explicit handoffs; they do not create another agent.

The visible docs assistant is a profile of the one shared DeepSpace agent runner and versioned model catalog. It is ephemeral and grounded by two read-only tools: corpus search and exact route read. The runner enforces a maximum of 20 tool calls. Never copy the runner, model list, provider policy, or agent route into the app or a feature.

Set `assistant.access` explicitly:

- `disabled`: emit neither UI nor route; this is the default.
- `authenticated`: use the same-origin session and bill the signed-in reader.
- `public`: owner-billed with the platform's bounded public request throttle.

The agent must search before answering, cite same-site page or heading URLs, and state when the corpus lacks evidence. It cannot access app records, integrations, or mutation tools.

Set `mcp.access` to `public` for the stateless Streamable HTTP `/mcp` endpoint or `disabled` to omit it. The public server exposes only `docs_search` and `docs_read` over the same compiled corpus. Builds also emit `llms.txt`, `llms-full.txt`, canonical per-page Markdown, generated Agent Skill discovery, sitemap/robots, redirects, a branded 404, and a manifest. Consume the manifest instead of inferring enablement from directory names.

## Generate API reference

Configure one local OpenAPI JSON/YAML source or an array:

```json
{
  "api": {
    "playground": { "display": "interactive" },
    "examples": {
      "languages": ["curl", "python", "javascript", "go"],
      "defaults": "required",
      "autogenerate": true
    }
  },
  "openapi": [{
    "source": "docs/openapi.yaml",
    "route": "/api-reference",
    "baseUrl": "https://api.staging.example.com"
  }]
}
```

The compiler creates one page per operation and `/_docs/openapi.json`. Examples derive from parameters, request schemas/examples, security schemes, and internal `$ref`s. Standard `x-codeSamples` replace the generated sample for the same language. A playground runs only when the config or specification provides an explicit base URL; keep preview and staging away from production APIs and keep credentials in browser memory.

## Deploy and verify staging

Deploy docs through the app's ordinary release:

```bash
npx deepspace deploy --env staging
```

With `docs.json` present, the result includes `docsUrl`. The app and docs share one app id, Source commit, release, asset bundle, and rollback. The platform URLs are:

- production: `<app>.app.space` and `docs.<app>.app.space`;
- staging: `<app>.spacestest.com` and `docs.<app>.spacestest.com`.

For a purchased hostname, target the logical surface explicitly with `deepspace app domain buy|attach ... --surface docs`; never infer authorization from hostname shape.

After a staging deploy, verify both returned URLs and the release identity. Check at minimum:

1. app and docs routes return the intended release;
2. a same-origin docs navigation keeps an open assistant and unsent draft, proving SPA navigation rather than reload;
3. search, page Markdown, `llms.txt`, MCP (when enabled), contextual actions, and the branded 404 work;
4. the assistant answers from the corpus with same-site citations and stays within the shared runner;
5. OpenAPI pages/examples/playground use only the intended staging base URL;
6. browser console, mobile navigation, keyboard focus, and WCAG checks are clean.

Touch staging only when authorized. Never use production as a routine docs test target.

## Respect the current boundary

These surfaces are intentionally deferred from the shipped 80/20 contract; record the product decision instead of inventing a partial implementation:

- versioned/localized URL trees;
- AsyncAPI generation;
- access control for the entire static docs surface (distinct from authenticated assistant access);
- docs analytics/query retention;
- changelog and changeset publication, which remains a separate release-system phase.

Do not report universal Mintlify parity. The shipped target covers the high-use authoring, reader, agent, API-reference, machine-consumer, and same-release deployment surfaces above.
