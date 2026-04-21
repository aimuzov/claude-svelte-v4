---
name: svelte-v4
description: Use when writing or modifying Svelte v4 components (.svelte files targeting svelte@^4). Triggers - "Svelte 4", "Svelte v4", "legacy Svelte", projects with `"svelte": "^4"` in package.json. CRITICAL - do NOT use Svelte 5 runes ($state, $derived, $effect, $props, $bindable, $inspect) or snippets ({#snippet}, {@render}) here. For Svelte 5 projects skip this skill.
---

# Svelte v4

Expert guidance for authoring Svelte v4 components. Covers core runtime and template syntax only — **SvelteKit is out of scope** for this skill.

## When to use

Activate on any of:

- File extension is `.svelte` **and** the project's `package.json` lists `svelte@^4` / `~4.x` / exact `4.x.y`.
- User explicitly says "Svelte 4", "Svelte v4", "legacy Svelte".
- Existing code uses `export let`, `$:`, `createEventDispatcher`, `<slot>` — all v4 idioms.

## When to skip

- `svelte@^5` in `package.json` — use runes/snippets idioms instead.
- File already uses `$state`, `$derived`, `$effect`, `{#snippet}`, `{@render}` — it is Svelte 5.
- User asks about SvelteKit routing, `load`, `form actions`, `hooks.server.ts` — that's a different domain.

## Version detection workflow

Before editing any `.svelte` file:

1. `Read package.json` at the project root.
2. Check `devDependencies.svelte` or `dependencies.svelte`.
3. If version is `^4.x`, `~4.x`, or an exact `4.x.y` → this skill applies.
4. If version is `^5.x` / `next` / `beta` → abort and use Svelte 5 idioms.
5. If no `svelte` entry or ambiguous (git dep, workspace `*`) → ask the user which major version.

## Hard DON'Ts in v4

Never generate or suggest:

- **Runes:** `$state`, `$state.raw`, `$state.snapshot`, `$derived`, `$derived.by`, `$effect`, `$effect.pre`, `$effect.root`, `$effect.tracking`, `$props`, `$bindable`, `$inspect`, `$host`.
- **Snippets:** `{#snippet name(params)}…{/snippet}`, `{@render snippet(args)}`, `children` prop as a function.
- **New utilities:** `untrack()`, `mount()`, `hydrate()`, `unmount()`, `getAllContexts()` usage outside init, `svelte/reactivity` classes (`SvelteMap`, `SvelteSet`, `SvelteDate`, `SvelteURL`).
- **New element API:** `$host()` for custom-element exports.

If the user asks for any of those, stop and confirm — they may be in the wrong skill.

## Core v4 primitives (cheat-sheet)

```svelte
<script lang="ts">
  import { onMount, createEventDispatcher, tick } from 'svelte';
  import { writable, derived } from 'svelte/store';

  export let count = 0;                // prop with default
  export let label: string;            // required prop
  export const version = '1.0';        // read-only export

  const dispatch = createEventDispatcher<{ change: number }>();

  const store = writable(0);
  const doubled = derived(store, ($s) => $s * 2);

  $: parity = count % 2 === 0 ? 'even' : 'odd';   // reactive declaration
  $: if (count > 10) dispatch('change', count);   // reactive side-effect

  onMount(() => {
    const id = setInterval(() => store.update((n) => n + 1), 1000);
    return () => clearInterval(id);
  });

  function increment() {
    count += 1;
  }
</script>

<button on:click={increment} class:big={count > 5}>
  {label}: {count} ({parity}) — store={$store}, doubled={$doubled}
</button>

<slot name="footer" {count} />

<style>
  button { padding: 0.5rem 1rem; }
  .big { font-size: 1.25rem; }
</style>
```

## Deep references

Load the relevant reference file when you need detail — don't guess. Paths are relative to this skill directory.

- **`references/component.md`** — component anatomy, props, events, slots, bindings, TypeScript, client-side API (`new Component`, `$set`, `$on`, `$destroy`).
- **`references/reactivity.md`** — `$:` labels, stores (`writable` / `readable` / `derived`), auto-subscription, common pitfalls.
- **`references/lifecycle.md`** — `onMount`, `onDestroy`, `beforeUpdate`, `afterUpdate`, `tick`, Context API, SSR gotchas.
- **`references/template.md`** — `{#if}` / `{#each}` / `{#await}` / `{#key}`, directives (`class:`, `style:`, `use:`, `transition:`, `animate:`), special elements (`<svelte:…>`), actions, built-in transitions/animations.
- **`references/motion.md`** — `svelte/motion` (`tweened`, `spring`) and `svelte/easing` (31 named functions).
- **`references/accessibility.md`** — full list of `a11y-*` compile warnings with descriptions, common fix patterns.

## External docs

Canonical reference: https://v4.svelte.dev/docs/. Svelte does not ship an `llms.txt` for v4 (only v5) — use `WebFetch` against specific v4 doc pages when you need detail beyond this skill:

- `https://v4.svelte.dev/docs/component-format`
- `https://v4.svelte.dev/docs/template-syntax`
- `https://v4.svelte.dev/docs/special-elements`
- `https://v4.svelte.dev/docs/svelte` (module: `onMount`, `tick`, `createEventDispatcher`, etc.)
- `https://v4.svelte.dev/docs/svelte-store`, `/svelte-motion`, `/svelte-transition`, `/svelte-animate`, `/svelte-easing`, `/svelte-action`
- `https://v4.svelte.dev/docs/client-side-component-api`
- `https://v4.svelte.dev/docs/typescript`
- `https://v4.svelte.dev/docs/accessibility-warnings`

## Migration guardrail

- If the user asks to migrate v4 → v5 (or vice versa), **do not rewrite silently**. Outline the diff (`export let` → `$props`, `$:` → `$derived`/`$effect`, slots → snippets, `createEventDispatcher` → callback props) and confirm scope first.
- Never mix v4 and v5 idioms in the same file — the compiler accepts one mode per component.
