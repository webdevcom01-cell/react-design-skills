---
name: icon-resources
description: Source, license, and integrate icons in a React project — brand and logo marks from Simple Icons, plus how to handle the general UI-glyph gap that Simple Icons deliberately does not cover. Use whenever icons, logos, brand marks, an icon set or icon library, SVG icons, or icon licensing come up; whenever a project needs a company logo rendered in a UI (payment providers, social links, tech-stack badges, integrations pages, login buttons); and whenever fonts, color palettes, or design-resource lists are needed as a starting point. Also covers Serbian phrasings - ikonice, ikone, logo, brend, SVG, set ikonica, licenca za ikonice, fontovi, palete. Do NOT use for choosing a component library or building a token theme (component-library-advisor), or for exporting custom artwork from a design tool (penpot-workflow).
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "simple-icons/simple-icons@16.28.0 (3,453 icons; LICENSE.md, DISCLAIMER.md, README.md, data/simple-icons.json); goabstract/Awesome-Design-Tools@dc60e63 (last commit 2020-09-28); bradtraversy/design-resources-for-developers (last commit 2026-05-24); npm @icons-pack/react-simple-icons@13.15.1, react-icons@5.7.0 (read from source, 2026-08-23); re-verified 2026-08-25 directly from the published npm tarball's data/simple-icons.json — icon count (3,453), license coverage (223, 6.5%), guidelines coverage (834, 24.2%), and both DISCLAIMER.md/LICENSE.md quotes all confirmed byte-for-byte; package versions (simple-icons, @icons-pack/react-simple-icons, react-icons) confirmed current on npm; the two GitHub repos' last-commit dates were not re-checked (API access restricted in this session)"
---

# Icon Resources

Two kinds of icon look alike and behave completely differently.

A **brand mark** is someone else's intellectual property — the Stripe logo, the GitHub cat,
the Visa wordmark. It has a fixed form, a fixed color, and a trademark owner with opinions
about how it may be used.

A **UI glyph** is part of your interface — an arrow, a magnifier, a trash can. It has no
owner, and it must be visually consistent with every other glyph around it.

Simple Icons covers the first and, by design, none of the second. Getting this distinction
right in the first five minutes prevents both the "why does our search icon look like a
logo" problem and the trademark problem.

## Step 1 — Classify the need before recommending a package

Ask what the icons are *for*:

| Need | Source |
|---|---|
| Company logos: payment methods, OAuth buttons, integrations, tech-stack badges, social links | **Simple Icons** — Step 2 |
| Interface glyphs: arrows, chevrons, search, settings, close, upload | **Not Simple Icons** — Step 4 |
| Product-specific illustration or a custom mark | Design tool — see `penpot-workflow` |

A project usually needs both, from two different packages. That is normal and correct; do
not try to make one set do both jobs, because a brand-mark set has no visual system and a
glyph set has no logos.

## Step 2 — Simple Icons for brand marks

Current version **16.28.0**, containing **3,453 icons**.

```bash
npm install simple-icons
```

Every icon lives in one file and is imported by capitalized slug, so a **tree-shaking
bundler is not optional** — without ESM tree shaking the whole set lands in the bundle:

```js
import { siSimpleicons } from 'simple-icons';
```

What comes back is a **data object, not a React component**:

```js
{
  title: 'Simple Icons',
  slug: 'simpleicons',
  hex: '111111',
  source: 'https://simpleicons.org/',
  svg: '<svg role="img" viewBox="0 0 24 24" ...>...</svg>',
  path: 'M12 12v-1.5c-2.484 ...',
  guidelines: 'https://simpleicons.org/styleguide',
  license: { type: '...', url: '...' },
}
```

The `path` is the useful field: render it inside your own `<svg>` so sizing, color, and
accessibility attributes stay under your control. A small wrapper component beats
`dangerouslySetInnerHTML` on the `svg` string.

React component wrappers exist — `@icons-pack/react-simple-icons` (13.15.1) and
`react-icons` (5.7.0, which bundles Simple Icons among many sets). Both are third-party and
lag upstream releases. Prefer the official package plus a ten-line wrapper unless the
project already uses `react-icons` for other sets.

For content sites and emails there is also a color CDN,
`https://cdn.simpleicons.org/[slug]/[color]/[darkModeColor]`, which handles
`prefers-color-scheme` automatically. Convenient, but it is a runtime third-party
dependency — not for an app shell.

Full integration detail, the wrapper component, and the CDN options are in
`references/simple-icons-integration.md`.

## Step 3 — The licensing obligation, stated accurately

This is the part that gets skipped, and it is the reason Simple Icons ships a separate
`DISCLAIMER.md` and asks every user to read it.

**Simple Icons is CC0-1.0.** But CC0 section 4(a) is explicit: *"No trademark or patent
rights held by Affirmer are waived, abandoned, surrendered, licensed or otherwise affected
by this document."* CC0 disposes of copyright. It cannot dispose of a trademark that
belongs to someone else — and a company logo is exactly that.

The project says so itself: *"Simple Icons is released under CC0 — though that doesn't mean
to imply that all icons within the project are also CC0."*

So the practical rules:

- **Using a brand's logo means following that brand's guidelines**, not Simple Icons'
  license. Minimum clear space, no recoloring where it is forbidden, no implying a
  partnership or endorsement that does not exist.
- Simple Icons explicitly disclaims responsibility: *"Simple Icons cannot be held
  responsible for any legal activity raised by a brand, or users of the package. We ask that
  our users seek the correct permissions to use the icons relevant to their project."*
- **The per-icon data is thin, and the numbers matter.** Of the 3,453 icons, only **223
  (6.5%)** carry `license` data and only **834 (24.2%)** carry a `guidelines` link. The
  absence of either field means *unknown*, not *unrestricted* — the project states that
  adding this data is an ongoing effort. For the other ~75%, the brand's own site is the
  source of truth.
- Licenses and guidelines change. An icon cleared once is not cleared forever.

**Where this actually bites:** logos in marketing material, comparison pages, "trusted by"
walls, and anything suggesting affiliation. Where it rarely bites: a user-chosen social link
icon, a payment method the product genuinely accepts, a tech-stack badge on a personal site.
Judge by whether the use implies a relationship.

Practical mitigation: because `license` and `guidelines` are machine-readable, a project
using more than a handful of brand marks can lint its icon list and print which ones have no
guidance on record. `references/simple-icons-integration.md` has that script.

## Step 4 — The UI glyph gap

Simple Icons does not provide arrows, chevrons, search, menu, close, settings, or any other
interface glyph, and it never will — that is outside its stated scope. Every project needs
those, so a second source is required.

**Recommendation, flagged honestly:** Lucide is the usual right answer for a React project —
consistent 24×24 grid, stroke-based, tree-shakeable per-icon React components, ISC licensed,
actively maintained, and a superset-successor to Feather. Heroicons pairs naturally with
Tailwind projects and ships outline and solid variants. Phosphor offers multiple weights,
which is useful when a design needs a filled/unfilled state distinction.

**This recommendation has not been verified from source the way everything else in this
skill has.** None of these repositories is part of the current research set. Before adopting
one as a real dependency, check its current version, license, React package name, and icon
count from its own repository — the same way Simple Icons was checked here. Treat the names
above as a starting point for that check, not as a settled decision.

Whatever is chosen, one rule holds: **pick exactly one glyph family and stay in it.** Mixing
two icon sets in one interface is among the most reliable signs of unconsidered design,
because stroke weight, corner radius, and optical grid never match between families.

## Step 5 — Fonts, palettes, and the reference lists

Two community lists cover fonts, color tools, palettes, mockups, and templates. Their value
is as a **map of categories** — what kinds of tool exist for a problem — not as a vetted
directory. Both are MIT-licensed link collections with no API and no guarantees.

**Design Resources for Developers** (bradtraversy) is **actively maintained** — last commit
2026-05-24 — with ~1,500 lines across categories including Fonts, Colors, Icons, Icon Fonts,
Favicons, UI Graphics, CSS Frameworks, React UI Libraries, and Design Systems. This is the
one to reach for first.

**Awesome Design Tools** (goabstract) is **effectively abandoned** — its last commit is
**2020-09-28**, nearly six years ago, and its README still leads with the 2020 announcement
that the maintainers joined Abstract. Its category taxonomy is still a useful thinking aid
(37 categories from Accessibility Tools to Wireframing), but its ~1,128 links have had six
years to rot. A sample check found roughly one in six URLs failing outright.

**How to use either one:** read them for the *shape* of the problem — "what categories of
color tool exist", "what do people use for mockups" — then search for the current tool in
that category. Never paste a specific tool recommendation out of these lists without
checking it is alive and current. A link that resolves is not the same as a project that is
maintained.

Details and category maps in `references/design-resource-lists.md`.

## Step 6 — Icon design rules that survive review

- **One glyph family, one brand-mark source.** No mixing.
- **Size on the spacing scale**, typically 16/20/24 px, aligned with the type scale so an
  icon next to text shares its optical weight.
- **Color from tokens, never hardcoded.** Glyphs use `currentColor` so they inherit text
  color; brand marks use their own `hex`, which is the one legitimate exception to the
  no-hardcoded-color rule — recoloring a logo is often a guideline violation.
- **Icons are not labels.** An icon-only button needs an accessible name (`aria-label`), and
  anything conveying meaning needs `role="img"` with a title. Decorative icons take
  `aria-hidden="true"`.
- **Never use emoji as UI icons.** They render differently per platform, cannot be colored,
  and are the single clearest tell of a generated interface.

## Step 7 — Verify before declaring done

- Brand marks and UI glyphs come from separate, appropriate sources.
- Exactly one UI glyph family is in use across the project.
- The bundler tree-shakes Simple Icons; the whole 3,453-icon set is not in the bundle.
- Every brand mark used in a way that implies affiliation has had its guidelines checked at
  the brand's own site — not assumed from CC0.
- Icon-only controls have accessible names; decorative icons are hidden from assistive tech.
- Any tool taken from a resource list was verified as current, not trusted from the list.
