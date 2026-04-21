# claude-svelte-v4

A [Claude Code](https://claude.com/claude-code) plugin that installs a **Svelte v4** expert skill into your assistant.

Svelte 5 introduced runes (`$state`, `$derived`, `$effect`, `$props`) and snippets (`{#snippet}`, `{@render}`). These are **not valid** in Svelte v4 — but Claude tends to emit them by default on any `.svelte` file. This plugin fixes that by teaching Claude the v4 idioms (`export let`, `$:`, `<slot>`, `createEventDispatcher`, stores) and explicitly forbidding v5-only syntax.

## What's inside

A single skill — `svelte-v4` — organized as an entry point plus focused reference files that Claude loads on demand:

| File | Covers |
|---|---|
| `skills/svelte-v4/SKILL.md` | Entry point: triggers, version detection, hard DON'Ts, cheat-sheet |
| `references/component.md` | Component anatomy, props, events, slots, bindings, TypeScript, client API |
| `references/reactivity.md` | `$:` labels, stores (`writable`/`readable`/`derived`), pitfalls |
| `references/lifecycle.md` | `onMount`, `onDestroy`, `beforeUpdate`, `afterUpdate`, `tick`, context |
| `references/template.md` | Control flow, directives, special elements, actions, transitions |
| `references/motion.md` | `svelte/motion` (`tweened`, `spring`) and `svelte/easing` |
| `references/accessibility.md` | All `a11y-*` compile warnings + fix patterns |

## Installation

In Claude Code:

```
/plugin marketplace add aimuzov/claude-svelte-v4
/plugin install svelte-v4@aimuzov-claude-svelte-v4
```

Or from a local clone:

```
/plugin marketplace add /path/to/claude-svelte-v4
/plugin install svelte-v4@aimuzov-claude-svelte-v4
```

That's it — the skill activates automatically the next time you open a `.svelte` file in a project where `package.json` lists `svelte@^4`.

## How it activates

Claude reads each skill's description and decides when to use it. `svelte-v4` triggers on:

- File extension `.svelte` **and** `package.json` contains `"svelte": "^4"` / `"~4.x"` / `"4.x.y"`.
- Explicit user mentions: "Svelte 4", "Svelte v4", "legacy Svelte".
- Existing code using v4 idioms (`export let`, `$:`, `<slot>`).

And it explicitly **skips** when it detects Svelte 5 markers (`$state`, `$derived`, `{#snippet}`, …) so it won't interfere with v5 projects.

## Uninstall

```
/plugin uninstall svelte-v4@aimuzov-claude-svelte-v4
/plugin marketplace remove aimuzov-claude-svelte-v4
```

## Contributing

PRs are welcome — especially corrections spotted against the [official Svelte v4 docs](https://v4.svelte.dev/docs/). Keep the skill focused on **core runtime and template syntax only**; SvelteKit is deliberately out of scope.

## License

[MIT](./LICENSE) — Aleksey Imuzov
