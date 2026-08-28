# Penpot token format reference

Verified against `penpot/penpot` @ 2.17.1 — `common/src/app/common/types/token.cljc`
(the authoritative type table), `library/test/_tokens-1.json` and `_tokens-2.json`
(real export fixtures), `docs/user-guide/design-systems/design-tokens.njk`,
`.serena/memories/common/tokens-schema-subtleties.md`.
Re-verified 2026-08-25 directly against `token.cljc` on the live `develop` branch — the
entire `$type` table, the singular backward-compat import aliases, and the composite-only
`lineHeights`/`lineHeight` member all match the source exactly.

## Contents
- What the file looks like
- The complete `$type` table and its W3C status
- Import aliases accepted for backwards compatibility
- Composite tokens
- Names, references, and math
- Sets, themes, and precedence
- File layouts: single, multifile, ZIP
- Known lossy behavior

## What the file looks like

A single-file export, root keys are **set names**:

```json
{
  "Global": {
    "color": {
      "300": { "$value": "red", "$type": "color", "$description": "my token description" }
    }
  },
  "Brands/A": {
    "color": {
      "accent": { "$value": "{color.300}", "$type": "color", "$description": "" }
    }
  },
  "$themes": [
    {
      "id": "48af6582-f247-8060-8006-ff4dd1d761a8",
      "name": "tes1",
      "description": "",
      "isSource": false,
      "selectedTokenSets": { "Global": "enabled", "Brands/A": "enabled" }
    }
  ],
  "$metadata": {
    "tokenSetOrder": ["Global", "Brands/A"],
    "activeThemes": ["/tes1"],
    "activeSets": ["Global", "Brands/A"]
  }
}
```

`$themes` and `$metadata` are **Tokens Studio extensions**. The W3C DTCG draft defines
neither. A strict DTCG parser will treat them as malformed groups.

## The complete `$type` table and its W3C status

From `token-type->dtcg-token-type` in `common/src/app/common/types/token.cljc` — this is
every type Penpot emits:

| Penpot internal | `$type` emitted | W3C DTCG type? |
|---|---|---|
| `:color` | `color` | ✅ yes |
| `:dimensions` | `dimension` | ✅ yes |
| `:number` | `number` | ✅ yes |
| `:shadow` | `shadow` | ✅ yes |
| `:typography` | `typography` | ✅ name matches; member names differ (see below) |
| `:border-radius` | `borderRadius` | ❌ Tokens Studio; W3C would use `dimension` |
| `:stroke-width` | `borderWidth` | ❌ Tokens Studio; W3C would use `dimension` |
| `:spacing` | `spacing` | ❌ Tokens Studio; W3C would use `dimension` |
| `:sizing` | `sizing` | ❌ Tokens Studio; W3C would use `dimension` |
| `:font-family` | `fontFamilies` | ❌ W3C singular is `fontFamily` |
| `:font-size` | `fontSizes` | ❌ W3C would use `dimension` |
| `:font-weight` | `fontWeights` | ❌ W3C singular is `fontWeight` |
| `:letter-spacing` | `letterSpacing` | ❌ Tokens Studio |
| `:opacity` | `opacity` | ❌ W3C would use `number` |
| `:rotation` | `rotation` | ❌ Tokens Studio |
| `:text-case` | `textCase` | ❌ Tokens Studio |
| `:text-decoration` | `textDecoration` | ❌ Tokens Studio |
| `:boolean` | `boolean` | ❌ not in W3C |
| `:string` | `string` | ❌ not in W3C |
| `:other` | `other` | ❌ not in W3C |

Practical consequence: build transforms against **this table**, not against the W3C spec.
Tooling written for Tokens Studio (notably `@tokens-studio/sd-transforms` feeding Style
Dictionary) is aimed at exactly this vocabulary; tooling written for strict W3C DTCG is not.

## Import aliases accepted for backwards compatibility

On **import** Penpot also accepts singular forms, mapping them to the same internal types:

- `fontWeight` → font weight
- `fontSize` → font size
- `fontFamily` → font family
- `boxShadow` → shadow

So a hand-written or W3C-flavored file using singular names will partly import. It will not
round-trip: the export comes back in the plural Tokens Studio form.

## Composite tokens

A `typography` token's `$value` is an object whose members use the Tokens Studio names:

```json
{
  "heading": {
    "$value": { "fontFamilies": ["Aboreto"], "fontSizes": "12", "fontWeights": "300" },
    "$type": "typography",
    "$description": ""
  }
}
```

Inside a composite, one extra member type exists that is not valid for a standalone token:
`lineHeights` (accepted as `lineHeight` on import). Note the values are **strings**, not
numbers — `"12"`, not `12`.

## Names, references, and math

**Name rules** (from the validation regex): letters, digits, `_`, `-`, and `$`; must not
start with `$`; dots separate groups and a name must not end with one. `$` has no special
meaning in a name. `button.primary.default.background-color` becomes four nested JSON
levels.

**References (aliases)** are a token name in braces: `{color.blue.500}`. They are **case
sensitive** — a wrong case is a broken reference, not a silent fallback.

**Math** is available on numeric token types: `+`, `-`, `*`, `/`, with references allowed
as operands — `{spacing.small} * 2`, `{spacing.small} * {spacing.scale}`. Expressing a scale
as math keeps it coherent when the base value changes.

## Sets, themes, and precedence

- A **set** is a named collection of tokens. Set names may contain `/` to form groups
  (`Brands/A`).
- Only **active** sets affect shapes and reference resolution.
- When the same token name exists in several active sets, **the later set in
  `tokenSetOrder` wins**. Order is data, not decoration — preserve it through any transform.
- A **theme** is a preset of active sets (`selectedTokenSets`). Activating a theme activates
  its sets but does not deactivate sets activated by other themes.
- Themes can be **grouped**, and at most one theme per group is active at a time. This is
  how independent axes are modeled: a `color-scheme` group (light/dark), a `density` group,
  a `brand` group, each contributing sets.
- Penpot keeps an internal **hidden theme** representing "sets toggled manually, no named
  theme active". Exports deliberately omit it from `$themes` and `activeThemes`, while
  `activeSets` records the effective set list. Do not try to reconstruct it.

## File layouts: single, multifile, ZIP

All three carry identical content; the choice is organizational.

**Single file** — root keys are set names, plus `$themes` and `$metadata`.

**Multifile folder** — one JSON per set, path and filename become the set name, plus
separate `$themes.json` and `$metadata.json`. Individual set files contain **only tokens**.

```
folder/
├── global/
│   ├── colors.json      // set "global/colors"
│   └── dimension.json   // set "global/dimension"
├── $themes.json
└── $metadata.json
```

The top folder name is not part of the set names.

**ZIP** — either shape, zipped. A ZIP with one JSON imports as a single set; a ZIP with a
folder structure follows the multifile rules.

For version control, the multifile layout produces far more readable diffs: a color change
touches one small file instead of one large one.

## Known lossy behavior

- Import **discards unsupported token types** instead of failing. Tokens from a tool with a
  richer type set vanish silently.
- Multi-set import normalizes set names, keeps `tokenSetOrder`, **rejects conflicting token
  path names**, and validates that themes reference sets that exist.
- Single-set import **throws** if it finds no supported tokens at all — a wholly
  incompatible file errors rather than importing empty.
- Tokens are applied to shapes **by name**, not by id. Renaming a token or a token group has
  to update every applied reference; a rename performed outside Penpot (editing the JSON by
  hand and reimporting) can therefore orphan applications.

Diff exports before and after any round-trip through another tool. Silent loss is the
failure mode to design against.
