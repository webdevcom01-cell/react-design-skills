# Design resource lists: status and how to use them

Verified 2026-08-23 by cloning both repositories and reading commit history and structure.

## Contents
- Status summary
- Design Resources for Developers
- Awesome Design Tools
- Link-rot sample
- How to use a list like this
- What these are not

## Status summary

| List | Last commit | Size | License | Verdict |
|---|---|---|---|---|
| bradtraversy/design-resources-for-developers | **2026-05-24** | ~1,526 lines, 33 categories | MIT | Actively maintained — use first |
| goabstract/Awesome-Design-Tools | **2020-09-28** | ~1,236 lines README + 3 side lists, 37 categories, ~1,128 links | MIT | Abandoned — taxonomy only |

Both are plain Markdown link collections. Neither has an API, a schema, or any vetting
process beyond pull-request review.

## Design Resources for Developers

Last commit 2026-05-24 (merge of PR #1588), so it is receiving contributions.

Categories, in file order: UI Graphics, Fonts, Colors, Icons, Logos, Favicons, Icon Fonts,
Stock Photos, Stock Videos, Stock Music & Sound Effects, Vectors & Clip Art, Product & Image
Mockups, HTML & CSS Templates, CSS Frameworks, CSS Methodologies, CSS Animations, Javascript
Animation Libraries, Javascript Chart Libraries, UI Components & Kits, React UI Libraries,
Vue UI Libraries, Angular UI Libraries, Svelte UI Libraries, React Native UI Libraries,
Design Systems & Style Guides, Online Design Tools, Downloadable Design Software, Design
Inspiration, Image Compression, Chrome Extensions, Firefox Extensions, AI Graphic Design
Tools, Others.

Most useful sections for this skill set: **Fonts**, **Colors**, **Icons**, **Icon Fonts**,
**Favicons**, **Design Systems & Style Guides**.

Being actively maintained does not make it curated — inclusion is by pull request, so
presence signals that someone submitted a link, not that anyone assessed the tool.

## Awesome Design Tools

Last commit `dc60e63c` dated **2020-09-28**. The README still opens with the announcement
that the maintainers (Flawless App) joined Abstract, which dates it precisely. Three
companion files exist and are equally stale: `Awesome-Design-Plugins.md` (~1,556 lines),
`Awesome-Design-UI-Kits.md` (~644), `Awesome-Design-Conferences.md` (~226).

Its 37 categories remain a genuinely good taxonomy of the problem space: Accessibility
Tools, Animation Tools, Augmented Reality, Collaboration Tools, Color Picker Tools, Design
Feedback Tools, Design Handoff Tools, Design Inspiration, Design System Tools, Design to
Code Tools, Design Version Control, Development Tools, Experience Monitoring, Font Tools,
Gradient Tools, Icons Tools, Illustrations, Information Architecture, Logo Design, Mockup
Tools, No Code Tools, Pixel Art Tools, Prototyping Tools, Screenshot Software, Sketching
Tools, SMM Design Tools, Sound Design, Stock Photos Tools, Stock Videos, Tools for Learning
Design, UI Design Tools, User Flow Tools, User Research Tools, Visual Debugging Tools,
Wireframing Tools, 3D Modeling Software.

Use the category list to ask "what kind of tool solves this?" Do not use the entries to
answer "which tool?" — six years of SaaS churn sits between the file and today.

## Link-rot sample

A spot check of sampled non-GitHub links (roughly 17–18 per list, following redirects, with
a browser user agent):

| List | Alive (200) | Failed |
|---|---|---|
| Awesome Design Tools | 11 | 6 |
| Design Resources for Developers | 16 | 2 |

Caveat on method: several failures were `403`, which usually means bot-blocking rather than
a dead site. Hard connection failures (`000` — domain no longer resolving or serving) were 3
of 17 for Awesome Design Tools and 1 of 18 for Design Resources for Developers.

The sample is small and only indicative. The direction is what matters and it matches the
commit dates: the unmaintained list has meaningfully more rot.

## How to use a list like this

Treat them as **methodology, not inventory**:

1. Find the *category* that matches the problem — "gradient tools", "icon fonts", "design
   system tools".
2. Read two or three entries to learn what that category's tools actually do and what
   vocabulary they use.
3. **Search for what is current in that category**, using that vocabulary.
4. Verify the candidate directly: its repository or site, its license, its last release, its
   pricing.

Step 4 is not optional and is where the value is. A tool named in a list may be dead,
acquired, relicensed, or turned into a paid product since it was added. Recommending it
without checking is how a client ends up on a service that shut down.

For fonts specifically, prefer the canonical sources over any list: Google Fonts for
open-licensed web fonts, Fontsource for self-hosting them as npm packages, and the foundry
directly for anything commercial. Font licensing has real teeth — web-embedding rights are
separate from desktop rights, and a list will not tell you which one a project has.

For color, the same: a palette generator is a starting point, but the token layer defined in
`component-library-advisor` and `penpot-workflow` is where color decisions actually live.

## What these are not

- Not APIs. There is nothing to query programmatically; they are Markdown.
- Not curated or vetted. Inclusion means a pull request was merged.
- Not stable references. Both are MIT-licensed and can be forked, which is the only way to
  freeze a snapshot.
- Not a substitute for checking a tool's own source, which is the standard applied to every
  other dependency in this skill set.
