# State, Localization & Interaction Features (3.1+)

> Everything here landed in **apex-grid 3.1 and later** on top of the 3.0 API; version tags mark what is newer. The setup rules from `SKILL.md` (register + bounded host height, no theme import) still apply. Row pinning / reordering, undo-redo, and validators are all **opt-in** via a config object, off by default.

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

Events: `rowPinning` (cancellable) → `rowPinned`, both `CustomEvent<{ row: T; position: 'top' | 'bottom' | null }>` (`null` when unpinning). Pins survive sort / filter / pagination as long as row identity is preserved. Since 3.6, replacing `grid.data` carries the pins onto the new rows: matched by `rowId` when it is set, otherwise a pin whose object left the data is dropped (re-pin those). While `rowPinning.enabled` is `false`, `pinnedRows` reports none and the rows return to the body, but the pins are kept (`getState` still saves them) and come back when pinning is re-enabled.

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

`moveRow(from, to, position?)` indices are into `grid.pageItems`; returns `boolean`, `false` when disabled, out of range, cancelled, or (3.6) a no-op move that would leave the row where it is, which fires no events. A manual order is **mutually exclusive with column sorting**: applying a sort clears the manual order, and since 3.6 a move on a sorted grid clears the sort and keeps the moved order (on 3.5 an `applyToData` move was undone by the re-sort). Rows hidden by a filter keep their place when a visible row moves, and replacing `grid.data` carries the order onto the new rows (by `rowId` when set).

Events: `rowMoving` (cancellable) → `rowMoved`, both `CustomEvent<ApexRowMovingEvent<T>>`:

```ts
interface ApexRowMovingEvent<T> {
  from: number;                     // view index before the move
  to: number;                       // the target row's view index
  position: 'before' | 'after';     // 3.6: which side of the target it lands
  toIndex: number;                  // 3.6: the view index it lands at (moveRow(0, 3) lands at 2)
  data: T;                          // the moved record
}

// Mirror a move in your own array ordered like the view (the indices are view indices):
grid.addEventListener('rowMoved', ({ detail: { from, toIndex } }) => {
  order.splice(toIndex, 0, ...order.splice(from, 1));
});
```

Pressing Escape after keyboard grab-and-move steps restores the original order (and `grid.data` with `applyToData`, and any sort) and fires `rowMoved` for the way back, so a persisted order stays in sync.

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

**Data reloads (3.6).** Replacing `grid.data` with new records clears the history and fires `historyChanged`; before, undo replayed onto the old, detached objects and nothing changed on screen. Data that keeps the edited records, such as `grid.data = [...grid.data, newRow]`, keeps its history, and so do in-place edits. A `cellValueChanged` fired by undo / redo reports `rowIndex: -1` when the record is pinned or on another page, so identify the row by `event.detail.data`, not by index.

## Batch edits: `applyEdits()` (3.5)

Writes many cells as a single operation. Each write still goes through the same choke point interactive editing uses, so it emits the cancellable `cellValueChanging` and then `cellValueChanged`, and runs the column's validators. What differs from looping `editCell` + `commitEdit` is that the whole set lands as **one undo step and one pipeline run**: a thousand round-trips would otherwise mean a thousand undo entries and a thousand re-renders.

```ts
const tally = grid.applyEdits([
  { rowIndex: 0, column: 'discount', value: 0 },
  { rowIndex: 3, column: 'discount', value: 0 },
]);
// { applied: 2, unchanged: 0, invalid: 0, cancelled: 0, skipped: 0 }

// 3.6: name the row by record to reach every selected row, on any page
grid.applyEdits(grid.selectedRows.map((row) => ({ row, column: 'discount', value: 0 })));
```

Each entry names its row one of two ways:

- `rowIndex` is **view-relative** and matches `grid.pageItems`, the same as `editCell`, so it reaches the current page only. It must be a whole number from `0` to `pageItems.length - 1`.
- `row` (3.6) is the record itself, the same object held in `grid.data`. It also reaches rows on other pages, pinned rows and rows under a collapsed tree parent; the events and validators for those see `rowIndex` -1.

Order is preserved, so a later edit to the same cell wins.

It returns a per-outcome tally rather than a count, because "nothing changed" and "everything was rejected" are different answers:

| Key | Means |
|---|---|
| `applied` | The value actually changed. |
| `unchanged` | The cell already held that value. |
| `invalid` | A `column.validators` rule rejected it. |
| `cancelled` | A `cellValueChanging` listener called `preventDefault()`. |
| `skipped` | Unknown column, a `rowIndex` that is negative, fractional or past the end of the page (3.5 counted a negative index from the end and rounded a fraction down), or a column the user could not edit either (editing off, not `editable`, or hidden). |

Skipping rather than throwing means a partially stale batch still applies what it can. Needs `editing.enabled` and per-column `editable`, the same gate as interactive editing; without `editing.history.enabled` the batch simply is not recorded.

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

`rowIndex` is view-relative, or -1 for a pinned row or a record on another page (3.6). In row edit mode `ctx.data` is the **staged row** (3.6): a copy of the record (same prototype) with the row's staged values applied, so a cross-field rule judges the row being saved rather than the stored record. A row commit also re-validates the row's other editable cells against it.

Event: `cellValidationFailed` — `CustomEvent<{ key; rowIndex; data: T; value: unknown; errors: readonly string[] }>`.

## Editing behaviour (3.6)

- **Row mode edits a staged row.** The cells of the row in edit show their staged values. A boolean checkbox click, or a `cellTemplate` calling `ctx.commit(value)`, joins the row edit instead of writing straight to the record, so `cancelEdit()` rolls it back and `commitEdit()` writes it with the rest. Every cell passes validation and its `cellValueChanging` gate before any is written: one invalid or cancelled cell aborts the whole row.
- **An open editor stays on its record.** When the data, sort, filter or page changes while a cell is in edit, the editor follows the record it was opened for (on 3.5 it could jump to another row, and Enter wrote into the wrong record). The edit is cancelled when that record leaves the view (removed, filtered out, or replaced by a copy). `editingCell`, `editingRow` and the `rowIndex` of the edit events are -1 while the record is pinned or on another page; identify it by the event's `data`.

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

Event: `stateChanged`: `CustomEvent<{ state: GridState }>`, debounced after the render pipeline settles. Fires for programmatic (including `setState`) and UI changes. The payload is only computed while a listener is attached (zero-cost otherwise); identical snapshots don't re-fire, so compare before persisting to avoid a save loop. Attach the listener to the grid element itself: the grid counts only listeners registered on it, so a delegated listener on an ancestor receives nothing while none is attached to the grid.

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

`gotoPage` / `setPageSize` / `nextPage` / `previousPage` / `firstPage` / `lastPage` emit `pageChanging` → `pageChanged`. `setQuickFilter` emits `quickFilterChanging` → `quickFilterChanged`, plus (3.6) `pageChanged` when it moves the grid back to the first page. Since 3.6 the `*-ed` events fire after the pipeline run they caused, so awaiting the returned promise, or reading `grid.pageItems` in the listener, sees the new rows even with an async pipeline hook. `pinColumn` / `unpinColumn` emit `columnPinning` → `columnPinned` and only change the visual render order (the `columns` array isn't reordered).

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

**3.6 key additions** (English and Spanish) are used only by enterprise panels: `toolPanel.valueColumn`, `toolPanel.columnLabelColumn`, `chart.mapCategoryNone`, `setFilter.more`, `pivot.columnsTruncated`. A partial `localeText` map needs none of them; a full translation typed as `Record<GridLocaleKey, string>` must add them to compile.

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
| `'selected'` | the selected rows in the order the grid shows them, then selected rows the view hides (3.6; on 3.5, selection order) |
| `'all'` | the raw `data` array |

```ts
grid.exportToCSV();                                       // download data.csv (view)
grid.exportToCSV({ filename: 'users', source: 'selected' });
const text = grid.exportToCSV({ filename: '' });          // string only, no download
grid.exportToCSV({ escapeFormulas: false });              // 3.6: opt out of formula escaping
```

`exportFormats` *(getter)* → `readonly ExportFormat[]` (`{ id, label }`). The community grid offers CSV only; it drives the toolbar export menu, one item per entry, dispatching to `exportAs(formatId, options?)`. Other `ExportOptions` fields: `columns?` (narrows to these keys among the visible, exportable columns, kept in display order), `includeHeader?` (default `true`), `formatter?`; CSV (`CSVExportOptions`) adds `delimiter?` (`','`), `bom?` (`true`), `newline?` (`'\r\n'`), `escapeFormulas?` (`true`, 3.6).

**Export output changes in 3.6:**

- **Formula escaping.** A CSV text cell or header that starts with `=`, `+`, `-`, `@`, a tab or a carriage return is written with a leading apostrophe, so a spreadsheet shows it as text instead of running it. Numbers are untouched, and so is text that is a plain signed decimal (`-5`, `+1.5`). Other text with a leading sign gains the apostrophe too: a phone number such as `+44 20 7946 0958`, or formatter output such as `-$5.00`. Pass `escapeFormulas: false` for the 3.5 output.
- **Typed values.** `boolean` columns export `true` only for the value `true` (as the checkbox shows), so strings such as `'false'` or `'yes'` and the number `1` export as `false`. `currency` and `progress` columns export as numbers, like `number` and `rating`; a value that is not a number exports as an empty cell.

## Master-detail: styling detail content and nested grids (3.6)

Parts used inside your own `detailTemplate` content are not reachable with `apex-grid::part()` until you list them in `expansion.exportParts`. The panel itself is always reachable as `detail-panel`.

```ts
grid.expansion = {
  enabled: true,
  exportParts: ['order-summary'],
  detailTemplate: ({ data }) => html`<div part="order-summary">${data.total}</div>`,
};
```

```css
apex-grid::part(detail-panel)  { padding: 12px; }
apex-grid::part(order-summary) { font-weight: 600; }
```

A grid rendered inside a detail panel is nested in the outer grid's rows, and grid events bubble and are composed, so a listener on the outer grid also receives the inner grid's events (`cellValueChanged`, `rowSelected`, `sorted`, ...). `grid.ownsEvent(event)` returns `true` only for the grid's own events; call it inside the listener, while the event is dispatching. The grid applies the same test to its own interactions, so since 3.6 clicks, Space selection, Ctrl+Z and pointer drags in a nested grid no longer drive the outer one.

```ts
outer.addEventListener('rowSelected', (e) => {
  if (!outer.ownsEvent(e)) return;          // a nested grid's selection
  updateToolbar(outer.selectedRows);
});
```
