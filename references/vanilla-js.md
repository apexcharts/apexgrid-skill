# Vanilla JavaScript — Without Lit's `render` in Your App Code

`apex-grid` is built with Lit internally, but you don't have to use Lit's `render` / `html` in your application code. You can write a pure-DOM page that imports the custom element, sets properties directly, and listens for events with `addEventListener`.

**One honest caveat:** `cellTemplate` and `headerTemplate` are the **only** places that still require Lit's `html` tag — they must return a `TemplateResult` and there is no string-template escape hatch. Returning a plain string renders as escaped text. If you only need the default rendering (no custom cells), you can avoid Lit entirely.

---

## 1. Pure-DOM, default rendering (no Lit at all)

Open this in a Vite dev server (`npm create vite@latest`, then `npm run dev`):

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Apex Grid — vanilla JS</title>
    <style>
      body { margin: 0; padding: 20px; font-family: system-ui; }
      apex-grid { height: 480px; }                    /* REQUIRED — bounded host. NEVER set `display` here — it overrides the component's `:host { display: grid }` and collapses the virtualizer. */
    </style>
  </head>
  <body>
    <h1>Users</h1>
    <apex-grid id="grid"></apex-grid>

    <script type="module">
      // 1. Register the element + adopt host sizing (no theme import needed)
      import { setup } from 'apex-grid';
      setup();

      // 2. Set properties on the element
      const grid = document.getElementById('grid');

      grid.columns = [
        { key: 'id',         type: 'number',  headerText: 'ID',         width: '80px',  sort: true, filter: true },
        { key: 'name',       type: 'string',  headerText: 'Name',       width: '240px', sort: true, filter: true },
        { key: 'age',        type: 'number',  headerText: 'Age',        width: '100px', sort: true, filter: true },
        { key: 'subscribed', type: 'boolean', headerText: 'Subscribed', width: '140px', sort: true, filter: true },
      ];

      grid.data = [
        { id: 1, name: 'Ada Lovelace',  age: 36, subscribed: true  },
        { id: 2, name: 'Carl Sagan',    age: 62, subscribed: false },
        { id: 3, name: 'Grace Hopper',  age: 85, subscribed: true  },
      ];

      // 3. Listen for events
      grid.addEventListener('sorted',   (e) => console.log('sorted',   e.detail));
      grid.addEventListener('filtered', (e) => console.log('filtered', e.detail));
    </script>
  </body>
</html>
```

This works without any `import { html, render } from 'lit'` in your code. The grid renders, sorts, and filters with the default cell display.

---

## 2. When you need a custom cell — minimal Lit usage

The moment you want to style a cell (e.g. render the `subscribed` column as a checkbox), you must import `html` from Lit. There's no way around it.

```js
// Add this to the imports above:
import { html } from 'lit';

grid.columns = [
  { key: 'id',   type: 'number',  width: '80px',  sort: true, filter: true },
  { key: 'name', type: 'string',  width: '240px', sort: true, filter: true,
    cellTemplate: ({ value }) => html`<strong>${value}</strong>`,
  },
  { key: 'age',  type: 'number',  width: '100px', sort: true, filter: true },
  { key: 'subscribed', type: 'boolean', width: '140px', sort: true, filter: true,
    cellTemplate: ({ value }) => html`<input type="checkbox" .checked=${value} disabled />`,
  },
];
```

You're still not calling Lit's `render(...)` anywhere — you only import `html` to author template values that the custom element internally renders for each cell.

### Why not a string template?

```js
// ❌ This renders as literal text: <strong>Ada Lovelace</strong>
cellTemplate: ({ value }) => `<strong>${value}</strong>`,
```

The cell renderer treats the return value as a Lit template, not as HTML markup. Strings are coerced to text nodes, so any `<tag>` characters are escaped. There is no `dangerouslySetInnerHTML` equivalent.

---

## 3. Auto-generation — the genuinely 3-line demo

If you don't care about specific columns and just want to see your data in a grid:

```html
<apex-grid auto-generate id="grid"></apex-grid>

<script type="module">
  import { setup } from 'apex-grid';
  setup();

  document.getElementById('grid').data = [
    { id: 1, name: 'Ada',  age: 36 },
    { id: 2, name: 'Carl', age: 62 },
  ];
</script>
```

Auto-generation infers columns from `Object.keys(data[0])` and runtime types. `sort` and `filter` default to `false`, so the auto path is read-only out of the box. To enable them with auto-generated columns, run `updateColumns` after the auto-pass:

```js
grid.updateColumns([
  { key: 'id',   sort: true, filter: true },
  { key: 'name', sort: true, filter: true },
  { key: 'age',  sort: true, filter: true },
]);
```

---

## 4. Programmatic sort & filter without templates

```js
grid.sort({ key: 'age', direction: 'descending' });

grid.filter({ key: 'age', condition: 'greaterThan', searchTerm: 30 });

grid.filter({ key: 'subscribed', condition: 'true' });    // boolean operands are unary

grid.clearSort();
grid.clearFilter();
```

Programmatic calls don't fire `sorting` / `sorted` / `filtering` / `filtered` events. If you need a hook for every state change regardless of source, watch `grid.dataView` after each programmatic call:

```js
grid.sort({ key: 'age', direction: 'descending' });
queueMicrotask(() => console.log('post-sort:', grid.dataView));
```

---

## 5. Common vanilla-JS gotchas

| ❌ | ✅ |
|---|---|
| Setting `grid.setAttribute('columns', JSON.stringify(cols))` | `grid.columns = cols` — must be a property, not an attribute |
| Forgetting `<script type="module">` | All `apex-grid` imports are ESM; you need `type="module"` |
| Opening the HTML with `file://` | Use a dev server — ESM imports fail CORS over `file://` |
| `cellTemplate: ({ value }) => '<b>' + value + '</b>'` | `cellTemplate: ({ value }) => html\`<b>${value}</b>\`` — Lit `html` is required |
| Setting `grid.columns` *before* the script tag's module imports resolve | Set them inside the same `<script type="module">` as the imports |
