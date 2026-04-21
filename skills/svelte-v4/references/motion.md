# Motion & Easing — Svelte v4

Two modules: `svelte/motion` (physics/tween stores) and `svelte/easing` (easing functions). Both integrate with `transition:`, `animate:`, and reactive values.

## `svelte/motion`

### `tweened(value?, options?)`

Time-based interpolation with an easing curve. Returns a store whose `set` and `update` return a `Promise` that resolves when the animation completes.

```ts
import { tweened } from 'svelte/motion';
import { cubicOut } from 'svelte/easing';

const progress = tweened(0, {
  delay: 0,
  duration: 400,
  easing: cubicOut,
});

await progress.set(1);              // resolves when animation finishes
await progress.update((p) => p + 0.1, { duration: 200 });  // per-call overrides
```

Options:

| Option | Type | Default | Notes |
|---|---|---|---|
| `delay` | `number` | `0` | ms before animation starts |
| `duration` | `number \| (from, to) => number` | `400` | ms, or function for dynamic timing |
| `easing` | `(t: number) => number` | `t => t` (linear) | any function from `svelte/easing` |
| `interpolate` | `(a, b) => (t) => value` | built-in | custom interpolator (dates, colors, etc.) |

Built-in interpolation handles numbers, arrays of numbers, and plain objects of numbers. For `Date`, colors, strings — supply `interpolate`.

### `spring(value?, options?)`

Physics-based motion — no fixed duration. Adjusts velocity toward target using stiffness/damping.

```ts
import { spring } from 'svelte/motion';

const coords = spring({ x: 0, y: 0 }, {
  stiffness: 0.1,
  damping: 0.25,
  precision: 0.001,
});

coords.set({ x: 100, y: 50 });
coords.update((c) => ({ x: c.x + 10, y: c.y }), { hard: true });    // jump, no animation
coords.set({ x: 0, y: 0 }, { soft: 0.5 });                           // preserve momentum 0.5s
```

Options:

| Option | Range | Default | Meaning |
|---|---|---|---|
| `stiffness` | `0..1` | `0.15` | higher = snappier |
| `damping` | `0..1` | `0.8` | lower = bouncier |
| `precision` | small positive | `0.01` | settle threshold |

`set`/`update` accept a second options arg:

- `{ hard: true }` — jump immediately.
- `{ soft: number | true }` — preserve velocity for `n` seconds (or indefinitely with `true`).

Both returns are `Promise<void>` resolving when the spring settles.

### Integration with transitions

`tweened` and `spring` are plain stores — subscribe with `$`. Use them to animate arbitrary numeric state (progress bars, drag coordinates, chart values) where `transition:` is not a fit.

```svelte
<script>
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';

  const progress = tweened(0, { duration: 300, easing: cubicOut });
  export let value;
  $: progress.set(value);
</script>

<progress value={$progress} max="1" />
```

## `svelte/easing`

31 named functions. Use with `transition:*={{ easing }}`, `tweened({ easing })`, or custom animations.

Families (each with `In`, `Out`, `InOut` variants): **back, bounce, circ, cubic, elastic, expo, quad, quart, quint, sine**. Plus `linear` (no variants).

Examples:

```ts
import {
  linear,
  cubicIn, cubicOut, cubicInOut,
  elasticOut, bounceInOut, quintIn, sineInOut,
} from 'svelte/easing';
```

Naming cheat:

- `*In` — starts slow, accelerates.
- `*Out` — starts fast, decelerates (most natural for enter transitions).
- `*InOut` — accelerate then decelerate (most natural for continuous motion).

Custom easing: any function `(t: number) => number` with `t` in `[0, 1]` and output typically in `[0, 1]` works (overshoot, e.g. elastic/back, is allowed).

## Pitfalls

- **Heavy interpolators in `tweened`** — a complex custom `interpolate` runs every frame. Keep it allocation-free.
- **Spring with large payloads** — each top-level numeric field animates independently. Don't spring a 1000-item array; spring only scalar values and compute the rest reactively.
- **Store identity** — re-creating the store on every render (e.g. inside `$:`) resets the animation. Declare it once at init.
