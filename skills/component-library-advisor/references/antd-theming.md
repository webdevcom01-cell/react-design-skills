# Ant Design theming reference

Verified against `ant-design/ant-design@6.6.1` — `docs/react/customize-theme.en-US.md`,
`docs/react/migration-v6.en-US.md`, `docs/react/getting-started.en-US.md`,
`components/theme/interface/seeds.ts`, `components/theme/themes/seed.ts`.
Spot-checked 2026-08-25 against live docs: seed token defaults, `zeroRuntime` (6.0.0+),
and per-component `algorithm: true` (5.8.0+) all confirmed accurate as stated.

## Contents
- Token architecture
- The full seed token list and defaults
- ConfigProvider theme config
- Algorithms
- Component tokens
- Static methods caveat
- Bundle size and runtime
- v6 requirements

## Token architecture

Three layers, derived in sequence:

- **Seed Token** — the design intent. `colorPrimary`, `borderRadius`, `fontSize`. Changing
  a seed cascades through everything below it.
- **Map Token** — gradients derived from seeds (a whole color palette from one primary,
  a radius set from one radius). Change these through `theme.algorithm` so the gradient
  relationships stay intact; override individual ones through `theme.token` only when
  there is a specific reason.
- **Alias Token** — batch aliases used by components, e.g. `colorLink`.

For most themes, setting seeds is enough. Reach for map/alias tokens only when a seed
cannot express the intent.

## Seed tokens (exact names, with defaults)

```
colorPrimary        '#1677ff'      colorSuccess     '#52c41a'
colorWarning        '#faad14'      colorError       '#ff4d4f'
colorInfo           '#1677ff'      colorLink        ''
colorTextBase       ''             colorBgBase      ''
fontFamily          system stack   fontFamilyCode   'SFMono-Regular', Consolas, ...
fontSize            14             borderRadius     6
lineWidth           1              lineType         'solid'
sizeUnit            4              sizeStep         4
sizePopupArrow      16             controlHeight    32
zIndexBase          0              zIndexPopupBase  1000
opacityImage        1              wireframe        false
focusOutline        true           motion           true
motionUnit          0.1            motionBase       0
motionEaseOutCirc / motionEaseInOutCirc / motionEaseInOut / motionEaseOutBack /
motionEaseInBack / motionEaseInQuint / motionEaseOutQuint / motionEaseOut  (cubic-bezier strings)
```

`sizeUnit: 4` and `sizeStep: 4` are why Ant Design already sits on a 4px grid — matching
Tailwind's default `0.25rem` scale. Do not fight it with arbitrary pixel values.

## ConfigProvider theme config

```tsx
import { ConfigProvider, type ThemeConfig } from 'antd';

const theme: ThemeConfig = {
  token: {
    colorPrimary: '#00b96b',
    borderRadius: 2,
    fontFamily: '"Inter", system-ui, sans-serif',
    colorBgContainer: '#f6ffed',
  },
};

export default function App({ children }: { children: React.ReactNode }) {
  return <ConfigProvider theme={theme}>{children}</ConfigProvider>;
}
```

`theme` accepts:

| Property | Type | Default | Note |
|---|---|---|---|
| `token` | `AliasToken` | – | seed / map / alias overrides |
| `components` | `ComponentsConfig` | – | per-component tokens |
| `algorithm` | `(token: SeedToken) => MapToken` or an array of them | `defaultAlgorithm` | |
| `cssVar` | `{ prefix?: string; key?: string }` | – | CSS-variable output |
| `hashed` | `boolean` | `true` | hash suffix on class names |
| `inherit` | `boolean` | `true` | inherit from an outer ConfigProvider |
| `zeroRuntime` | `boolean` | `false` | 6.0.0+; see below |

Define the theme object **outside** the component, or memoize it — a new object identity on
every render re-generates styles.

Never pass `theme={undefined}` conditionally. When `theme` is `undefined`, antd skips a
Provider layer, so toggling between `undefined` and an object changes the React tree shape
and remounts the subtree. Use `{}` instead.

## Algorithms

```tsx
import { theme } from 'antd';

const { defaultAlgorithm, darkAlgorithm, compactAlgorithm } = theme;

const config: ThemeConfig = { algorithm: [darkAlgorithm, compactAlgorithm] };
```

Three presets: `defaultAlgorithm`, `darkAlgorithm`, `compactAlgorithm`. They compose in an
array — dark + compact is a common pairing for dense internal tools.

Dark mode is an algorithm swap, not a second hand-written palette. Prefer it over
maintaining two token sets.

## Component tokens

```tsx
const config: ThemeConfig = {
  components: {
    Button: { colorPrimary: '#00b96b', algorithm: true },
    Input:  { colorPrimary: '#eb2f96' },
  },
};
```

By default a component token only **overrides** the global token — it is not expanded by the
algorithm, so a component-level `colorPrimary` will not generate hover/active shades.
Setting `algorithm: true` (antd ≥ 5.8.0) runs the global algorithm for that component;
an algorithm or array of algorithms can also be passed to override it.

## Reading tokens

Inside React:

```tsx
const { token } = theme.useToken();
```

Outside the React lifecycle (build scripts, canvas, non-React code):

```tsx
import { theme } from 'antd';

const globalToken = theme.getDesignToken(config); // config optional
```

## Static methods do not see ConfigProvider

`message.xxx`, `Modal.xxx`, and `notification.xxx` render through their own React root, so
they do not inherit ConfigProvider context — including the theme. Use the hook forms
(`Modal.useModal()`, `message.useMessage()`) and place the returned `contextHolder` inside
the provider, or wrap the app in antd's `App` component, which does this once for all three.

## Bundle size and runtime

- `antd` supports ES-module tree shaking by default: `import { Button } from 'antd';` drops
  unused code. `babel-plugin-import` is a v3/v4 relic — do not add it.
- Keep imports named from `'antd'`. Deep paths like `antd/es/button` are not needed and
  break style resolution.
- `zeroRuntime: true` (6.0.0+) stops runtime style generation; styles must then be imported
  manually via `import 'antd/dist/antd.css'`. That file carries every component's styles
  with no hashed class names. To ship less, generate a subset with
  `@ant-design/static-style-extract`:

```tsx
import fs from 'fs';
import { extractStyle } from '@ant-design/static-style-extract';

fs.writeFileSync('/path/to/antd.css', extractStyle({ includes: ['Button'] }));
```

- For SSR, follow `docs/react/server-side-rendering` — style extraction differs per
  framework and getting it wrong produces a flash of unstyled content.

## v6 requirements

- React ≥ 18 (v6 dropped React 17 and earlier).
- `@ant-design/icons` ≥ 6 — and `@ant-design/icons@6` is **not** compatible with `antd@5`.
  Upgrade the pair together; a version mismatch surfaces as build errors.
- `@ant-design/v5-patch-for-react-19` is no longer needed; remove it.
- CSS variables are on by default; IE is unsupported.
- DOM structure changed for many components in v6 — CSS selectors written against v5 markup
  may break.

## Agent tooling from the antd team

`npx skills add ant-design/ant-design-cli` installs their own skill, and
`npm i -g @ant-design/cli` gives offline per-component metadata (`antd info Button`,
`antd token DatePicker`, `antd changelog 5.0.0 6.0.0 Select`, `antd lint ./src`). Useful
for verifying a prop or token against the installed version instead of guessing.
