# Template syntax — Svelte v4

Covers control flow, template tags, directives, special elements, actions, transitions, animations.

## Control flow blocks

### `{#if}`

```svelte
{#if user}
  <p>Hi {user.name}</p>
{:else if guest}
  <p>Hello, guest</p>
{:else}
  <p>Please log in</p>
{/if}
```

### `{#each}`

```svelte
{#each items as item, index (item.id)}
  <Row {item} {index} />
{:else}
  <p>No items</p>
{/each}

<!-- Destructuring + no key (only when order is stable) -->
{#each rows as { id, label } (id)}
  <li>{label}</li>
{/each}
```

- The keyed form `(expression)` is **strongly preferred** — without a key, Svelte reuses DOM by index, which breaks animations and component identity.
- `{:else}` runs when the iterable is empty.
- The iterable can be any array-like (`Array`, `NodeList`, `Map` via `[...map]`, etc.). Plain objects: iterate over `Object.entries(obj)`.

### `{#await}`

```svelte
{#await promise}
  <Spinner />
{:then value}
  <Data {value} />
{:catch error}
  <Error {error} />
{/await}

<!-- Shorthand: skip pending state -->
{#await promise then value}
  <Data {value} />
{/await}

<!-- Shorthand: skip catch -->
{#await promise}
  <Spinner />
{:then value}
  <Data {value} />
{/await}
```

Reassigning the promise starts a new resolution cycle.

### `{#key}`

Destroys and recreates the block whenever the expression changes. Use to force-remount a child or restart transitions:

```svelte
{#key selectedTab}
  <Panel />
{/key}
```

## Template tags

```svelte
{@html '<em>trusted markup</em>'}          <!-- raw HTML, no sanitization -->
{@debug user, cart}                         <!-- pauses in devtools when any listed var changes -->
{#each rows as row (row.id)}
  {@const total = row.price * row.qty}      <!-- local binding inside the block -->
  <Row {...row} {total} />
{/each}
```

- `{@html}` is unsafe with untrusted input — sanitize first.
- `{@debug}` with no args pauses on every update.
- `{@const}` is valid only inside `{#each}`, `{#if}`, `{#await}`, `{:then}`, `{:catch}`, or the top level of `<Component>` child content.

## Directives

### `class:`

```svelte
<div class:active={isActive} class:big={size > 10}>...</div>
<div class:selected>{/* shorthand: uses the identically-named local var */}</div>
```

### `style:`

```svelte
<div style:color={accent} style:--spacing="{n}px" style:padding|important="8px">
```

Takes any CSS property; with `|important` emits `!important`. Mixes safely with a static `style="..."` attribute.

### `use:` (actions)

```svelte
<script>
  import type { Action } from 'svelte/action';

  const clickoutside: Action<HTMLElement, () => void> = (node, cb) => {
    const onDoc = (e: MouseEvent) => {
      if (!node.contains(e.target as Node)) cb();
    };
    document.addEventListener('click', onDoc, true);
    return {
      update(newCb) { cb = newCb; },
      destroy() { document.removeEventListener('click', onDoc, true); },
    };
  };
</script>

<div use:clickoutside={() => (open = false)}>…</div>
```

- Action signature: `(node, params?) => void | { update?(newParams), destroy?() }`.
- `update` runs when `params` change; `destroy` when the element unmounts.
- Import `Action<Element, Params>` and `ActionReturn` from `svelte/action` for typing.

### `transition:` / `in:` / `out:`

```svelte
<script>
  import { fade, fly } from 'svelte/transition';
  import { quintOut } from 'svelte/easing';
</script>

{#if open}
  <div transition:fade={{ duration: 200 }}>...</div>
{/if}

{#each items as item (item.id)}
  <li in:fly={{ y: -10 }} out:fade={{ duration: 150 }}>{item.label}</li>
{/each}
```

- `transition:` binds both enter and leave; `in:` / `out:` split them.
- Modifiers: `|local` (only animate when the block itself enters/leaves, not ancestors) and `|global` (animate on every mount/unmount, default for `in:`/`out:`).
- Custom transition: function returning `{ delay?, duration?, easing?, css?(t, u), tick?(t, u) }`.
- Built-ins in `svelte/transition`: `fade`, `blur`, `fly`, `slide`, `scale`, `draw` (SVG), `crossfade` (factory returning `[send, receive]`).

### `animate:`

```svelte
<script>
  import { flip } from 'svelte/animate';
</script>

{#each list as item (item.id)}
  <li animate:flip={{ duration: 200 }}>{item.label}</li>
{/each}
```

- `animate:` only works on immediate children of a **keyed** `{#each}`.
- Built-in `flip` (first-last-invert-play) from `svelte/animate`. Custom animations implement the same signature as transitions plus access to `from`/`to` DOMRect.

## Special elements

| Element | Purpose | Placement constraint |
|---|---|---|
| `<svelte:self>` | Recursive component reference. | **Must be inside `{#if}`/`{#each}` or similar** — never at component top level (would infinite-loop). |
| `<svelte:component this={Comp}>` | Render a dynamic component. If `this` is `null`/`undefined`, nothing renders. | Anywhere. |
| `<svelte:element this={tag}>` | Render a dynamic HTML tag. `this` must resolve to a valid HTML tag name. Only `bind:this` is supported — no `bind:value` etc. Dev-mode runtime error if a void tag (`<br>`, `<img>`) gets children. | Anywhere. |
| `<svelte:window>` | Event listeners and bindings on `window`. Writable bindings: `scrollX`, `scrollY`. Readonly bindings: `innerWidth`, `innerHeight`, `outerWidth`, `outerHeight`, `online`, `devicePixelRatio`. | **Top level of component only** — not inside any block/element. |
| `<svelte:document>` | Event listeners + readonly bindings (`fullscreenElement`, `visibilityState`). | Top level only. |
| `<svelte:body>` | Event listeners on `document.body` (`on:mouseenter`, `on:mouseleave`, etc.). | Top level only. |
| `<svelte:head>` | Inject content into `<head>` (title, meta, link). SSR emits it separately from the body HTML. | Top level only. |
| `<svelte:options>` | Per-component compiler options (`immutable`, `accessors`, `namespace`, `customElement`). | Top level only. |
| `<svelte:fragment>` | Group multiple nodes into a named slot without a DOM wrapper. | Inside a component that provides a `slot`. |

Examples:

```svelte
<svelte:window on:keydown={handleKey} bind:innerWidth={w} />

<svelte:head>
  <title>{pageTitle}</title>
  <meta name="description" content={desc} />
</svelte:head>

<svelte:component this={variant === 'a' ? CompA : CompB} {...props} />

<svelte:element this={level} class="heading">{text}</svelte:element>
```

## Putting it together

```svelte
<script lang="ts">
  import { fly } from 'svelte/transition';
  import { flip } from 'svelte/animate';

  let todos: { id: number; text: string; done: boolean }[] = [];
  let next = '';

  function add() {
    if (!next.trim()) return;
    todos = [...todos, { id: Date.now(), text: next, done: false }];
    next = '';
  }
</script>

<form on:submit|preventDefault={add}>
  <input bind:value={next} placeholder="Add todo" />
  <button>Add</button>
</form>

{#if todos.length === 0}
  <p>No todos yet.</p>
{:else}
  <ul>
    {#each todos as todo (todo.id)}
      <li animate:flip={{ duration: 150 }} in:fly={{ y: -5 }}>
        <input type="checkbox" bind:checked={todo.done} />
        <span class:done={todo.done}>{todo.text}</span>
      </li>
    {/each}
  </ul>
{/if}

<style>
  .done { text-decoration: line-through; opacity: 0.6; }
</style>
```
