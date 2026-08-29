# Native vs. JS scroll: the browser-support matrix and code

Verified against `@mdn/browser-compat-data@8.0.13` (published 2026-08-27);
`framer-motion@13.1.1` `dist/es/value/use-scroll.mjs`.

## Contents

- Why this uses MDN's raw data, not caniuse prose
- CSS scroll-driven animations — support table and syntax
- The `@supports` fallback pattern
- Motion's `useScroll` — the native-acceleration finding, in full
- View Transitions API — support table and syntax
- The two APIs are not the same thing

## Why this uses MDN's raw data, not caniuse prose

Every support claim below comes from `@mdn/browser-compat-data`, version
`8.0.13`, published 2026-08-27 — read directly from the package's `data.json`,
not summarized from a browser-support website that could be caching stale
data. The package version number and its publish date are both worth checking
again before trusting this table long after 2026-08-29 — browser support
tables are the single most perishable fact type in this entire skill.

## CSS scroll-driven animations — support table and syntax

```json
"animation-timeline": {
  "chrome": "115",
  "edge": "115",
  "safari": "26",
  "firefox": "preview",   // <- not shipped in stable
  "firefox_android": false
}
```

`"preview"` in MDN's data means behind a flag / in a preview channel only —
**not available in stable Firefox** as of the data checked. This is not
Baseline "Widely available," and treating it as safe to ship alone is the
single most likely native-scroll mistake in this skill.

```css
.reveal {
  animation: fade-in linear;
  animation-timeline: view();
  animation-range: entry 0% cover 40%;
}

@keyframes fade-in {
  from { opacity: 0; }
  to { opacity: 1; }
}
```

`scroll()` ties the timeline to a scrollable ancestor's scroll position;
`view()` ties it to the animated element's own position within the nearest
scroller's viewport — `view()` is the one that reads like a scroll-triggered
reveal, `scroll()` reads like a progress bar.

## The `@supports` fallback pattern

```css
@supports (animation-timeline: view()) {
  .reveal {
    animation: fade-in linear;
    animation-timeline: view();
  }
}

@supports not (animation-timeline: view()) {
  /* either: leave it static, or hand off to a JS fallback below */
}
```

For Firefox specifically, the practical choices are: accept the effect being
absent there (fine for a genuinely decorative reveal), or drive the same
visual effect with GSAP ScrollTrigger behind the `@supports not` branch. Do
not build one effect two different ways as a matter of course — reserve the
JS fallback for effects the design actually depends on.

## Motion's `useScroll` — the native-acceleration finding, in full

This is worth documenting in more depth than the main skill has room for,
because it is genuinely non-obvious and changes when Motion is a legitimate
third option alongside native CSS and GSAP.

Read directly from `framer-motion`'s shipped source
(`dist/es/value/use-scroll.mjs`, 13.1.1):

```js
function canAccelerateScroll(target, offset) {
  if (typeof window === 'undefined') return false
  return target
    ? supportsViewTimeline() && !!offsetToViewTimelineRange(offset)
    : supportsScrollTimeline()
}

function useScroll({ container, target, ...options } = {}) {
  const values = useConstant(createScrollMotionValues)
  if (canAccelerateScroll(target, options.offset)) {
    values.scrollXProgress.accelerate = makeAccelerateConfig(/* ... */)
    values.scrollYProgress.accelerate = makeAccelerateConfig(/* ... */)
  }
  // ...falls back to a plain scroll-event-driven update loop otherwise
}
```

In plain terms: `useScroll()` checks, per-call, whether the browser supports
the native `ScrollTimeline`/`ViewTimeline` primitives that back CSS
scroll-driven animations, and if so, hands the actual scroll-progress tracking
off to the browser's compositor instead of a JS scroll listener. Where it
doesn't (Firefox, per the table above), it transparently falls back to a
`requestAnimationFrame`-driven listener. The calling code (`useTransform`,
etc.) doesn't change either way.

```jsx
import { useScroll, useTransform, motion } from 'motion/react'

function Section() {
  const ref = useRef(null)
  const { scrollYProgress } = useScroll({
    target: ref,
    offset: ['start end', 'end start'],
  })
  const opacity = useTransform(scrollYProgress, [0, 1], [0, 1])

  return <motion.div ref={ref} style={{ opacity }} />
}
```

The practical consequence: a project already using Motion for its entrance
motion (per `motion-principles`) gets native-thread scroll performance on
Chrome/Safari and a correct fallback on Firefox from this one API, with no
manual `@supports` branching. What it does not get is pinning, snapping, or
GSAP's timeline-label-based sequencing — if the effect needs any of those,
this isn't the option, regardless of how convenient the acceleration is.

## View Transitions API — support table and syntax

```json
"Document.startViewTransition": {
  "chrome": "111",
  "edge": "111",
  "safari": "18",
  "firefox": "144"        // shipped 2025-10-14
}
```

Unlike scroll-driven animations, this one is genuinely safe today — Firefox
144 shipped over the current release cycle's worth of versions ago (Firefox
was already past 150 as of the same data pull). All four major engines ship
it in stable.

```js
function navigate(url) {
  if (!document.startViewTransition) {
    location.href = url
    return
  }
  document.startViewTransition(async () => {
    await router.push(url) // or whatever completes the DOM swap
  })
}
```

```css
::view-transition-old(root) {
  animation: fade-out 0.3s ease-out;
}
::view-transition-new(root) {
  animation: fade-in 0.3s ease-in;
}
```

Named transitions (`view-transition-name`) scope the effect to a specific
element (a shared hero image between a list and detail view) rather than the
whole page — worth reaching for once the plain root cross-fade is working,
not before.

**What isn't verified here:** wrapping a specific framework router's
navigation correctly (Next.js App Router, React Router, whichever the project
uses) — the exact point at which the DOM commit needs to land inside the
`startViewTransition` callback is framework- and version-specific. Check the
framework's current docs before writing this as a settled pattern; this
reference stops at the native, framework-agnostic API.

## The two APIs are not the same thing

Worth stating explicitly because the names invite confusion: `scroll-timeline`
/ `view-timeline` (this file's first section) animate an element **as the
user scrolls past it** — no navigation involved. The View Transitions API
animates **the swap between two DOM states**, typically but not exclusively
across a route change. A project can use either without the other, and a
route change animated with `startViewTransition` has nothing to do with
whether that page also has `animation-timeline`-driven reveals once the user
starts scrolling it.
