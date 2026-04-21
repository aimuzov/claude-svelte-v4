# Reactivity — Svelte v4

Svelte v4 reactivity is **assignment-based**: the compiler rewrites every `=`, `+=`, `-=`, `++`, `--` on top-level `let`/`var` bindings into "invalidate" calls. Mutations without reassignment are invisible to the compiler.

## Local state

```svelte
<script>
  let count = 0;       // reactive
  const max = 100;     // NOT reactive (const)
  let items = [];      // reactive, but mutation != update
</script>

<button on:click={() => count++}>+</button>
```

Rules:

- Only top-level `let` / `var` is tracked. Inside functions it isn't — assign the *outer* binding.
- `const` is not reactive; if you need reassignment, use `let`.
- Destructured bindings from a reactive object are tracked individually: `let { a, b } = state;` then `a = 1;` triggers update for `a`.

## Updating arrays and objects

```svelte
<script>
  let items = [{ id: 1 }];

  // Bad — mutation, no update
  items.push({ id: 2 });

  // Good — reassignment triggers reactivity
  items = [...items, { id: 2 }];
  items = items.filter((x) => x.id !== 1);

  // Also works — reassign the same reference
  items.push({ id: 2 });
  items = items;
</script>
```

Rule of thumb: after any mutation, write `x = x`.

## Reactive statements — `$:`

`$:` declares code that re-runs whenever any referenced reactive value changes. The compiler topologically sorts `$:` blocks by their dependencies.

```svelte
<script>
  export let price = 100;
  export let qty = 1;

  $: total = price * qty;                        // computed value
  $: discount = total > 500 ? 0.1 : 0;           // depends on total
  $: console.log(`total=${total}, discount=${discount}`);  // side effect

  $: {                                           // block
    if (qty > 0) triggerRefresh();
  }

  $: if (total < 0) total = 0;                   // reactive `if`
</script>
```

Caveats:

- Only **referenced** identifiers become dependencies. Values used via closures or dynamic property lookups (`obj[key]`) may not track as expected.
- Async inside `$:` is valid but awaited values do not become dependencies for subsequent runs — use a `derived` store for that.
- `$:` can also appear at the top of a `<script>` in any order; it does not have to follow the `let`.

## `tick()`

```svelte
<script>
  import { tick } from 'svelte';

  async function focusInput() {
    showInput = true;
    await tick();          // wait for DOM to flush
    inputEl.focus();
  }
</script>
```

`tick()` resolves after the current microtask-batch of pending state changes is applied to the DOM.

## Stores

Stores are any object with a `subscribe(fn)` method. Built-in factories live in `svelte/store`.

### `writable`

```ts
import { writable } from 'svelte/store';

export const count = writable(0, (set) => {
  // `start` fn runs when the first subscriber attaches.
  const id = setInterval(() => set(Date.now()), 1000);
  return () => clearInterval(id);   // `stop` when last subscriber leaves
});

count.set(10);
count.update((n) => n + 1);
```

### `readable`

```ts
import { readable } from 'svelte/store';

export const now = readable(new Date(), (set) => {
  const id = setInterval(() => set(new Date()), 1000);
  return () => clearInterval(id);
});
```

### `derived`

```ts
import { derived } from 'svelte/store';

// Sync: return value
export const doubled = derived(count, ($c) => $c * 2);

// Multiple inputs
export const summary = derived([a, b], ([$a, $b]) => `${$a}-${$b}`);

// Async: use set (or set + cleanup)
export const results = derived(query, ($q, set) => {
  let cancelled = false;
  fetch(`/api?q=${$q}`).then((r) => r.json()).then((v) => { if (!cancelled) set(v); });
  return () => { cancelled = true; };
}, [] /* initial value */);
```

### `get`

```ts
import { get } from 'svelte/store';
const snapshot = get(count);   // synchronous, but briefly subscribes+unsubscribes — avoid in hot paths
```

### Auto-subscription (`$store`)

Inside `<script>` and markup, the `$` prefix auto-subscribes and auto-unsubscribes on component teardown:

```svelte
<script>
  import { count } from './stores';
</script>

<p>{$count}</p>
<button on:click={() => $count += 1}>+</button>

{#if $count > 10}<Banner />{/if}
```

- Works only for top-level store variables (cannot do `$obj.store`).
- Works only inside components — not in plain `.ts`/`.js` modules (use `get()` or `subscribe()` there).

### Custom stores

Any object with a `subscribe` method is a store. Wrap a writable to expose a narrower API:

```ts
function createTodos() {
  const { subscribe, update } = writable<Todo[]>([]);
  return {
    subscribe,
    add: (text: string) => update((xs) => [...xs, { id: Date.now(), text }]),
    remove: (id: number) => update((xs) => xs.filter((t) => t.id !== id)),
  };
}

export const todos = createTodos();
// Still works with `$todos` and store contract.
```

## Pitfalls

- **Mutation without reassign:** `obj.field = 1` won't update the DOM. Write `obj = { ...obj, field: 1 }` or `obj.field = 1; obj = obj;`.
- **Async in `$:`:** after `await`, subsequent reactive reads are not tracked. Move to a `derived` store or split into two `$:` blocks.
- **`$store` outside a component:** invalid in plain `.ts` files — use `get(store)` or `store.subscribe(...)`.
- **Ordering of `$:`:** topological, not textual. If two blocks mutate the same state, extract an explicit function.
- **Stores and SSR:** module-level stores are shared across requests on the server. For request-scoped state use the Context API (see `lifecycle.md`).
