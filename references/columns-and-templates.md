# Column Configuration & Lit Templates

> **Before you start**: this reference assumes the Quickstart setup from `SKILL.md` is done — element registered and the host element sized (e.g. `setup()`). **No theme CSS import is needed**; the grid self-styles via `--ag-*` CSS variables. Without a sized host, no column configuration will produce a visible grid.

## A note on column widths

If you omit `width`, the column flex-fills the remaining space. With one un-widthed column among several explicit ones, that column **stretches to fill the grid** — including pushing its sort arrow to the far edge of the viewport, which looks like a phantom column. For predictable layouts, set explicit widths on every column.

## `ColumnConfiguration<T, K>`

Apex Grid is generic over a row type `T extends object`. Every column is typed against a key `K extends keyof T` so the cell template's `value` is correctly inferred.

```ts
interface BaseColumnConfiguration<T extends object, K extends keyof T = keyof T> {
  key: K;                                          // REQUIRED — keyof T
  type?: DataType;                                 // default 'string' (see "Column types" below)
  headerText?: string;                             // default: String(key)
  width?: string;                                  // any CSS width unit
  hidden?: boolean;                                // default false
  resizable?: boolean;                             // default false
  pinned?: 'start' | 'end' | null;                 // freeze to a side during horizontal scroll
  reorderable?: boolean;                           // per-column opt-out of drag reorder
  exportable?: boolean;                            // default true; false omits from CSV export
  group?: string;                                  // id of a columnGroups spanning header (state-and-features.md)
  editable?: boolean;                              // requires grid editing.enabled
  validators?: Validator<T, K>[];                  // edit-time validation (see below)
  sort?: boolean | { caseSensitive?: boolean; comparer?: (a: T[K], b: T[K]) => number };
  filter?: boolean | { caseSensitive?: boolean };
  headerTemplate?: (ctx: ApexHeaderContext<T>) => TemplateResult | unknown;
  cellTemplate?: (ctx: ApexCellContext<T, K>) => TemplateResult | unknown;
  editorTemplate?: (ctx: ApexEditorContext<T, K>) => TemplateResult | unknown;
  // type-specific (ignored unless `type` matches):
  options?: (V | { value: V; label?: string })[]; // 'select'
  max?: number;                                    // 'rating' (stars, def 5) / 'progress' (full, def 100)
  format?: 'short' | 'medium' | 'long' | 'full';   // 'date' (def 'medium')
  shape?: 'square' | 'circle';                     // 'image' (def 'square')
  alt?: string;                                    // 'image' alt text
  currency?: string;                               // 'currency' ISO 4217 (def 'USD')
  locale?: string;                                 // 'currency' BCP 47 locale; 4.0: also the separators for numbers typed into the filter panel, or checked by min() and max() (enterprise: paste, advanced filter)
  badgeVariant?: 'gold' | 'brand' | 'neutral' | 'muted' | ((v) => …);  // 'badge'
  statusVariant?: 'active' | 'trial' | 'churn' | ((v) => …);           // 'status'
  showDelta?: boolean;                             // 'sparkline' (def true)
  sparkline?: { label?: string | ((value, data: T) => string) };   // 'sparkline' (5.1): accessible name
  chart?: {                                        // 'chart' (4.1): an ApexCharts chart per cell
    type?: string; height?: number; width?: number | string;   // def 'line', 28, '100%'
    options?: object | ((value, data: T) => object);           // ApexOptions over the row defaults
    label?: (value, data: T) => string;                         // accessible name
  };
}
```

### Column types (`DataType`)

`'string'` / `'number'` / `'boolean'` are the primitive types driving sort, filter, and the default editor. The rest are presentation renderers over a primitive value (and sort/filter as that value): `'select'` (label from `options`, `<select>` editor), `'rating'` (stars, 0..`max`), `'date'` (locale date via `format`; accepts `Date`/ISO/timestamp; `<input type="date">` editor), `'image'` (`<img>`, `shape`/`alt`), `'currency'` (`Intl.NumberFormat`, `currency`/`locale`), `'avatar'`, `'badge'` (`badgeVariant`), `'progress'` (0..`max`), `'sparkline'` (`number[]`, `showDelta`), `'chart'` (4.1: an ApexCharts chart per cell, configured with `chart`; needs `setApexCharts(ApexCharts)` or a script-tag global, ApexCharts 8.4.1+), `'status'` (`statusVariant`). Of these, `'currency'`, `'rating'` and `'progress'` are numeric: since 3.6 they filter with the number operands and export as numbers, like `'number'`.

Defaults applied on assignment (`DEFAULT_COLUMN_CONFIG`):

```ts
{ type: 'string', resizable: false, hidden: false, sort: false, filter: false }
```

## Type field — what it actually does

| `type` | Default sort comparer | Filter operands |
|---|---|---|
| `'string'` *(default)* | `String#localeCompare` (case-insensitive unless `caseSensitive: true`) | `contains`, `doesNotContain`, `startsWith`, `endsWith`, `equals`, `doesNotEqual`, `empty`, `notEmpty` |
| `'number'` | numeric `<` / `>` | `equals`, `doesNotEqual`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `empty`, `notEmpty` |
| `'boolean'` | `false < true` | `all`, `true`, `false`, `empty`, `notEmpty` |

The table above lists the three **primitive** types. The presentation types (`'select'`, `'rating'`, `'date'`, `'image'`, `'currency'`, `'avatar'`, `'badge'`, `'progress'`, `'sparkline'`, `'chart'`, `'status'`) sort as their underlying value. For filtering, `'currency'`, `'rating'` and `'progress'` take the `'number'` operands (since 3.6; on 3.5 they took the text ones), and the others take the `'string'` operands. With the built-in comparison, empty cells (`null`, `undefined`, `NaN`, invalid `Date`) sort last in both directions (3.6). For dates, use the real `'date'` type: it accepts `Date`, ISO strings, or millisecond timestamps, renders a locale date (`format`), and sorts chronologically. No timestamp/comparer workaround needed:

```ts
{ key: 'createdAt', type: 'date', format: 'medium', sort: true, filter: true }
```

## Sort configuration per column

```ts
// Boolean shorthand — uses the type-default comparer
{ key: 'name', sort: true }

// Object form — case-sensitive comparison
{ key: 'name', sort: { caseSensitive: true } }

// Custom comparer (overrides the type-default)
{ key: 'priority',
  type: 'string',
  sort: { comparer: (a, b) => ['low','standard','high'].indexOf(a) - ['low','standard','high'].indexOf(b) },
}
```

The `comparer` signature is fully typed against `T[K]`. Since 3.6 it receives the **raw** cell values, original casing and `null` / `undefined` included, and owns their order (a descending sort reverses its result). On 3.5 it received strings already lower-cased (unless `caseSensitive`), so a lookup written in the data's casing (`'High'`, `'Low'`) never matched and the sort did nothing. Now the lookup above matches only values stored in lower case: write it in the data's own casing, and decide where empty values go.

## Filter configuration per column

```ts
{ key: 'name', filter: true }                              // case-insensitive
{ key: 'name', filter: { caseSensitive: true } }           // case-sensitive
```

There is no per-column `defaultOperand` field — operands come from `type`.

## Cell template (`cellTemplate`)

Receives `ApexCellContext<T, K>`:

```ts
interface BaseApexCellContext<T, K> {
  readonly data: T;                 // 4.0: the record this cell was rendered for; stays it after an await
  parent: ApexGridCell<T>;          // <apex-grid-cell> element (may later show another record)
  row:    ApexGridRow<T>;           // <apex-grid-row> element (may later show another record)
  column: ColumnConfiguration<T, K>;
  value:  T[K];                     // T[K] — fully typed
  commit?: (value: T[K]) => Promise<boolean>;   // editable columns only: write without opening the editor; writes to ctx.data
}
```

`ApexEditorContext` extends it with `commit(value)` and `cancel()`, so an `editorTemplate` gets `ctx.data` too. Unlike the cell context's `commit`, the editor's `commit` commits whichever cell is in edit when it is called, so do not call it after an `await` that can outlive the edit.

### Row recycling (Changed in 4.0.0)

Body rows are recycled by default (`grid.rowRecycling`, default `true`): a scroll, sort or filter re-binds the rendered row elements to the records now in view instead of building new rows, which makes long jumps and sorts cheaper on large grids. A row element and its cells can therefore show another record later. Templates must:

- **derive what they render from the context on each render**, and read the record from `ctx.data` (on 3.x the usual spelling was `ctx.row.data`, which now reads whatever record the element shows at that moment);
- **not hold `ctx.row` or `ctx.parent` across an `await`** (a `ctx.commit` after an `await` is safe: it writes to `ctx.data`);
- **not keep state in the elements they create** (an input's typed value, a third-party widget), or wrap the template in `keyed(ctx.data, ...)` so the state never shows on the wrong record. For an ApexCharts chart, use `type: 'chart'` (4.1) instead: it keeps one chart per cell and re-points it, where `keyed` builds a new chart every time a row shows another record (thousands per scroll on a large grid).

```ts
import { html } from 'lit';
import { keyed } from 'lit/directives/keyed.js';

{ key: 'notes',
  cellTemplate: (ctx) => keyed(ctx.data, html`<my-rich-note .value=${ctx.value}></my-rich-note>`) }
```

To keep record-keyed rows (each element stays with its record), opt out: `grid.rowRecycling = false` or `<apex-grid row-recycling="false">`. Detail panel content follows its record and keeps its DOM state without any change, and a grid created inside a `detailTemplate` keeps its own setting. Pinned rows are not recycled, but a pinned band can re-bind its elements (when a row above is unpinned), which is why `ctx.data` is the safe read everywhere.

**Custom elements in a cell (5.1).** A scroll step moves the rows that left the view to its other end, and moving a row disconnects and reconnects every custom element in it. From 5.1 a moved row already shows its new record when it reconnects, so `connectedCallback` reads the right data (on 4.x and 5.0 it usually saw the record of the row it was leaving). Two more rules keep an element that hosts a widget (a chart, a map, anything built on connect) correct and cheap:

- **Defer tear-down.** In `disconnectedCallback`, check `isConnected` a microtask later and tear down only if the element is still off the page. A move reconnects it within the same task.
- **React to the data property** (a setter, or a Lit reactive property) rather than reading it once on connect. A sort, a filter that keeps the rendered range, and a long jump (a scrollbar drag, setting `scrollTop`, a page change) re-bind rows where they stand, without disconnecting them, so connect never runs for the new record.

```ts
// `Widget` and `createWidget` stand for whatever the cell draws with.
class SparkCell extends HTMLElement {
  #record?: Row;
  #widget?: Widget;
  set record(record: Row) {
    this.#record = record;
    this.#widget?.update(record); // a re-bind, in place or ahead of a move
  }
  connectedCallback() {
    this.#widget ??= createWidget(this, this.#record); // a move reconnects an element that has its widget
  }
  disconnectedCallback() {
    queueMicrotask(() => {
      if (this.isConnected) return; // moved, not removed
      this.#widget?.destroy();
      this.#widget = undefined;
    });
  }
}
```

Built-in `type: 'chart'` cells already work this way.

Since 3.6 a `cellTemplate` runs again for every visible row, body and pinned, after each pipeline run (sort, filter, quick filter, edit, tree toggle), so a template that reads other fields of `ctx.data` stays current after an edit to another cell. Keep templates cheap; plain and typed cells are not re-rendered this way. In row edit mode, `commit(value)` stages the value with the row edit instead of writing straight to the record (3.6), so `cancelEdit()` rolls it back.

```ts
import { html } from 'lit';

const columns: ColumnConfiguration<User>[] = [
  // Direct value — no template needed
  { key: 'id', type: 'number' },

  // Inline emphasis
  { key: 'name',
    cellTemplate: ({ value }) => html`<strong>${value}</strong>` },

  // Conditional render with the record (ctx.data, 4.0)
  { key: 'subscribed', type: 'boolean',
    cellTemplate: ({ value, data }) => html`
      <input type="checkbox" .checked=${value} disabled />
      <small>${data.name}</small>` },

  // Prefer the built-in type over a template where one exists:
  // `avatar` and `rating` are column types, no template needed.
  { key: 'avatar', type: 'avatar' },
  { key: 'satisfaction', type: 'rating' },

  // Any custom element works, but YOU must install and register it:
  // the grid ships no third-party components (see SKILL.md §1.1).
  { key: 'status',
    cellTemplate: ({ value }) => html`<my-status-chip .value=${value}></my-status-chip>` },
];
```

**Templates must return `TemplateResult`** (the value of an `html\`...\`` literal). Returning a string renders as escaped text.

## Header template (`headerTemplate`)

Receives `ApexHeaderContext<T>`:

```ts
interface ApexHeaderContext<T> {
  parent: ApexGridHeader<T>;
  column: ColumnConfiguration<T>;
}
```

```ts
{
  key: 'age',
  headerTemplate: ({ column }) => html`
    <span style="display:flex;align-items:center;gap:4px;">
      <span>${column.headerText ?? String(column.key)}</span>
      <small style="opacity:.6;">(yrs)</small>
    </span>`,
}
```

## Column validators (`validators`) — 3.2+

`column.validators` run before a candidate value is written (inline edit, row-mode, and bulk paste / fill). Each returns an error `string` (reject) or `null` (pass); all run and every message is collected. A failing commit keeps the editor open, marks the cell `aria-invalid`, and emits `cellValidationFailed`. Requires the column to be `editable` and grid `editing.enabled`.

```ts
import { required, min, max, pattern, custom } from 'apex-grid';

{ key: 'age', type: 'number', editable: true, validators: [required(), min(0), max(120)] }
{ key: 'email', editable: true, validators: [pattern(/^\S+@\S+$/, 'Invalid email')] }
{ key: 'code', editable: true,
  validators: [custom((value, ctx) => (value === ctx.data.id ? 'Cannot equal id' : null))] }
```

Built-in factories: `required(message?)`, `min(limit, message?)`, `max(limit, message?)`, `pattern(regex, message?)`, `custom(fn)`. `min`/`max`/`pattern` pass on empty values, so compose with `required`. **Changed in 4.0.0:** `min` / `max` fail a value that is not a number (on 3.x `'abc'` passed both), and read text as typed: full-width digits (`１２３`), the column `locale`'s grouping and decimal separator, and a currency mark. Called without a `message`, the built-ins report the `validation.*` locale keys (4.0), so `localeText` translates them. In row edit mode `ctx.data` is the staged row (the record with the row's pending values applied) since 3.6, so a cross-field rule such as `ctx.data.end >= ctx.data.start` sees the row being saved. See `references/state-and-features.md` for the `Validator` / `ValidatorContext` shapes and the `cellValidationFailed` payload.

**Numeric editing (3.4):** clearing a `number` or `currency` editor now commits `null`, not `NaN`. An empty or unparseable numeric input leaves the cell empty instead of writing `NaN` into the row. Editing also opens from the keyboard now (Enter or F2 on the active cell), not only by pointer; see SKILL.md §5 for the full keyboard model. Separately, a spreadsheet-style error value (an object carrying a `#...` code, such as an enterprise formula engine's `#VALUE!`) renders as its code in typed `number` / `currency` columns rather than coercing to `NaN` and showing an empty cell.

## Auto-generation — the fastest demo

The shortest possible "just show me a table" example:

```ts
import { html, render } from 'lit';
import { setup } from 'apex-grid';
setup();

const data = [
  { id: 1, name: 'Ada',  age: 36 },
  { id: 2, name: 'Carl', age: 62 },
];

render(
  html`<apex-grid auto-generate .data=${data}></apex-grid>`,
  document.getElementById('app')!,
);
```

> Remember the host CSS — `apex-grid { height: 480px; }` — or you'll see no rows even with `auto-generate`.

**How it works:**

- Reads `Object.keys(data[0])` and creates a column for each, using the value's runtime type to pick `type`.
- **Skipped entirely if `columns` is non-empty.** Set `grid.columns = []` before swapping data to re-trigger.
- Late-bound data (e.g. fetched after first render) infers correctly because the watcher re-runs on `data` changes — but only when `columns` is still empty.

**Caveats:**

- `sort` and `filter` default to `false`, so the auto path is read-only. To get sortable / filterable columns from inferred data, run auto-generate first then patch via `grid.updateColumns([{ key: 'name', sort: true, filter: true }, ...])`.
- Auto-generated columns get no explicit `width` — they flex-fill across the grid. For mixed-type columns that's usually not what you want; assign explicit `columns` instead.

## Updating columns at runtime

```ts
// Replace the whole array (any change re-applies DEFAULT_COLUMN_CONFIG)
grid.columns = [...newColumns];

// Patch existing columns by key (preserves order, merges fields)
grid.updateColumns({ key: 'name', hidden: true });
grid.updateColumns([
  { key: 'age',  width: '120px' },
  { key: 'name', headerText: 'Full Name' },
]);
```

`updateColumns` triggers the data pipeline (so any sort/filter state is re-applied). It does **not** add new columns — only existing keys are patched.

## Hidden columns

```ts
{ key: 'internalId', hidden: true }
```

Column data is still accessible via `grid.dataView` and the cell template's `data` context, but no cell or header is rendered for it. Toggle at runtime via `updateColumns`.

## Resizable columns

```ts
{ key: 'name', resizable: true, width: '200px' }
```

The grid renders a drag handle on the right edge of the column header. Minimum resize width is hard-coded to `80px`.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `{ key: 'fullName' }` when `fullName` isn't in `T` | Add to `T`, or compute via `cellTemplate: ({ data }) => html\`${data.firstName} ${data.lastName}\`` |
| `cellTemplate: ({ row }) => row.name` | `row` is the `<apex-grid-row>` element, not the record: read `data.name` (4.0) |
| Reading `ctx.row.data` after an `await`, or keeping state in a template's elements | With row recycling (4.0 default) the element can show another record by then: read `ctx.data`, use `keyed(ctx.data, ...)`, or set `grid.rowRecycling = false` |
| `cellTemplate: ({ value }) => \`<b>\${value}</b>\`` (string) | `cellTemplate: ({ value }) => html\`<b>\${value}</b>\`` |
| `{ key: 'age', sort: true }` (default `type: 'string'`) | Set `type: 'number'` |
| `{ key: 'created', type: 'string', sort: { comparer } }` to fake dates | `{ key: 'created', type: 'date', format: 'medium' }` — a real type (accepts `Date`/ISO/timestamp) |
| `auto-generate` with existing columns | Reset `grid.columns = []` first |
| `updateColumns({ key: 'newField', ... })` to add | `updateColumns` only patches existing — assign a new `columns` array to add |
