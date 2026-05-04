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

`grid.sort(...)` does **not** fire `sorting` / `sorted` events.

### Reading the active sort state

```ts
grid.sortExpressions;            // SortExpression<T>[]
```

The array is in the order operations were applied (insertion order on the underlying `Map`).

### Clearing sort

```ts
grid.clearSort();                // clear everything
grid.clearSort('name');          // clear only the `name` column
```

> **Heads-up:** assigning an empty array (`grid.sortExpressions = []`) is a no-op. Always call `clearSort()` to reset.

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
  criteria?: 'and' | 'or';                                 // for combining with previous expression on same column
  caseSensitive?: boolean;                                 // overrides column config
}
```

### Operand sets

```ts
import { StringOperands, NumberOperands, BooleanOperands } from 'apex-grid';
```

| Type | Operand keys |
|---|---|
| `'string'` | `contains`, `doesNotContain`, `startsWith`, `endsWith`, `equals`, `doesNotEqual`, `empty`, `notEmpty` |
| `'number'` | `equals`, `doesNotEqual`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `empty`, `notEmpty` |
| `'boolean'` | `all`, `true`, `false`, `empty`, `notEmpty` |

Unary operands (no `searchTerm` needed): `empty`, `notEmpty`, `all`, `true`, `false`.

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

`grid.filter(...)` does **not** fire `filtering` / `filtered` events.

### Reading the active filter state

```ts
grid.filterExpressions;          // FilterExpression<T>[]
```

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

If you need a side-effect on every change regardless of source, watch `grid.dataView` (after `requestUpdate` / via `updated()` in a Lit parent) or wrap your programmatic calls.

## Combining UI + programmatic operations

UI operations and programmatic operations write to the **same underlying state** (the `StateController`). Order matters:

```ts
// Persisting and restoring state across mounts
const savedSort = grid.sortExpressions;
const savedFilter = grid.filterExpressions;

// ... later, after re-mounting
grid.sort(savedSort);
grid.filter(savedFilter);
```

## Common pitfalls

| ❌ | ✅ |
|---|---|
| Listening for events on programmatic calls | Wrap the call yourself, or watch `grid.dataView` |
| `grid.filter({ key: 'age', condition: 'contains' })` (string operand on number column) | Match operand to type — `condition: 'greaterThan'` for number columns |
| `grid.sortExpressions = []` to clear | `grid.clearSort()` |
| Multi-column sort with `sortConfiguration.multiple = false` | Either set `multiple: true` or call `grid.sort` with a single expression |
| Forgetting `searchTerm` on a non-unary operand | Add `searchTerm`, or use a unary operand (`empty` / `notEmpty` / `true` / `false` / `all`) |
| Mutating `e.detail` in a `sorted` / `filtered` listener | Only `sorting` / `filtering` are cancellable / mutable; the post-events are read-only by design |
