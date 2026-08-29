# Text reveal patterns

Verified against `gsap@3.15.0` `SplitText.js`; own Vite+React+TS reference
build, browser-tested 2026-08-29 (desktop 1440×900 and mobile 375×812
viewports, both light and dark `prefers-color-scheme`).

## Contents

- The DOM shape `SplitText` actually produces
- The line-height prerequisite, in full
- The color prerequisite, in full
- Character, word, and line reveals
- Mixing reveal granularity in one heading
- Reduced motion for a text reveal specifically

Every `SplitText.create()` call below assumes
`gsap.registerPlugin(SplitText, useGSAP)` has already run — see the main
`SKILL.md` Step 3 for why this is required, not optional, and omitted here
for brevity.

## The DOM shape `SplitText` actually produces

Don't target `.char`/`.word`/`.line` CSS classes — the current version
doesn't use them. `SplitText.create()` returns an object whose `chars`,
`words`, and `lines` properties are live arrays of the actual wrapper
elements it created; animate those directly:

```jsx
const split = SplitText.create('.hero-title', { type: 'chars,words' })
// split.chars -> [div, div, div, ...] — one per character
// split.words -> [div, div, ...] — one per word
gsap.from(split.chars, { /* ... */ })
```

With `mask: 'chars'` (or `'words'`/`'lines'`), each unit gets wrapped in an
additional `overflow: clip` container sized to the line box, and the
animated element sits inside that mask — confirmed by inspecting the
rendered output: nested `<div>`s, `aria-hidden="true"` on the generated
wrappers (the original text remains available to assistive tech via the
source element, or restore it — `split.revert()` — for screen-reader-first
contexts if the visual effect isn't essential to the content's meaning).

## The line-height prerequisite, in full

This is the more subtle of the two verified defects, because the mechanism
is a genuine CSS specificity/inheritance behavior, not a bug in any library.

**The mechanism:** a `<percentage>` value for `line-height` computes to a
*fixed length* at the element it's declared on, using that element's own
font-size — and it is the fixed, already-computed length that inherits to
descendants, not the percentage itself. So:

```css
:root {
  font: 18px/145%; /* shorthand for font-size: 18px; line-height: 145% */
}
```

computes `line-height` to `18px × 1.45 ≈ 26.1px` **at `:root`**, and every
descendant inherits that literal `26.1px` — including a heading at
`font-size: 5rem` (90px) that never set its own `line-height`. The heading's
much larger font renders inside a line box sized for an 18px font.

**Why this specifically breaks `SplitText`'s `mask` option:** the mask
wrapper GSAP generates around each character is sized to the element's line
box. A line box that's a fraction of the actual glyph height clips every
character at the top and bottom — visibly, consistently, and with zero
signal in the DOM. Verified directly: `getBoundingClientRect()` on the
generated mask wrapper and its inner character both reported the exact
26.1px height in a case where the visible text was clipped, while
`opacity`/`transform` on the animated character were both already at their
correct, fully-settled resting values. The animation is not the thing that's
broken.

**The fix, and why it's not optional:**

```jsx
<h1 className="hero-title" style={{ fontSize: '5rem', lineHeight: 1.1 }}>
```

Set an explicit, unitless `line-height` (unitless is relative to the
element's *own* font-size at every level, so it doesn't have this
inheritance trap) on any element that will be `SplitText`-masked. Do this
even when the boilerplate's default line-height "looks fine" for normal
paragraph text elsewhere on the page — the trap only becomes visible at
sizes and line-heights different enough from the boilerplate's own
assumptions, which a hero heading almost always is.

This isn't specific to Vite's starter CSS — any global reset or design
system that sets a percentage or shorthand `line-height` at a root/body
level has the same inheritance behavior. Treat "what line-height did this
element actually inherit" as a thing to check with DevTools computed styles,
not assume from reading the source CSS.

## The color prerequisite, in full

Verified independently, on the same reference build, and easy to miss for a
different reason: it can look completely correct in one testing context and
be completely broken in another, with identical code.

**The mechanism:** a common boilerplate pattern sets heading color from a
theme variable that flips with the OS color scheme:

```css
:root { --text-h: #08060d; } /* near-black, light mode */
@media (prefers-color-scheme: dark) {
  :root { --text-h: #f3f4f6; } /* near-white, dark mode */
}
h1, h2 { color: var(--text-h); }
```

A hero heading that never sets its own `color` inherits nothing from a
parent's inline `color: white` — the `h1, h2 { color: var(--text-h) }` rule
targets the element directly and wins over any inherited value regardless of
how that inherited value was set. If the developer's browser or OS happens
to be in dark mode while building the page, `--text-h` resolves to a
near-white value, the heading looks correctly white on a dark hero
background, and the missing explicit color is invisible. Switch to light
mode — or hand the identical code to any user whose system is in light
mode — and `--text-h` resolves to near-black, rendering the same markup as
near-invisible text on the same dark background.

**Verified directly, as a controlled experiment:** identical code, identical
viewport, only the emulated color scheme changed from light to dark —
`getComputedStyle` on the heading and its `SplitText`-generated characters
reported `color: rgb(8, 6, 13)` (matching `--text-h`'s light-mode value
exactly) in the broken state, and the text was visually unreadable against
the black hero background. No animation property was involved in either
state — `opacity: 1` and a fully-settled `transform` in both.

**The fix:** set `color` explicitly on any element being `SplitText`-masked,
the same non-negotiable rule as `line-height`:

```jsx
<h1 className="hero-title" style={{ fontSize: '5rem', lineHeight: 1.1, color: '#fff' }}>
```

**The general lesson, stated once:** a hero/showcase heading is exactly the
kind of element a boilerplate's generic heading styles were never designed
for — it's usually far larger, often on an unusual background, and often the
one piece of text where "it happened to render correctly during dev" is not
evidence of correctness. Test masked/reveal text in both color schemes if
the project supports both, and prefer setting typography properties
explicitly on showcase headings over trusting inherited defaults, full stop.

## Character, word, and line reveals

```jsx
// Character-level — most granular, most expensive at scale
SplitText.create('.hero-title', { type: 'chars', mask: 'chars' })

// Word-level — cheaper, reads as more editorial/restrained
SplitText.create('.hero-subtitle', { type: 'words', mask: 'words' })

// Line-level — for paragraph-length copy, not headlines
SplitText.create('.hero-body', { type: 'lines', mask: 'lines' })
```

Character-level staggers get expensive fast — a 40-character headline is 40
tween targets. Keep `stagger` tight (0.03–0.05, per the main skill's Step 5)
and consider word- or line-level splitting for anything longer than a short
headline; the visual "typewriter reveal" read most people associate with
`SplitText` comes from character count and stagger tightness together, not
from splitting by character being inherently correct for every case.

## Mixing reveal granularity in one heading

A common, effective pattern: split by both `chars` and `words` in one call
(`type: 'chars,words'`), stagger the character-level animation, but use the
word-level array for layout-related logic (e.g. detecting line breaks) —
`SplitText` returns both arrays from a single split, no need for two
separate `SplitText.create()` calls on the same element.

## Reduced motion for a text reveal specifically

Per the main skill's Step 7: keep the content visible, remove the movement.
For a character-level reveal specifically, that means the `reduceMotion`
branch of `gsap.matchMedia()` should still run `gsap.set()` (not `gsap.from`)
to place every character at its resting `opacity: 1` state instantly, rather
than skipping the `SplitText.create()` call altogether — skipping it removes
the mask wrappers entirely, which is fine functionally but means the
reduced-motion path and the full-motion path render subtly different DOM,
worth being a deliberate choice rather than an accident of how the branch
was written.
