# Data Pipeline — Server-Side, Async, Virtualization

## How the pipeline runs

When `data`, `columns`, sort state, filter state, the quick-filter value, or (in remote pagination mode) the page changes, Apex Grid runs an async pipeline. Since 4.0 a local page change only re-slices the view and does not rerun it. The full pipeline:

```
data ─► (quickFilter) ─► (filter) ─► (sort) ─► (pagination) ─► virtualizer ─► rendered rows
```

By default every stage runs in-memory. Override any stage with `dataPipelineConfiguration`.

## `dataPipelineConfiguration`

```ts
type DataPipelineParams<T> = { data: T[]; grid: ApexGrid<T>; type: 'sort' | 'filter' | 'quickFilter' };
type DataPipelineHook<T>   = (state: DataPipelineParams<T>) => T[] | Promise<T[]>;

interface DataPipelineConfiguration<T extends object> {
  sort?:        DataPipelineHook<T>;
  filter?:      DataPipelineHook<T>;   // column filters
  quickFilter?: DataPipelineHook<T>;   // global search (runs before column filter)
}
```

The grid invokes the matching hook in place of its built-in pipeline stage. Whatever the hook returns becomes the input for the next stage (or the final view). All three hooks are optional: omit a stage to keep it in-memory.

> **Changed in 4.0.0 (types only):** `DataPipelineConfiguration.pagination` and the `'pagination'` value of `DataPipelineParams.type` are removed. The hook was documented but never called, so nothing changes at runtime, but TypeScript that sets it stops compiling. Page on the server with remote pagination (below).

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

The grid runs the stages in a fixed order: **quick-filter → column filter → sort → pagination**. If your server endpoint does several of these together, do the work in the earliest relevant hook (returning the fully processed array) and omit the later hooks.

### Server-driven paging: `pagination.mode: 'remote'`

In remote mode the grid does not slice `data`: it renders whatever `data` holds as the current page. Fetch that page on `pageChanged`, assign it to `data`, and keep `pagination.totalItems` at the number of rows the server holds for the current filter, so the paginator can count the pages.

```ts
grid.pagination = { enabled: true, mode: 'remote', pageSize: 25, totalItems: 0 };

async function load(page: number, pageSize: number) {
  const res = await fetch(`/api/users?page=${page}&size=${pageSize}`);
  const { rows, total } = await res.json();
  grid.pagination = { ...grid.pagination!, page, pageSize, totalItems: total };  // a `page` in the config is applied on every assignment
  grid.data = rows;
}

grid.addEventListener('pageChanged', (e) => load(e.detail.page, e.detail.pageSize));
load(0, 25);
```

In remote mode a page or page size change reruns the pipeline (so your `sort` / `filter` hooks run again). **Changed in 4.0.0:** in local mode it no longer does, so custom `sort` and `filter` hooks are not called on a local page change, and with pagination on, pinned rows no longer count toward `totalItems`, `pageCount` or the summary (they show on every page).

### Hooks fire on every state change

The pipeline runs whenever `data`, `columns`, sort state, or filter state changes, including programmatic `sort()` / `filter()` calls. Since 3.6 only the latest run's result is applied: a slower earlier run that resolves after a newer one is discarded, and no longer holds back the newer run's events. If your server endpoint is expensive, still abort or debounce inside the hook, so the superseded requests are not sent at all:

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

Apex Grid uses [`@lit-labs/virtualizer`](https://www.npmjs.com/package/@lit-labs/virtualizer) (`<apex-virtualizer>`, pinned to exactly 2.1.1 in 4.0) to render rows. **Only currently visible rows + a small overscan are present in the DOM.**

| API | What it returns |
|---|---|
| `grid.rows` | `ApexGridRow<T>[]` — only the rendered chunk |
| `grid.dataView` | `readonly T[]` — full post-pipeline data (all rows) |
| `grid.totalItems` | `number`: `dataView.length`; with pagination, without pinned rows (4.0); in remote mode, the `pagination.totalItems` you supply |

**Row recycling (Changed in 4.0.0).** `grid.rowRecycling` defaults to `true`: a scroll, sort or filter re-binds the rendered row elements to the records now in view (and only the rows that scroll in render). So a row element can show a different record after any of those, not only after a scroll. `grid.rowRecycling = false` (attribute `row-recycling="false"`) keeps record-keyed rows. Templates read the record from `ctx.data` (`references/columns-and-templates.md`).

Implications:

- **Don't size the grid based on `grid.rows.length`.** Use `grid.totalItems`.
- **Don't hold references to `<apex-grid-row>` elements** across a scroll, sort or filter: read `row.data` each time, never keep the element as a handle to a record.
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

Since 3.6 this is reliable with async hooks too: `sorted`, `filtered`, `pageChanged` and `quickFilterChanged` fire after the pipeline run they caused, so `dataView` already holds the new rows. On 3.5 they could fire before an async hook resolved and read the previous view. (They remain UI-only, except the page and quick-filter methods listed in SKILL.md §5.)

Assigning `grid.data = null` or `undefined` (a framework prop still loading) shows an empty grid and reads back as `[]` since 3.6; on 3.5 it threw and kept the previous rows.

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Reading `grid.rows.length` for total count | `grid.totalItems` (or `grid.dataView.length`) |
| Caching `<apex-grid-row>` references across scrolls, sorts or filters | Look them up fresh from `grid.rows` (or operate on `grid.dataView` data) |
| `dataPipelineConfiguration.pagination` to page on the server | Removed in 4.0 (it was never called): `pagination.mode: 'remote'` + `pageChanged` + `pagination.totalItems` |
| Re-fetching from the server only on UI events | Hooks fire for both UI and programmatic — debounce / abort inside the hook |
| Sorting in the `filter` hook expecting it to override the built-in sort | The grid still runs the built-in sort after your filter hook unless you also supply a `sort` hook |
| Returning a Promise that rejects | Errors propagate; the grid keeps the previous `dataView`. Catch + log inside the hook to surface failures |
