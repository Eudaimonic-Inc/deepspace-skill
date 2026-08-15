# References moved into the documentation

The topic references that lived here were absorbed into the documentation at
<https://docs.deep.space>, which is the opinionated authority for both facts
and approach. Start from `SKILL.md`'s documentation router, the
<https://docs.deep.space/llms.txt> index, or the MCP server at
`https://docs.deep.space/mcp` (`documentation_search`, `documentation_read`).

The integration endpoint catalog that lived in `assets/integrations/` is
served live by the CLI: `npx deepspace integrations list` and
`npx deepspace integrations info <integration>/<endpoint>`.

This file is ordering insurance: images built from a pre-cutover
create-build-service assert `references/*.md` at bake time, and this file
keeps that guard green. Once the cutover's relaxed guard (SKILL.md only) is
everywhere, the file is just a browse-time pointer and may be dropped.
