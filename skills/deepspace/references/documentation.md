_Load when installing, authoring, customizing, testing, or deploying the DeepSpace documentation feature; when editing `documentation.json`, `documentation/**/*.md(x)`, or `documentation.tsx`; or when debugging `/docs`, search, page actions, the assistant, MCP, Agent Skill artifacts, or OpenAPI pages._

# Documentation

DeepSpace compiles app-owned documentation into the app's ordinary immutable release. The public surface is `/docs` on the app hostname. It is not a second app, release, agent, or platform-managed subdomain.

## Install and run

Inspect and install the optional feature instead of wiring its Vite plugin, Worker routes, Durable Object, or reserved assets by hand:

```bash
npx deepspace add --info documentation
npx deepspace add documentation
npx deepspace dev start
```

Open `/docs` on the URL returned by `dev start`. Use the ordinary test and deploy lifecycle:

```bash
npx deepspace test run
npx deepspace deploy
```

There is no documentation-specific CLI command group. The presence of `documentation.json` enables compilation; apps without it keep ownership of `/docs`.

## Author in the app

The installed structure is intentionally small:

```text
documentation.json
documentation/
  index.mdx
  quickstart.mdx
documentation.tsx  # optional
```

File paths become routes; `index` represents its directory. Front matter supports `title`, `description`, `slug`, `hidden`, and `noindex`.

- Use `.md` for inert GFM content.
- Use `.mdx` for trusted same-repository React, the exported documentation components, and direct local imports.
- Keep local media inside the source tree so the compiler can publish it safely.
- Keep navigation complete: every public page appears exactly once when navigation is explicit.
- Fix unsafe paths or markup, broken links, redirect loops, duplicate routes, and navigation errors rather than bypassing validation.

MDX and the optional root component execute app-owned code. Treat documentation changes with the same review and testing standards as application code.

## Customize without a plugin layer

Omit `documentation.tsx` to use the SDK-owned reader. Add it only when the whole surface needs an app-specific shell:

```tsx
import {
  DefaultDocumentation,
  type DocumentationSiteProps,
} from 'deepspace/documentation/react'

export default function Documentation(props: DocumentationSiteProps) {
  return <div className="product-documentation"><DefaultDocumentation {...props} /></div>
}
```

MDX can import app components directly. DeepSpace also exports `Accordion`, `AccordionGroup`, `Card`, `CodeGroup`, `Info`, `Note`, `Step`, `Steps`, `Tab`, `Tabs`, `Tip`, and `Warning` from `deepspace/documentation/react`. Do not create a component registry, compatibility runtime, or parallel router.

## Configure the public surface

`documentation.json` is the sole configuration authority. The feature starts with `source: "documentation"`, an empty `domains` list, explicit navigation, and authenticated assistant access. Set access modes explicitly when changing them:

```json
{
  "name": "Acme",
  "description": "Build with Acme",
  "domains": [],
  "navigation": [{ "group": "Get started", "pages": ["index", "quickstart"] }],
  "assistant": { "access": "authenticated" },
  "mcp": { "access": "public" }
}
```

The same file owns branding, fonts, links, footer, redirects, OpenAPI inputs, contextual actions, and SEO. Installed `deepspace/documentation` types and validation are authoritative for less common fields; do not invent aliases or fallback config files during migration.

The reader includes responsive navigation, search, article navigation, table of contents, theme controls, and page actions. Contextual actions may use `copy`, `view`, `assistant`, `chatgpt`, `claude`, `mcp`, `add-mcp`, `cursor`, and `vscode`.

## Agent and machine-readable surfaces

The assistant is a read-only profile of the shared DeepSpace agent runner, grounded by `documentation_search` and `documentation_read` with at most twenty tool calls. Access is `disabled`, `authenticated`, or owner-billed `public`; the compiler default is disabled, while the installed starter opts into authenticated access. It cannot access app records, integrations, or mutation tools.

MCP is either `disabled` or `public` and serves the same two read-only tools at `/docs/mcp`. Builds also publish page Markdown, `llms.txt`, `llms-full.txt`, Agent Skill discovery, sitemap/robots, redirects, search data, and a branded 404 under `/docs`. Consume generated URLs and manifests rather than internal `/_documentation` asset paths.

For OpenAPI, point `openapi` at local JSON or YAML and give an explicit non-production `baseUrl` when enabling an interactive playground in preview or staging. Generated operations, examples, and playgrounds remain part of the same documentation graph.

## Domains, deploys, and migration

The normal URLs are `https://<app>.app.space/docs` in production and `https://<app>.spacestest.com/docs` in staging. Documentation shares the app id, Source commit, asset bundle, release, and rollback.

Only list a custom hostname in `domains` after attaching that exact hostname to the app through the existing `deepspace app domain` workflow. A listed attached hostname mounts documentation at `/`; the feature does not provision DNS, TLS, a second app, or a hostname inferred from a `docs.` prefix.

For staging, use the ordinary environment lifecycle described in `deploy.md` and verify the returned app URL plus `/docs`. Test authored routes, internal navigation, search, generated Markdown and LLM artifacts, configured page actions, MCP/assistant access, OpenAPI base URLs, mobile navigation, focus behavior, and console errors. Never use production as a routine documentation test target.

When migrating from Mintlify, move Markdown/MDX and local components into this app-owned structure, translate configuration intentionally, and validate the result through the normal app lifecycle. Do not copy Mintlify runtime code or claim unshipped versioning, localization, AsyncAPI, static-site access control, analytics, or changelog support.
