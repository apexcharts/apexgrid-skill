# Data Pipeline — Server-Side, Async, Virtualization

## How the pipeline runs

When `data`, `columns`, sort state, or filter state changes, Apex Grid runs an async pipeline:

```
data ─► (filter) ─► (sort) ─► dataState ─► virtualizer ─► rendered rows
```

By default both stages run in-memory. Override either stage (or both) with `dataPipelineConfiguration`.

## `dataPipelineConfiguration`

```ts
interface DataPipelineConfiguration<T extends object> {
  sort?:   (state: { data: T[]; grid: ApexGrid<T>; type: 'sort'   }) => T[] | Promise<T[]>;
  filter?: (state: { data: T[]; grid: ApexGrid<T>; type: 'filter' }) => T[] | Promise<T[]>;
}
```

The grid invokes the matching hook in place of its built-in pipeline stage. Whatever the hook returns becomes the new `dataState` for the next stage (or the final view).

```ts
grid.dataPipelineConfiguration = {
  sort: async ({ data, grid }) => {
    const expr = grid.sortExpressions;
    const res = await fetch('/api/users/sort', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(expr),
    });
    return res.json() as Promise<User[]>;
  },
  filter: async ({ data, grid }) => {
    const expr = grid.filterExpressions;
    const res = await fetch('/api/users/filter', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(expr),
    });
    return res.json() as Promise<User[]>;
  },
};
```

### Hook-only sort, in-memory filter (or vice versa)

Just omit one of the two:

```ts
grid.dataPipelineConfiguration = {
  filter: async ({ grid }) => fetchFilteredFromServer(grid.filterExpressions),
  // sort omitted — uses the built-in in-memory sort on the filtered data the hook returns
};
```

### Order of operations

The grid runs **filter first, then sort** — and that ordering is fixed. If your server endpoint sorts and filters together, do both in the `filter` hook (returning the fully sorted+filtered array) and skip the `sort` hook.

### Hooks fire on every state change

The pipeline runs whenever `data`, `columns`, sort state, or filter state changes — including programmatic `sort()` / `filter()` calls. If your server endpoint is expensive, debounce inside the hook:

```ts
let pending: AbortController | null = null;
grid.dataPipelineConfiguration = {
  filter: async ({ grid }) => {
    pending?.abort();
    pending = new AbortController();
    const res = await fetch('/api/users/filter', {
      method: 'POST',
      body: JSON.stringify(grid.filterExpressions),
      signal: pending.signal,
    });
    return res.json();
  },
};
```

### Hooks have access to the grid instance

Use it to read the latest sort / filter state, query metadata, or trigger follow-up operations:

```ts
grid.dataPipelineConfiguration = {
  sort: async ({ data, grid }) => {
    const expressions = grid.sortExpressions;
    const customComparer = grid.getColumn('priority')?.sort;       // ColumnSortConfiguration
    // ...
  },
};
```

## Virtualization

Apex Grid uses [`@lit-labs/virtualizer`](https://www.npmjs.com/package/@lit-labs/virtualizer) (`<apex-virtualizer>`) for row recycling. **Only currently visible rows + a small overscan are present in the DOM.**

| API | What it returns |
|---|---|
| `grid.rows` | `ApexGridRow<T>[]` — only the rendered chunk |
| `grid.dataView` | `readonly T[]` — full post-pipeline data (all rows) |
| `grid.totalItems` | `number` — `dataView.length` |

Implications:

- **Don't size the grid based on `grid.rows.length`.** Use `grid.totalItems`.
- **Don't hold references to `<apex-grid-row>` elements** across scrolls — they get recycled.
- **DOM-driven row queries** (`document.querySelectorAll('apex-grid-row')`) only see visible rows.

## When the pipeline runs

Internally the grid watches the `'pipeline'` reactive symbol. It runs after:

- `data` is reassigned
- `columns` is reassigned (or `updateColumns` is called)
- A sort operation is applied (UI or programmatic)
- A filter operation is applied (UI or programmatic)
- Sort / filter state is cleared

It does **not** run when only column properties like `width` / `hidden` / `resizable` change, since those are presentational.

## `dataView` for derived UI

`dataView` is the canonical source of truth for "what's on screen after the pipeline":

```ts
// React-style consumer outside the grid (e.g. a footer summary)
const updateSummary = () => {
  const view = grid.dataView;
  document.getElementById('count')!.textContent = `${view.length} rows`;
  document.getElementById('avg')!.textContent =
    String(view.reduce((s, u) => s + u.age, 0) / view.length);
};

grid.addEventListener('sorted', updateSummary);
grid.addEventListener('filtered', updateSummary);
```

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Reading `grid.rows.length` for total count | `grid.totalItems` (or `grid.dataView.length`) |
| Caching `<apex-grid-row>` references across scrolls | Look them up fresh from `grid.rows` (or operate on `grid.dataView` data) |
| Re-fetching from the server only on UI events | Hooks fire for both UI and programmatic — debounce / abort inside the hook |
| Sorting in the `filter` hook expecting it to override the built-in sort | The grid still runs the built-in sort after your filter hook unless you also supply a `sort` hook |
| Returning a Promise that rejects | Errors propagate; the grid keeps the previous `dataView`. Catch + log inside the hook to surface failures |
