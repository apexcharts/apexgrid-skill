---
name: apex-grid
description: >
  AI skill for building UIs with the `apex-grid` web component (a Lit-based
  data grid by ApexCharts). Use whenever the user asks to create, configure, or
  troubleshoot a sortable / filterable / paginated / editable / virtualized data
  table with `<apex-grid>`. Covers the register-and-size setup (no theme import —
  the grid styles itself via `--ag-*` CSS variables), the optional free
  `apex-grid/ignite-ui` add-on that follows an Ignite UI theme, the generic
  `ColumnConfiguration<T>` shape with its 13 column types, Lit `cellTemplate`
  requirements (read the record from `ctx.data`; body rows are recycled),
  theming and dark mode (`--ag-*` / `--ag-dark-*` tokens, `theme="dark"`),
  sorting / filtering / quick-filter, pagination (full or compact paginator),
  inline editing, row selection, master-detail and tree rows, CSV export, batch
  writes via `applyEdits`, right-to-left layout, the UI-only event model, the
  `dataPipelineConfiguration` hooks for server-side data, and upgrading from
  apex-grid 3.x to 4.0.
metadata:
  author: ApexCharts
  version: "2.6.0"
  library_version: "4.0.0"
  category: data-visualization
  tags: [grid, data-grid, table, web-component, lit, apex-grid]
  docs: https://github.com/apexcharts/apexgrid
  npm: apex-grid
  github: https://github.com/apexcharts/apexgrid
---

# Apex Grid AI Skill

## 1. Quickstart — produce a visible, styled grid

The grid styles itself out of the box (borders, row separators, sort/filter UI) — there is **no theme CSS to import**. The one real gotcha is host sizing: the grid virtualizes rows, so it needs a bounded height or it collapses and renders nothing. Two setup steps: **register** the element and give it a **bounded height**, then bind `.data` / `.columns`.

### 1.1 Install

```bash
npm install apex-grid lit
```

That is the whole install. `apex-grid` brings its own runtime dependencies (`lit`, `@lit/context`, `@lit-labs/virtualizer`) and there is **no theme CSS to import** from anywhere.

> **Changed in 3.5.0:** earlier versions also pulled in `igniteui-webcomponents` and `igniteui-theming` transitively. Both are gone. The grid never registered an Ignite UI element, so nothing you wrote against the grid is affected, but code that relied on `igc-*` elements being installed as a side effect must now depend on that package directly. An Ignite UI theme on the page never fully themed the grid: through 3.x it only tinted the brand and replaced the font. From 4.0 the only way to make the grid follow an Ignite UI theme is the `apex-grid/ignite-ui` add-on (§1.3).

> Confirmed working versions: `apex-grid@4.0.0`, `lit@^3.3.0` (and `apex-grid-enterprise@0.9.0`, which needs apex-grid 4.0). 4.0 is a major version: moving an app from 3.x, read §1.6 first.

### 1.2 Register the element + size the host — `setup()` does both

The one-call convenience export registers `<apex-grid>` and adopts a default host stylesheet (`height: 100%` with a `min-height: 240px` fallback) so the virtualizer has a bounded height:

```ts
import { setup } from 'apex-grid';
setup();                       // registers <apex-grid> + adopts host sizing. Idempotent.
```

Prefer manual control? Register with the side-effect import (or the static method) and size the host yourself:

```ts
import 'apex-grid/define';     // side-effect import, idempotent
// or: import { ApexGrid } from 'apex-grid'; ApexGrid.register();
```

```css
apex-grid { height: 480px; }   /* any explicit height; % works if the parent is sized too */
```

`setup({ hostStyles: false })` registers the element but skips the injected host CSS, leaving sizing entirely to you. Without a bounded height (from `setup()` or your own CSS) the grid collapses and shows no rows.

Since 3.6.0 the `setup()` sizing is adopted into whichever root each grid connects in, so a grid rendered inside another web component's shadow root gets it too. On 3.5 and earlier it reached the page only, and a grid inside a shadow root needed its own height rule there.

> **Do not set `display` on `apex-grid`** — the component declares `:host { display: grid }` for its internal track layout; an outer `display` rule overrides it and collapses the virtualizer.

### 1.3 Styling — `--ag-*` CSS custom properties (no theme import)

The grid is styled out of the box and customized entirely through `--ag-*` CSS custom properties. There is **no theme file to import and no `configureTheme()` call**. Set the variables on `apex-grid` or on any ancestor: `:root`, a dashboard panel, or the shadow host of a web component the grid renders in.

```css
apex-grid {
  height: 480px;
  --ag-brand: #7c3aed;   /* selection, focus, accents; the brand washes follow it */
  --ag-radius: 12px;     /* outer card radius */
  --ag-row-h: 40px;      /* row height; see the package README for the full token list */
}
```

> **Changed in 3.6.0:** the token defaults moved off the grid host onto `:where(:root)`, adopted into the document (through `adoptedStyleSheets`, so no CSP nonce) when the first grid connects. On 3.5 and earlier a value set on an ancestor never reached the grid, because the host's own default beat it; only values set on the `apex-grid` element took effect. Now both do, through shadow roots however deep.

**Dark mode.** Add the `theme="dark"` attribute (an attribute, not a property). `theme="tinted"` mixes the brand into the chrome, and the two combine as `theme="dark tinted"`. A dark grid reads its own colour inputs, the `--ag-dark-*` tokens (`--ag-dark-surface`, `--ag-dark-surface-elevated`, `--ag-dark-grid-bg`, `--ag-dark-text`, `--ag-dark-text-muted`, `--ag-dark-hairline`, `--ag-dark-border`, `--ag-dark-row-hover`, `--ag-dark-shadow-card`, ...). Sizing, typography, motion, the brand and the base state colours are shared by both modes, so one place can theme light and dark grids:

```css
.dashboard {
  --ag-brand: #7c3aed;
  --ag-surface: #fdfcff;        /* light grids */
  --ag-dark-surface: #140f24;   /* grids with theme="dark" */
}
apex-grid { --ag-grid-shadow: var(--ag-shadow-card); }                     /* elevated card edge */
apex-grid[theme~='dark'] { --ag-grid-shadow: var(--ag-dark-shadow-card); } /* the same, on dark grids */
```

> **Changed in 3.6.0:** a light colour token (`--ag-surface`, `--ag-text`, `--ag-border`, ...) no longer applies to a dark grid, whether it is set on an ancestor or **on the dark grid element itself**. A rule such as `apex-grid[theme='dark'] { --ag-surface: #111; }` is now ignored: write `--ag-dark-surface`. A `theme="tinted"` grid ignores `--ag-border` and `--ag-dark-border` (its row separators are the brand alone).

**Reading tokens in your own CSS** (page styles, markup in a `cellTemplate`): `var(--ag-surface)`, `var(--ag-text-muted)` and the other inputs resolve to their defaults anywhere on the page. `--ag-brand`, `--ag-brand-strong` and the derived washes (`--ag-brand-soft`, `--ag-brand-softer`, `--ag-brand-line`, `--ag-row-hover`, `--ag-surface-alt`) have no root default, so give them a fallback: `var(--ag-brand, #0d6fd3)`. To manage the defaults yourself, `setup({ tokenStyles: false })` skips the adopted sheet and `import 'apex-grid/tokens.css'` ships the same defaults as a file (also the way to get them into server-rendered HTML). The grid renders the same either way.

**CSS parts.** Since 3.6.0 every part a body row renders is reachable with `apex-grid::part(...)`: `cell`, `selection-cell`, `selection-checkbox`, `expansion-cell`, `expansion-toggle`, `expansion-chevron`, `detail-panel`, `row-number-cell`, `row-number`, `reorder-handle-cell`, `reorder-handle`, `reorder-grip`. Before, only `cell` was. Parts inside your own `detailTemplate` content are reached by listing them in `expansion.exportParts` (`references/state-and-features.md`).

Since 4.0.0 the paginator and toolbar parts are reachable too. Paginator: `paginator-host` (the element), `paginator`, `paginator-size`, `paginator-info`, `paginator-controls`, `paginator-page`, `paginator-button` (every button) plus one part per button, `paginator-first`, `paginator-prev`, `paginator-next`, `paginator-last`. Toolbar: `toolbar-host`, `toolbar`, `toolbar-search`, `search-field`, `search-icon`, `search-input`, `toolbar-actions`, `toolbar-action`, `export`, `export-trigger`, `export-icon`, `export-caret`, `export-menu`, `export-menu-item`. Also new: `table` (the element with the grid role) and `drag-ghost` (the column drag ghost).

```css
apex-grid::part(paginator-button) { border-radius: 999px; }
apex-grid::part(paginator-next)   { font-weight: 600; }
apex-grid::part(search-input)     { font-size: 14px; }
```

> The deprecated `setup({ theme })` option does **not** change the grid's appearance. It only ever forwarded to Ignite UI's `configureTheme()`, and since 3.5.0 it is inert and warns once. Omit it; use `--ag-*` variables (and the `theme="dark"` attribute) instead, and to follow an Ignite UI theme, the `apex-grid/ignite-ui` add-on below.

**Ignite UI theming (new in 4.0.0).** The add-on ships with apex-grid 4.0; on 3.x `import 'apex-grid/ignite-ui'` does not resolve. To make grids follow an Ignite UI theme loaded on the page, import the add-on once. It is free and needs no licence key.

```ts
import 'apex-grid/ignite-ui';
```

Every grid on the page, in the document or any shadow root, then follows the theme, including grids that connected before the import: the brand (`--ig-primary-*`), text and muted text, gridlines and the pinned-column edge, the header (`--ig-surface-raised`, Ignite UI 7.5+), the state colours, the font, and the `apex-grid-enterprise` popups (`--ig-surface-overlay`, 7.5+). Card and row surfaces keep the grid's own light or dark background. The add-on registers on a shared `globalThis` slot, so it reaches grids from a separately built bundle too. The grid ships as ES modules only: load it from your bundle or a `<script type="module">`.

- **Ranking**, highest first: an `--ag-*` token you set > the family `--apx-*` tokens (`--apx-accent`, `--apx-fore`, `--apx-grid`, `--apx-surface`, `--apx-font-family`, `--apx-radius-factor`, also new in that release) > Ignite UI > built-in values. (Across the Apex family: explicit option > product CSS variable > `--apx-*` > named theme/palette > `--ig-*` > defaults.)
- **Mode:** a grid without a `theme` attribute follows the Ignite UI variant (`--ig-theme-variant`) declared on `:root` or an ancestor, live, on a theme swap too. `theme="light"` keeps a grid light and `theme="dark"` keeps it dark, so do not pin `theme="light"` if the grid should go dark with a dark Ignite UI theme.
- **Per grid:** `design-system="off"` (property `designSystem = false`) turns the Ignite UI colours and font off (the variant still picks the mode); `tokens="off"` (`tokens = false`) turns off the `--apx-*` tokens and Ignite UI together, the variant included, for the grid and the content it renders.
- **No `refreshTokens()` needed:** the grid follows the cascade live.
- **Your own CSS:** while the add-on is loaded, the tokens that follow Ignite UI (text, lines, surfaces, header, state colours, pinned edge) leave the root defaults, so page CSS that reads one needs its own fallback: `var(--ag-text, #111827)`.
- **Enterprise popups:** new tokens `--ag-overlay` and `--ag-overlay-text` (dark grids: `--ag-dark-overlay`, `--ag-dark-overlay-text`) colour the `apex-grid-enterprise` popups (apex-grid-enterprise 0.9.0); unset, each keeps its built-in colours. For the enterprise chart panel to read the theme itself, also import `apexcharts/features/ignite-ui` (ApexCharts 8.8.0 or later; the grid never loads it).
- **`--ag-font`:** in 4.0 an `--ag-font` you set wins over the family and Ignite UI fonts, and it no longer has a root default (the built-in Inter stack is unchanged inside the grid), so page CSS that reads it needs a fallback, as with `--ag-brand`.

> **Breaking in 4.0.0:** the grid no longer reads Ignite UI tokens by itself. Through 3.x an Ignite UI theme on the page re-tinted the brand (`--ig-primary-500`, `--ig-primary-700`) and replaced the font (`--ig-font-family`) with no import. From 4.0 an Ignite UI stylesheet changes nothing until you `import 'apex-grid/ignite-ui'`, which brings the brand and font back along with the rest of the theme. The 4.0 grid also follows the family `--apx-*` tokens: a page that sets them on `:root` for its charts sees the grid follow them too (set the matching `--ag-*` tokens to keep the grid as it was).

### 1.4 Minimal example — a styled, sortable, filterable table

```ts
import { html, render } from 'lit';
import { setup } from 'apex-grid';
import type { ColumnConfiguration } from 'apex-grid';

setup();   // register + host sizing; no theme import needed

type User = { id: number; name: string; age: number; subscribed: boolean };

const data: User[] = [
  { id: 1, name: 'Ada Lovelace',  age: 36, subscribed: true  },
  { id: 2, name: 'Carl Sagan',    age: 62, subscribed: false },
  { id: 3, name: 'Grace Hopper',  age: 85, subscribed: true  },
];

// Explicit widths prevent the "last column stretches to fill" trap.
const columns: ColumnConfiguration<User>[] = [
  { key: 'id',         type: 'number',  headerText: 'ID',         width: '80px',  sort: true, filter: true },
  { key: 'name',       type: 'string',  headerText: 'Name',       width: '240px', sort: true, filter: true },
  { key: 'age',        type: 'number',  headerText: 'Age',        width: '100px', sort: true, filter: true },
  { key: 'subscribed', type: 'boolean', headerText: 'Subscribed', width: '140px', sort: true, filter: true },
];

render(
  html`<apex-grid .data=${data} .columns=${columns}></apex-grid>`,
  document.getElementById('app')!,
);
```

### 1.5 What success looks like

- **Visible borders** between rows and columns, and **hover state** on rows
- **Sort arrows** (↕) next to each header (because `sort: true`)
- A **filter row** below the headers with a "Filter" chip per column (because `filter: true`)
- **Smooth virtualized scrolling** — only ~20 `<apex-grid-row>` elements exist in the DOM at once, even with thousands of rows

If rows don't render at all, it's almost always the host height (§1.2). If the grid looks bare, you don't need a theme import — check your `--ag-*` overrides aren't fighting the defaults.

> One non-obvious case, fixed in **3.5.0**: a grid first laid out while it is off-screen (inside an iframe the host page has not sized yet, or below the fold on a page still settling) could render zero rows while reporting a correct row count and scroll height, and stay that way until something forced a re-render. On 3.5.0 and later the body detects this and re-measures itself. On earlier versions, reassign `grid.data = grid.data.slice()` once the layout settles.

> **Dev-server reminder.** ES module imports need an HTTP server — opening via `file://` fails with CORS errors. Use Vite (`npm create vite@latest`), any static server, or your framework's dev server.

### 1.6 Upgrading from 3.x (4.0.0)

4.0 is a major version. Check these when moving an app from 3.6:

- **Body rows are recycled by default** (`rowRecycling`, default `true`). A scroll, sort or filter re-binds the rendered row elements to the records now in view, so a row element and its cells can show another record later. In `cellTemplate` / `editorTemplate`, read the record from `ctx.data` (not `ctx.row.data`), derive everything from the context on each render, and do not hold `ctx.row`, `ctx.parent` or `grid.rows` elements across an `await` or a scroll. A template that keeps element state (an input's typed value, a third-party widget) can wrap itself in `keyed(ctx.data, ...)` (`lit/directives/keyed.js`), or opt the grid out with `grid.rowRecycling = false` (attribute `row-recycling="false"`) for record-keyed rows. Detail panels follow their record and need no change.
- **The grid role moved off the host.** `role` (`grid`, or `treegrid` for tree, master-detail and grouped grids), `aria-rowcount`, `aria-colcount`, `aria-multiselectable` and the accessible name are on the inner `[part~="table"]` element. An `aria-label` / `aria-labelledby` on the host still names the grid and `aria-describedby` describes it. Tests that read these from the host query `grid.shadowRoot.querySelector('[part~="table"]')`.
- **An Ignite UI theme does nothing until you `import 'apex-grid/ignite-ui'`** (3.x took its brand and font on its own), and **`--ag-font` has no root default**: page CSS that reads it writes `var(--ag-font, Inter, sans-serif)` (§1.3).
- **Pagination.** Pinned rows no longer count toward `totalItems`, `pageCount` or the summary (they show on every page). A local page change no longer reruns the pipeline, so custom `sort` / `filter` hooks are not called on it (remote mode still reruns). `DataPipelineConfiguration.pagination` and the `'pagination'` value of `DataPipelineParams.type` are removed: the hook was never called, but TypeScript that sets it stops compiling. For server paging use `pagination.mode: 'remote'` (`references/data-pipeline.md`).
- **`min()` and `max()` fail a value that is not a number.** On 3.x text such as `'abc'` passed both. Empty values still pass.
- **Smaller type and DOM changes:** `ctx.data` is a required field of the cell context (code that builds one by hand supplies it), `ApexGridPaginator.pageSizeOptions` is `readonly number[]`, the paginator's previous button id is `prev` (was `previous`), and the protected `ApexGrid.headerRowCount` and `ApexGridHeaderRow.renderDragGhost` are gone (subclasses only).
- **Noticeable, not breaking:** `pageItems` keeps the same array while nothing that shapes it changes; the column header row is no longer a Tab stop; the column menu opens in the top layer (it can extend outside the grid's box); a page change is announced once.

New in 4.0: `showHeader`, `selection.isRowSelectable`, `pagination.paginator: 'compact'` (§5), `::part()` for every paginator and toolbar part (§1.3), `ctx.data` (§4), the Ignite UI add-on and the family `--apx-*` tokens (§1.3), IME input and numbers typed in full width or with the locale's grouping (§5), and locale keys for text that stayed English (`references/state-and-features.md`).

---

## 2. Naming — `apex-grid` vs `<apex-grid>` vs `ApexGrid`

| Form | What it is |
|---|---|
| `apex-grid` | The **npm package name**. `import { ApexGrid } from 'apex-grid'`. With a hyphen. **There is no `apexgrid` package.** |
| `<apex-grid>` | The **HTML custom-element tag** rendered into the DOM. |
| `ApexGrid<T>` | The **exported class**. Used for types, the static `register()` method, and `ApexGrid.tagName`. |

AI models trained before the package was published may suggest `apexgrid` (no hyphen) or treat `ApexGrid` as a class constructor — both are wrong.

---

## 3. Critical Rules

1. **Apex Grid is a Lit web component, not a JS class with a constructor.** You don't `new ApexGrid(...)`. Render `<apex-grid>` in the DOM and set `.data` / `.columns` as properties.

2. **Two setup steps for a visible grid:** register the element and give the host a bounded height — `setup()` does both (§1.2). **No theme CSS import is required** (§1.3); the grid styles itself via `--ag-*` variables. The classic failure is an unsized host → no rows.

3. **`columns` and `data` are properties, not attributes.** Use property binding (`.columns=${...}` in Lit, `[columns]=` in Angular, `:columns.prop=` in Vue, `el.columns = ...` in vanilla JS). A stringified `columns="[...]"` HTML attribute becomes the literal string and the grid renders nothing.

4. **`column.key` must be a real `keyof T`.** No `field` / `dataIndex` / `accessor` aliases. To display a derived value, compute it in `cellTemplate` from `ctx.data`, the record the cell renders (4.0). `ctx.row` is the `<apex-grid-row>` element, which recycling can re-bind to another record.

5. **Set `type` per column.** The primitive types `'string'`, `'number'`, `'boolean'` drive the default sort comparer, filter operands, and editor. The default is `'string'` — leaving a numeric column unset sorts as strings ("10" before "9"). There are also presentation types — `'select'`, `'rating'`, `'date'`, `'image'`, `'currency'`, `'avatar'`, `'badge'`, `'progress'`, `'sparkline'`, `'status'` — that change rendering while sorting/filtering as their underlying value (§4).

6. **`cellTemplate` / `headerTemplate` / `editorTemplate` must return Lit `TemplateResult`s** — values from an `` html`...` `` tagged template. Returning a plain string renders as escaped text.

7. **Sort/filter (and the other `*-ing`/`*-ed`) events fire only for UI-initiated operations.** Programmatic `grid.sort(...)` / `grid.filter(...)` calls apply silently. To react to every change regardless of source, watch `grid.dataView`.

8. **Features are opt-in via config objects.** Sorting/filtering are per-column (`sort`/`filter`); editing, selection, pagination, tree, and master-detail expansion are enabled via the grid-level `editing` / `selection` / `pagination` / `tree` / `expansion` properties (§5).

---

## 4. Data & Columns

Apex Grid is generic over a row type `T extends object`:

```ts
interface ColumnConfiguration<T extends object, K extends keyof T = keyof T> {
  key: K;                                  // REQUIRED — must be a keyof T
  type?: DataType;                         // default 'string' (see column types below)
  headerText?: string;                     // default: String(key)
  width?: string;                          // any CSS width — recommended in practice
  hidden?: boolean;                        // default false
  resizable?: boolean;                     // default false
  pinned?: 'start' | 'end' | null;         // freeze to a side during horizontal scroll
  reorderable?: boolean;                   // per-column opt-out of drag reorder
  exportable?: boolean;                    // default true; false omits from CSV export
  group?: string;                          // id of a columnGroups spanning header (§5)
  sort?:   boolean | { caseSensitive?: boolean; comparer?: (a: T[K], b: T[K]) => number };
  filter?: boolean | { caseSensitive?: boolean };
  editable?: boolean;                      // requires grid `editing.enabled`
  validators?: Validator<T, K>[];          // declarative edit-time validation (§5)
  headerTemplate?:  (ctx: ApexHeaderContext<T>) => TemplateResult | unknown;
  cellTemplate?:    (ctx: ApexCellContext<T, K>) => TemplateResult | unknown;
  editorTemplate?:  (ctx: ApexEditorContext<T, K>) => TemplateResult | unknown;
  // Type-specific options (each is ignored unless `type` matches):
  options?: (V | { value: V; label?: string })[];   // type: 'select'
  max?: number;                            // 'rating' star count (5) / 'progress' full value (100)
  format?: 'short' | 'medium' | 'long' | 'full';     // 'date' display (default 'medium')
  shape?: 'square' | 'circle';             // 'image' crop (default 'square')
  alt?: string;                            // 'image' alt text
  currency?: string;                       // 'currency' ISO 4217 code (default 'USD')
  locale?: string;                         // 'currency' BCP 47 locale; 4.0: also how numbers typed into the filter panel, or checked by min() and max(), are read (enterprise: paste, advanced filter)
  badgeVariant?: 'gold' | 'brand' | 'neutral' | 'muted' | ((v) => …);   // 'badge'
  statusVariant?: 'active' | 'trial' | 'churn' | ((v) => …);            // 'status'
  showDelta?: boolean;                     // 'sparkline' trailing delta label (default true)
}
```

Applied defaults (`DEFAULT_COLUMN_CONFIG`): `{ type: 'string', resizable: false, hidden: false, sort: false, filter: false }`. Sort and filter are **off** until you opt in. (`exportable` isn't in that object but behaves as `true` unless set to `false`.)

### Column types (`DataType`)

`type` is both the data type and, for the built-ins, the presentation renderer. The three primitive types drive sort/filter/editing; the rest are renderers over a primitive value (and sort/filter as that underlying value):

| `type` | Renders / behaves as |
|---|---|
| `'string'` | text (default); text editor |
| `'number'` | number; number editor |
| `'boolean'` | check-mark icon (true) / dimmed (false); checkbox editor |
| `'select'` | label from `options`; `<select>` editor. Sorts/filters as its value |
| `'rating'` | 0..`max` (default 5) as stars; star-picker editor. Numeric |
| `'date'` | locale date (`format` preset); `<input type="date">` editor. Accepts `Date`/ISO/timestamp |
| `'image'` | `<img>` (`shape` `'square'`/`'circle'`, `alt`) |
| `'currency'` | `Intl.NumberFormat` (`currency`, `locale`). Numeric |
| `'avatar'` | first letter in a tinted circle |
| `'badge'` | colored pill (`badgeVariant`) |
| `'progress'` | health bar, 0..`max` (default 100). Numeric |
| `'sparkline'` | inline trend chart of a `number[]` (`showDelta`) |
| `'status'` | dot + pill (`statusVariant`, or inferred) |

> A `'date'` column **does** exist in 3.x — use `type: 'date'` (it accepts `Date`, ISO strings, or millisecond timestamps), not a string column with a custom comparer.

### Column type → filter operands

| `type` | Operand keys in the filter UI |
|---|---|
| `'string'`, and every type not listed below (`select`, `date`, `image`, `avatar`, `badge`, `sparkline`, `status`) | `contains`, `doesNotContain`, `startsWith`, `endsWith`, `equals`, `doesNotEqual`, `empty`, `notEmpty` |
| `'number'`, and the numeric types `'currency'`, `'rating'`, `'progress'` | `equals`, `doesNotEqual`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `empty`, `notEmpty` |
| `'boolean'` | `all`, `true`, `false`, `empty`, `notEmpty` (all **unary**: no `searchTerm`) |

```ts
grid.filter({ key: 'subscribed', condition: 'true' });   // unary: no searchTerm
grid.filter({ key: 'price', condition: 'greaterThan', searchTerm: 100 });   // a 'currency' column
```

> **Changed in 3.6.0:** `currency`, `rating` and `progress` columns filter as numbers. On 3.5 they took the text operands. A text condition such as `contains` on one of them, from `filter()` or a saved state, is now dropped with a console warning.

### Cell template

Receives `ApexCellContext<T, K>` (`{ data, value, column, parent, row, commit? }`). `data` (4.0) is the record the cell was rendered for; `parent` and `row` are the cell and `<apex-grid-row>` elements. Must return a Lit `TemplateResult`:

```ts
import { html } from 'lit';

{ key: 'name', cellTemplate: ({ value, data }) => html`<strong>${value}</strong> <small>#${data.id}</small>` }
```

> **Changed in 4.0.0:** body rows are recycled by default (`rowRecycling`), so a row element and its cells can show another record after a scroll, sort or filter. Read the record from `ctx.data`, which stays that record even after an `await`; `ctx.row.data` is whatever the element shows now. Do not keep state in the elements a template creates, or wrap it in `keyed(ctx.data, ...)`, or set `grid.rowRecycling = false` (§1.6).

Since 3.6.0 a `cellTemplate` runs again for every visible row after each pipeline run (sort, filter, edit, quick filter, tree toggle), so a template that reads other fields of its row stays current. Keep templates cheap.

### Auto-generated columns — the 3-line demo

```ts
render(html`<apex-grid auto-generate .data=${data}></apex-grid>`, document.getElementById('app')!);
```

Auto-generates columns from `Object.keys(data[0])` and runtime types. `sort` / `filter` default to `false`, so the auto path is read-only. To re-trigger after a data swap, reset `grid.columns = []` first.

For full column patterns — derived columns, header/editor templates, per-column styling — see `references/columns-and-templates.md`.

---

## 5. Lifecycle & API

### Lifecycle pattern

```ts
import { setup } from 'apex-grid';
setup();                                                  // register + host sizing; no theme import

import { html, render } from 'lit';
render(html`<apex-grid></apex-grid>`, document.getElementById('app')!);

const grid = document.querySelector('apex-grid')!;
grid.columns = columns;                                   // properties, after element is in DOM
grid.data = data;

grid.addEventListener('sorted', (e) => {});               // UI events
grid.sort({ key: 'name', direction: 'ascending' });       // programmatic (silent)

// No destroy() — removing the element triggers Lit's disconnectedCallback automatically.
```

### Core properties

| Property | Type | Notes |
|---|---|---|
| `data` | `T[]` | Setting triggers a pipeline run. `null` / `undefined` (a framework prop still loading) shows an empty grid and reads back as `[]` (3.6). Replacing it with new records clears undo history and drops expansion of rows that left (`references/state-and-features.md`). |
| `columns` | `ColumnConfiguration<T>[]` | Each entry merged with column defaults. |
| `autoGenerate` (attr `auto-generate`) | `boolean` | Ignored if `columns` is non-empty. |
| `sortConfiguration` | `{ multiple, triState }` | Multi-column + tri-state sorting toggles. |
| `dataPipelineConfiguration` | `{ sort?, filter?, quickFilter? }` | Server-side / async hooks (§ data-pipeline). 4.0 removed the never-called `pagination` hook: page on the server with `pagination.mode: 'remote'`. |
| `rowRecycling` (attr `row-recycling`) | `boolean` | Default `true` (4.0): body row elements are re-bound to other records on scroll, sort and filter. `false` (`row-recycling="false"`) keeps record-keyed rows for templates that hold element state (§1.6). |
| `showHeader` (attr `show-header`) | `boolean` | Default `true`. `false` (4.0) leaves out the column header row, the column group row and the filter panel, so a short grid (a dashboard tile) gives that room to the rows. Screen readers still announce the column names; sort, filter, widths and the page are kept. While hidden, resizing, the column menu, drag reorder and the header filter button are unavailable. |
| `quickFilter` (attr `quick-filter`) | `string` | Global substring search across visible columns. |
| `showQuickFilter` / `showExport` | `boolean` | Toolbar UI toggles for quick-filter input / export menu. |
| `columnReordering` (attr `column-reordering`) | `boolean` | Drag-reorder headers; per-column opt-out via `reorderable`. |
| `columnMenu` | `boolean` | Header kebab-menu button. Default `true`; set `false` to hide. |
| `columnSeparator` (attr `column-separator`) | `boolean` | Persistent header dividers (also the resize handle). Default `true`; theme via `--ag-header-separator`. |
| `columnGroups` | `ColumnGroupConfiguration[]` | Spanning headers over contiguous columns (join via `column.group`). See `references/state-and-features.md`. |
| `localeText` | `GridLocaleText` | Partial locale-key → string overrides; ready-made `esLocale` export. Omitted keys fall back to English. |
| `rowId` | `(row: T) => string \| number` | Stable row-identity resolver so `getState` selection/expansion survives a data reload. |
| `displayColumns` *(getter)* | `readonly ColumnConfiguration<T>[]` | Columns in visual render order (start-pinned → unpinned → end-pinned). |
| `hasColumnGroups` / `columnGroupDepth` *(getters)* | `boolean` / `number` | Whether a group header row renders / its depth (0–1). |
| `pinnedRows` *(getter)* | `{ top: T[]; bottom: T[] }` | Currently pinned rows per band. |
| `canUndo` / `canRedo` *(getters)* | `boolean` | Undo/redo availability (needs `editing.history.enabled`). |
| `exportFormats` *(getter)* | `readonly { id, label }[]` | Export menu formats (CSV in the community grid). |
| `sortExpressions` / `filterExpressions` | `…[]` (get/set) | Current state. Assigning **replaces** the whole sort / filter, and `[]` or `null` clears it (3.6; on 3.5 assignment merged and `[]` did nothing). `sort()` / `filter()` add instead. `clearSort()` / `clearFilter()` also clear. |
| `selectedRows` / `expandedRows` | `T[]` (get/set) | Snapshots; set to replace (goes through cancellable events). |
| `rows` *(getter)* | `ApexGridRow<T>[]` | **Currently rendered only** (virtualized). With recycling (4.0 default) an element can show another record later: read its `data` each time, never keep it as a handle to a record. |
| `dataView` *(getter)* | `readonly T[]` | Full post-filter/sort data. |
| `pageItems` *(getter)* | `readonly T[]` | Current page slice (= `dataView` if no pagination), without pinned rows. Since 4.0 the same array while nothing that shapes it changes. |
| `totalItems` / `pageCount` *(getters)* | `number` | Row / page totals. With pagination, pinned rows are not counted (4.0); in remote mode `totalItems` is the `pagination.totalItems` you supply. |
| `page` / `pageSize` | `number` (get/set) | Current page (0-based) / page size. |

### Feature config properties (opt-in)

| Property | Shape |
|---|---|
| `editing` | `{ enabled, mode?: 'cell' \| 'row', trigger?: 'click' \| 'doubleClick', history?: { enabled, stackSize? } }` — plus per-column `editable: true` |
| `selection` | `{ enabled, mode?: 'single' \| 'multiple', showCheckboxColumn?, isRowSelectable? }`. `isRowSelectable(row)` (4.0) keeps chosen rows (a pinned total row) out of selection: no checkbox, and clicks, Shift/Ctrl clicks, Space, Select all, `selectRow`, `selectedRows = ...` and `setState` skip them; the header checkbox counts only selectable rows. Unset, every row is selectable, pinned rows included |
| `pagination` | `{ enabled, mode?: 'local' \| 'remote', page?, pageSize?, pageSizeOptions?, totalItems?, paginator?: 'full' \| 'compact' }`. `'full'` (default) drops the page size select, then first / last, when the grid is narrow; `'compact'` (4.0) always shows summary, previous, page / count, next |
| `tree` | `{ enabled, getDataPath: (row) => string[], groupColumnKey?, defaultExpanded?, childIndent? }`: flat data, hierarchy derived. Under a filter, the ancestors of a match show as their own rows (3.6) |
| `expansion` | `{ enabled, detailTemplate: (ctx) => TemplateResult, isExpandable?, showToggleColumn?, exportParts? }`: master-detail. `exportParts` (3.6) re-exports parts used in the detail content |
| `rowPinning` | `{ enabled }` — sticky top/bottom bands via `pinRow()` / `unpinRow()` |
| `rowReordering` | `{ enabled, applyToData?, handle? }`: drag/keyboard reorder + `moveRow()`. A sort clears the manual order, and a move clears the sort (3.6) |

```ts
grid.editing    = { enabled: true, mode: 'cell', trigger: 'doubleClick', history: { enabled: true } };
grid.selection  = { enabled: true, mode: 'multiple', showCheckboxColumn: true,
                    isRowSelectable: (row) => row !== totalRow };      // 4.0
grid.pagination = { enabled: true, pageSize: 25, paginator: 'compact' }; // 'compact': 4.0
grid.showHeader = false;                                                 // 4.0: no header rows (dashboard tile)
grid.rowPinning    = { enabled: true };
grid.rowReordering = { enabled: true };
```

Declarative validators (`column.validators`), column groups (`columnGroups`), undo/redo, and state persistence (`getState` / `setState` / `getSchema`) are covered in `references/state-and-features.md`.

### Methods

| Method | Description |
|---|---|
| `sort(expr \| expr[])` / `filter(expr \| expr[])` | Programmatic. **No events.** |
| `clearSort(key?)` / `clearFilter(key?)` | Clear all, or one column's, sort/filter state. |
| `getColumn(keyOrIndex)` | Find a column. |
| `updateColumns(col \| col[])` | Merge new fields into existing columns by `key`. Triggers a pipeline run. |
| `moveColumn(fromKey, toKey, position?)` | Reorder a column (within its pinning group). |
| `pinColumn(key, 'start' \| 'end' \| null)` / `unpinColumn(key)` | Programmatic column pinning → `Promise<boolean>`; emits `columnPinning`/`columnPinned`. |
| `exportToCSV(options?)` / `exportAs(formatId, options?)` | Export the data. CSV is built in. `options.source`: `'view'`(def)/`'page'`/`'selected'`/`'all'`. Formula-like CSV text (leading `=`, `+`, `-`, `@`, tab or CR; a plain signed number such as `-5` excepted) gets a leading apostrophe unless `escapeFormulas: false` (3.6). |
| pagination | `gotoPage(n)`, `setPageSize(n)`, `nextPage()`, `previousPage()`, `firstPage()`, `lastPage()` → `Promise<boolean>`; emit `pageChanging`/`pageChanged`. |
| `setQuickFilter(value)` | Apply global search (`''` clears) → `Promise<boolean>`; emits `quickFilterChanging`/`quickFilterChanged`. |
| selection | `selectRow`, `deselectRow`, `toggleRowSelection`, `selectAllRows`, `clearSelection`, `isRowSelected` |
| expansion | `expandRow`, `collapseRow`, `toggleRowExpansion`, `expandAllRows`, `collapseAllRows`, `isRowExpanded` |
| tree | `expandTreeRow`, `collapseTreeRow`, `toggleTreeRow`, `expandAllTreeRows`, `collapseAllTreeRows`, `isTreeRowExpanded` |
| row pinning | `pinRow(row, 'top' \| 'bottom')`, `unpinRow(row)` (needs `rowPinning.enabled`) |
| row reorder | `moveRow(from, to, position?)` (needs `rowReordering.enabled`) |
| editing | `commitEdit()`, `cancelEdit()` (used with `mode: 'row'`) |
| `applyEdits(edits)` | Write many cells as **one** operation: one undo step, one pipeline run. Returns a tally, not a count. (3.5) Each entry names its row by `rowIndex` (current page) or, since 3.6, by record: `{ row, column, value }` reaches any page, pinned rows and collapsed tree children. |
| `ownsEvent(event)` | `true` when an event came from this grid, `false` when from a grid nested in one of its master-detail panels. (3.6) |
| undo/redo | `undo()`, `redo()`, `clearHistory()` (needs `editing.history.enabled`) |
| state | `getState(options?)` → `GridState`, `setState(partial, options?)` → `SetStateResult`, `getSchema()` → `GridSchema` |
| `ApexGrid.register()` / `ApexGrid.tagName` | Static element registration / `'apex-grid'`. |

### Events — UI-initiated only

Every event fires only for operations driven through the grid's own UI; programmatic API calls are silent. The `*-ing` events are **cancellable** (call `event.preventDefault()`); the `*-ed` events are notifications. `event.detail` carries the relevant payload.

| Pairs (`-ing` cancellable / `-ed` after) | Triggered by |
|---|---|
| `sorting` / `sorted` | header sort |
| `filtering` / `filtered` | filter-row change |
| `quickFilterChanging` / `quickFilterChanged` | quick-filter input |
| `pageChanging` / `pageChanged` | paginator |
| `columnPinning` / `columnPinned` | pin/unpin |
| `columnMoving` / `columnMoved` | header drag reorder |
| `cellValueChanging` / `cellValueChanged` | inline edit commit |
| `rowEditStarted` / `rowEditEnded` | row-edit session (`mode: 'row'`) |
| `rowSelecting` / `rowSelected` | row selection |
| `rowExpanding` / `rowExpanded` | master-detail expand |
| `treeRowExpanding` / `treeRowExpanded` | tree-row expand |
| `rowPinning` / `rowPinned` | row pin / unpin (also fires for `pinRow()`/`unpinRow()`) |
| `rowMoving` / `rowMoved` | row reorder (also fires for `moveRow()`) |

**Exceptions to "UI-only".** The dedicated programmatic methods added for pagination (`gotoPage`, `nextPage`, …), quick-filter (`setQuickFilter`), column pinning (`pinColumn` / `unpinColumn`), row pinning (`pinRow` / `unpinRow`), and row reorder (`moveRow`) **do** emit their `*-ing` / `*-ed` pairs (unlike `sort()` / `filter()`, which stay silent). Since 3.6, `setQuickFilter` also fires `pageChanged` when it moves the grid back to the first page; assigning the `quickFilter` property stays silent.

**Timing (3.6).** `sorted`, `filtered`, `pageChanged` and `quickFilterChanged` fire after the pipeline run they caused, so a listener that reads `grid.dataView`, `grid.pageItems` or `grid.totalItems` sees the new rows, even with an async `dataPipelineConfiguration` hook. On 3.5 they could fire before an async hook resolved.

**Nested grids (3.6).** Grid events bubble and are composed, so a listener on a grid also hears the events of a grid nested in one of its master-detail panels. Filter them inside the listener:

```ts
grid.addEventListener('cellValueChanged', (e) => {
  if (!grid.ownsEvent(e)) return;          // came from a grid inside a detail panel
  save(e.detail.data);
});
```

Single-shot events (no `-ing` pair):

| Event | `event.detail` | Fires when |
|---|---|---|
| `cellValidationFailed` | `{ key, rowIndex, data, value, errors: string[] }` | a candidate value is rejected by `column.validators` (keeps editor open) |
| `historyChanged` | `{ canUndo, canRedo }` | undo/redo stacks change (record / undo / redo / clear) |
| `stateChanged` | `{ state: GridState }` | restorable state changes (debounced; UI or programmatic incl. `setState`) |

Operand reference, multi-column sort, and server-side hooks: `references/sort-and-filter.md` and `references/data-pipeline.md`. Row pinning/reorder, undo/redo, validators, column groups, state/schema, and localization: `references/state-and-features.md`.

### Keyboard & accessibility (3.4 and later)

The grid ships full keyboard operation and screen-reader support out of the box; there's nothing to enable. The body uses a **roving-tabindex** focus model: the active cell is the body's single tab stop, so `Tab` moves into and out of the grid in one step, and real focus follows arrow-key navigation (screen readers track cell-to-cell movement).

| Key | Moves / does |
|---|---|
| Arrow keys | Move the active cell one cell up / down / left / right |
| Home / End | Move to the **first / last cell of the current row** |
| Ctrl/Cmd+Home / Ctrl/Cmd+End | Jump to the grid's first / last cell |
| PageUp / PageDown | Move the active cell by the rows that fit in the visible body (below the sticky header and pinned rows) |
| Enter or F2 | Open the active cell's editor (editing is no longer pointer-only) |

Keyboard-driven commit (Enter) or cancel (Escape) returns focus to the cell, so navigation continues from where you left off. Since 3.6.0, after tabbing into the body with no cell active yet, the first arrow, Home or PageUp key activates the first cell instead of moving past it (ArrowDown lands on row 0).

**Editing behavior:** clearing a `number` or `currency` editor now commits `null` (previously it wrote `NaN`). An empty or unparseable numeric input leaves the cell empty rather than poisoning the row. Since 3.6.0 an open editor stays on the record it was opened for when the data, sort, filter or page change under it, and the edit is cancelled if that record leaves the view; row mode edits a staged copy (`references/state-and-features.md`).

**Announcements:** sorting, filtering, selection, paging, row pinning, row reorder, expansion, and undo / redo each announce through a localized live region (a page change once, "Page X of N"). The strings are locale keys (`announce.*`, `header.*`, `grid.label`, `editor.rating`, and since 4.0 `cell.rating`, `cell.true` / `cell.false`); override them via `localeText` (see `references/state-and-features.md`).

> **Changed in 4.0.0:** the `grid` / `treegrid` role, `aria-rowcount`, `aria-colcount`, `aria-multiselectable` (multi-row selection) and the localized accessible name ("Data grid", `grid.label`) are on the inner `[part~="table"]` element, not the `<apex-grid>` host. It holds only the header rows, the pinned row bands and the body, so the toolbar, paginator, filter panel and live region are no longer read as table content. An `aria-label` / `aria-labelledby` you put on the host still names the grid. The column header row is no longer a Tab stop.

**Input methods and typed numbers (4.0):** Japanese, Chinese or Korean input through an IME works in the cell editor, the filter panel and the quick filter: the Enter, Escape and Tab that confirm or cancel a conversion stay with the IME (they do not commit or cancel an edit, close the filter panel or clear the quick filter), and the filter panel and quick filter apply the text once the conversion ends. A number typed in the filter panel, or checked by `min()` / `max()`, is read as a person writes it: full-width digits and signs (`１２３`, `－５`), the grouping and decimal separator of the column's `locale` (else the runtime locale, so `1.234,5` is 1234.5 under `'de-DE'`), and a leading or trailing `¥`, `$`, `€`, `円` or the column currency's symbol. A plain ASCII decimal reads as before in every locale (`1.000` stays 1 even under `'de-DE'`), and other text splits: the filter panel falls back to its leading number as in 3.x (`12.5%` filters by 12.5, `10²` by 10), while `min()` / `max()` read `12.5%` or `10²` as not a number, so the value fails. Programmatic `filter()` search terms are not parsed this way: pass numbers.

**New in 3.5, all automatic:** the grid honours `prefers-reduced-motion` (every transition routes through the `--ag-dur-*` tokens, which zero under the media query), it stays usable under forced colors / Windows High Contrast (the active-cell ring, invalid-cell ring, selected-row bar and focus rings are painted as outlines, which forced colors recolours rather than drops), and interactive targets meet the WCAG 2.2 AA 24x24 minimum. The only visible change is that the paginator bar is 2px taller.

### Right-to-left (3.5)

Set `dir="rtl"` on the grid or any ancestor and it mirrors completely. There is no property to set and nothing to import: direction is read from the computed style, so an inherited `dir` works.

```html
<html dir="rtl">
  <apex-grid></apex-grid>   <!-- inherits; no grid-level config -->
</html>
```

Arrow-key navigation, column resize and column reorder all follow inline order rather than screen direction, so ArrowLeft moves to the *next* column in RTL. Column pinning `'start'` / `'end'` were already logical and are unchanged.

---

## 6. Pitfalls — ❌ Wrong vs ✅ Correct

### 1. Treating it like a class API
❌ `const grid = new ApexGrid({ container: '#app', columns, data })`
✅ Render `<apex-grid .data=${data} .columns=${columns}></apex-grid>`.

### 2. Forgetting to register the element
❌ Importing `ApexGrid` but never registering.
✅ `setup()` (or `import 'apex-grid/define'`) once at app startup.

### 3. Importing a theme CSS that no longer exists
❌ `import 'igniteui-webcomponents/themes/light/bootstrap.css'` + `setup({ theme: 'bootstrap' })` is obsolete; it never affected the grid, and as of 3.5.0 the option is inert and that package is not even installed.
✅ Nothing to import for a styled grid: it styles itself; retheme via `--ag-*` CSS variables. The only required CSS is a host **height**. To follow an Ignite UI theme that the page loads, the path is the `apex-grid/ignite-ui` add-on (apex-grid 4.0, §1.3), not a theme import into the grid.

### 4. Stringified `columns` attribute
❌ `<apex-grid columns="[...]">` — becomes the literal string `"[object Object],..."`.
✅ Property binding: `.columns=${columns}` / `:columns.prop="columns"` / `[columns]="columns"` / `el.columns = columns`.

### 5. `column.key` not a real key of `T`
❌ `{ key: 'fullName' }` when `User` has only `firstName` + `lastName`.
✅ Compute via template: `cellTemplate: ({ data }) => html\`${data.firstName} ${data.lastName}\``. The context's `data` is the record (4.0); `row` is the row element, not the record.

### 6. String from `cellTemplate`
❌ `cellTemplate: ({ value }) => \`<strong>\${value}</strong>\`` — renders as escaped text.
✅ `cellTemplate: ({ value }) => html\`<strong>\${value}</strong>\``.

### 7. Numbers sorted as strings
❌ `{ key: 'age', sort: true }` (default `type: 'string'`) — "10" sorts before "9".
✅ `{ key: 'age', type: 'number', sort: true }`.

### 8. Reaching for a custom comparer to display dates
❌ `{ key: 'created', type: 'string', sort: { comparer: ... } }` to fake date handling.
✅ `{ key: 'created', type: 'date', format: 'medium' }` — a real type; accepts `Date`, ISO strings, or millisecond timestamps and sorts chronologically.

### 9. Clearing sort/filter
❌ Assuming you must rebuild state to clear it.
✅ `grid.clearSort()` / `grid.clearFilter()` (optionally per `key`), or `grid.sortExpressions = []` / `grid.filterExpressions = []` (3.6).

### 10. Expecting events from programmatic operations
❌ Listening for `sorting` after calling `grid.sort(...)` — never fires.
✅ Wrap the programmatic call yourself, or watch `grid.dataView` for any state change.

### 11. Building a multi-column sort one assignment at a time
❌ `grid.sortExpressions = [byCity]; grid.sortExpressions = [byAge];` expecting both: since 3.6 each assignment **replaces** the sort (the same for `filterExpressions`), so only `byAge` applies.
✅ Assign the full list, `grid.sortExpressions = [byCity, byAge]`, or add with `grid.sort(byAge)` / `grid.filter(expr)`.

### 12. Theming a dark grid with the light tokens
❌ `apex-grid[theme='dark'] { --ag-surface: #111827; --ag-text: #f9fafb; }`: since 3.6 a dark grid ignores the light colour tokens, even on its own element.
✅ `apex-grid { --ag-dark-surface: #111827; --ag-dark-text: #f9fafb; }` (or on any ancestor). Brand, sizing and typography tokens are shared, so `--ag-brand` still applies to both.

### 13. Expecting an Ignite UI theme to restyle the grid without the add-on (4.0)
❌ Loading an Ignite UI theme stylesheet and expecting the 4.0 grid to follow it: without `apex-grid/ignite-ui` every `--ig-*` token is silently ignored (3.x picked up the brand and font on its own; 4.0 does not). Also ❌ `theme="light"` on a page that switches to a dark Ignite UI theme: the pinned mode blocks the dark variant. Also ❌ `import 'apex-grid/ignite-ui'` on apex-grid 3.x: the module does not exist there.
✅ apex-grid 4.0, then `import 'apex-grid/ignite-ui'` once (free, no licence key) and leave the `theme` attribute off so the grid follows `--ig-theme-variant`. Use `design-system="off"` or `tokens="off"` to opt one grid out.

### 14. Treating a row element as a record (4.0)
❌ A click handler in a `cellTemplate` that reads `ctx.row.data` after an `await`, or code that keeps `grid.rows[0]` as a handle: with recycling (the 4.0 default) that element can show another record after a scroll, sort or filter, so the code acts on the wrong row. Also ❌ an `<input>` or third-party widget in a `cellTemplate` that keeps its own typed state: the state shows on another record after a scroll.
✅ Read `ctx.data` (it stays the record the cell rendered, across an `await`), keep state in your data rather than in the elements, wrap a stateful template in `keyed(ctx.data, html\`...\`)` (`lit/directives/keyed.js`), or set `grid.rowRecycling = false` for record-keyed rows.

### 15. Reading the grid's ARIA from the host (4.0)
❌ `expect(grid.getAttribute('role')).toBe('grid')` or `grid.getAttribute('aria-rowcount')` in a test: the host no longer carries them.
✅ `grid.shadowRoot!.querySelector('[part~="table"]')!.getAttribute('role')`. Name the grid with `aria-label` / `aria-labelledby` on the host as before.

---

## 7. Reference Routing Table

| Topic | Reference File |
|---|---|
| Upgrading an app from 3.x to 4.0 | §1.6 |
| Column types, templates (`ctx.data`, row recycling), auto-generation, per-column styling, header/editor templates | `references/columns-and-templates.md` |
| Sort & filter — operands, expressions, multi-column, quick-filter, events | `references/sort-and-filter.md` |
| Server-side data hooks (sort/filter/quickFilter), remote pagination, virtualization, `dataView` | `references/data-pipeline.md` |
| Row pinning / reordering, undo-redo, batch edits (`applyEdits`), validators, row-mode editing, column groups, state & schema (`getState`/`setState`/`getSchema`), localization, export source and CSV escaping, master-detail parts and nested grids (`exportParts`, `ownsEvent`) | `references/state-and-features.md` |
| End-to-end vanilla JS (no Lit `render`) | `references/vanilla-js.md` |
| Lit, React, Vue, Angular integration | `references/framework-integration.md` |
