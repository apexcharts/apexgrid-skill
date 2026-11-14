# Framework Integration — Lit, React, Vue, Angular

`apex-grid` is a standard custom element, and every framework can consume it directly (React also has an optional first-party wrapper package, `react-apex-grid`, below). Used directly, every framework consumes it the same way:

1. **Register the element + size the host once at app startup** — `setup()` does both (registers `<apex-grid>` and adopts a bounded host height). No theme CSS import is needed; the grid self-styles via `--ag-*` variables.
2. **Bind `data` and `columns` as properties** (not attributes).
3. **Listen for `sorting` / `sorted` / `filtering` / `filtered` (and the other UI) events** as standard DOM events.

The `setup()` call ideally lives in your application entry file (`main.ts` / `main.js` / `app.module.ts`) so it runs once, not per component.

---

## App-entry setup (all frameworks)

Put this at the top of your application entry point — `src/main.ts` for Vite/Lit/Vue, `src/index.tsx` for React, `src/main.ts` for Angular:

```ts
import { setup } from 'apex-grid';
setup();                         // registers <apex-grid> + adopts a bounded host height
```

No theme import needed: the grid self-styles. Retheme by overriding `--ag-*` CSS variables on `apex-grid` or on any ancestor (an app shell, a component's shadow host), and use the `--ag-dark-*` variables for grids with `theme="dark"` (SKILL.md §1.3). If you prefer manual control over sizing, call `setup({ hostStyles: false })` (or use `import 'apex-grid/define'`) and add your own rule:

```css
apex-grid {
  height: 480px;     /* or whatever fits your layout; + any --ag-* overrides */
}
```

The framework examples below assume this app-entry setup is already done.

---

## Lit

The most natural host — Lit `html` templates already understand `.property` and `@event` binding syntax.

```ts
// src/main.ts — done once
import { setup } from 'apex-grid';
setup();

// component
import { html, render } from 'lit';
import type { ColumnConfiguration } from 'apex-grid';

type User = { id: number; name: string; age: number; subscribed: boolean };

const data: User[] = [/* … */];
const columns: ColumnConfiguration<User>[] = [
  { key: 'id',         type: 'number',  width: '80px',  sort: true, filter: true },
  { key: 'name',       type: 'string',  width: '240px', sort: true, filter: true,
    cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
  { key: 'age',        type: 'number',  width: '100px', sort: true, filter: true },
  { key: 'subscribed', type: 'boolean', width: '140px', sort: true, filter: true },
];

render(
  html`<apex-grid
    .data=${data}
    .columns=${columns}
    @sorted=${(e) => console.log('sorted', e.detail)}
    @filtered=${(e) => console.log('filtered', e.detail)}
  ></apex-grid>`,
  document.getElementById('app')!,
);
```

---

## React

React 19+ supports custom elements natively, including property binding for non-string values. For broad compatibility, use `ref` + `useEffect`:

```tsx
import { useEffect, useRef } from 'react';
import { html } from 'lit';                              // required for cellTemplate
import type { ApexGrid, ColumnConfiguration } from 'apex-grid';

type User = { id: number; name: string; age: number; subscribed: boolean };

export function UsersGrid({ data }: { data: User[] }) {
  const ref = useRef<ApexGrid<User> | null>(null);

  useEffect(() => {
    const grid = ref.current;
    if (!grid) return;

    grid.columns = [
      { key: 'id',         type: 'number',  width: '80px',  sort: true, filter: true },
      { key: 'name',       type: 'string',  width: '240px', sort: true, filter: true,
        cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
      { key: 'age',        type: 'number',  width: '100px', sort: true, filter: true },
      { key: 'subscribed', type: 'boolean', width: '140px', sort: true, filter: true },
    ];

    const onSorted = (e: Event) => console.log('sorted', (e as CustomEvent).detail);
    grid.addEventListener('sorted', onSorted);
    return () => grid.removeEventListener('sorted', onSorted);
  }, []);

  useEffect(() => {
    if (ref.current) ref.current.data = data;            // set property whenever data changes
  }, [data]);

  // The global `apex-grid { height: ... }` CSS handles sizing.
  // @ts-expect-error — apex-grid is a custom element
  return <apex-grid ref={ref} />;
}
```

If you'd rather size per instance instead of globally, replace the global CSS rule with a `style` prop: `style={{ height: '500px' }}`. Never add `display` there: the grid needs its own `display: grid`, collapses without it, and logs a console warning on first render when something overrides it.

For full TypeScript / JSX intrinsic-element support:

```ts
// types/apex-grid.d.ts
import type { ApexGrid } from 'apex-grid';

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'apex-grid': React.DetailedHTMLProps<React.HTMLAttributes<ApexGrid<any>>, ApexGrid<any>>;
    }
  }
}
```

### Optional: the `react-apex-grid` wrapper

`react-apex-grid` (0.3.0) wraps the element with `@lit/react`: props instead of a ref for `data` / `columns`, `on<Event>` handlers, and a typed ref. It registers `<apex-grid>` itself in the browser (and nothing on a server). Its peers are `apex-grid` `^3.5.0 || ^4.0.0 || ^5.0.0`, `react` and `react-dom` 18 or later.

```tsx
import { createApexGrid } from 'react-apex-grid';
const UserGrid = createApexGrid<User>();          // data: User[], columns: ColumnConfiguration<User>[]

<UserGrid data={users} columns={columns} style={{ height: 480 }}
          onSorted={(e) => console.log(e.detail)} />
```

- **Props are the grid's settable properties only** (0.2.0), plus the `theme` attribute, `on<Event>` handlers and standard HTML props. Read-only members (`totalItems`, `pageCount`, `canUndo`, `dataView`, ...) and methods (`sort`, `getState`, ...) are TypeScript errors as props (on 0.1.x passing a read-only one threw at runtime and unmounted the React root): read or call them through the ref (`ApexGridElement<T>`).
- **No `ApexGridToolbar` / `ApexGridPaginator` components** (removed in 0.2.0): they rendered an empty element outside the grid. The grid renders its own toolbar and paginator; configure them with `showQuickFilter`, `showExport`, `exportFilename` and `pagination`.
- **Enterprise: `react-apex-grid/enterprise`** (0.3.0) wraps `<apex-grid-enterprise>` (`ApexGridEnterprise`, and the row-typed `createApexGridEnterprise<T>()`) and the six companion elements: `ApexGridToolPanel`, `ApexGridStatusBar`, `ApexGridSetFilter`, `ApexGridFilterBuilder`, `ApexGridChart` and `ApexGridAI`. `apex-grid-enterprise` (0.10.0 or later, below 1.0) is an optional peer that only this entry imports, so community apps install nothing new. Importing the entry turns on every enterprise feature module and registers the elements in the browser only. Enterprise properties (`groupBy`, `aggregations`, `pivotOn`, `rangeSelection`, ...) are props and enterprise events map to handlers by the same rule (`apex-range-changed` is `onApexRangeChanged`). A companion follows a grid through its `grid` prop: keep the grid element in state with a ref callback so the companion re-renders once the grid mounts.

```tsx
import { useState } from 'react';
import { ApexGridStatusBar, type ApexGridEnterpriseElement, createApexGridEnterprise } from 'react-apex-grid/enterprise';
const SalesGrid = createApexGridEnterprise<Sale>();

function Sales({ sales }: { sales: Sale[] }) {
  const [grid, setGrid] = useState<ApexGridEnterpriseElement<Sale> | null>(null);
  return (<>
    <SalesGrid ref={setGrid} data={sales} columns={columns} groupBy={['region']} style={{ height: 480 }}
               onApexRangeChanged={(e) => console.log(e.detail.stats.sum)} />
    <ApexGridStatusBar grid={grid} />
  </>);
}
```

---

## Vue 3

### Vite plugin config

Tell the Vue compiler that `apex-grid` is a custom element so it doesn't try to resolve it as a Vue component:

```ts
// vite.config.ts
export default defineConfig({
  plugins: [
    vue({
      template: { compilerOptions: { isCustomElement: (tag) => tag.startsWith('apex-') } },
    }),
  ],
});
```

### Component

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { html } from 'lit';                              // required for cellTemplate
import type { ApexGrid, ColumnConfiguration } from 'apex-grid';

type User = { id: number; name: string; age: number; subscribed: boolean };

const grid = ref<ApexGrid<User> | null>(null);

const data = ref<User[]>([/* … */]);
const columns: ColumnConfiguration<User>[] = [
  { key: 'id',         type: 'number',  width: '80px',  sort: true, filter: true },
  { key: 'name',       type: 'string',  width: '240px', sort: true, filter: true,
    cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
  { key: 'age',        type: 'number',  width: '100px', sort: true, filter: true },
  { key: 'subscribed', type: 'boolean', width: '140px', sort: true, filter: true },
];

onMounted(() => {
  grid.value!.columns = columns;                         // set as property after mount
});

function onSorted(e: CustomEvent) {
  console.log('sorted', e.detail);
}
</script>

<template>
  <apex-grid
    ref="grid"
    :data.prop="data"
    @sorted="onSorted"
  />
</template>
```

> Use `:data.prop="data"` to bind as a property. Vue's default `:data="data"` sets an attribute, which becomes the literal string `"[object Object]"` and the grid renders nothing.

---

## Angular

Add `CUSTOM_ELEMENTS_SCHEMA` to the component (or module). Use `[data]="..."` for property binding and `(sorted)="..."` for events:

```ts
import { Component, CUSTOM_ELEMENTS_SCHEMA, OnInit, ViewChild, ElementRef } from '@angular/core';
import { html } from 'lit';                              // required for cellTemplate
import type { ApexGrid, ColumnConfiguration } from 'apex-grid';

type User = { id: number; name: string; age: number; subscribed: boolean };

@Component({
  selector: 'app-users-grid',
  standalone: true,
  schemas: [CUSTOM_ELEMENTS_SCHEMA],
  template: `
    <apex-grid
      #grid
      [data]="data"
      (sorted)="onSorted($event)">
    </apex-grid>
  `,
})
export class UsersGridComponent implements OnInit {
  @ViewChild('grid', { static: true }) grid!: ElementRef<ApexGrid<User>>;

  data: User[] = [/* … */];

  ngOnInit() {
    this.grid.nativeElement.columns = [
      { key: 'id',         type: 'number',  width: '80px',  sort: true, filter: true },
      { key: 'name',       type: 'string',  width: '240px', sort: true, filter: true,
        cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
      { key: 'age',        type: 'number',  width: '100px', sort: true, filter: true },
      { key: 'subscribed', type: 'boolean', width: '140px', sort: true, filter: true },
    ];
  }

  onSorted(e: Event) {
    console.log('sorted', (e as CustomEvent).detail);
  }
}
```

---

## Common pitfalls across frameworks

| ❌ | ✅ |
|---|---|
| Forgetting to register the element | Top-level `import 'apex-grid/define'` once in the app entry |
| Importing an Ignite UI theme + `configureTheme(name)` | Obsolete: the grid self-styles; retheme via `--ag-*` CSS variables. To follow an Ignite UI theme on the page, `import 'apex-grid/ignite-ui'` once in the app entry (apex-grid 4.0; no extra licence key). Without it the 4.0 grid ignores `--ig-*` |
| A stateful widget (a framework component mounted into a cell, a third-party picker) inside `cellTemplate` showing the wrong row's state after a scroll or sort | Body rows are recycled by default in 4.0: read the record from `ctx.data`, keep the state in your data, wrap the template in `keyed(ctx.data, ...)`, or set `grid.rowRecycling = false` (`row-recycling="false"`) |
| `show-header={false}` / `:show-header="false"` expecting a boolean | Boolean attributes such as `show-header` and `row-recycling` turn off only on the string `"false"`; bind the property instead (`.showHeader=${false}`, `:showHeader.prop="false"`, `[showHeader]="false"`, or `el.showHeader = false`) |
| Pinning `theme="light"` on a page with a dark Ignite UI theme | Leave the `theme` attribute off so the grid follows `--ig-theme-variant`; `design-system="off"` / `tokens="off"` opt one grid out |
| Forgetting `setup()` / no host-height rule for `apex-grid` | `setup()` registers + sizes the host. Or `import 'apex-grid/define'` + `apex-grid { height: ... }` (never set `display` — it overrides the component's `:host { display: grid }`) |
| Vue: `:data="data"` (attribute, becomes `"[object Object]"`) | `:data.prop="data"`, or set via `ref` in `onMounted` |
| React: `<apex-grid columns={[...]}>` before React 19 | Use `ref` + `useEffect` to set as property |
| Angular: missing `CUSTOM_ELEMENTS_SCHEMA` | Add to the component / module schemas |
| Writing string templates for cells | Always `import { html } from 'lit'` and use `html\`...\`` tagged literals |
| Treating `cellTemplate` as framework-specific JSX | The template runs inside the web component — only Lit `html` works |
