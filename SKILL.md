---
name: apex-grid
description: >
  AI skill for building UIs with the `apex-grid` web component (a Lit-based
  data grid by ApexCharts). Use whenever the user asks to create, configure, or
  troubleshoot a sortable / filterable / paginated / editable / virtualized data
  table with `<apex-grid>`. Covers the register-and-size setup (no theme import —
  the grid styles itself via `--ag-*` CSS variables), the generic
  `ColumnConfiguration<T>` shape with its 13 column types, Lit `cellTemplate`
  requirements, sorting / filtering / quick-filter, pagination, inline editing,
  row selection, master-detail and tree rows, CSV export, the UI-only event
  model, and the `dataPipelineConfiguration` hooks for server-side data.
metadata:
  author: ApexCharts
  version: "2.1.0"
  library_version: "3.3.0"
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

`igniteui-webcomponents` is a regular (transitive) dependency — it installs automatically, and you do **not** import any theme CSS from it.

> Confirmed working versions: `apex-grid@3.0.1`, `lit@^3.0.0`.

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

> **Do not set `display` on `apex-grid`** — the component declares `:host { display: grid }` for its internal track layout; an outer `display` rule overrides it and collapses the virtualizer.

### 1.3 Styling — `--ag-*` CSS custom properties (no theme import)

The grid is styled out of the box and customized entirely through `--ag-*` CSS custom properties — there is **no theme file to import and no `configureTheme()` call**. Set the variables on `apex-grid` (or an ancestor):

```css
apex-grid {
  height: 480px;
  /* override any --ag-* variables to retheme; see the package README for the full list */
}
```

> The deprecated `theme` option (and `igniteui-webcomponents`' `configureTheme()`) does **not** change the grid's appearance — it only forwards to igniteui for apps that embed igniteui components alongside the grid. Omit it; use `--ag-*` variables instead.

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

> **Dev-server reminder.** ES module imports need an HTTP server — opening via `file://` fails with CORS errors. Use Vite (`npm create vite@latest`), any static server, or your framework's dev server.

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

4. **`column.key` must be a real `keyof T`.** No `field` / `dataIndex` / `accessor` aliases. To display a derived value, compute it in `cellTemplate` (which has `row` access).

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
  locale?: string;                         // 'currency' BCP 47 locale
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
| `'progress'` | health bar, 0..`max` (default 100) |
| `'sparkline'` | inline trend chart of a `number[]` (`showDelta`) |
| `'status'` | dot + pill (`statusVariant`, or inferred) |

> A `'date'` column **does** exist in 3.x — use `type: 'date'` (it accepts `Date`, ISO strings, or millisecond timestamps), not a string column with a custom comparer.

### Column type → filter operands (primitive types)

| `type` | Operand keys in the filter UI |
|---|---|
| `'string'` | `contains`, `doesNotContain`, `startsWith`, `endsWith`, `equals`, `doesNotEqual`, `empty`, `notEmpty` |
| `'number'` | `equals`, `doesNotEqual`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `empty`, `notEmpty` |
| `'boolean'` | `all`, `true`, `false`, `empty`, `notEmpty` (all **unary** — no `searchTerm`) |

```ts
grid.filter({ key: 'subscribed', condition: 'true' });   // unary — no searchTerm
```

### Cell template

Receives `ApexCellContext<T, K>` (`{ parent, row, column, value, commit? }`). Must return a Lit `TemplateResult`:

```ts
import { html } from 'lit';

{ key: 'name', cellTemplate: ({ value, row }) => html`<strong>${value}</strong>` }
```

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
| `data` | `T[]` | Setting triggers a pipeline run. |
| `columns` | `ColumnConfiguration<T>[]` | Each entry merged with column defaults. |
| `autoGenerate` (attr `auto-generate`) | `boolean` | Ignored if `columns` is non-empty. |
| `sortConfiguration` | `{ multiple, triState }` | Multi-column + tri-state sorting toggles. |
| `dataPipelineConfiguration` | `{ sort?, filter?, pagination?, quickFilter? }` | Server-side / async hooks (§ data-pipeline). |
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
| `sortExpressions` / `filterExpressions` | `…[]` (get/set) | Current state. To clear, use `clearSort()` / `clearFilter()`. |
| `selectedRows` / `expandedRows` | `T[]` (get/set) | Snapshots; set to replace (goes through cancellable events). |
| `rows` *(getter)* | `ApexGridRow<T>[]` | **Currently rendered only** (virtualized). |
| `dataView` *(getter)* | `readonly T[]` | Full post-filter/sort data. |
| `pageItems` *(getter)* | `readonly T[]` | Current page slice (= `dataView` if no pagination). |
| `totalItems` / `pageCount` *(getters)* | `number` | Row / page totals. |
| `page` / `pageSize` | `number` (get/set) | Current page (0-based) / page size. |

### Feature config properties (opt-in)

| Property | Shape |
|---|---|
| `editing` | `{ enabled, mode?: 'cell' \| 'row', trigger?: 'click' \| 'doubleClick', history?: { enabled, stackSize? } }` — plus per-column `editable: true` |
| `selection` | `{ enabled, mode?: 'single' \| 'multiple', showCheckboxColumn? }` |
| `pagination` | `{ enabled, mode?: 'local' \| 'remote', page?, pageSize?, pageSizeOptions?, totalItems? }` |
| `tree` | `{ enabled, getDataPath: (row) => string[], groupColumnKey?, defaultExpanded?, childIndent? }` — flat data, hierarchy derived |
| `expansion` | `{ enabled, detailTemplate: (ctx) => TemplateResult, isExpandable?, showToggleColumn? }` — master-detail |
| `rowPinning` | `{ enabled }` — sticky top/bottom bands via `pinRow()` / `unpinRow()` |
| `rowReordering` | `{ enabled, applyToData?, handle? }` — drag/keyboard reorder + `moveRow()`; manual order is mutually exclusive with sort |

```ts
grid.editing    = { enabled: true, mode: 'cell', trigger: 'doubleClick', history: { enabled: true } };
grid.selection  = { enabled: true, mode: 'multiple', showCheckboxColumn: true };
grid.pagination = { enabled: true, pageSize: 25 };
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
| `exportToCSV(options?)` / `exportAs(formatId, options?)` | Export the data. CSV is built in. `options.source`: `'view'`(def)/`'page'`/`'selected'`/`'all'`. |
| pagination | `gotoPage(n)`, `setPageSize(n)`, `nextPage()`, `previousPage()`, `firstPage()`, `lastPage()` → `Promise<boolean>`; emit `pageChanging`/`pageChanged`. |
| `setQuickFilter(value)` | Apply global search (`''` clears) → `Promise<boolean>`; emits `quickFilterChanging`/`quickFilterChanged`. |
| selection | `selectRow`, `deselectRow`, `toggleRowSelection`, `selectAllRows`, `clearSelection`, `isRowSelected` |
| expansion | `expandRow`, `collapseRow`, `toggleRowExpansion`, `expandAllRows`, `collapseAllRows`, `isRowExpanded` |
| tree | `expandTreeRow`, `collapseTreeRow`, `toggleTreeRow`, `expandAllTreeRows`, `collapseAllTreeRows`, `isTreeRowExpanded` |
| row pinning | `pinRow(row, 'top' \| 'bottom')`, `unpinRow(row)` (needs `rowPinning.enabled`) |
| row reorder | `moveRow(from, to, position?)` (needs `rowReordering.enabled`) |
| editing | `commitEdit()`, `cancelEdit()` (used with `mode: 'row'`) |
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

**Exceptions to "UI-only".** The dedicated programmatic methods added for pagination (`gotoPage`, `nextPage`, …), quick-filter (`setQuickFilter`), column pinning (`pinColumn` / `unpinColumn`), row pinning (`pinRow` / `unpinRow`), and row reorder (`moveRow`) **do** emit their `*-ing` / `*-ed` pairs (unlike `sort()` / `filter()`, which stay silent).

Single-shot events (no `-ing` pair):

| Event | `event.detail` | Fires when |
|---|---|---|
| `cellValidationFailed` | `{ key, rowIndex, data, value, errors: string[] }` | a candidate value is rejected by `column.validators` (keeps editor open) |
| `historyChanged` | `{ canUndo, canRedo }` | undo/redo stacks change (record / undo / redo / clear) |
| `stateChanged` | `{ state: GridState }` | restorable state changes (debounced; UI or programmatic incl. `setState`) |

Operand reference, multi-column sort, and server-side hooks: `references/sort-and-filter.md` and `references/data-pipeline.md`. Row pinning/reorder, undo/redo, validators, column groups, state/schema, and localization: `references/state-and-features.md`.

---

## 6. Pitfalls — ❌ Wrong vs ✅ Correct

### 1. Treating it like a class API
❌ `const grid = new ApexGrid({ container: '#app', columns, data })`
✅ Render `<apex-grid .data=${data} .columns=${columns}></apex-grid>`.

### 2. Forgetting to register the element
❌ Importing `ApexGrid` but never registering.
✅ `setup()` (or `import 'apex-grid/define'`) once at app startup.

### 3. Importing a theme CSS that no longer exists
❌ `import 'igniteui-webcomponents/themes/light/bootstrap.css'` + `configureTheme('bootstrap')` — obsolete; does not affect the grid.
✅ Nothing to import — the grid styles itself; retheme via `--ag-*` CSS variables. The only required CSS is a host **height**.

### 4. Stringified `columns` attribute
❌ `<apex-grid columns="[...]">` — becomes the literal string `"[object Object],..."`.
✅ Property binding: `.columns=${columns}` / `:columns.prop="columns"` / `[columns]="columns"` / `el.columns = columns`.

### 5. `column.key` not a real key of `T`
❌ `{ key: 'fullName' }` when `User` has only `firstName` + `lastName`.
✅ Compute via template: `cellTemplate: ({ row }) => html\`${row.firstName} ${row.lastName}\``.

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
✅ `grid.clearSort()` / `grid.clearFilter()` (optionally per `key`).

### 10. Expecting events from programmatic operations
❌ Listening for `sorting` after calling `grid.sort(...)` — never fires.
✅ Wrap the programmatic call yourself, or watch `grid.dataView` for any state change.

---

## 7. Reference Routing Table

| Topic | Reference File |
|---|---|
| Column types, templates, auto-generation, per-column styling, header/editor templates | `references/columns-and-templates.md` |
| Sort & filter — operands, expressions, multi-column, quick-filter, events | `references/sort-and-filter.md` |
| Server-side data hooks (sort/filter/pagination/quickFilter), virtualization, `dataView` | `references/data-pipeline.md` |
| Row pinning / reordering, undo-redo, validators, column groups, state & schema (`getState`/`setState`/`getSchema`), localization, export source | `references/state-and-features.md` |
| End-to-end vanilla JS (no Lit `render`) | `references/vanilla-js.md` |
| Lit, React, Vue, Angular integration | `references/framework-integration.md` |
