# Framework Integration — Lit, React, Vue, Angular

`apex-grid` is a standard custom element with no first-party framework wrappers. Every framework consumes it the same way: register the tag once, then bind `data` and `columns` as **properties** (not attributes) and listen for `sorting` / `sorted` / `filtering` / `filtered` events.

## Lit (recommended)

The most natural host — Lit `html` templates already understand the `.property` and `@event` binding syntax.

```ts
import 'apex-grid/define';
import { html, render } from 'lit';
import type { ColumnConfiguration } from 'apex-grid';

type User = { id: number; name: string; age: number; subscribed: boolean };

const data: User[] = [/* … */];
const columns: ColumnConfiguration<User>[] = [
  { key: 'id', type: 'number', sort: true, filter: true },
  { key: 'name', type: 'string', sort: true, filter: true,
    cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
  { key: 'age', type: 'number', sort: true, filter: true },
  { key: 'subscribed', type: 'boolean', sort: true, filter: true },
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

## React

React 19+ supports custom elements natively, including property binding via `ref` for non-string values. For broad compatibility, use a `useEffect` + `ref` pattern:

```tsx
import { useEffect, useRef } from 'react';
import 'apex-grid/define';
import type { ApexGrid, ColumnConfiguration } from 'apex-grid';
import { html } from 'lit';

type User = { id: number; name: string; age: number; subscribed: boolean };

export function UsersGrid({ data }: { data: User[] }) {
  const ref = useRef<ApexGrid<User> | null>(null);

  useEffect(() => {
    const grid = ref.current;
    if (!grid) return;

    grid.columns = [
      { key: 'id',         type: 'number',  sort: true, filter: true },
      { key: 'name',       type: 'string',  sort: true, filter: true,
        cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
      { key: 'age',        type: 'number',  sort: true, filter: true },
      { key: 'subscribed', type: 'boolean', sort: true, filter: true },
    ];

    const onSorted = (e: Event) => console.log('sorted', (e as CustomEvent).detail);
    grid.addEventListener('sorted', onSorted);
    return () => grid.removeEventListener('sorted', onSorted);
  }, []);

  // Set data whenever it changes
  useEffect(() => {
    if (ref.current) ref.current.data = data;
  }, [data]);

  // @ts-expect-error — apex-grid is a custom element
  return <apex-grid ref={ref} style={{ height: '500px', display: 'block' }} />;
}
```

For full TypeScript / JSX intrinsic-element support, declare:

```ts
// apex-grid.d.ts
import type { ApexGrid } from 'apex-grid';

declare module 'react' {
  namespace JSX {
    interface IntrinsicElements {
      'apex-grid': React.DetailedHTMLProps<React.HTMLAttributes<ApexGrid<any>>, ApexGrid<any>>;
    }
  }
}
```

## Vue 3

Vue understands property binding for custom elements out of the box. Mark `apex-grid` as a custom element in the compiler config so Vue doesn't try to resolve it as a Vue component:

```ts
// vite.config.ts
export default defineConfig({
  plugins: [
    vue({ template: { compilerOptions: { isCustomElement: (tag) => tag.startsWith('apex-') } } }),
  ],
});
```

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import 'apex-grid/define';
import { html } from 'lit';
import type { ApexGrid, ColumnConfiguration } from 'apex-grid';

type User = { id: number; name: string; age: number; subscribed: boolean };

const grid = ref<ApexGrid<User> | null>(null);

const data = ref<User[]>([/* … */]);
const columns: ColumnConfiguration<User>[] = [
  { key: 'id',         type: 'number',  sort: true, filter: true },
  { key: 'name',       type: 'string',  sort: true, filter: true,
    cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
  { key: 'age',        type: 'number',  sort: true, filter: true },
  { key: 'subscribed', type: 'boolean', sort: true, filter: true },
];

onMounted(() => {
  grid.value!.columns = columns;
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
    style="height: 500px; display: block;"
  />
</template>
```

> Use `:data.prop="data"` (or set via `ref` in `onMounted`) to bind as a property. Vue's default `:data="data"` sets an attribute (a stringified object) which `apex-grid` will not read.

## Angular

Add `CUSTOM_ELEMENTS_SCHEMA` to the module / standalone component, then use property binding `[data]="..."` and event binding `(sorted)="..."`:

```ts
// app.component.ts
import { Component, CUSTOM_ELEMENTS_SCHEMA, OnInit, ViewChild, ElementRef } from '@angular/core';
import 'apex-grid/define';
import { html } from 'lit';
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
      (sorted)="onSorted($event)"
      style="height: 500px; display: block;"
    ></apex-grid>
  `,
})
export class UsersGridComponent implements OnInit {
  @ViewChild('grid', { static: true }) grid!: ElementRef<ApexGrid<User>>;

  data: User[] = [/* … */];

  ngOnInit() {
    this.grid.nativeElement.columns = [
      { key: 'id',         type: 'number',  sort: true, filter: true },
      { key: 'name',       type: 'string',  sort: true, filter: true,
        cellTemplate: ({ value }) => html`<strong>${value}</strong>` },
      { key: 'age',        type: 'number',  sort: true, filter: true },
      { key: 'subscribed', type: 'boolean', sort: true, filter: true },
    ];
  }

  onSorted(e: Event) {
    console.log('sorted', (e as CustomEvent).detail);
  }
}
```

## Common pitfalls across frameworks

| ❌ | ✅ |
|---|---|
| Vue: `:data="data"` (attribute, becomes `"[object Object]"`) | `:data.prop="data"` or set via `ref` in `onMounted` |
| React: `<apex-grid columns={[...]}>` before React 19 (passes as attribute) | Use `ref` + `useEffect` to set as property, or React 19+ |
| Angular: missing `CUSTOM_ELEMENTS_SCHEMA` | Add to the component / module schemas |
| All: forgetting to register the element | Top-level `import 'apex-grid/define'` once |
| Lit: writing string templates instead of `html` literals | Always import `html` from `lit` and use tagged literals |
| Treating `cellTemplate` as framework-specific JSX | Always use Lit `html\`...\`` — the template runs inside the web component, not in the host framework |
