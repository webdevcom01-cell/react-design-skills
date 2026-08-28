# Simple Icons integration

Verified against `simple-icons/simple-icons@16.28.0` — `README.md`, `LICENSE.md`,
`DISCLAIMER.md`, `data/simple-icons.json`, `package.json` exports; npm
`@icons-pack/react-simple-icons@13.15.1`, `react-icons@5.7.0`.

## Contents
- What is in the package
- npm usage and tree shaking
- A React wrapper component
- Package exports
- CDN options
- React wrapper packages
- Auditing licenses and guidelines
- The legal position, in full

## What is in the package

Version 16.28.0 ships **3,453 brand icons**. Each entry in `data/simple-icons.json` carries
at minimum `title`, `hex`, `source`, and often `aliases`; the built package objects add
`slug`, `svg`, `path`, and — where known — `guidelines` and `license`.

Coverage of the optional legal fields, counted from the data file:

| Field | Icons with it | Share |
|---|---|---|
| `license` | 223 | 6.5% |
| `guidelines` | 834 | 24.2% |

Both are described by the project as ongoing efforts. **Absence means unknown, not
unrestricted.**

## npm usage and tree shaking

```bash
npm install simple-icons
```

```js
// import { si[ICON SLUG] } from 'simple-icons'
import { siSimpleicons } from 'simple-icons';   // ESM — tree shakeable
const { siSimpleicons } = require('simple-icons'); // CJS — not tree shakeable
```

The README is explicit: all icons are imported from a single file, and *"We highly recommend
using a bundler that can tree shake"*. With ESM and a modern bundler (Vite, webpack 5,
Rollup, Next.js) only the imported icons survive. With CJS, or with a dynamic lookup like
`icons['si' + name]`, the whole set ships — over 16 MB unpacked.

If icon names must be dynamic, do not index into the barrel. Build an explicit map of the
icons the project actually uses:

```ts
import { siStripe, siPaypal, siVisa } from 'simple-icons';

const PAYMENT_ICONS = { stripe: siStripe, paypal: siPaypal, visa: siVisa } as const;
```

This keeps tree shaking intact and makes the set of used brands greppable — which is what
the licence audit below depends on.

## A React wrapper component

The package returns data, not components. Render `path` inside your own SVG so size, color,
and accessibility stay under your control:

```tsx
import type { SimpleIcon } from 'simple-icons';

interface BrandIconProps {
  icon: SimpleIcon;
  size?: number;
  /** Brand marks usually keep their own color; guidelines often require it. */
  useBrandColor?: boolean;
  /** Omit for decorative use; the icon is then hidden from assistive tech. */
  label?: string;
}

export function BrandIcon({ icon, size = 24, useBrandColor = true, label }: BrandIconProps) {
  return (
    <svg
      role={label ? 'img' : undefined}
      aria-label={label}
      aria-hidden={label ? undefined : true}
      width={size}
      height={size}
      viewBox="0 0 24 24"
      fill={useBrandColor ? `#${icon.hex}` : 'currentColor'}
      xmlns="http://www.w3.org/2000/svg"
    >
      {label ? <title>{icon.title}</title> : null}
      <path d={icon.path} />
    </svg>
  );
}
```

Why not `dangerouslySetInnerHTML` with `icon.svg`: the bundled SVG string carries its own
attributes, so overriding size, color, and ARIA means string manipulation, and the escape
hatch invites injecting untrusted strings later.

All icons use a 24×24 viewBox, so they align with a 24 px glyph set without adjustment.

## Package exports

From `package.json`:

| Specifier | Contents |
|---|---|
| `simple-icons` | Full barrel — icon objects by `si*` name |
| `simple-icons/icons` | Icons only |
| `simple-icons/icons/*` | Individual icon files |
| `simple-icons/sdk` | SDK helpers (`sdk.mjs`, typed by `sdk.d.ts`) |

Types ship with the package (`types.d.ts`, `data/simple-icons.d.ts`); no `@types` package is
needed.

## CDN options

Raw SVG, pinned to a major version:

```html
<img height="32" width="32" src="https://cdn.jsdelivr.net/npm/simple-icons@v16/icons/[SLUG].svg" />
<img height="32" width="32" src="https://unpkg.com/simple-icons@v16/icons/[SLUG].svg" />
```

Colored, with dark-mode support, from the project's own CDN:

```html
<img src="https://cdn.simpleicons.org/[SLUG]" />
<img src="https://cdn.simpleicons.org/[SLUG]/[COLOR]" />
<img src="https://cdn.simpleicons.org/[SLUG]/[COLOR]/[DARK_MODE_COLOR]" />
```

`[COLOR]` accepts hex values or CSS color keywords and defaults to the brand color. When a
dark-mode color is supplied, the CDN serves it under `prefers-color-scheme: dark`.

Pinning `@v16` means no updates after the next major. Using `@latest` gets updates
indefinitely but **404s if an icon is removed** — and removals do happen, at major releases,
typically twice a year. For anything load-bearing, install the package.

## React wrapper packages

| Package | Version (checked 2026-08-23) | Published | Note |
|---|---|---|---|
| `@icons-pack/react-simple-icons` | 13.15.1 | 2026-08-16 | Simple Icons as React components |
| `react-icons` | 5.7.0 | 2026-06-30 | Many icon sets including Simple Icons |

Both release often. Read these from `https://registry.npmjs.org/<pkg>` rather than from a
local `npm view` when the number matters — a proxied or cached registry can be months
behind, which looks like a disagreement about facts rather than a stale mirror.

Both are third-party and track upstream on their own schedule, so a brand added to Simple
Icons this week may not be in either yet. `react-icons` is a reasonable choice when the
project already uses it for other sets; otherwise the official package plus the wrapper
above avoids a dependency that can fall behind.

**Do not use `simple-icons-react`.** The name is taken but the package is gone. Its
registry record shows it was created 2022-01-02, published four patch versions (0.1.0
through 0.1.3) within about three hours, and was fully unpublished the same day at 18:45
UTC — the tombstone lists `0.1.2` and `0.1.3` as the versions removed in that final
unpublish. Today the record has no `versions` and no `dist-tags`, so `npm install
simple-icons-react` fails with E404. It is an abandoned name, not an alternative.

## Auditing licenses and guidelines

Because the data is machine-readable, a project can check its own brand-mark usage:

```js
// scripts/audit-brand-icons.mjs
import * as icons from 'simple-icons';

const USED = ['siStripe', 'siPaypal', 'siVisa', 'siGithub'];

for (const name of USED) {
  const icon = icons[name];
  if (!icon) { console.error(`MISSING  ${name}`); continue; }
  const guidelines = icon.guidelines ?? '(none on record)';
  const license = icon.license?.type ?? '(none on record)';
  console.log(`${icon.title.padEnd(20)} license=${license.padEnd(16)} guidelines=${guidelines}`);
}
```

Run it when the icon list changes. It will not tell you whether a use is permitted — only
which brands have guidance on record and which need a visit to the brand's own site. Given
that 75% of icons have no guidelines link, expect the second category to dominate.

## The legal position, in full

`LICENSE.md` is CC0 1.0 Universal. Section 4(a):

> No trademark or patent rights held by Affirmer are waived, abandoned, surrendered,
> licensed or otherwise affected by this document.

CC0 waives copyright. Trademarks belong to the brands, and CC0 cannot and does not touch
them.

`DISCLAIMER.md`, which the README asks every user to read, states:

> Simple Icons is released under CC0 - though that doesn't mean to imply that all icons
> within the project are also CC0. Please see individual licenses where available.

> Simple Icons cannot be held responsible for any legal activity raised by a brand, or users
> of the package. We ask that our users seek the correct permissions to use the icons
> relevant to their project.

> Simple Icons provides a link to a brand's branding guidelines (or similar) if the brand
> provides one. We ask our users read these guidelines and ensure their usage of the brand's
> icon is in accordance with them. As guidelines are subject to change we also ask our users
> to regularly check if the brand guidelines of the icons they use have been updated.

The disclaimer also notes that where an icon includes a registered trademark (`®`) or
trademark (`™`) symbol, inclusion follows the project's contributing guidelines, and that
brands may request updates or removal — removals generally happen at major releases.

The honest summary for a client: *the icon files are free to copy; using a company's logo is
governed by that company's rules, and those rules are the project's responsibility, not
Simple Icons'.*
