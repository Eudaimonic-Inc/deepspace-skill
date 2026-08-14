# References moved into the documentation

The topic references that lived here were absorbed into the documentation at
<https://docs.deep.space>, which is the opinionated authority for both facts
and approach. Start from `SKILL.md`'s documentation router, the
<https://docs.deep.space/llms.txt> index, or the MCP server at
`https://docs.deep.space/mcp` (`documentation_search`, `documentation_read`).

The integration endpoint catalog that lived in `assets/integrations/` is
served live by the CLI: `npx deepspace integrations list` and
`npx deepspace integrations info <integration>/<endpoint>`.

This file also keeps `references/*.md` non-empty for older bake-time checks
that assert the directory's presence.
