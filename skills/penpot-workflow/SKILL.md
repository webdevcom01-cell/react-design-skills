---
name: penpot-workflow
description: Use Penpot as the single source of truth for a project's design tokens — define them there, export the token JSON, and generate the Ant Design ConfigProvider theme and the DaisyUI CSS theme from it instead of hand-maintaining either. Use whenever Penpot, design tokens, token export or import, DTCG, Tokens Studio, design handoff, or designer-to-code sync comes up; whenever a project needs one place where colors, spacing, radii, and typography are defined for both design and code; and whenever designs, SVG, or assets need to come out of a design tool into a React codebase. Also covers Serbian phrasings - Penpot, dizajn tokeni, izvoz tokena, handoff, dizajn sistem, jedan izvor istine. Do NOT use for choosing a component library or writing a theme by hand (component-library-advisor covers that), or for whiteboarding and diagramming (tldraw-workflow).
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "penpot/penpot @ 2.17.1 (2.18.0 unreleased); common/src/app/common/types/token.cljc, plugins/libs/plugin-types/index.d.ts, docs/user-guide/design-systems/design-tokens.njk, library/test/_tokens-*.json; npm @penpot/mcp@2.15.4, @penpot/plugin-types@1.4.2 (read from source, 2026-08-23); re-verified 2026-08-25 — MPL-2.0 license confirmed against the repo's LICENSE file, npm package versions for @penpot/mcp and @penpot/plugin-types confirmed current; Penpot itself has since shipped a 2.17.2 patch release (bugfixes only, no token-system changes) so 2.17.1 is one patch behind current"
---

# Penpot Workflow

Most design tools are used as handoff: the designer produces a picture, the developer reads
values off it and retypes them. Every retype is a chance to drift, and within two sprints
the design file and the code disagree about what the primary color is.

Penpot's token system removes the retyping. Tokens are defined once, exported as JSON, and
compiled into whatever the code needs — an Ant Design `ThemeConfig`, a DaisyUI
`@plugin "daisyui/theme"` block, CSS custom properties. The design file stops being a
picture of the system and becomes the system's source.

That only works if the pipeline is real. A Penpot file whose tokens are exported once,
pasted by hand, and never re-exported is handoff with extra steps.

## Step 1 — Know what the format actually is

Penpot's documentation says its tokens "adhere to the Design Tokens Format Module", the W3C
DTCG draft. That is true of the **syntax** and misleading about the **vocabulary**. Read
`references/token-format.md` before writing any transform — the differences decide whether
a downstream tool can read the file.

The short version:

- **DTCG syntax, yes.** Tokens are `{ "$value": ..., "$type": ..., "$description": ... }`,
  nested objects form groups, and aliases are `{token.name}` references.
- **Tokens Studio vocabulary, not W3C.** The `$type` values include `fontSizes`,
  `fontWeights`, `fontFamilies`, `borderRadius`, `borderWidth`, `spacing`, `sizing`,
  `opacity`, `letterSpacing`, `textCase`, `textDecoration`, `rotation`. None of those are
  W3C DTCG types — W3C would express most of them as `dimension` or `number`. Penpot does
  emit the genuine W3C names `color`, `dimension`, `number`, `shadow`, and `typography`.
- **`$themes` and `$metadata` at the root are Tokens Studio extensions**, not W3C. They
  carry the theme list and the set order, and the set order matters: when the same token
  name exists in several active sets, the later one wins.

So the practical statement is: **Penpot speaks the Tokens Studio dialect of DTCG.** That is
good news — it means the mature Tokens Studio tooling reads it — but a strict W3C-only
parser will reject or silently drop most of a Penpot export. Never promise "standard W3C
tokens" to a downstream team without this caveat.

Penpot's importer also **discards token types it does not support** rather than failing.
A round-trip through another tool can therefore lose tokens quietly. Diff exports before
and after any round-trip.

## Step 2 — Structure the tokens so they can compile

Design the token file for its consumers, not just for the design canvas.

**Two layers, always.** A primitive layer holds raw values (`color.blue.500`, `size.4`).
A semantic layer references them (`color.action.primary` → `{color.blue.500}`,
`spacing.card-padding` → `{size.4}`). Code and components consume the **semantic** layer
only. Without this split, rebranding means touching every component; with it, it means
changing the primitive layer.

**Sets carry variation, themes carry combinations.** Put light and dark in separate sets,
brand variants in separate sets, density in separate sets. A Penpot theme then activates a
combination. Themes can be grouped, and only one theme per group is active at a time —
which is how independent axes (color scheme × density × brand) are modeled without a
combinatorial set explosion.

**Name for the target.** Dots create groups: `button.primary.default.background-color`
nests four levels deep in the JSON. Names may contain letters, digits, `_`, `-`, and `$`,
must not start with `$`, and must not end with a dot. References are case sensitive — a
mismatched case is a broken alias, not a fallback.

**Use aliases and math instead of duplicated numbers.** Numeric tokens accept equations:
`{spacing.small} * 2`, `{spacing.small} * {spacing.scale}`. A spacing scale expressed as
math stays consistent when the base changes; a scale typed as ten literals does not.

## Step 3 — Get the tokens out

Four routes, in increasing order of automation. Details and code in
`references/token-pipeline.md`.

1. **Manual export (UI).** Tokens tab → **Tools** → **Export**, as a single JSON file or a
   folder of files (one per set, plus `$themes.json` and `$metadata.json`). Content is
   identical; the choice is organizational. Fine to start with, and fine forever for a
   project whose tokens change twice a year.
2. **Plugin API.** `penpot.library.local.tokens` exposes a `TokenCatalog` with `themes` and
   `sets`, each set exposing `tokens` and `tokensByType`, plus `addSet`, `addToken`, and
   `addTheme`. This is the real automation surface: a plugin can read the whole catalog and
   push it anywhere. Note that the user-facing docs still say the tokens plugin API is
   "coming soon" — the docs are stale, the typed API exists.
3. **MCP server.** `npx -y @penpot/mcp@latest` runs Penpot's official MCP server, which
   drives the Plugin API through a companion plugin loaded in the browser over a WebSocket.
   It lets an agent read and modify a design file directly. Match the MCP version to the
   Penpot version — the npm package trails the repo, and a mismatch is a supported failure
   mode, not a bug.
4. **REST API.** Personal access tokens (Your account → Access tokens) authenticate against
   `/api/rpc/command/...` with an `Authorization: Token <token>` header. Suitable for CI.
   Treat these tokens as passwords: environment variables or a secret manager, never in the
   repo and never pasted into a shell command that gets recorded.

Whichever route, the exported JSON belongs **in the repository**, committed. That is what
makes token changes reviewable in a pull request instead of invisible.

## Step 4 — Compile tokens into the component library's theme

This is the step that closes the loop with `component-library-advisor`. The exported JSON
is the input; the outputs are generated files that no one edits by hand:

- **Ant Design** → a `ThemeConfig` object: semantic tokens map onto seed tokens
  (`colorPrimary`, `borderRadius`, `fontFamily`, `fontSize`, `controlHeight`). Let antd's
  algorithm derive the rest of the palette rather than exporting every shade.
- **DaisyUI** → a `@plugin "daisyui/theme" { ... }` block: semantic tokens map onto
  `--color-*`, `--radius-*`, `--size-*`, `--border`. Every variable in the block must be
  present, so the generator needs a complete mapping or explicit defaults.

`references/token-pipeline.md` covers both mappings, the Style Dictionary route with
`@tokens-studio/sd-transforms` for the dialect, and when a 60-line custom script beats a
build tool.

Mark generated files as generated — a header comment and a build script — so the next
person edits the token source rather than the output.

## Step 5 — Designs and assets

Penpot is a full design tool, not only a token store, and its file format is SVG-based, so
exports are open rather than proprietary.

- Export shapes and boards as **SVG** for anything that should scale or inherit color.
  SVG that will be recolored by CSS must use `currentColor` rather than a baked hex.
- Prefer SVG-as-component (via an SVGR-style transform) over `<img>` when the asset needs
  props, accessible labels, or theme-driven color.
- Raster exports (PNG/WebP) at defined scales for photographic content only.
- Icons drawn in Penpot are a legitimate source, but for standard UI icons and brand logos
  check the `icon-resources` skill first — a maintained icon set beats hand-drawn glyphs.

## Step 6 — Licensing, stated correctly

Penpot itself is licensed **MPL-2.0**. This governs the Penpot *software*: relevant when
self-hosting a modified build and distributing it, since MPL-2.0 is a file-level copyleft
that requires modified source files to be shared under the same license.

It does **not** attach to designs, files, tokens, or exports produced with Penpot. Work
created in the tool belongs to whoever created it, exactly as with any other design tool.
Say this plainly if a client raises it — the confusion is common and the answer is simple.

The hosted service at design.penpot.app is a separate matter from the software license;
check its current terms and any paid plan limits before committing a team to it.

## Step 7 — Verify before declaring done

- The exported token JSON is committed to the repository.
- Tokens are split into primitive and semantic layers, and components consume only semantic.
- The AntD theme file and/or the DaisyUI theme block are **generated**, and regenerating
  them produces no diff.
- A token change in Penpot, re-exported and recompiled, visibly changes the running app —
  test this once; a pipeline that has never been run end to end does not work.
- Nothing downstream was told the export is "W3C standard tokens" without the dialect caveat.
- Access tokens are in environment variables, not in the repository.
