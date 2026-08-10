_Load when adding, migrating, customizing, testing, or deploying the DeepSpace documentation feature; when editing `documentation.json`, documentation Markdown/MDX, or `documentation.tsx`; or when debugging its reader, search, assistant, MCP, machine-readable artifacts, or OpenAPI pages._

# Documentation feature

Documentation is part of the app's ordinary source tree, build, Worker, and
immutable release. Do not create a separate docs app, command namespace,
runtime, model catalog, or deployment pipeline.

## Discover and install

Inspect the package-matched feature before changing the app:

```bash
npx deepspace add --info documentation
npx deepspace add documentation
```

The installer creates `documentation.json` and starter Markdown/MDX under
`documentation/`, then wires the documented Vite and Worker adapters. Use
ordinary lifecycle commands:

```bash
npx deepspace dev start
npx deepspace test run
npx deepspace deploy
```

There is no `deepspace docs` command group. Use current `add --info`, CLI
`--help`, installed package types, and the live DeepSpace manual instead of
inventing or recalling commands.

## Author and customize

- Keep `documentation.json`, `documentation/**/*.md`,
  `documentation/**/*.mdx`, and optional root `documentation.tsx` together.
- Markdown is the inert default. Use MDX when trusted same-repository React is
  genuinely useful.
- Add ordinary local components and CSS directly; there is no docs plugin
  registry.
- Omit `documentation.tsx` for the default reader. Add it only to wrap or
  replace the documented React surface.
- Keep media referenced by documentation in app-owned source and never edit
  generated `public/_documentation*` output.

`documentation.json` owns brand, navigation, links, redirects, assistant and
MCP access, contextual actions, SEO, domains, and OpenAPI inputs. Inspect the
installed `DocumentationConfig` types and feature starter before adding keys;
do not preserve `docs.json` or Mintlify aliases as a second authority.

Explicit navigation must contain every public page exactly once unless the
page is hidden. Treat unsafe paths/markup, broken local links, redirect loops,
duplicate routes, and missing navigation entries as defects rather than
bypassing validation.

## Public surfaces

The normal app hostname serves documentation under `/docs`. A hostname listed
in `documentation.json` may serve the same release at its root only after the
owner explicitly attaches that domain to the app. Do not assume or manufacture
a `docs.<app>` hostname.

The feature may emit search, per-page Markdown, `llms.txt`, MCP, sitemap,
robots, OpenAPI pages, contextual actions, and an optional assistant according
to the installed config contract. Consume the generated manifest or runtime
routes; do not infer enablement from output directories.

The documentation assistant uses the shared DeepSpace agent runner and model
catalog. Never copy that runner, its model list, or provider policy into app
code. Keep its access and billing mode explicit in `documentation.json`; do not
silently turn on an owner-billed public assistant.

## Verify

Use a local build/test before any authorized staging deploy. Verify at minimum:

1. ordinary app routes and `/docs` serve the intended release;
2. navigation, search, Markdown/MDX, and the branded not-found behavior work;
3. configured machine surfaces and OpenAPI examples use the intended base URL;
4. assistant/MCP access matches the config and exposes no app mutation tools;
5. console, keyboard, mobile, focus, and accessibility checks are clean.

Do not use production as a routine documentation test target. When exact
config keys, exports, limits, or URLs matter, inspect the package version in
the app and the live manual rather than copying volatile facts into this
reference.
