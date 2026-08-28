# DaisyUI theming reference

Verified against `saadeghi/daisyui@5.7.20` (`skills/daisyui/{install,config,colors}/SKILL.md`)
and `daisyui@4.12.24` (`src/index.d.ts`).

## Contents
- Version gate
- Install (DaisyUI 5 / Tailwind v4)
- Plugin config vs theme config — two different blocks
- Custom theme: the complete variable set
- Overriding a built-in theme
- Semantic color rules
- Dark mode
- DaisyUI 4 (Tailwind v3) form

## Version gate

| Tailwind | DaisyUI | Config location |
|---|---|---|
| v4 | 5 | CSS — `@plugin "daisyui" { ... }` |
| v3 | 4 | JS — `tailwind.config.js`, `daisyui: { ... }` |

DaisyUI 5 requires Tailwind CSS 4. `tailwind.config.js` is deprecated in Tailwind v4 —
with v4 as a Node dependency, the CSS file needs only `@import "tailwindcss";`.

## Install (DaisyUI 5)

```bash
npm i -D daisyui@latest
```

```css
@import "tailwindcss";
@plugin "daisyui";
```

That is the whole installation. No config file, no `content` array.

## Plugin config vs theme config

These are two separate at-rules and confusing them is the most common mistake:

- `@plugin "daisyui" { ... }` — **which themes are enabled** and plugin behavior.
- `@plugin "daisyui/theme" { ... }` — **the definition of one theme**.

Plugin options, with their defaults:

```css
@plugin "daisyui" {
  themes: light --default, dark --prefersdark;
  root: ":root";
  include: ;
  exclude: ;
  prefix: ;
  logs: true;
}
```

- `--default` marks the theme applied with no `data-theme`; `--prefersdark` marks the one
  used under `prefers-color-scheme: dark`.
- `themes: all;` enables every built-in theme. `themes: false;` disables all built-ins —
  do this before defining only custom themes, otherwise a large unused theme block ships.
- `exclude:` takes feature names, e.g. `rootscrollgutter, checkbox`.
- `prefix: daisy-;` namespaces DaisyUI class names when another CSS library collides.
- Apply a non-default enabled theme with `data-theme="THEME_NAME"` on `<html>`; themes
  nest at any depth.

## Custom theme — the complete variable set

Every variable below must be present; DaisyUI does not fill gaps for a new theme name.
Colors may be OKLCH, hex, or any CSS color format.

```css
@import "tailwindcss";
@plugin "daisyui";
@plugin "daisyui/theme" {
  name: "mytheme";
  default: true;
  prefersdark: false;
  color-scheme: light;

  --color-base-100: oklch(98% 0.02 240);
  --color-base-200: oklch(95% 0.03 240);
  --color-base-300: oklch(92% 0.04 240);
  --color-base-content: oklch(20% 0.05 240);
  --color-primary: oklch(55% 0.3 240);
  --color-primary-content: oklch(98% 0.01 240);
  --color-secondary: oklch(70% 0.25 200);
  --color-secondary-content: oklch(98% 0.01 200);
  --color-accent: oklch(65% 0.25 160);
  --color-accent-content: oklch(98% 0.01 160);
  --color-neutral: oklch(50% 0.05 240);
  --color-neutral-content: oklch(98% 0.01 240);
  --color-info: oklch(70% 0.2 220);
  --color-info-content: oklch(98% 0.01 220);
  --color-success: oklch(65% 0.25 140);
  --color-success-content: oklch(98% 0.01 140);
  --color-warning: oklch(80% 0.25 80);
  --color-warning-content: oklch(20% 0.05 80);
  --color-error: oklch(65% 0.3 30);
  --color-error-content: oklch(98% 0.01 30);

  --radius-selector: 1rem;
  --radius-field: 0.25rem;
  --radius-box: 0.5rem;

  --size-selector: 0.25rem;
  --size-field: 0.25rem;

  --border: 1px;

  --depth: 1;
  --noise: 0;
}
```

Value guidance from the DaisyUI source:

- `--radius-*` preferred values: `0rem`, `0.25rem`, `0.5rem`, `1rem`, `2rem`. The three
  roles are deliberately separate — selectors (checkbox, toggle, badge), fields (button,
  input, select, tab), boxes (card, modal, alert). Giving all three the same value is what
  makes a UI read as untouched.
- `--size-selector` / `--size-field` should stay at `0.25rem` unless density is being
  changed on purpose; bigger `0.28125` / `0.3125`, smaller `0.21875` / `0.1875`.
- `--border` stays `1px` unless deliberate; thicker `1.5px` / `2px`, thinner `0.5px`.
- `--depth` and `--noise` are `0` or `1` only. `--depth: 1` adds a subtle 3D shadow effect;
  `--noise: 1` adds grain. Both are strong stylistic statements — choose, do not leave at
  default by accident.

Ship generated themes without the explanatory comments.

The visual builder at https://daisyui.com/theme-generator/ produces this exact block.

## Overriding a built-in theme

Name an existing theme and set only what changes; the rest is inherited:

```css
@plugin "daisyui/theme" {
  name: "light";
  default: true;
  --color-primary: blue;
  --color-secondary: teal;
}
```

## Semantic color rules

The color names are variables, so they follow the active theme. That is the whole point,
and it breaks the moment fixed Tailwind colors are mixed in.

Names: `primary`, `secondary`, `accent`, `neutral`, `base-100`, `base-200`, `base-300`,
`info`, `success`, `warning`, `error` — each with a matching `*-content` foreground color.
`base-100` is the page surface; `base-200` and `base-300` are progressively elevated.

- Use them like any Tailwind color: `bg-primary`, `text-base-content`.
- Use `base-*` for most of the page. Use the default variant for most elements. Use
  `primary` for the single most important element on a page — once.
- `*-content` colors must contrast clearly against their partner color.
- Avoid fixed Tailwind colors for text. `text-gray-800` on `bg-base-100` becomes unreadable
  the moment a dark theme is active.
- Fixed colors are legitimate when something must not change across themes — an SVG brand
  mark, a chart series.

## Dark mode

Do **not** use Tailwind's `dark:` variant with DaisyUI color names — the theme system
already handles the swap, and the two fight. Dark mode is a second theme with
`prefersdark: true`.

If `dark:` must follow a specific theme (for third-party components that expect it):

```css
@custom-variant dark (&:where([data-theme=night], [data-theme=night] *));
```

For a CDN setup with no build step, define the same variables in a selector matching the
theme name and the theme-controller input:

```css
:root:has(input.theme-controller[value=mytheme]:checked),
[data-theme="mytheme"] {
  color-scheme: light;
  --color-primary: oklch(55% 0.3 240);
  /* remaining variables */
}
```

## DaisyUI 4 (Tailwind v3)

```bash
npm i -D daisyui@4
```

```js
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{js,ts,jsx,tsx}'],
  plugins: [require('daisyui')],
  daisyui: {
    themes: ['light', 'dark', { mytheme: { primary: '#00b96b', 'base-100': '#ffffff' } }],
    darkTheme: 'dark',
    base: true,
    styled: true,
    utils: true,
    rtl: false,
    prefix: '',
    logs: true,
    themeRoot: ':root',
  },
};
```

Option semantics from `src/index.d.ts`:

- `themes` — `true` enables all, `false` enables only light and dark, an array enables the
  listed ones with the **first as default**. A custom theme is a plain object of
  color-name → color-value.
- `darkTheme` — which enabled theme answers the system dark preference (default `'dark'`).
- `base` — inject DaisyUI's base styles. `styled` — components come pre-styled;
  `false` gives unstyled skeletons. `utils` — include responsive/utility classes.
- `prefix` — prefixes component and modifier classes only, not color utilities.
- `themeRoot` — element that receives the theme CSS variables.

DaisyUI 4 has no `--radius-*` / `--size-*` / `--depth` / `--noise` variables — that
structured token surface arrived in 5. On Tailwind v3 those decisions are made in the
Tailwind theme config instead.
