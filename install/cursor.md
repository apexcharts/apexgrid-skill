# Installing Apex Grid Skill for Cursor

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexgrid-skill/main/.cursorrules
```

Restart Cursor or open a new window. Cursor automatically reads `.cursorrules` files in the project root as context.

## For Windsurf

Same approach — Windsurf also reads `.cursorrules`.

## Verification

Ask Cursor to generate an Apex Grid with multi-column sort and a server-side filter pipeline. It should:
- Set `sortConfiguration: { multiple: true, triState: true }` (or rely on the default)
- Provide an async `dataPipelineConfiguration.filter` hook that returns the new data array
- Use property bindings (`.columns=${...}`), not attribute syntax
