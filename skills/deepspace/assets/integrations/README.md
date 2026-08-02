# Integration endpoint catalog

These hand-maintained YAML files are the offline fallback for `deepspace integrations list|info`:

- `index.yaml`: endpoint keys and summaries. Search this first.
- `<integration>.yaml`: input/output schema, billing, and OAuth details for one integration. Load only the file you need.

Endpoint keys are exactly `<integration>/<endpoint>` and calls use `integration.post(key, body)`. Entries describe the POST method, validated input schema, optional output schema, and OAuth requirement. Responses use:

```ts
{ success: true, data: unknown }
{ success: false, error: string, issues?: Array<{ path?: unknown; message: string; code?: string }> }
```

The live CLI catalog is authoritative if it differs from these files. These are skill metadata, not runtime app assets; do not copy them into an app.
