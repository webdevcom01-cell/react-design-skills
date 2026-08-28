# From Penpot export to a themed React app

Verified against `penpot/penpot` @ 2.17.1 — `plugins/libs/plugin-types/index.d.ts`,
`mcp/README.md`, `docs/technical-guide/integration.md`,
`docs/user-guide/design-systems/design-tokens.njk`; npm `@penpot/mcp@2.15.4`,
`@penpot/plugin-types@1.4.2`, `style-dictionary@5.5.2`, `@tokens-studio/sd-transforms@2.0.3`.

## Contents
- Pipeline shape
- Getting tokens out: four routes
- Transform: Style Dictionary or a custom script
- Mapping to Ant Design
- Mapping to DaisyUI
- Making it repeatable

## Pipeline shape

```
Penpot tokens  →  tokens.json (committed)  →  transform  →  generated theme files  →  app
```

Three properties make it a pipeline rather than a handoff:

1. `tokens.json` lives in the repository, so token changes show up in code review.
2. The theme files are **generated** and never hand-edited.
3. Regenerating produces no diff unless the source changed. That is the test.

## Getting tokens out: four routes

### 1. Manual export (UI)

Tokens tab → **Tools** → **Export** → single JSON file or multiple files. Both offer a
preview before download. Commit the result.

Good enough for most projects. Do not build automation for a token set that changes twice a
year — the automation will rot faster than the tokens.

### 2. Plugin API

`penpot.library.local.tokens` returns a `TokenCatalog`:

```ts
interface TokenCatalog {
  readonly themes: TokenTheme[];
  readonly sets: TokenSet[];
  addTheme({ group, name }: { group: string; name: string }): TokenTheme;
  addSet({ name, active }: { name: string; active?: boolean }): TokenSet;
  getThemeById(id: string): TokenTheme | undefined;
  getSetById(id: string): TokenSet | undefined;
}

interface TokenSet {
  readonly id: string;          // internal only, never exported or synced
  name: string;                 // may contain a group path separated by "/"
  active: boolean;              // only active sets affect shapes and resolution
  readonly tokens: Token[];     // alphabetical
  readonly tokensByType: [string, Token[]][];
  toggleActive(): void;
  getTokenById(id: string): Token | undefined;
  addToken({ type, name, value }: { type: TokenType; name: string; value: TokenValueString }): Token;
  duplicate(): TokenSet;
  remove(): void;
}
```

Two things worth knowing before writing a plugin:

- **Values are strings, always** — including numeric types. `addToken` expects `"16"` or
  `"16px"`, not `16`. A plain number is coerced, but write the string.
- **New sets are inactive by default.** Pass `active: true` or set `set.active = true`
  afterwards, or the tokens exist and affect nothing.

Set ids are internal and are never exported or synced with external token sources — key
anything durable on **names**, not ids.

The user guide still describes the tokens plugin API as "coming soon". That text is stale;
the typed API is shipped. Trust `plugins/libs/plugin-types/index.d.ts` over the prose docs.

### 3. MCP server

```bash
npx -y @penpot/mcp@latest
```

Penpot's official MCP server drives the Plugin API from an AI client. The architecture is
indirect: the MCP server talks over a WebSocket to a companion **Penpot MCP Plugin** loaded
in the browser, which executes code against the Plugin API inside the design file. So the
browser tab must be open and the plugin connected — this is not a headless server.

Practical constraints from the project's own README:

- **Version match matters.** Run the MCP version that matches the Penpot version. The npm
  package trails the repository (2.15.4 published against a 2.17.x repo at the time of
  writing), so pin deliberately.
- **Local-network restrictions bite.** Chromium 142+ hardened private network access;
  connecting `localhost` from `design.penpot.app` needs an explicit permission grant, and
  some browsers (Brave's Shields) block it outright. Firefox is the documented fallback.
- The MCP integration has had bugs when the Penpot tab is backgrounded or frozen by the
  browser. Keep the tab foregrounded during a session.

Useful for agent-driven design work. Not the right tool for a CI token export.

### 4. REST API

Personal access tokens are created under **Your account → Access tokens** and used as a
header:

```bash
curl -H "Authorization: Token $PENPOT_TOKEN" \
  https://design.penpot.app/api/rpc/command/get-profile
```

This is the route for CI. Treat the token as a password: environment variable or secret
manager, never committed, and never pasted inline into a shell command that gets recorded
in shell history or a tool-approval file.

## Transform: Style Dictionary or a custom script

Penpot emits the **Tokens Studio dialect** (see `token-format.md`), which is what
`@tokens-studio/sd-transforms` exists to normalize before handing off to Style Dictionary.
That is the natural pairing, and the right first thing to try for a project with many
tokens, several themes, and multiple output targets.

Verify it against the actual export rather than assuming: run one real Penpot file through
it and check that `$themes` / `$metadata`, the plural type names, and the composite
`typography` shape all survive. Adjust or add transforms where they do not.

For a small project — one theme, thirty semantic tokens, two outputs — a plain Node script
that reads the JSON, resolves `{alias}` references, and writes two files is roughly 60 lines
and has no dependency to maintain. Prefer it when that is the actual scope. The point is
generated output, not any particular build tool.

Whichever route, the resolver has to handle three things the format guarantees:

- **Alias resolution**, including chains (`a` → `{b}` → `{c}`) and case-sensitive names.
- **Math evaluation** for numeric tokens (`{spacing.small} * 2`).
- **Set precedence** — merge active sets in `tokenSetOrder`, later sets overriding earlier.

## Mapping to Ant Design

Map semantic tokens onto **seed tokens** and let antd's algorithm derive the rest. Exporting
every shade defeats the algorithm and produces uneven palettes.

```ts
// src/theme/antdTheme.generated.ts — GENERATED FROM tokens.json, DO NOT EDIT
import { theme, type ThemeConfig } from 'antd';

export const lightTheme: ThemeConfig = {
  token: {
    colorPrimary: '#00b96b',    // ← color.action.primary
    colorSuccess: '#52c41a',    // ← color.feedback.success
    colorError:   '#ff4d4f',    // ← color.feedback.error
    borderRadius: 6,            // ← radius.field       (number, not "6px")
    fontFamily:   '"Inter", system-ui, sans-serif',
    fontSize:     14,           // ← font.size.body
    controlHeight: 32,          // ← density.control
  },
};

export const darkTheme: ThemeConfig = {
  ...lightTheme,
  algorithm: theme.darkAlgorithm,
};
```

Two conversion notes:

- antd seed tokens are **numbers** for sizes and radii, while Penpot values are strings
  possibly carrying units. Strip units and convert; a `"6px"` passed as `borderRadius`
  produces a silently wrong theme.
- Dark mode should be `theme.darkAlgorithm` over the same seeds, not a second exported
  palette. Export one palette; derive the dark one.

## Mapping to DaisyUI

Generate the whole `@plugin "daisyui/theme"` block. **Every variable must be present** — the
generator needs a complete mapping or explicit defaults, or the theme is incomplete and
DaisyUI falls back in ways that are hard to trace.

```css
/* src/styles/theme.generated.css — GENERATED FROM tokens.json, DO NOT EDIT */
@plugin "daisyui/theme" {
  name: "brand";
  default: true;
  color-scheme: light;

  --color-base-100: #ffffff;        /* ← color.surface.base */
  --color-base-200: #f5f5f4;        /* ← color.surface.raised */
  --color-base-300: #e7e5e4;        /* ← color.surface.overlay */
  --color-base-content: #1c1917;    /* ← color.text.default */
  --color-primary: #00b96b;         /* ← color.action.primary */
  --color-primary-content: #ffffff; /* ← color.action.primary-on */
  /* … secondary, accent, neutral, info, success, warning, error and their -content … */

  --radius-selector: 1rem;          /* ← radius.selector */
  --radius-field: 0.25rem;          /* ← radius.field */
  --radius-box: 0.5rem;             /* ← radius.box */
  --size-selector: 0.25rem;
  --size-field: 0.25rem;            /* ← density.control */
  --border: 1px;                    /* ← border.width.default */
  --depth: 0;
  --noise: 0;
}
```

The `*-content` foreground colors are a contrast obligation, not a free choice. If the token
source does not define them, the generator must compute an accessible pair rather than
defaulting to white — that is where generated DaisyUI themes usually fail review.

Import the generated file after the plugin declaration in the app's CSS entry, and import
that same entry in `.storybook/preview.tsx` so Storybook shows the real theme.

## Making it repeatable

```json
{
  "scripts": {
    "tokens:build": "node scripts/build-tokens.mjs",
    "tokens:check": "npm run tokens:build && git diff --exit-code src/theme src/styles"
  }
}
```

`tokens:check` in CI is what keeps the pipeline honest: it fails when someone hand-edits a
generated file or forgets to regenerate after a token change. Without it, the generated
files drift back into hand-maintained files within a couple of months and the whole
arrangement quietly reverts to handoff.
