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
  locale?: string;                                 // 'currency' BCP 47 locale
  badgeVariant?: 'gold' | 'brand' | 'neutral' | 'muted' | ((v) => …);  // 'badge'
  statusVariant?: 'active' | 'trial' | 'churn' | ((v) => …);           // 'status'
  showDelta?: boolean;                             // 'sparkline' (def true)
}
```

### Column types (`DataType`)

`'string'` / `'number'` / `'boolean'` are the primitive types driving sort, filter, and the default editor. The rest are presentation renderers over a primitive value (and sort/filter as that value): `'select'` (label from `options`, `<select>` editor), `'rating'` (stars, 0..`max`), `'date'` (locale date via `format`; accepts `Date`/ISO/timestamp; `<input type="date">` editor), `'image'` (`<img>`, `shape`/`alt`), `'currency'` (`Intl.NumberFormat`, `currency`/`locale`), `'avatar'`, `'badge'` (`badgeVariant`), `'progress'` (0..`max`), `'sparkline'` (`number[]`, `showDelta`), `'status'` (`statusVariant`).

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

The table above lists the three **primitive** types. The presentation types (`'select'`, `'rating'`, `'date'`, `'image'`, `'currency'`, `'avatar'`, `'badge'`, `'progress'`, `'sparkline'`, `'status'`) sort and filter as their underlying value. For dates, use the real `'date'` type — it accepts `Date`, ISO strings, or millisecond timestamps, renders a locale date (`format`), and sorts chronologically. No timestamp/comparer workaround needed:

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

The `comparer` signature is fully typed against `T[K]`.

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
  parent: ApexGridCell<T>;          // <apex-grid-cell> element
  row:    ApexGridRow<T>;           // <apex-grid-row> element
  column: ColumnConfiguration<T, K>;
  value:  T[K];                     // T[K] — fully typed
}
```

```ts
import { html } from 'lit';

const columns: ColumnConfiguration<User>[] = [
  // Direct value — no template needed
  { key: 'id', type: 'number' },

  // Inline emphasis
  { key: 'name',
    cellTemplate: ({ value }) => html`<strong>${value}</strong>` },

  // Conditional render with row context
  { key: 'subscribed', type: 'boolean',
    cellTemplate: ({ value, row }) => html`
      <input type="checkbox" .checked=${value} disabled />
      <small>${row.name}</small>` },

  // Embedded igniteui-webcomponents control
  { key: 'avatar',
    cellTemplate: ({ value }) => html`<igc-avatar shape="circle" .src=${value}></igc-avatar>` },

  // Rating widget tied to data type
  { key: 'satisfaction', type: 'number',
    cellTemplate: ({ value }) => html`<igc-rating readonly .value=${value}></igc-rating>` },
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

Built-in factories: `required(message?)`, `min(limit, message?)`, `max(limit, message?)`, `pattern(regex, message?)`, `custom(fn)`. `min`/`max`/`pattern` pass on empty / non-numeric values — compose with `required`. See `references/state-and-features.md` for the `Validator` / `ValidatorContext` shapes and the `cellValidationFailed` payload.

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

Column data is still accessible via `grid.dataView` and the cell template's `row` context, but no cell or header is rendered for it. Toggle at runtime via `updateColumns`.

## Resizable columns

```ts
{ key: 'name', resizable: true, width: '200px' }
```

The grid renders a drag handle on the right edge of the column header. Minimum resize width is hard-coded to `80px`.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| `{ key: 'fullName' }` when `fullName` isn't in `T` | Add to `T`, or compute via `cellTemplate: ({ row }) => html\`${row.firstName} ${row.lastName}\`` |
| `cellTemplate: ({ value }) => \`<b>\${value}</b>\`` (string) | `cellTemplate: ({ value }) => html\`<b>\${value}</b>\`` |
| `{ key: 'age', sort: true }` (default `type: 'string'`) | Set `type: 'number'` |
| `{ key: 'created', type: 'string', sort: { comparer } }` to fake dates | `{ key: 'created', type: 'date', format: 'medium' }` — a real type (accepts `Date`/ISO/timestamp) |
| `auto-generate` with existing columns | Reset `grid.columns = []` first |
| `updateColumns({ key: 'newField', ... })` to add | `updateColumns` only patches existing — assign a new `columns` array to add |
