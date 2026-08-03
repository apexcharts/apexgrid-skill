# State, Localization & Interaction Features (3.1+)

> Everything here landed in **apex-grid 3.1 – 3.4** on top of the 3.0 API. The setup rules from `SKILL.md` (register + bounded host height, no theme import) still apply. Row pinning / reordering, undo-redo, and validators are all **opt-in** via a config object — off by default.

## Row pinning — sticky top / bottom bands

Lift rows out of the scrolling body into sticky top / bottom bands. Identity is **by reference**, so pass the same object held in `grid.data`.

```ts
grid.rowPinning = { enabled: true };            // GridRowPinningConfiguration
grid.pinRow(grid.data[0], 'top');               // 'top' | 'bottom' (default 'top')
grid.pinRow(grid.data[1], 'bottom');
grid.unpinRow(grid.data[0]);
grid.pinnedRows;                                // getter → { top: T[]; bottom: T[] }
```

| Member | Signature | Notes |
|---|---|---|
| `rowPinning` | `{ enabled: boolean }` | Disabled when omitted. |
| `pinRow(row, position?)` | `(T, 'top' \| 'bottom') => boolean` | Moves between bands if already pinned. `false` when disabled / cancelled. |
| `unpinRow(row)` | `(T) => boolean` | No-op (returns `true`) when not pinned. |
| `pinnedRows` *(getter)* | `{ top: T[]; bottom: T[] }` | In pin order. |

Events: `rowPinning` (cancellable) → `rowPinned`, both `CustomEvent<{ row: T; position: 'top' | 'bottom' | null }>` (`null` when unpinning). Pins survive sort / filter / pagination as long as row identity is preserved; re-pin after a wholesale `data` swap.

## Row reordering — drag / keyboard / API

```ts
grid.rowReordering = { enabled: true, applyToData: false, handle: true };
grid.moveRow(2, 0, 'before');                   // (from, to, position?: 'before' | 'after')
```

| Field | Type | Default | Notes |
|---|---|---|---|
| `enabled` | `boolean` | — | Disabled when omitted. |
| `applyToData` | `boolean` | `false` | `true` also splices `grid.data` in place; otherwise the order lives in the grid and you persist via `rowMoved`. |
| `handle` | `boolean` | `true` | `true` renders a leading grip cell (drag only from it); `false` drags from anywhere on the row. |

`moveRow(from, to, position?)` indices are into `grid.pageItems`; returns `boolean`. A manual order is **mutually exclusive with column sorting** — applying a sort clears it. Events: `rowMoving` (cancellable) → `rowMoved`, both `CustomEvent<{ from: number; to: number; data: T }>`.

## Inline-edit undo / redo

Opt in via `editing.history` (requires `editing.enabled`). Tracks every committed cell edit — single, row-mode, and bulk paste / fill.

```ts
grid.editing = { enabled: true, history: { enabled: true, stackSize: 100 } };
grid.undo();                                    // Ctrl/Cmd+Z
grid.redo();                                    // Ctrl/Cmd+Shift+Z or Ctrl+Y
grid.clearHistory();
grid.canUndo;  grid.canRedo;                    // getters (false unless history.enabled)
```

`stackSize` defaults to `100` (older commands evicted past the cap). Event: `historyChanged` — `CustomEvent<{ canUndo: boolean; canRedo: boolean }>`, fires after any record / undo / redo / clear (drive toolbar button enablement from it).

## Declarative column validators

`column.validators: Validator[]` run **before** a candidate value is written. Each returns an error `string` (reject) or `null` (pass). All run and every message is collected, so one commit can surface multiple errors. A failing commit keeps the editor open, marks the cell `aria-invalid`, and emits `cellValidationFailed`. Covers bulk edits (paste / fill) too.

```ts
import { required, min, max, pattern, custom } from 'apex-grid';

const columns: ColumnConfiguration<User>[] = [
  { key: 'name', editable: true, validators: [required('Name is required')] },
  { key: 'age',  type: 'number', editable: true,
    validators: [required(), min(0), max(120)] },
  { key: 'email', editable: true,
    validators: [pattern(/^\S+@\S+$/, 'Invalid email')] },
  { key: 'code', editable: true,
    validators: [custom((value, ctx) => value === ctx.data.id ? 'Cannot equal id' : null)] },
];
```

| Factory | Signature | Fails when |
|---|---|---|
| `required(message?)` | `(msg?) => Validator` | value is `null` / `undefined` / empty-or-whitespace string |
| `min(limit, message?)` | `(number, msg?) => Validator` | numeric value `< limit` (non-numeric / empty pass — compose with `required`) |
| `max(limit, message?)` | `(number, msg?) => Validator` | numeric value `> limit` (non-numeric / empty pass) |
| `pattern(regex, message?)` | `(RegExp, msg?) => Validator` | `String(value)` doesn't match `regex` (empty passes) |
| `custom(fn)` | `(Validator) => Validator` | wraps a predicate; an inline arrow works identically |

```ts
// Validator signature
type Validator<T, K> = (value: unknown, context: ValidatorContext<T, K>) => string | null;
interface ValidatorContext<T, K> { column: ColumnConfiguration<T, K>; data: T; rowIndex: number; }
```

Event: `cellValidationFailed` — `CustomEvent<{ key; rowIndex; data: T; value: unknown; errors: readonly string[] }>`.

## Column groups — spanning headers

A presentational header layer above the column header row. `columns` stays flat; a column joins a group via its `group` id. Members must be **contiguous within one pin region** (the grid warns and skips the spanning cell otherwise). v1 ships static spanning headers only (`collapsible` is reserved / inert).

```ts
grid.columnGroups = [
  { id: 'name', headerText: 'Name' },
  { id: 'contact', headerText: 'Contact',
    headerTemplate: ({ group, span }) => html`<em>${group.headerText} (${span})</em>` },
];
grid.columns = [
  { key: 'first', group: 'name' },
  { key: 'last',  group: 'name' },
  { key: 'email', group: 'contact' },
  { key: 'phone', group: 'contact' },
];
```

| `ColumnGroupConfiguration` | Type | Notes |
|---|---|---|
| `id` | `string` | Referenced by a column's `group`. |
| `headerText` | `string` | Text in the spanning cell. |
| `headerTemplate?` | `(ctx: { group; span }) => TemplateResult` | Overrides `headerText`. |
| `collapsible?` | `boolean` | Reserved for a future collapse affordance; inert in v1. |

Getters: `hasColumnGroups` (a group has ≥1 visible member), `columnGroupDepth` (0 or 1 in v1), `displayColumns` (`readonly ColumnConfiguration<T>[]` in visual render order: start-pinned, unpinned, end-pinned).

## State & schema — persist / restore / drive AI

`getState()` returns a serializable, JSON-safe snapshot (functions / templates omitted); `setState()` applies a partial one. `getSchema()` returns a machine-readable descriptor of the grid (columns, per-column operations, capabilities, current state) — the contract an AI layer hands an LLM and validates a returned patch against.

```ts
const snapshot = grid.getState();               // GridState
localStorage.setItem('grid', JSON.stringify(snapshot));
// later …
grid.setState(JSON.parse(localStorage.getItem('grid')!));   // → SetStateResult

// Selection / expansion survive a wholesale data reload only with a stable id:
grid.rowId = (row) => row.id;
```

`GridState` (`version: 1`) captures: `columns` (order/width/pinning/visibility), `sort`, `filter` (condition by operand name), `quickFilter`, `pagination` (`{ page, pageSize }`), `selection`, `expansion`, `treeExpanded` / `treeExpandedKeys`, `rowPinning`, `rowOrder`, and per-`modules` state. Rows are captured as `RowRef` — a stable `id` when `rowId` is set, else a positional `index` into `data` (round-trips within a session, not across a data swap).

`setState(state, options?)` is **partial and defensive** — only present slices apply (omit a slice to leave it untouched; a present-but-empty `sort`/`filter` clears it). It never throws on malformed / stale / LLM input unless `options.strict` is set; unknown columns/operands, out-of-range pages, and unresolvable rows are dropped / clamped and reported. `rowOrder` is mutually exclusive with `sort` (a snapshot with both drops `rowOrder`).

```ts
interface SetStateResult { applied: string[]; skipped: string[]; warnings: string[]; }
interface GetStateOptions<T> { rowId?: (row: T) => string | number; }
interface SetStateOptions<T> { rowId?: (row: T) => string | number; strict?: boolean; }
```

Event: `stateChanged` — `CustomEvent<{ state: GridState }>`, debounced after the render pipeline settles. Fires for programmatic (including `setState`) and UI changes. The payload is only computed while a listener is attached (zero-cost otherwise); identical snapshots don't re-fire, so compare before persisting to avoid a save loop.

## Programmatic pagination, quick-filter, and column pinning

All emit the same cancellable-then-notify event pairs as their UI equivalents and return `Promise<boolean>`.

```ts
await grid.gotoPage(2);          // clamped into [0, pageCount - 1]
await grid.nextPage();
await grid.previousPage();
await grid.firstPage();
await grid.lastPage();
await grid.setPageSize(50);      // returns to the first page
await grid.setQuickFilter('john');   // '' to clear
await grid.pinColumn('name', 'start');   // 'start' | 'end' | null
await grid.unpinColumn('actions');       // = pinColumn(key, null)
```

`gotoPage` / `setPageSize` / `nextPage` / `previousPage` / `firstPage` / `lastPage` emit `pageChanging` → `pageChanged`. `setQuickFilter` emits `quickFilterChanging` → `quickFilterChanged`. `pinColumn` / `unpinColumn` emit `columnPinning` → `columnPinned` and only change the visual render order (the `columns` array isn't reordered).

## Localization — `localeText`

Every built-in UI string (pagination, filter operators, toolbar, row/selection labels) is localizable through a `GridLocaleText` map keyed by dot-namespaced `GridLocaleKey`s. Partial maps are valid — any omitted key falls back to the English default.

```ts
import { esLocale } from 'apex-grid';

grid.localeText = esLocale;                                   // whole-UI Spanish
grid.localeText = { 'toolbar.searchPlaceholder': 'Buscar…' };  // or override a few keys

grid.localize('pagination.summary', { start: 1, end: 25, total: 100 });   // resolve a key
```

Exports: `EN_LOCALE` (the default dictionary / source of truth for keys), `esLocale` (ready-made Spanish), `interpolate` (`{token}` substitution), `localize`, and the types `GridLocaleKey` / `GridLocaleText` / `LocaleParams`. `grid.localize(key, params?, fallback?)` applies any `localeText` override over the English default and interpolates `{placeholder}` tokens; `fallback` covers keys outside the built-in set (e.g. a custom operand label).

**3.4 key changes** (both English and Spanish): the accessibility work added live-region announcement keys (`announce.*`, e.g. `announce.sortedAscending`, `announce.page`, `announce.rowSelected`, `announce.undoOne`), header control / sort-state labels (`header.*`, e.g. `header.sortedAscending`, `header.filterColumn`, `header.columnMenu`), the host label `grid.label` ("Data grid"), and `editor.rating`. The `ai.noAdapter` key was **removed** (drop it from any `localeText` override you kept). `EN_LOCALE` remains the source of truth for the full, current key list.

## Column menu & column separator

| Property | Type | Default | Notes |
|---|---|---|---|
| `columnMenu` | `boolean` (JS property, no attr) | `true` | Kebab-menu button in each header, always visible. Set `false` to hide it. |
| `columnSeparator` (attr `column-separator`) | `boolean` | `true` | Persistent trailing divider on every header. On resizable columns it doubles as the resize handle. Theme via `--ag-header-separator` / `--header-separator-color` and inset via `--apex-header-separator-inset`. |

```ts
grid.columnMenu = false;
grid.columnSeparator = false;
```

## Export source & formats

`exportToCSV` / `exportAs` accept a `source` selecting which rows to export:

| `source` | Rows exported |
|---|---|
| `'view'` *(default)* | post-filter, post-sort rows across all pages |
| `'page'` | current page slice only |
| `'selected'` | current row selection (insertion order) |
| `'all'` | the raw `data` array |

```ts
grid.exportToCSV();                                       // download data.csv (view)
grid.exportToCSV({ filename: 'users', source: 'selected' });
const text = grid.exportToCSV({ filename: '' });          // string only, no download
```

`exportFormats` *(getter)* → `readonly ExportFormat[]` (`{ id, label }`) — the community grid offers CSV only; it drives the toolbar export menu, one item per entry, dispatching to `exportAs(formatId, options?)`. Other `ExportOptions` fields: `columns?` (keys to include), `includeHeader?` (default `true`), `formatter?`; CSV adds `delimiter?` (`','`), `bom?` (`true`), `newline?` (`'\r\n'`).
