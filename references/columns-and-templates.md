# Column Configuration & Lit Templates

## `ColumnConfiguration<T, K>`

Apex Grid is generic over a row type `T extends object`. Every column is typed against a key `K extends keyof T` so the cell template's `value` is correctly inferred.

```ts
interface BaseColumnConfiguration<T extends object, K extends keyof T = keyof T> {
  key: K;                                          // REQUIRED — keyof T
  type?: 'string' | 'number' | 'boolean';          // default 'string'
  headerText?: string;                             // default: String(key)
  width?: string;                                  // any CSS width unit
  hidden?: boolean;                                // default false
  resizable?: boolean;                             // default false
  sort?: boolean | { caseSensitive?: boolean; comparer?: (a: T[K], b: T[K]) => number };
  filter?: boolean | { caseSensitive?: boolean };
  headerTemplate?: (ctx: ApexHeaderContext<T>) => TemplateResult | unknown;
  cellTemplate?: (ctx: ApexCellContext<T, K>) => TemplateResult | unknown;
}
```

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

There is **no `'date'` type**. For dates:

```ts
// Option A — store as timestamps
{ key: 'createdAt', type: 'number', sort: true, filter: true,
  cellTemplate: ({ value }) => html`${new Date(value).toLocaleDateString()}` }

// Option B — store as ISO strings + custom comparer
{ key: 'createdAt', type: 'string',
  sort: { comparer: (a, b) => a.localeCompare(b) },
  cellTemplate: ({ value }) => html`${new Date(value).toLocaleDateString()}` }
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

## Auto-generation

```ts
<apex-grid auto-generate .data=${data}></apex-grid>
```

- Reads `Object.keys(data[0])` and creates a column for each, using the value's runtime type to pick `type`.
- **Skipped entirely if `columns` is non-empty.** Set `grid.columns = []` before swapping data to re-trigger.
- Late-bound data (e.g. fetched after first render) infers correctly because the watcher re-runs on `data` changes — but only when `columns` is still empty.

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
| `{ key: 'created', type: 'date' }` | No `'date'` type — use timestamp + `'number'` or ISO + custom `comparer` |
| `auto-generate` with existing columns | Reset `grid.columns = []` first |
| `updateColumns({ key: 'newField', ... })` to add | `updateColumns` only patches existing — assign a new `columns` array to add |
