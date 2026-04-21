# Accessibility — Svelte v4

Svelte v4 emits **compile-time a11y warnings** whenever markup violates common WAI-ARIA or HTML semantics rules. Fix the root cause — don't silence with `<!-- svelte-ignore ... -->` unless you have a documented reason.

## Ignoring a warning

Add a comment immediately above the offending element. You may list multiple codes separated by whitespace.

```svelte
<!-- svelte-ignore a11y-autofocus a11y-missing-attribute -->
<input autofocus />
```

## Full list of warnings

| Code | What it catches |
|---|---|
| `a11y-accesskey` | `accesskey` attribute — conflicts with assistive tech shortcuts. |
| `a11y-aria-activedescendant-has-tabindex` | Element using `aria-activedescendant` must itself be tabbable. |
| `a11y-aria-attributes` | Element's role does not allow the ARIA attribute (e.g. `aria-hidden` on `<meta>`). |
| `a11y-autofocus` | `autofocus` disorients keyboard/screen-reader users. |
| `a11y-click-events-have-key-events` | Interactive element with `on:click` also needs `on:keydown`/`on:keyup`/`on:keypress`. |
| `a11y-distracting-elements` | `<marquee>`, `<blink>` — obsolete and hostile to users. |
| `a11y-hidden` | `aria-hidden="true"` on a heading hides landmark from AT. |
| `a11y-img-redundant-alt` | `alt="photo of..."` is redundant — screen readers already announce "image". |
| `a11y-incorrect-aria-attribute-type` | `aria-*` value does not match its expected type (e.g. boolean, tristate, integer). |
| `a11y-invalid-attribute` | Attribute value is invalid (`href="javascript:void(0)"`, `href="#"`). |
| `a11y-interactive-supports-focus` | Interactive ARIA role requires a `tabindex`. |
| `a11y-label-has-associated-control` | `<label>` must wrap a control or reference one via `for`. |
| `a11y-media-has-caption` | `<video>` / `<audio>` needs `<track kind="captions">`. |
| `a11y-misplaced-role` | `role` attribute on an element whose role is reserved (`<html>`, `<meta>`). |
| `a11y-misplaced-scope` | `scope` attribute only allowed on `<th>`. |
| `a11y-missing-attribute` | Required a11y attribute missing (`<img>` without `alt`, `<a>` without `href`). |
| `a11y-missing-content` | `<h1>`…`<h6>`, `<a>` empty of text content. |
| `a11y-mouse-events-have-key-events` | `on:mouseover`/`on:mouseout` requires `on:focus`/`on:blur` equivalents. |
| `a11y-no-redundant-roles` | `role="button"` on `<button>` is redundant. |
| `a11y-no-interactive-element-to-noninteractive-role` | Do not demote `<button>` to `role="presentation"`. |
| `a11y-no-noninteractive-element-interactions` | `<div on:click>` without a role — use `<button>` instead. |
| `a11y-no-noninteractive-element-to-interactive-role` | Do not promote `<ul>` to `role="button"`. |
| `a11y-no-noninteractive-tabindex` | Only interactive elements should be tab-focusable. |
| `a11y-no-static-element-interactions` | Static `<div>` with `on:click` needs a role and keyboard handling. |
| `a11y-positive-tabindex` | `tabindex > 0` breaks natural tab order. Use `0` or `-1`. |
| `a11y-role-has-required-aria-props` | `role="checkbox"` requires `aria-checked`. |
| `a11y-role-supports-aria-props` | `role="list"` cannot have `aria-expanded`. |
| `a11y-structure` | Elements must follow required parent/child rules (`<li>` inside `<ul>`/`<ol>`, `<dt>`/`<dd>` inside `<dl>`). |
| `a11y-unknown-aria-attribute` | Typo'd ARIA attribute (`aria-lable`). |
| `a11y-unknown-role` | Unknown ARIA role value. |

## Common patterns

### Focusable div for custom widgets

```svelte
<div
  role="button"
  tabindex="0"
  on:click={toggle}
  on:keydown={(e) => (e.key === 'Enter' || e.key === ' ') && toggle()}
  aria-pressed={active}
>
  {label}
</div>
```

But prefer a real `<button>` with `type="button"` unless you have a strong reason.

### Labels

```svelte
<!-- Implicit association -->
<label>
  Name
  <input bind:value={name} />
</label>

<!-- Explicit -->
<label for="email">Email</label>
<input id="email" type="email" bind:value={email} />
```

### Images

```svelte
<img src={url} alt="" />                <!-- decorative, empty alt is fine -->
<img src={url} alt="Team photo" />       <!-- informative -->
<img src={url} role="presentation" alt="" />  <!-- redundant: empty alt already does this -->
```

### Keyboard + pointer parity

Every `on:click` on an interactive element should have a `on:keydown` counterpart (Enter/Space activation). Every `on:mouseenter`/`on:mouseleave` should pair with `on:focus`/`on:blur`.

### Dialog / modal

Use `<dialog>` where possible (native focus trap, ESC handling). Otherwise wrap in `role="dialog" aria-modal="true"` with explicit focus management in `onMount`.
