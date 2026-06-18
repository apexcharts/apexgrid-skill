# Apex Grid AI Skill

AI coding skill for building UIs with the [`apex-grid`](https://github.com/apexcharts/apexgrid) web component (the Lit-based ApexCharts data grid). Works with Claude Code, Cursor, GitHub Copilot, and any AI coding assistant.

> **Naming heads-up.** The npm package is `apex-grid` (with a hyphen), the custom-element tag is `<apex-grid>`, and the exported class is `ApexGrid<T>`.

> **Separate skill — one of the ApexCharts ecosystem skills.** This is the dedicated skill for **Apex Grid** (`apex-grid`), shipped as its own `apexgrid-skill` package and repo — distinct from the core `apexcharts-skill` and the other product skills. Each product has its own library and skill; use the one that matches yours:
>
> | Product | npm library | Skill package & repo |
> |---|---|---|
> | ApexCharts — charts | `apexcharts` | [`apexcharts-skill`](https://github.com/apexcharts/apexcharts-skill) |
> | ApexGantt — Gantt / timeline | `apexgantt` | [`apexgantt-skill`](https://github.com/apexcharts/apexgantt-skill) |
> | ApexTree — hierarchy / org charts | `apextree` | [`apextree-skill`](https://github.com/apexcharts/apextree-skill) |
> | ApexSankey — flow / Sankey | `apexsankey` | [`apexsankey-skill`](https://github.com/apexcharts/apexsankey-skill) |
> | **Apex Grid** — data grid · *this skill* | `apex-grid` | `apexgrid-skill` |

## What This Does

AI models routinely get web-component grid code wrong: they treat `<apex-grid>` like a class with a constructor, forget to register the custom element, set `columns` as a stringified attribute, return strings (not Lit `TemplateResult`s) from `cellTemplate`, or expect events from programmatic `sort()` calls. This skill ships structured reference files so the assistant generates correct `apex-grid` code on the first try.

### Coverage

- **Quickstart** — register + size the host (`setup()` does both); no theme import — the grid self-styles via `--ag-*` CSS variables
- **Custom-element registration** — `setup()`, `import 'apex-grid/define'`, or `ApexGrid.register()`
- **Generic `ColumnConfiguration<T>`** — `key`, the 13 column `type`s (incl. `date`, `select`, `currency`, …), templates, `sort`, `filter`, `pinned`, explicit widths
- **Lit cell, header & editor templates** with the right context shapes
- **Sort, filter & quick-filter** — programmatic API, UI events, operands, multi-column
- **Editing, selection, pagination, tree & master-detail** — the opt-in feature config objects
- **Server-side data** — `dataPipelineConfiguration` async hooks (sort/filter/pagination/quickFilter)
- **CSV export & column reordering/pinning**
- **Virtualization** — `grid.rows` vs `grid.dataView` vs `grid.totalItems`
- **Vanilla JS** — full setup without using Lit's `render()` in app code
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
│   ├── columns-and-templates.md       # ColumnConfiguration, types, Lit templates, widths
│   ├── sort-and-filter.md             # operands, expressions, events, multi-column
│   ├── data-pipeline.md               # async hooks, server-side, virtualization
│   ├── vanilla-js.md                  # full setup without Lit render() in app code
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
- [igniteui-webcomponents](https://www.npmjs.com/package/igniteui-webcomponents) (transitive dep — installs automatically)

## License

MIT
