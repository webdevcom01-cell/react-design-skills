# Wiring the project's theme into Storybook

Verified against `storybookjs/storybook` v10 docs (`docs/essentials/themes.mdx`,
`docs/writing-stories/mocking-data-and-modules/mocking-providers.mdx`,
`docs/_snippets/storybook-addon-themes-*.md`) and npm `@storybook/addon-themes@10.5.10`.

## Contents
- The rule that matters
- Install
- Ant Design — `withThemeFromJSXProvider`
- DaisyUI — `withThemeByDataAttribute`
- Per-story theme override without the addon
- Failure modes

## The rule that matters

`.storybook/preview.tsx` imports the **same** theme object and the **same** global CSS the
application imports. It never declares its own copy.

A duplicated theme in `.storybook/` looks harmless on day one and is a second design system
by month two: the app's primary changes, Storybook's does not, and every review after that
is reviewing a product that does not exist.

If the project keeps its tokens in a W3C token JSON (Penpot / Tokens Studio export), that
file feeds both the app config and this preview file. Same source, two consumers.

## Install

```bash
npx storybook@latest add @storybook/addon-themes
```

The `add` command installs the package and registers it in `.storybook/main.ts`. Keep its
version equal to the Storybook core version — the addons ship in lockstep.

## Ant Design — `withThemeFromJSXProvider`

Ant Design is themed through a React provider, so it takes the JSX-provider decorator. The
`themes` map holds `ThemeConfig` objects; the `Provider` is `ConfigProvider`.

```tsx
// .storybook/preview.tsx
import React from 'react';
import type { Preview, Renderer } from '@storybook/react-vite';
import { withThemeFromJSXProvider } from '@storybook/addon-themes';
import { ConfigProvider } from 'antd';

// The same objects the app imports — do not redefine them here.
import { lightTheme, darkTheme } from '../src/theme/antdTheme';

const preview: Preview = {
  decorators: [
    withThemeFromJSXProvider<Renderer>({
      themes: { light: lightTheme, dark: darkTheme },
      defaultTheme: 'light',
      Provider: ConfigProvider,
    }),
  ],
  tags: ['autodocs'],
};

export default preview;
```

`withThemeFromJSXProvider` passes the selected entry from `themes` to the provider as its
`theme` prop, which is exactly `ConfigProvider`'s API. It also accepts a `GlobalStyles`
component if the project has one.

Notes specific to Ant Design:

- Dark mode should come from `theme.darkAlgorithm`, not a hand-written dark palette. Build
  `darkTheme` in the app's theme module as `{ ...lightTheme, algorithm: theme.darkAlgorithm }`
  and import it here, so Storybook and the app switch identically.
- Components rendered through antd's static methods (`message.xxx`, `Modal.xxx`,
  `notification.xxx`) do not receive `ConfigProvider` context anywhere, Storybook included.
  Stories for anything using them must use the hook forms or wrap in antd's `App` component,
  or the story will render an unthemed dialog and look like a Storybook bug.
- If the project sets `zeroRuntime: true`, the corresponding CSS file has to be imported in
  `preview.tsx` too, or stories render unstyled.

## DaisyUI — `withThemeByDataAttribute`

DaisyUI is themed by a `data-theme` attribute on an ancestor element, so it takes the
data-attribute decorator. **The global CSS import is mandatory** — without it Tailwind and
DaisyUI never load and every story renders as unstyled HTML.

```tsx
// .storybook/preview.tsx
import type { Preview } from '@storybook/react-vite';
import { withThemeByDataAttribute } from '@storybook/addon-themes';

// Mandatory: the app's Tailwind entry, containing @import "tailwindcss";
// @plugin "daisyui"; and the @plugin "daisyui/theme" block.
import '../src/index.css';

const preview: Preview = {
  decorators: [
    withThemeByDataAttribute({
      themes: { light: 'mytheme', dark: 'mytheme-dark' },
      defaultTheme: 'light',
      attributeName: 'data-theme',
    }),
  ],
  tags: ['autodocs'],
};

export default preview;
```

The **values** in the `themes` map are theme names as DaisyUI knows them — they must match
names enabled in the `@plugin "daisyui" { themes: ... }` block or defined by a
`@plugin "daisyui/theme" { name: "..." }` block. The **keys** are only labels for the
toolbar. A typo in a value fails silently: the attribute is set, no theme matches it, and
the built-in default renders instead.

Vite-based Storybook picks up the Tailwind v4 plugin from the project's Vite config. If
stories are unstyled while the app is fine, check that `.storybook/main.ts` is not
overriding `viteFinal` in a way that drops `@tailwindcss/vite`.

## Per-story theme override without the addon

For a one-off — a story that must render in dark while the rest default to light — read
`parameters` in a decorator instead of adding a second decorator:

```tsx
// .storybook/preview.tsx
const preview: Preview = {
  decorators: [
    (Story, { parameters }) => {
      const { theme = 'light' } = parameters;
      return (
        <ConfigProvider theme={themes[theme]}>
          <Story />
        </ConfigProvider>
      );
    },
  ],
};
```

```tsx
// Button.stories.tsx
export const OnDarkSurface: Story = {
  parameters: { theme: 'dark' },
};
```

This is the documented pattern for configuring a mocked provider per story, and it
generalizes beyond themes — user roles, locales, feature flags.

Note the file extension: a preview file containing JSX must be `.tsx` / `.jsx`, not `.ts`.

## Failure modes

| Symptom | Cause |
|---|---|
| Stories render as unstyled HTML | Global CSS not imported in `preview.tsx` (DaisyUI/Tailwind) |
| Toolbar switches but nothing changes | Theme name in `themes` map does not match a DaisyUI theme name |
| Storybook colors differ from the app | `preview.tsx` declares its own theme instead of importing the app's |
| Modals/toasts render unthemed | antd static methods bypass `ConfigProvider`; use hooks or `App` |
| Type errors on `Meta`/`StoryObj` | Imported from `@storybook/react` instead of the framework package |
| Addon behaves oddly after upgrade | Addon major does not match Storybook core version |
