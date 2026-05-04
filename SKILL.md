---
name: apex-grid
description: >
  AI skill for building UIs with the `apex-grid` web component (the Lit-based
  ApexCharts data grid). Use whenever the user asks to create, configure, or
  troubleshoot a sortable / filterable / virtualized data table with
  `<apex-grid>`. Covers the generic `ColumnConfiguration<T>` shape, cell &
  header templates that return Lit `TemplateResult`s, the
  `sort` / `filter` / `sorted` / `filtered` events, programmatic
  `sort()` / `filter()` / `clearSort()` / `clearFilter()` / `updateColumns()`
  APIs, the `dataPipelineConfiguration` hooks for server-side data, and the
  one-time `ApexGrid.register()` custom-element registration.
metadata:
  author: ApexCharts
  version: "1.0.0"
  library_version: ">=0.0.0"
  category: data-visualization
  tags: [grid, data-grid, table, web-component, lit, apex-grid]
  docs: https://github.com/apexcharts/apexgrid
  npm: apex-grid
  github: https://github.com/apexcharts/apexgrid
---

# Apex Grid AI Skill

> **Heads-up on naming.** The npm package is **`apex-grid`** (with a hyphen). The custom-element tag is **`<apex-grid>`**. The exported class is **`ApexGrid<T>`**. There is no `apexgrid` package — using `import ... from 'apexgrid'` will fail.

## 1. Critical Rules

1. **Apex Grid is a Lit web component, not a JS class with a constructor.** You don't `new ApexGrid(...)`. You register the custom element once and then use `<apex-grid>` in the DOM, setting `.data` and `.columns` as JS properties.
2. **Register the element exactly once at app startup**: either `import 'apex-grid/define'` (auto-registers as a side-effect) or `import { ApexGrid } from 'apex-grid'; ApexGrid.register();`. Without this, the tag renders as an empty inert element.
3. **`columns` and `data` are properties, not attributes.** Always bind with the property syntax (`.columns=${...}` in Lit, `[columns]="..."` in Angular, `:columns=` in Vue, ref-based in plain JS) — never as a stringified `columns="..."` attribute.
4. **`column.key` must be a real key of `T`** (the data row type). It is what the grid reads from each row, what sort/filter expressions reference, and what's emitted in events. There is no `field` / `dataIndex` / `accessor` alias.
5. **`type` defaults to `'string'`.** Set it to `'number'` or `'boolean'` to get the right filter operands and comparer. There is no `'date'` type — use `'number'` (timestamps) or `'string'` (ISO) and a custom `comparer` / `cellTemplate`.
6. **`sort` and `filter` per-column are `boolean | configObject`.** They are `false` by default — no opt-in, no UI. Set `sort: true` / `filter: true` to enable, or pass a config object for fine-grained control.
7. **`cellTemplate` and `headerTemplate` must return Lit `TemplateResult`s** (`html\`...\``) — not strings, DOM nodes, or framework-specific JSX. Plain HTML strings render as escaped text.
8. **Sort/filter events fire on UI-initiated operations only.** Programmatic calls (`grid.sort(...)`, `grid.filter(...)`) do not emit `sorting` / `sorted` / `filtering` / `filtered` events.
9. **`sorting` and `filtering` events are cancellable** (`event.preventDefault()`) and their `event.detail` is mutable so you can rewrite the expression before it runs.
10. **For server-side data, set `dataPipelineConfiguration.sort` / `.filter`** — async hooks that receive `{ data, grid, type }` and return the new array. The grid skips its built-in pipeline entirely when a hook is provided for that operation.
11. **`autoGenerate` is ignored when `columns` is non-empty.** To re-trigger auto-generation after a data swap, set `grid.columns = []` first, then assign the new `data`.
12. **`grid.rows` only returns currently rendered rows.** The grid uses `@lit-labs/virtualizer`, so off-screen rows are not in the DOM.
13. **`apex-grid` depends on `igniteui-webcomponents`** for filter/sort UI controls. The package lists it as a runtime dependency — don't try to remove it.

---

## 2. Data Format & Column Configuration

Apex Grid is generic over a row type `T`:

```ts
interface ColumnConfiguration<T extends object, K extends keyof T = keyof T> {
  key: K;                                          // REQUIRED — must be a key of T
  type?: 'string' | 'number' | 'boolean';          // default 'string'
  headerText?: string;                             // default: the key itself
  width?: string;                                  // any CSS width
  hidden?: boolean;                                // default false
  resizable?: boolean;                             // default false
  sort?: boolean | { caseSensitive?: boolean; comparer?: (a: T[K], b: T[K]) => number };
  filter?: boolean | { caseSensitive?: boolean };
  headerTemplate?: (ctx: ApexHeaderContext<T>) => TemplateResult | unknown;
  cellTemplate?: (ctx: ApexCellContext<T, K>) => TemplateResult | unknown;
}
```

Minimal example:

```ts
import { html, render } from 'lit';
import 'apex-grid/define';                         // registers <apex-grid>
import type { ColumnConfiguration } from 'apex-grid';

type User = { id: number; name: string; age: number; subscribed: boolean };

const data: User[] = [
  { id: 1, name: 'Ada',  age: 36, subscribed: true  },
  { id: 2, name: 'Carl', age: 41, subscribed: false },
];

const columns: ColumnConfiguration<User>[] = [
  { key: 'id',         type: 'number',  sort: true, filter: true, headerText: 'ID' },
  { key: 'name',       type: 'string',  sort: true, filter: true },
  { key: 'age',        type: 'number',  sort: true, filter: true },
  { key: 'subscribed', type: 'boolean', sort: true, filter: true },
];

render(
  html`<apex-grid .data=${data} .columns=${columns}></apex-grid>`,
  document.getElementById('app')!,
);
```

### Column types and filter operands

The `type` field selects which filter operand set is exposed in the filter UI:

| `type` | Operands available |
|---|---|
| `'string'` *(default)* | `contains`, `doesNotContain`, `startsWith`, `endsWith`, `equals`, `doesNotEqual`, `empty`, `notEmpty` |
| `'number'` | `equals`, `doesNotEqual`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `empty`, `notEmpty` |
| `'boolean'` | `all`, `true`, `false`, `empty`, `notEmpty` |

Programmatic filter expressions can reference operands by name (`condition: 'contains'`) or by importing them as a function:

```ts
import { StringOperands } from 'apex-grid';

grid.filter({ key: 'name', condition: StringOperands.contains, searchTerm: 'ada' });
// or shorthand
grid.filter({ key: 'name', condition: 'contains', searchTerm: 'ada' });
```

### Cell & header templates (Lit)

Templates are Lit render callbacks. They receive a context object and must return a `TemplateResult`.

```ts
const columns: ColumnConfiguration<User>[] = [
  {
    key: 'name',
    cellTemplate: ({ value, row, parent, column }) => html`
      <strong style="color:#1f4ed8">${value}</strong>
    `,
  },
  {
    key: 'subscribed',
    type: 'boolean',
    cellTemplate: ({ value }) => html`
      <input type="checkbox" .checked=${value} disabled />
    `,
  },
  {
    key: 'age',
    headerTemplate: ({ column }) => html`
      <em>${column.headerText ?? String(column.key)}</em>
    `,
  },
];
```

Context shapes:

```ts
interface ApexCellContext<T, K extends keyof T> {
  parent: ApexGridCell<T>;        // the cell element
  row:    ApexGridRow<T>;         // the row element
  column: ColumnConfiguration<T, K>;
  value:  T[K];
}

interface ApexHeaderContext<T> {
  parent: ApexGridHeader<T>;
  column: ColumnConfiguration<T>;
}
```

### Auto-generated columns

`auto-generate` (or `.autoGenerate=${true}`) infers columns from the first data row's keys. **It is ignored if `columns` is non-empty.** To re-trigger after a data swap:

```ts
grid.columns = [];                   // reset first
grid.data = newData;                 // then replace data — columns are inferred
```

---

## 3. Public API

### Properties

| Property | Type | Default | Notes |
|---|---|---|---|
| `data` | `T[]` | `[]` | Row data. Setting it triggers a pipeline run. |
| `columns` | `ColumnConfiguration<T>[]` | `[]` | Column definitions. Each entry is merged with `DEFAULT_COLUMN_CONFIG`. |
| `autoGenerate` | `boolean` (attr `auto-generate`) | `false` | Infer columns from data. Ignored when `columns` is non-empty. |
| `sortConfiguration` | `{ multiple: boolean; triState: boolean }` | `{ true, true }` | Grid-wide sort behaviour. |
| `dataPipelineConfiguration` | `{ sort?, filter? }` | — | Server-side / async data hooks. |
| `sortExpressions` | `SortExpression<T>[]` | `[]` | Get/set the active sort state. Setting an empty array is a no-op — use `clearSort()`. |
| `filterExpressions` | `FilterExpression<T>[]` | `[]` | Get/set the active filter state. Empty-set semantics same as above. |
| `rows` | `ApexGridRow<T>[]` *(getter)* | — | Currently rendered rows only (virtualized). |
| `dataView` | `readonly T[]` *(getter)* | — | Data after sort + filter pipeline. |
| `totalItems` | `number` *(getter)* | — | `dataView.length`. |

### Methods

| Method | Description |
|---|---|
| `sort(expr \| expr[])` | Apply a sort. Does **not** fire `sorting` / `sorted` events. |
| `filter(expr \| expr[])` | Apply a filter. Does **not** fire `filtering` / `filtered` events. |
| `clearSort(key?)` | Clear all sort state, or just the column with `key`. |
| `clearFilter(key?)` | Clear all filter state, or just the column with `key`. |
| `getColumn(keyOrIndex)` | Find a column by `key` or numeric index. |
| `updateColumns(col \| col[])` | Merge new properties into existing columns by `key`. Triggers a pipeline run. |

### Static

| Member | Description |
|---|---|
| `ApexGrid.tagName` | The string `'apex-grid'`. |
| `ApexGrid.register()` | Registers `<apex-grid>` and all internal sub-components. Idempotent. Either call this once, or import `'apex-grid/define'` for auto-registration. |

---

## 4. Events

Events fire **only** when the user interacts with sort / filter UI. Programmatic `grid.sort(...)` / `grid.filter(...)` calls are silent.

```ts
import type { ApexGridEventMap } from 'apex-grid';

grid.addEventListener('sorting', (e) => {
  // CustomEvent<SortExpression<T>>
  // e.detail is the expression about to be applied — mutate to rewrite, preventDefault to cancel
  if (e.detail.direction === 'descending') {
    e.preventDefault();
  }
});

grid.addEventListener('sorted', (e) => {
  // CustomEvent<SortExpression<T>>
});

grid.addEventListener('filtering', (e) => {
  // CustomEvent<{ key, expressions, type: 'add'|'modify'|'remove' }>
});

grid.addEventListener('filtered', (e) => {
  // CustomEvent<{ key, state: FilterExpression<T>[] }>
});
```

| Event | Cancellable | `event.detail` |
|---|---|---|
| `sorting` | yes | `SortExpression<T>` (mutable) |
| `sorted` | no | `SortExpression<T>` |
| `filtering` | yes | `{ key: keyof T; expressions: FilterExpression<T>[]; type: 'add' \| 'modify' \| 'remove' }` |
| `filtered` | no | `{ key: keyof T; state: FilterExpression<T>[] }` |

---

## 5. Server-Side / Async Data — `dataPipelineConfiguration`

When sort or filter should run on the server, supply async hooks. Each hook receives `{ data, grid, type }` and returns the new array. **The built-in pipeline is skipped for whichever operations have a hook.**

```ts
grid.dataPipelineConfiguration = {
  sort: async ({ data, grid }) => {
    const expressions = grid.sortExpressions;
    return fetch('/api/users/sort', {
      method: 'POST',
      body: JSON.stringify(expressions),
    }).then((r) => r.json());
  },
  filter: async ({ data, grid }) => {
    const expressions = grid.filterExpressions;
    return fetch('/api/users/filter', {
      method: 'POST',
      body: JSON.stringify(expressions),
    }).then((r) => r.json());
  },
};
```

You can supply just `sort`, just `filter`, or both. Anything you don't override falls back to the in-memory pipeline.

---

## 6. Programmatic Sort & Filter

```ts
import { StringOperands } from 'apex-grid';

// Single sort
grid.sort({ key: 'name', direction: 'ascending' });

// Multiple (multi-column) sort — works only when sortConfiguration.multiple is true
grid.sort([
  { key: 'subscribed', direction: 'descending' },
  { key: 'age',        direction: 'ascending'  },
]);

// Filter — by operand name
grid.filter({ key: 'name', condition: 'contains', searchTerm: 'ad' });

// Filter — by operand object (typed)
grid.filter({ key: 'name', condition: StringOperands.contains, searchTerm: 'ad' });

// Multiple filters with AND/OR
grid.filter([
  { key: 'age', condition: 'greaterThan', searchTerm: 30 },
  { key: 'age', condition: 'lessThan',    searchTerm: 50, criteria: 'and' },
]);

// Clear
grid.clearSort();              // all columns
grid.clearSort('name');        // just `name`
grid.clearFilter();
grid.clearFilter('name');
```

`SortExpression<T>` and `FilterExpression<T>` are exported from `apex-grid` for typed arrays.

---

## 7. Lifecycle Pattern

```ts
// 1. Register the custom element ONCE at app startup
import 'apex-grid/define';
// — or —
import { ApexGrid } from 'apex-grid';
ApexGrid.register();

// 2. Stamp the element into the DOM
import { html, render } from 'lit';
render(html`<apex-grid></apex-grid>`, document.getElementById('app')!);

// 3. Set properties (after the element is in the DOM)
const grid = document.querySelector('apex-grid')!;
grid.columns = [...];
grid.data = [...];

// 4. Listen for UI events
grid.addEventListener('sorted', (e) => console.log(e.detail));

// 5. Optional: programmatic operations
grid.sort({ key: 'name', direction: 'ascending' });

// 6. Removing the element from the DOM tears down its observers automatically.
//    There is no destroy() — Lit's disconnectedCallback handles cleanup.
```

---

## 8. Pitfalls — ❌ Wrong vs ✅ Correct

### 1. Treating it like a class-based JS API
❌ `const grid = new ApexGrid({ container: '#app', columns, data })` — this is a custom element, not a constructor.
✅ Render `<apex-grid .data=${data} .columns=${columns}></apex-grid>` and let the DOM instantiate it.

### 2. Forgetting to register the element
❌ Importing `ApexGrid` but never calling `register()` — `<apex-grid>` renders as an inert tag.
✅ `import 'apex-grid/define'` once, or `ApexGrid.register()` at app startup.

### 3. Wrong package name
❌ `import { ApexGrid } from 'apexgrid'` — the package is `apex-grid` (with a hyphen).
✅ `import { ApexGrid } from 'apex-grid'`.

### 4. Stringified `columns` attribute
❌ `<apex-grid columns="[...]">` — attributes are strings; `columns` is an object array.
✅ Property binding: `.columns=${columns}` (Lit), `:columns="columns"` (Vue), `[columns]="columns"` (Angular), or `el.columns = columns` (vanilla).

### 5. `column.key` not a real key of `T`
❌ `{ key: 'fullName' }` when `User` has only `firstName` / `lastName`.
✅ Either rename the field on `User` or compute it via `cellTemplate`: `cellTemplate: ({ row }) => html\`${row.firstName} ${row.lastName}\``.

### 6. Returning a string from `cellTemplate`
❌ `cellTemplate: ({ value }) => \`<strong>\${value}</strong>\`` — renders as escaped text.
✅ `cellTemplate: ({ value }) => html\`<strong>\${value}</strong>\``.

### 7. Expecting events from programmatic calls
❌ Listening for `sorting` after calling `grid.sort(...)` — never fires.
✅ The UI-initiated path fires events; the programmatic path is silent. If you need a side-effect on every change, watch `grid.dataView` or wrap the call yourself.

### 8. `sort: true` without setting `type`
❌ Sorting numbers as strings: `{ key: 'age', sort: true }` with no `type`.
✅ `{ key: 'age', type: 'number', sort: true }` — gets the correct comparer and filter operands.

### 9. Looking for a `'date'` column type
❌ `{ key: 'created', type: 'date' }` — there is no `'date'` type.
✅ Store dates as numbers (timestamps) with `type: 'number'`, or as ISO strings with a custom `sort.comparer` and a `cellTemplate` that formats:
```ts
{
  key: 'created',
  type: 'string',
  sort: { comparer: (a, b) => a.localeCompare(b) },
  cellTemplate: ({ value }) => html`${new Date(value).toLocaleDateString()}`,
}
```

### 10. Calling `clearSort()` to flush an empty array
❌ `grid.sortExpressions = []` — the setter is a no-op for empty arrays.
✅ `grid.clearSort()`.

### 11. Querying all rows
❌ `grid.rows.length === grid.data.length` — false; rows are virtualized.
✅ Use `grid.totalItems` for the post-pipeline count, or `grid.dataView` for the post-pipeline rows.

### 12. Auto-generating after replacing data
❌ `grid.data = newData` with `autoGenerate: true` and existing columns — the old columns persist.
✅ Reset first: `grid.columns = []; grid.data = newData;`.

### 13. Mixing operand keys across types
❌ `grid.filter({ key: 'age' /* number */, condition: 'contains' /* string operand */ })` — runtime error.
✅ Match the operand to the column type (or import the typed operand: `NumberOperands.greaterThan`).

### 14. Rendering before `register()` in the same task
❌ Stamping `<apex-grid>` before the registration import has resolved — the element is upgraded later but properties set before upgrade may be lost.
✅ Import `'apex-grid/define'` at the top of your entry file, then render.

---

## 9. Reference Routing Table

| Topic | Reference File |
|---|---|
| Column configuration, types, templates, auto-generation | `references/columns-and-templates.md` |
| Sort & filter — UI events, programmatic API, expressions, operands | `references/sort-and-filter.md` |
| Server-side data pipeline, async hooks, virtualization notes | `references/data-pipeline.md` |
| Framework integration — Lit, React, Vue, Angular | `references/framework-integration.md` |
