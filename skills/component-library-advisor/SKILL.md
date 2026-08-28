---
name: component-library-advisor
description: Choose, install, and theme the right React component library — Ant Design or DaisyUI — and build the token layer (color scale, type scale, spacing, radius, density) that keeps the result from looking generic. Use whenever a React, Next.js, Remix, or Vite project needs a UI library picked, swapped, or restyled; whenever antd, ConfigProvider, design tokens, DaisyUI, Tailwind theming, or a custom theme come up; whenever someone says the UI looks default, generic, bootstrap-y, template-y, or AI-generated; and whenever a new React app is scaffolded with no library chosen yet. Also covers Serbian phrasings - koju biblioteku komponenti, AntD ili DaisyUI, tema, tokeni, izgleda genericki, izgleda kao template. Do NOT use for Storybook setup, motion and animation work, or icon selection; separate skills cover those.
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "ant-design/ant-design@6.6.1, saadeghi/daisyui@5.7.20 and daisyui@4.12.24 (read from source, 2026-08-23); spot-checked against npm registry + official docs 2026-08-25, current versions antd@6.6.2 / daisyui@5.7.22 — peer deps, icons version, sizeUnit/sizeStep/controlHeight defaults all confirmed accurate"
---

# Component Library Advisor

Picking a component library is a design decision disguised as a dependency choice. The
library ships a visual language; whatever token layer is left at its defaults becomes the
product's identity by accident. The job here is to make that choice deliberately, then
replace the defaults with a small, coherent token set before any screen gets built.

Work in this order: **read the project → pick the library → set the token layer → build
screens.** Skipping to screens is what produces work that has to be redone.

## Step 1 — Read the project before recommending anything

Never recommend from the project's description alone. The Tailwind major version is a hard
gate on DaisyUI, and an existing library is usually a stronger constraint than any
preference.

```bash
grep -E '"(react|next|vite|antd|daisyui|tailwindcss|@tailwindcss/[a-z]+)"' package.json
ls tailwind.config.* postcss.config.* 2>/dev/null
grep -rl '@import "tailwindcss"\|@tailwind base' src app styles 2>/dev/null | head
```

How to read the result:

| Signal | Means |
|---|---|
| `@import "tailwindcss";` in CSS, `@tailwindcss/vite` or `@tailwindcss/postcss` dep, no config file needed | Tailwind v4 |
| `tailwind.config.js` with `content: [...]`, `@tailwind base;` directives in CSS | Tailwind v3 |
| No Tailwind at all | Free choice; adding Tailwind is a real cost, weigh it |
| `antd` or `daisyui` already present | Default to keeping it; switching needs a stated reason |

Report what was found before recommending. A recommendation that contradicts the installed
stack without acknowledging it reads as guesswork.

## Step 2 — Choose the library

These two are not competitors on the same axis. Ant Design ships **behavior**; DaisyUI
ships **class names**. That difference decides most cases.

**Ant Design** when the product is data-dense and the widgets are the product: admin
panels, internal tools, CRUD-heavy dashboards, multi-step enterprise forms, tables that
need sorting, filtering, row selection and virtual scroll, date/time range pickers,
tree-selects, i18n and RTL. Buying these behaviors is worth the opinionated look, and the
look can be moved a long way through tokens.

**DaisyUI** when visual identity matters more than widget behavior, and Tailwind is already
in the project: marketing sites, landing pages, docs, content products, and app UIs where
the interactive parts are few and custom. DaisyUI is CSS only — no JavaScript, no focus
management, no portal logic. Anything stateful beyond its CSS-only patterns needs a
headless library (Radix, Base UI, React Aria) underneath. Budget for that instead of
discovering it at the first combobox.

**Neither** when the product is a small set of highly custom surfaces. Tailwind plus a
headless primitive library and ~10 hand-built components beats fighting a library's
opinions. Say so when it is true; recommending a library that will mostly be overridden is
bad advice.

**Mixing:** split by surface, never by component. A marketing site on DaisyUI and an app on
Ant Design is fine when they are separate routes or separate builds with separate CSS
entries. Loading both into one screen collides on resets and on the semantic color names,
and doubles the bundle for no gain.

Present the recommendation as a decision with its tradeoff named, then confirm before
installing. If the user has already decided, skip to the token layer.

## Step 3 — Version gate

**DaisyUI is pinned to the Tailwind major version. This is not negotiable.**

- Tailwind **v4** → DaisyUI **5** (`npm i -D daisyui@latest`). CSS-first config:
  `@plugin "daisyui" { ... }`. `tailwind.config.js` is deprecated in Tailwind v4 — do not
  create one, and do not write DaisyUI config into one.
- Tailwind **v3** → DaisyUI **4** (`npm i -D daisyui@4`). JS config: the plugin goes in
  `tailwind.config.js` under `plugins: [require("daisyui")]` with a sibling `daisyui: {}`
  options object.

Mixing these produces a plugin that silently does nothing, which is hard to debug from the
symptom. When a project is on Tailwind v3 and wants DaisyUI 5, the real task is a Tailwind
v4 upgrade — name that as its own piece of work rather than half-doing it.

**Ant Design v6** (current: 6.6.2 — check `npm view antd version` before pinning, it moves
often) declares `react >= 18.0.0` and `react-dom >= 18.0.0` as
peer dependencies. It uses CSS variables by default and drops IE support.

`@ant-design/icons` is a regular dependency of antd (currently `^6.3.2`), not a peer
dependency — so in most projects the version resolves on its own and there is nothing to
upgrade by hand. The manual step only applies when the project lists `@ant-design/icons`
explicitly in its own `package.json`: a pinned v5 stays locked there and the build breaks,
because `@ant-design/icons@6` is not compatible with `antd@5`. Check for an explicit entry
before telling anyone to upgrade it.

On React 17, stay on the v5 line — install from the live dist-tag rather than a hardcoded
number: `npm i antd@latest-5` (currently 5.29.3).

## Step 4 — Set the token layer

Read the matching reference and follow it; both contain the full, verified config shape:

- `references/antd-theming.md` — ConfigProvider, seed/map/alias tokens, algorithms,
  component tokens, static-method caveat, zero-runtime and bundle notes.
- `references/daisyui-theming.md` — `@plugin "daisyui"` vs `@plugin "daisyui/theme"`, the
  complete custom-theme variable set, semantic color rules, and the DaisyUI 4 JS form.

Whichever library is chosen, the token layer is authored **once**, in **one file**, and
every screen reads from it. Two sources of truth is how a codebase drifts into
inconsistency within a month. If the project has a Penpot or Tokens Studio export, that
W3C token JSON is the source and both configs below are generated from it.

## Step 5 — Anti-slop rules

Generated UI has a recognizable signature: the library's default primary, one uniform
border radius on every element, a single gray, three font sizes that all look the same
weight, a drop shadow on everything, and even spacing that gives no hierarchy. Each rule
below removes one of those tells. They apply to both libraries.

**Color — replace the default primary, then derive the rest.**
Ant Design's `#1677ff` and DaisyUI's stock themes are the two most recognizable "no one
chose this" signals in React. Pick a primary from the brand, then let the library derive
its scale: AntD's algorithm expands `colorPrimary` into the full palette, and DaisyUI
derives states from `--color-primary`. Hand-picking ten shades produces uneven steps.
Beyond the primary, most of a good UI is neutrals — spend the effort there: one neutral
ramp with a deliberate temperature (slightly warm or slightly cool, not pure gray) does
more for perceived quality than a second accent color.

**Typography — 5 to 6 steps from one ratio.**
Choose the ratio by density: ~1.2 for data-dense app UI, 1.25–1.333 for marketing. Six
steps is enough for body, small, large, and three heading levels. Hierarchy comes from
size *plus* weight *plus* color — but never all three at every level, or everything shouts.
Body text stays at one size across the whole product.

**Spacing — 4px base, 8px rhythm, no arbitrary values.**
Both stacks already agree on this: Ant Design's `sizeUnit` and `sizeStep` are both `4`, and
Tailwind's default scale is `0.25rem` = 4px. So use the scale and never reach for
`p-[13px]` or `style={{ margin: 13 }}`. Vertical rhythm carries grouping: related elements
close, unrelated far. Uniform gaps everywhere is the flattest possible hierarchy.

**Radius — one decision, applied by element role.**
Pick a single radius character (sharp, soft, or pill) and let the library apply it by role.
DaisyUI encodes the roles literally (`--radius-selector`, `--radius-field`, `--radius-box`);
Ant Design derives them from the `borderRadius` seed. Setting the same radius on a checkbox,
a button, and a card is the tell — real design systems differentiate.

**Elevation — pick borders or shadows, not both everywhere.**
Choose one primary means of separating surfaces and use the other sparingly for genuinely
floating things (dropdowns, modals). Shadows on every card is the single loudest slop
signal. DaisyUI's `--depth: 0` and Ant Design's flat container tokens both support a
border-led look.

**Density — one dial, set once.**
Ant Design: `controlHeight` (default 32) is the density dial; `theme.compactAlgorithm` is
the pre-built dense variant. DaisyUI: `--size-field`. Set it from the product type — dense
for data tools, roomy for marketing — and do not adjust per component.

## Step 6 — Verify before declaring done

- The chosen library matches the project's actual stack, and the Tailwind/DaisyUI major
  versions line up.
- No default primary color survives anywhere in the output.
- The token file is the only place colors, radii, and spacing values are defined; screens
  contain no hard-coded hex values or arbitrary pixel values.
- Type scale has 5–6 steps from one ratio, and body size is consistent.
- Light and dark (if in scope) both come from the token layer, not from `dark:` overrides
  scattered through components — with DaisyUI in particular, `dark:` on semantic color
  names fights the theme system.
- Contrast holds for text on every surface color that was changed.

Motion, Storybook, and icon choices are handled by their own skills — hand off rather than
improvising them here.
