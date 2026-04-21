# Component anatomy — Svelte v4

## File structure

A `.svelte` file contains up to four top-level blocks, in any order:

```svelte
<script context="module">
  // Module-level code. Runs once per file, not per instance.
  // Can export named exports consumed by other modules.
  export function helper() {}
</script>

<script lang="ts">
  // Instance code. Runs per component instance.
</script>

<!-- Markup -->
<div>...</div>

<style>
  /* Scoped to the component by default. */
</style>
```

Notes:

- `<script context="module">` cannot use `$:` and cannot `export let` props — those live in the instance script.
- `<script lang="ts">` enables TypeScript (requires `svelte-preprocess` or `svelte-check` setup).
- `<style>` is scoped automatically; use `:global(.name)` for global selectors, or a top-level `<style global>` in older setups.

## `<svelte:options>`

```svelte
<svelte:options
  immutable={true}
  accessors={true}
  namespace="svg"
  customElement="my-counter"
/>
```

- `immutable={true}` — promise the compiler that props change by reference, enables `==` optimizations.
- `accessors={true}` — generate getters/setters for props on the component instance (mainly for vanilla JS consumers).
- `namespace="svg"` — required when the component renders SVG at the root.
- `customElement="tag-name"` — compile the component as a custom element.

## Props

```svelte
<script>
  export let value;                  // required (no default)
  export let count = 0;              // optional, default 0
  export let id: string = crypto.randomUUID(); // typed + default
  export const version = '1.0';      // read-only export (consumers cannot bind)

  let { a, b, ...rest } = $$props;   // read-only snapshot of all props
  $: console.log($$restProps);       // props NOT declared with `export let`
</script>
```

- `export let` is the declaration syntax. Dashes in prop names are rewritten to camelCase (`<Foo aria-label="x" />` → `ariaLabel`).
- `export const` / `export function` — read-only, invoked on the component instance from outside.
- `$$props` — reactive object with all props; avoid unless you need dynamic forwarding.
- `$$restProps` — props not captured by explicit `export let`; use for forwarding to a wrapped element: `<input {...$$restProps} />`.

## Events

```svelte
<script lang="ts">
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher<{
    select: { id: number };
    close: void;
  }>();

  function onPick(id: number) {
    dispatch('select', { id });
  }
</script>

<button on:click={() => onPick(1)}>Pick</button>
<button on:click={() => dispatch('close')}>Close</button>

<!-- DOM modifiers -->
<form on:submit|preventDefault={save}>
  <input on:keydown|once|self={handle} />
</form>

<!-- Event forwarding: re-emit without handler -->
<button on:click>Forwarded</button>
```

Available modifiers: `preventDefault`, `stopPropagation`, `stopImmediatePropagation`, `passive`, `nonpassive`, `capture`, `once`, `self`, `trusted`. Combine with `|`: `on:click|once|capture={fn}`.

Consumer listens with `on:select={(e) => …}`, reading `e.detail` for payload.

## Slots

```svelte
<!-- Inside Card.svelte -->
<article>
  <header><slot name="header">Default header</slot></header>
  <slot {count} /> <!-- default slot, exposes `count` as slot prop -->
  <footer>
    {#if $$slots.actions}
      <slot name="actions" />
    {/if}
  </footer>
</article>

<!-- Usage -->
<Card let:count>
  <h1 slot="header">Title</h1>
  <p>Count is {count}</p>
  <svelte:fragment slot="actions">
    <button>OK</button>
    <button>Cancel</button>
  </svelte:fragment>
</Card>
```

- `$$slots` — `{ [name]: true }` map of provided slots; useful to conditionally render wrappers.
- `<svelte:fragment slot="...">` — groups elements into a named slot without introducing a wrapper DOM node.
- Slot props (`let:count`) flow **parent → child → parent scope** inside the slot content.

## Bindings

```svelte
<!-- Two-way for form controls -->
<input bind:value={name} />
<input type="checkbox" bind:checked={agreed} />
<input type="number" bind:value={age} />   <!-- auto-coerced to number -->
<select bind:value={choice}>…</select>
<textarea bind:value={note} />

<!-- Radio / checkbox groups -->
<input type="radio" bind:group={color} value="red" />
<input type="checkbox" bind:group={flavors} value="vanilla" />

<!-- Element / component references -->
<canvas bind:this={canvasEl}></canvas>
<Child bind:exposedProp={parentState} />

<!-- Media / element metrics (read-only unless noted) -->
<video
  bind:currentTime
  bind:duration
  bind:paused
  bind:played
  bind:buffered
  bind:seekable
  bind:seeking
  bind:ended
  bind:readyState
  bind:playbackRate
  bind:volume
  bind:muted
/>
<div bind:clientWidth bind:clientHeight bind:offsetWidth bind:offsetHeight></div>
```

- Component bindings require the child to declare the prop with `export let` (not `export const`).
- `bind:this` fires after mount; access the ref inside `onMount` or a reactive block guarded by `if (el)`.

## TypeScript

### Setup

`lang="ts"` requires a preprocessor — use `vitePreprocess` (SvelteKit default) or `svelte-preprocess`. Run `svelte-check` in CI for project-wide type validation; template expressions (markup) cannot be typed, only `<script>` code.

```svelte
<script lang="ts">
  import { createEventDispatcher, type ComponentType, type ComponentProps, type ComponentEvents } from 'svelte';

  export let items: ReadonlyArray<{ id: number; label: string }>;
  export let variant: 'primary' | 'ghost' = 'primary';

  const dispatch = createEventDispatcher<{
    pick: { id: number };
    close: void;                 // payload-less events: use `void` or `null`
  }>();

  // Cannot inline-type a reactive declaration. Declare separately:
  let doubled: number;
  $: doubled = items.length * 2;
</script>
```

### Utility types from `svelte`

| Type | Purpose |
|---|---|
| `ComponentType<T extends SvelteComponent = SvelteComponent>` | The class type of a component (for `<svelte:component this={...}>`). |
| `ComponentProps<C>` | Extracts the props shape of a component class. |
| `ComponentEvents<C>` | Extracts the event map of a component class. |
| `SvelteComponent` / `SvelteComponentTyped<Props, Events, Slots>` | Base class for augmenting library component types. |

```ts
import type Button from './Button.svelte';
import type { ComponentProps, ComponentEvents } from 'svelte';

type ButtonProps = ComponentProps<Button>;
type ButtonEvents = ComponentEvents<Button>;

function usage(props: ButtonProps, handler: (e: ButtonEvents['click']) => void) {}
```

### Typing library components (`.d.ts`)

```ts
// MyLib.d.ts
import { SvelteComponentTyped } from 'svelte';

export class MyWidget extends SvelteComponentTyped<
  { label: string; count?: number },
  { pick: CustomEvent<{ id: number }> },
  { default: { active: boolean } }
> {}
```

### Augmenting DOM types

Since v4.2, augment `svelte/elements` for custom attributes/events on standard tags:

```ts
// src/app.d.ts
import type { HTMLAttributes } from 'svelte/elements';

declare module 'svelte/elements' {
  export interface HTMLAttributes<T> {
    'on:custom-event'?: (e: CustomEvent<{ payload: string }>) => void;
    'data-analytics'?: string;
  }
}
```

For older code using the `svelteHTML` namespace, the same idea applies but targeting `svelteHTML.IntrinsicElements`.

### Generics limitation

True generic components are **not** supported in v4. Workarounds:

- Use `unknown` + a type guard inside the component.
- Expose a factory function in a `.ts` file that returns a narrowed component type.
- If generics are essential for the API, consider upgrading to Svelte 5.

## Client-side API (imperative mounting)

Every `.svelte` file compiled with `generate: 'dom'` (the default) exports a class. Mount it imperatively for progressive enhancement, widget embeds, or tests:

```ts
import App from './App.svelte';

const app = new App({
  target: document.body,
  anchor: document.getElementById('placeholder') ?? undefined,
  props: { answer: 42 },
  context: new Map([['theme', 'dark']]),
  hydrate: false,        // true when upgrading SSR markup
  intro: true,           // run transitions on initial mount
});

const off = app.$on('pick', (e) => console.log(e.detail));
app.$set({ answer: 100 });   // async: applies on next microtask
off();                       // remove listener
app.$destroy();              // unmount + run onDestroy hooks
```

Notes:

- `$set` is asynchronous by default. Compile with `<svelte:options accessors />` (or the global `accessors: true`) to gain synchronous prop getters/setters via `app.answer = 100`.
- `target` must be an `HTMLElement` or `ShadowRoot`.
- `context` is a root-level `Map<any, any>` consumed by `getContext(...)` at init.
- For custom elements, don't use this API — use `<svelte:options customElement="my-tag" />` and register via the DOM.

## Styling

```svelte
<style>
  .btn { color: var(--accent, blue); }

  /* Escape scoping */
  :global(body.dark .btn) { color: white; }

  /* Keyframes must be :global to be reusable */
  @keyframes -global-pulse { … }
</style>
```

- Unused selectors warn at compile time — silence with `/* svelte-ignore css-unused-selector */` above the rule.
- Dynamic CSS via `style:` directive or custom properties (`--color: {accent}` in markup → `var(--color)` in `<style>`).
