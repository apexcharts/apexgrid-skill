# Apex Grid AI Skill

AI coding skill for building UIs with the [`apex-grid`](https://github.com/apexcharts/apexgrid) web component (the Lit-based ApexCharts data grid). Works with Claude Code, Cursor, GitHub Copilot, and any AI coding assistant.

> **Naming heads-up.** The npm package is `apex-grid` (with a hyphen), the custom-element tag is `<apex-grid>`, and the exported class is `ApexGrid<T>`.

## What This Does

AI models routinely get web-component grid code wrong: they treat `<apex-grid>` like a class with a constructor, forget to register the custom element, set `columns` as a stringified attribute, return strings (not Lit `TemplateResult`s) from `cellTemplate`, or expect events from programmatic `sort()` calls. This skill ships structured reference files so the assistant generates correct `apex-grid` code on the first try.

### Coverage

- **Custom-element registration** — `import 'apex-grid/define'` vs `ApexGrid.register()`
- **Generic `ColumnConfiguration<T>`** — `key`, `type`, templates, `sort`, `filter`
- **Lit cell & header templates** with the right context shapes
- **Sort & filter** — programmatic API, UI events, operands, multi-column
- **Server-side data** — `dataPipelineConfiguration` async hooks
- **Virtualization** — `grid.rows` vs `grid.dataView` vs `grid.totalItems`
- **Framework integration** — Lit, React, Vue, Angular

## Installation

### Claude Code

```bash
mkdir -p .claude/skills
cd .claude/skills
git clone https://github.com/apexcharts/apexgrid-skill.git
```

### Cursor / Windsurf

```bash
curl -o .cursorrules https://raw.githubusercontent.com/apexcharts/apexgrid-skill/main/.cursorrules
```

### GitHub Copilot

Reference `SKILL.md` in Copilot Chat: `@workspace #file:SKILL.md`, or paste the contents of `.cursorrules` into Copilot's custom instructions.

### As an npm dependency

```bash
npm install apexgrid-skill
```

```js
import { skillFile, referencePath } from 'apexgrid-skill';
import { readFile } from 'node:fs/promises';

const skill = await readFile(skillFile, 'utf8');
const cols  = await readFile(referencePath('columns-and-templates.md'), 'utf8');
```

## Repository Structure

```
├── SKILL.md                           # Main entry point
├── .cursorrules                       # Self-contained Cursor / Windsurf version
├── references/
│   ├── columns-and-templates.md       # ColumnConfiguration, types, Lit templates
│   ├── sort-and-filter.md             # operands, expressions, events, multi-column
│   ├── data-pipeline.md               # async hooks, server-side, virtualization
│   └── framework-integration.md       # Lit, React, Vue, Angular
└── install/
    ├── claude-code.md
    ├── cursor.md
    └── copilot.md
```

## Links

- [apex-grid GitHub](https://github.com/apexcharts/apexgrid)
- [npm: apex-grid](https://www.npmjs.com/package/apex-grid)
- [Lit framework](https://lit.dev)
- [igniteui-webcomponents](https://www.npmjs.com/package/igniteui-webcomponents) (runtime peer)

## License

MIT
