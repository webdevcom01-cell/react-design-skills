---
name: storybook-workflow
description: Set up and structure Storybook for a React component library — CSF 3 story files, variant and state coverage, and a preview that renders the project's real theme instead of Storybook defaults. Use whenever a reusable component library, design system, or shared UI package is being built, documented, or reviewed; whenever .stories.tsx, CSF, Storybook, autodocs, story variants, addon-themes, component isolation, or visual regression testing come up; and whenever components need to be developed outside the app before they are wired into it. Also covers Serbian phrasings - komponentna biblioteka, dizajn sistem, storybook prica, varijante komponente, vizuelni regresioni test. Do NOT use for landing pages, marketing sites, or one-off application screens; Storybook is pure overhead there, and saying so is the correct answer.
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "storybookjs/storybook docs @ v10 branch; npm storybook@10.5.10, @storybook/addon-themes@10.5.10, chromatic@18.5.0 (read from source, 2026-08-23); adversarially re-verified 2026-08-25, 5 factual claims corrected"
---

# Storybook Workflow

Storybook earns its cost when components have **more than one consumer**. A shared button
that six screens import needs a place where its variants are visible side by side, where a
new team member can see what exists before rebuilding it, and where a change can be
reviewed without opening the app. That is a component library.

A landing page has none of those properties. Neither does a screen used once. Setting up
Storybook there adds a build target, a pile of devDependencies to keep updated, and story
files that go stale within a sprint, in exchange for nothing.

## Step 1 — Decide whether Storybook belongs here at all

Ask what is being built. Storybook is right when **two or more** of these hold:

- The components are imported by multiple screens, apps, or teams.
- The output is a package (`@company/ui`) or an internal design system.
- Components have real variant/state surface — sizes, tones, loading, disabled, error.
- Designers or non-frontend people need to see the components without running the app.

Storybook is wrong for: landing and marketing pages, a single admin screen, a prototype,
or a project whose components are each used exactly once. Say this plainly and stop — the
useful answer is often "you don't need this yet." Recommend it later, when the second
consumer of a component appears.

If the project is a mix — a design system package plus a marketing site — Storybook covers
the package only. Do not add stories for page-level compositions that will never be reused.

## Step 2 — Install

```bash
npm create storybook@latest
```

That command detects the framework, installs it, writes `.storybook/`, and adds scripts.
Prefer it to manual setup; it also picks the right framework package, which is easy to get
wrong.

Current line is **Storybook 10** (10.5.10 at time of writing). Storybook's own addons ship
in lockstep with the core version — `@storybook/addon-themes`, `@storybook/addon-a11y`, and
`@storybook/addon-docs` all track the same number. Mismatched addon majors are a common and
confusing failure; keep them equal.

Framework packages for React: `@storybook/react-vite` (Vite), `@storybook/nextjs` (Next.js
with Webpack), `@storybook/nextjs-vite` (Next.js with Vite). React ≥ 16.8 and Vite ≥ 5 for
the Vite framework.

**Import from the framework package, not the renderer.** `import type { Meta, StoryObj }
from '@storybook/react-vite'` — not from `@storybook/react`. This is the single most common
stale-tutorial habit (older tutorials predate the framework packages). The framework
packages currently re-export these types from `@storybook/react`, so it will not always
error — but it is the version Storybook's own docs use and the one guaranteed to stay
correct as the packages diverge, so default to it rather than relying on the re-export.

## Step 3 — Write stories

Story files live **next to the component**, not in a parallel test tree:

```
components/
└─ Button/
   ├─ Button.tsx
   └─ Button.stories.tsx
```

Colocation is what keeps stories alive — a component edited in one folder with its stories
visible gets its stories updated. A `__stories__` directory two levels away does not.

Use **CSF 3**, the stable format:

```tsx
import type { Meta, StoryObj } from '@storybook/react-vite';
import { Button } from './Button';

const meta = {
  component: Button,
  tags: ['autodocs'],
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: { variant: 'primary', children: 'Save changes' },
};
```

`satisfies Meta<typeof Button>` plus `StoryObj<typeof meta>` is what gives args real type
checking against the component's props. Without the `satisfies`, args silently degrade to
loose typing.

A `component` property lets Storybook compute the title automatically — prefer that to
hardcoding `title`, so moving a file does not break its sidebar path or its URL.

**CSF Next** (`defineMain` / `definePreview` / `preview.meta()` / `meta.story()`) exists in
v10 and gives stronger addon typing, but the docs label it a **preview feature whose API
may still change**. Use CSF 3 for anything a team depends on; reach for CSF Next only when
the user asks for it knowing that.

For which stories to write, and how to cover variants and states without combinatorial
explosion, read `references/story-patterns.md`.

## Step 4 — Make Storybook render the project's real theme

This is the step that decides whether Storybook is useful or actively misleading. A
Storybook showing default Ant Design blue while the product ships a custom token theme is
worse than no Storybook — every review of it is a review of something that does not exist.

Two rules carry most of the value:

**Import the app's theme, never redefine it.** `.storybook/preview.tsx` imports the same
theme object and the same global CSS file the application imports. The moment Storybook
holds its own copy of the tokens, there are two design systems drifting apart.

**Use `@storybook/addon-themes` rather than a hand-rolled decorator.** It ships the three
wiring shapes and adds a toolbar switcher for free:

- `withThemeFromJSXProvider` — for libraries themed through a React provider. This is the
  Ant Design path: the provider is `ConfigProvider`, the theme values are the same
  `ThemeConfig` object the app uses.
- `withThemeByDataAttribute` — for libraries themed through an attribute on a parent. This
  is the DaisyUI path: `attributeName: 'data-theme'`, with the theme names from the
  `@plugin "daisyui"` block.
- `withThemeByClassName` — for class-driven theming.

Full working `preview.tsx` files for both Ant Design and DaisyUI, including the global-CSS
import that DaisyUI requires, are in `references/theme-decorators.md`. Read it before
writing the preview file — the failure modes there (missing CSS import, theme names that do
not match the plugin config) are silent.

The token layer itself is chosen and authored by the `component-library-advisor` skill.
This skill consumes it; it does not invent one.

## Step 5 — Documentation and checks worth turning on

**Autodocs.** Tagging stories with `autodocs` generates a docs page from the component's
props and stories. Set it once in `.storybook/preview.ts` (`tags: ['autodocs']`) rather than
per file. A component with no story tagged `autodocs` and no hand-written MDX docs page
gets no docs page at all — the tag is the automatic path, not the only path.

**Accessibility.** `npx storybook@latest add @storybook/addon-a11y` installs and registers
the addon; axe-core itself then runs automatically against whichever story is currently
open in the Storybook UI, or across all stories when the addon's test-runner/Vitest
integration is run. It catches roughly half of WCAG issues automatically — a first line of
QA, not a substitute for the accessibility review skill.
Its results split into Violations, Passes, and **Incomplete**; the incomplete ones need a
human and are the ones teams learn to ignore. Say so when handing results over.

**Visual regression (optional).** `npx storybook@latest add @chromatic-com/storybook` adds
Chromatic's Visual Tests addon, which snapshots each story and flags pixel changes across
commits. It is a hosted commercial service from the Storybook maintainers, so it is a
budget decision, not just a technical one — check current pricing and plan limits before
recommending it. The value is real for a library with many consumers, where a token change
can quietly alter twenty components; it is overkill for a small internal set.

## Step 6 — Verify before declaring done

- Stories render with the project's actual theme; a screenshot of Storybook and a
  screenshot of the app show the same colors, radii, and type.
- The theme object and global CSS in `.storybook/` are imported from the app, not duplicated.
- Every component has stories for its meaningful variants **and** its non-happy states —
  loading, disabled, error, empty — not just the default.
- Story files sit beside their components.
- Types come from the framework package, and `satisfies Meta<typeof X>` is present.
- Core and addon versions match.
- `npm run build-storybook` succeeds — needed for self-hosting a static Storybook anywhere
  outside the Storybook UI's own built-in Share link, which was the point.
