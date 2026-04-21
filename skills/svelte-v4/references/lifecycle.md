# Lifecycle & Context — Svelte v4

All lifecycle functions must be called **synchronously during component initialization** — that is, during the first `<script>` pass, before `await` or inside regular functions. Calling them later throws.

## Hooks

### `onMount(fn)`

Runs after the component is mounted in the DOM. **Not called during SSR.**

```svelte
<script>
  import { onMount } from 'svelte';

  let data;

  onMount(async () => {
    const res = await fetch('/api/data');
    data = await res.json();
  });

  // Optional cleanup: return a function (or async-return it synchronously)
  onMount(() => {
    const id = setInterval(tick, 1000);
    return () => clearInterval(id);
  });
</script>
```

- The return-for-cleanup must be a **synchronous** return. `async () => ...` cannot return a cleanup fn because the Promise wraps it. Write the async work inside and return cleanup synchronously:

```ts
onMount(() => {
  let cancelled = false;
  (async () => {
    const r = await fetch('/x');
    if (!cancelled) data = await r.json();
  })();
  return () => { cancelled = true; };
});
```

### `onDestroy(fn)`

Runs before the component is removed. Called on both client and server.

```svelte
<script>
  import { onDestroy } from 'svelte';
  const unsub = store.subscribe(...);
  onDestroy(unsub);
</script>
```

### `beforeUpdate(fn)` / `afterUpdate(fn)`

Run on every reactive update cycle, before and after DOM patching respectively.

```svelte
<script>
  import { beforeUpdate, afterUpdate } from 'svelte';
  import { tick } from 'svelte';

  let autoscroll = false;
  let list;

  beforeUpdate(() => {
    autoscroll = list && list.offsetHeight + list.scrollTop > list.scrollHeight - 20;
  });

  afterUpdate(() => {
    if (autoscroll) list.scrollTo(0, list.scrollHeight);
  });
</script>
```

- Called at least once on initial mount (after `onMount` for `afterUpdate`).
- Avoid heavy work here — prefer reactive statements.

### `tick()`

```svelte
<script>
  import { tick } from 'svelte';

  async function addAndFocus() {
    items = [...items, newItem];
    await tick();              // DOM reflects the new item
    newInputEl.focus();
  }
</script>
```

`tick()` resolves once pending reactive state has been applied to the DOM. It is safe to `await` outside init.

## Context API

Context provides component-tree-scoped data without prop drilling. Keys are compared by **identity** (use `Symbol` or a shared constant, not a string literal, to avoid collisions).

```ts
// key.ts
export const THEME = Symbol('theme');
```

```svelte
<!-- Parent.svelte -->
<script>
  import { setContext } from 'svelte';
  import { THEME } from './key';
  import { writable } from 'svelte/store';

  const theme = writable<'light' | 'dark'>('light');
  setContext(THEME, theme);   // pass a store so children get reactivity
</script>

<slot />
```

```svelte
<!-- DeepChild.svelte -->
<script>
  import { getContext, hasContext } from 'svelte';
  import { THEME } from './key';

  const theme = getContext(THEME);       // same store instance
</script>

<p class:dark={$theme === 'dark'}>Themed</p>
```

API:

- `setContext(key, value)` — register on the current component. Must run during init.
- `getContext(key)` — read nearest ancestor's value. Must run during init.
- `hasContext(key)` — boolean check. Init-only.
- `getAllContexts()` — returns a `Map` of the current init scope. Init-only. Useful to clone context into a dynamically created child.

Patterns:

- **Share a store, not a plain value**, so that updates propagate reactively.
- **Scope per-request on SSR:** because stores at module level are shared across requests, move per-request state into a context provider at the root of each route component.
- **Typed context:** wrap `setContext`/`getContext` in a helper that narrows the key type:

```ts
import { getContext, setContext } from 'svelte';

export function createContextPair<T>(name: string) {
  const key = Symbol(name);
  return {
    set: (value: T) => setContext(key, value),
    get: () => getContext<T>(key),
  };
}
```

## SSR pitfalls

- `window`, `document`, `navigator`, `localStorage` — do **not** touch at module scope or in `<script>` top-level. Wrap in `onMount`, or gate with `typeof window !== 'undefined'`.
- Module-level `writable(...)` stores are singletons per server instance → leak data across requests. Move to per-request context.
- `setTimeout` / `setInterval` scheduled at init run on the server too. Guard or move into `onMount`.
- Fetching at init: prefer `onMount` on the client, or lift data loading to a framework layer (SvelteKit `load`, etc. — outside this skill's scope).
