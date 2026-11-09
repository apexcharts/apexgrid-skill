# Sort & Filter — Operands, Expressions, Events

## Sort

### Sort expression shape

```ts
interface BaseSortExpression<T, K extends keyof T> {
  key: K;                                          // target column
  direction: 'ascending' | 'descending' | 'none';
  caseSensitive?: boolean;                         // overrides column config
  comparer?: (a: T[K], b: T[K]) => number;         // overrides column config
}
```

`'none'` is the explicit reset state and only meaningful when `sortConfiguration.triState: true`.

### Programmatic sort

```ts
grid.sort({ key: 'name', direction: 'ascending' });
grid.sort({ key: 'age', direction: 'descending' });

// Multi-column (requires sortConfiguration.multiple === true, default true)
grid.sort([
  { key: 'subscribed', direction: 'descending' },
  { key: 'age',        direction: 'ascending' },
]);

// Override the column comparer for this operation
grid.sort({
  key: 'priority',
  direction: 'ascending',
  comparer: (a, b) => ['low', 'standard', 'high'].indexOf(a) - ['low', 'standard', 'high'].indexOf(b),
});
```

`grid.sort(...)` does **not** fire `sorting` / `sorted` events. It **adds** to the current sort (a key already sorted is updated in place); to replace the sort, assign `sortExpressions`.

### How rows compare

With the built-in comparison, strings compare with `localeCompare` (case folded unless `caseSensitive: true`) and other values with `<` / `>`. Since 3.6:

- **Empty cells** (`null`, `undefined`, `NaN`, an invalid `Date`) sort after every real value in **both** directions. Rows that tie keep their incoming order (a stable sort).
- **A custom `comparer` receives the raw cell values**, original casing and empty values included, and owns their order; a descending sort reverses its result. On 3.5 it received strings already lower-cased, so a lookup comparer over mixed-case values (`'High'`, `'Low'`) ranked every row the same. Write the lookup in the data's own casing, and decide where `null` goes.

### Reading and assigning the sort state

```ts
grid.sortExpressions;            // SortExpression<T>[]

grid.sortExpressions = [         // 3.6: REPLACES the current sort
  { key: 'subscribed', direction: 'descending' },
  { key: 'age',        direction: 'ascending' },
];
grid.sortExpressions = [];       // clears it (so does null)
```

The array is in the order operations were applied (insertion order on the underlying `Map`).

> **Changed in 3.6.0:** assigning `sortExpressions` replaces the sort, as `setState` does; on 3.5 it merged into it and `[]` was a no-op. Code that assigned one expression at a time to build a multi-column sort now keeps only the last one: assign the full list, or call `grid.sort()`, which still adds. The list applies at the next update against the columns as they are then, so a framework template may bind `sortExpressions` before `columns`. Entries naming an unknown column, or with an unknown direction, are dropped with a console warning (an omitted direction sorts ascending).

### Clearing sort

```ts
grid.clearSort();                // clear everything
grid.clearSort('name');          // clear only the `name` column
grid.sortExpressions = [];       // 3.6: same as clearSort()
```

### `sortConfiguration`

```ts
grid.sortConfiguration = {
  multiple: true,                // default — allow multi-column sort
  triState: true,                // default — clicking a sorted-asc column twice goes ascending → descending → none
};
```

## Filter

### Filter expression shape

```ts
interface BaseFilterExpression<T, K extends keyof T> {
  key: K;
  condition: FilterOperation<T[K]> | OperandKeys<T[K]>;   // operand object or string key
  searchTerm?: T[K];                                       // optional for unary operands (empty / notEmpty / true / false / all)
  criteria?: 'and' | 'or';                                 // default 'and'; combines expressions of the SAME column
  caseSensitive?: boolean;                                 // overrides column config
}
```

Within one column, a row passes when **all** of its `and` expressions pass **or any one** of its `or` expressions passes. Columns always combine with AND: criteria never reach across columns (3.6; on 3.5 an `or` on one column could cancel the `or` conditions of another).

### Operand sets

```ts
import { StringOperands, NumberOperands, BooleanOperands } from 'apex-grid';
```

| Column `type` | Operand keys |
|---|---|
| `'string'` and every other type not listed below | `contains`, `doesNotContain`, `startsWith`, `endsWith`, `equals`, `doesNotEqual`, `empty`, `notEmpty` |
| `'number'`, `'currency'`, `'rating'`, `'progress'` | `equals`, `doesNotEqual`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `empty`, `notEmpty` |
| `'boolean'` | `all`, `true`, `false`, `empty`, `notEmpty` |

Unary operands (no `searchTerm` needed): `empty`, `notEmpty`, `all`, `true`, `false`.

Since 3.6 the numeric presentation types `currency`, `rating` and `progress` take the number operands (on 3.5 they took the text ones). A condition name that does not belong to the column's set, such as `contains` on a `currency` column, is dropped with a console warning, from `filter()` and from a saved state alike. Text conditions read `null` / `undefined` as empty text and match other values (numbers, dates) on their text instead of throwing, and the range conditions (`greaterThan`, `lessThan`, ...) no longer match an empty cell as if it were 0.

**Typed numbers in the filter panel (Changed in 4.0.0).** On a numeric column (`number`, `currency`, `rating`, `progress`) the filter panel reads a number as a person writes it: full-width digits and signs (`１２３`, `－５`), the grouping and decimal separator of the column's `locale` (or the runtime locale), and a leading or trailing `¥`, `$`, `€`, `円` or the column currency's symbol. On 3.6 `１２３` did not filter and `1,000` filtered by 1. A plain ASCII decimal reads as before in every locale (`1.000` filters as 1 even under `'de-DE'`), the input keeps the text as typed, and text the parser does not read falls back to its leading number as in 3.x (`12.5%` filters by 12.5, `10²` by 10). The `min()` / `max()` validators have no such fallback: `12.5%` or `10²` there is not a number, so the value fails. Programmatic `filter()` / `filterExpressions` terms are not parsed: pass a number as `searchTerm`.

```ts
{ key: 'price', type: 'currency', currency: 'EUR', locale: 'de-DE', filter: true }   // typing 1.234,5 filters by 1234.5
```

**IME input (4.0).** Japanese, Chinese or Korean typed through an input method works in the filter panel and the quick filter: the Enter, Escape and Tab that confirm or cancel a conversion stay with the IME (they do not close the panel or clear the quick filter), the text applies once the conversion ends rather than on every unconverted keystroke, and a quick filter search still waiting out its debounce is dropped when a conversion starts. Typing without an IME filters as before.

### Programmatic filter

```ts
// String operand by name
grid.filter({ key: 'name', condition: 'contains', searchTerm: 'ada' });

// String operand by reference (typed)
grid.filter({ key: 'name', condition: StringOperands.contains, searchTerm: 'ada' });

// Number operand
grid.filter({ key: 'age', condition: 'greaterThan', searchTerm: 30 });

// Boolean operand
grid.filter({ key: 'subscribed', condition: 'true' });

// Unary — no searchTerm
grid.filter({ key: 'avatar', condition: 'empty' });

// Combined AND / OR
grid.filter([
  { key: 'age', condition: 'greaterThan', searchTerm: 30 },
  { key: 'age', condition: 'lessThan',    searchTerm: 50, criteria: 'and' },
]);
grid.filter([
  { key: 'priority', condition: 'equals', searchTerm: 'high' },
  { key: 'priority', condition: 'equals', searchTerm: 'standard', criteria: 'or' },
]);
```

`grid.filter(...)` does **not** fire `filtering` / `filtered` events. Like `grid.sort()`, it **adds** to the current filter; assign `filterExpressions` to replace it.

### Reading and assigning the filter state

```ts
grid.filterExpressions;          // FilterExpression<T>[]

grid.filterExpressions = [       // 3.6: REPLACES the current filter
  { key: 'age', condition: 'greaterThan', searchTerm: 30 },
];
grid.filterExpressions = [];     // clears it (so does null)
```

Assignment follows the same 3.6 rules as `sortExpressions`: it replaces (3.5 merged, and `[]` was a no-op), applies at the next update, and assigning a list equal to the current one (a wrapper that re-binds it on every render) keeps the scroll position. Any other filter change scrolls the grid back to its first row.

Combined across columns. To get state for a single column, find by `key`:

```ts
const ageFilters = grid.filterExpressions.filter((e) => e.key === 'age');
```

### Clearing filter

```ts
grid.clearFilter();              // clear everything
grid.clearFilter('age');         // clear only the `age` column
```

## Events

```ts
// UI-initiated sort about to apply — cancellable, mutable
grid.addEventListener('sorting', (e) => {
  // e: CustomEvent<SortExpression<T>>
  if (e.detail.key === 'id') e.preventDefault();        // veto
  if (e.detail.direction === 'descending')              // rewrite
    e.detail.direction = 'ascending';
});

// UI-initiated sort completed
grid.addEventListener('sorted', (e) => {
  // e: CustomEvent<SortExpression<T>>
});

// UI-initiated filter about to apply — cancellable, mutable
grid.addEventListener('filtering', (e) => {
  // e: CustomEvent<{ key, expressions, type: 'add'|'modify'|'remove' }>
});

// UI-initiated filter completed
grid.addEventListener('filtered', (e) => {
  // e: CustomEvent<{ key, state: FilterExpression<T>[] }>
});
```

| Event | Cancellable | `event.detail` | Fires for programmatic? |
|---|---|---|---|
| `sorting` | yes | `SortExpression<T>` (mutable) | no |
| `sorted` | no | `SortExpression<T>` | no |
| `filtering` | yes | `{ key, expressions, type }` | no |
| `filtered` | no | `{ key, state }` | no |

Since 3.6, `sorted` and `filtered` fire after the pipeline run they caused, so a listener reading `grid.dataView` or `grid.totalItems` sees the new rows, even when a `dataPipelineConfiguration` hook is async. On 3.5 they fired after the next render, which could come before an async hook resolved.

If you need a side-effect on every change regardless of source, watch `grid.dataView` (after `requestUpdate` / via `updated()` in a Lit parent) or wrap your programmatic calls.

## Combining UI + programmatic operations

UI operations and programmatic operations write to the **same underlying state** (the `StateController`). Order matters:

```ts
// Persisting and restoring state across mounts
const savedSort = grid.sortExpressions;
const savedFilter = grid.filterExpressions;

// ... later, after re-mounting: assignment replaces whatever is there (3.6)
grid.sortExpressions = savedSort;
grid.filterExpressions = savedFilter;
```

`grid.sort(savedSort)` / `grid.filter(savedFilter)` would add onto any sort or filter already applied instead. For a full snapshot (columns, paging, selection too), use `getState()` / `setState()` (`references/state-and-features.md`).

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Listening for events on programmatic calls | Wrap the call yourself, or watch `grid.dataView` |
| `grid.filter({ key: 'age', condition: 'contains' })` (string operand on number column) | Match operand to type: `condition: 'greaterThan'` for `number`, `currency`, `rating` and `progress` columns (a mismatched name is dropped with a console warning) |
| `grid.sortExpressions = [a]; grid.sortExpressions = [b];` to build a two-column sort | Each assignment replaces (3.6): assign `[a, b]`, or `grid.sort(b)` to add |
| Restoring with `grid.sort(saved)` onto a grid that already has a sort | `grid.sortExpressions = saved` replaces it (3.6) |
| A `comparer` that expects lower-cased strings | Since 3.6 it receives raw values (original casing, `null` included) |
| Multi-column sort with `sortConfiguration.multiple = false` | Either set `multiple: true` or call `grid.sort` with a single expression |
| Forgetting `searchTerm` on a non-unary operand | Add `searchTerm`, or use a unary operand (`empty` / `notEmpty` / `true` / `false` / `all`) |
| Mutating `e.detail` in a `sorted` / `filtered` listener | Only `sorting` / `filtering` are cancellable / mutable; the post-events are read-only by design |
