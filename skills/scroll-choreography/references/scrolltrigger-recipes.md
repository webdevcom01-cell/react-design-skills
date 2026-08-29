# ScrollTrigger recipes

Verified against `gsap@3.15.0` — `ScrollTrigger.js`, `ScrollSmoother.js`,
`Observer.js`; `@gsap/react@2.1.2` `src/index.js`; `gsap.com/showcase`
curated gallery (60-entry sample, 2026-08-29).

## Contents

- `useGSAP` cleanup, verified
- Vertical reveal (the common case)
- Horizontal pin + scrub
- Snap
- The two official "mistakes," in full
- Observer recipes
- The GSAP showcase sample, full breakdown

## `useGSAP` cleanup, verified

Every recipe below assumes `useGSAP` from `@gsap/react`, not a hand-rolled
`useEffect`. Read directly from `@gsap/react`'s own source (`src/index.js`,
2.1.2): the hook creates a `gsap.context()` scoped to the component (or an
explicit `scope` ref), adds the callback to it, and on cleanup calls
`context.revert()` — which kills every tween, timeline, and ScrollTrigger
created inside that context, automatically. There is no manual
`ScrollTrigger.kill()` bookkeeping to do.

```jsx
import { useGSAP } from '@gsap/react'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'

gsap.registerPlugin(ScrollTrigger, useGSAP)

function Section() {
  const container = useRef<HTMLDivElement>(null)

  useGSAP(
    () => {
      // any gsap.to/from/timeline + ScrollTrigger created here
      // is automatically reverted on unmount
    },
    { scope: container },
  )

  return <section ref={container}>...</section>
}
```

`@gsap/react` ships no `"use client"` directive of its own (confirmed: zero
matches in both `src/index.js` and `dist/index.js`) — the component using
`useGSAP` must add it itself in a Next.js App Router project, the same rule as
Motion's provider pattern in `motion-principles`.

## Vertical reveal — the common case

```jsx
useGSAP(
  () => {
    gsap.from('.reveal-target', {
      opacity: 0,
      y: 40,
      duration: 0.6,
      scrollTrigger: {
        trigger: container.current,
        start: 'top 80%', // fires when the trigger's top hits 80% down the viewport
      },
    })
  },
  { scope: container },
)
```

No `pin`, no `scrub` — this is a one-shot entrance that plays once when the
element enters the viewport. This is the effect most "reveal on scroll"
requests actually want; reach for pin/scrub only when the animation's
*progress* should track scroll position, not just its *start*.

## Horizontal pin + scrub

```jsx
useGSAP(
  () => {
    const panels = gsap.utils.toArray<HTMLElement>('.panel')

    gsap.to(panels, {
      xPercent: -100 * (panels.length - 1),
      ease: 'none', // scrub-driven tweens should not have their own easing —
                     // the scroll input IS the easing curve
      scrollTrigger: {
        trigger: container.current,
        pin: true,
        scrub: 1,
        end: () => '+=' + (container.current!.offsetWidth * (panels.length - 1)),
        invalidateOnRefresh: true,
      },
    })
  },
  { scope: container },
)
```

- `pin: true` sets the trigger element to `position: fixed` internally and
  inserts a spacer div (`.pin-spacer`, visible in DevTools) so the document
  height accounts for the pinned duration. `pinSpacing` (default `true`)
  controls whether that spacer is added — confirmed a real option in the
  shipped source, along with `anticipatePin`, which starts the pin transition
  slightly early to smooth out fast-scroll jank on the pin boundary.
- `end` as a function (not a fixed string) recalculates against
  `container.current!.offsetWidth` on every refresh — necessary because a
  fixed pixel `end` value goes stale the moment the viewport resizes.
- `ease: 'none'` on a scrubbed tween is the rule, not a stylistic choice: with
  `scrub` set, scroll position *is* the timeline's playhead, so an eased tween
  would apply easing twice — once from scroll, once from the tween's own
  curve — and read as sluggish or inconsistent.

## Snap

```js
scrollTrigger: {
  // ...pin/scrub as above
  snap: {
    snapTo: 1 / (panels.length - 1),
    duration: 0.4,
    ease: 'power1.inOut',
  },
}
```

`snap` (confirmed a real ScrollTrigger option) settles the scrubbed progress
to the nearest panel boundary once the user stops scrolling — this is what
makes a horizontal panel sequence feel like discrete slides rather than a
scrub the user has to stop at exactly the right spot themselves.

## The two official "mistakes," in full

Both confirmed against GSAP's own `gsap.com/resources/st-mistakes/` page, not
community threads.

**Lazy-loaded / dynamically-sized content shifts trigger positions.**
ScrollTrigger measures every trigger's position once, on load (and on
`refresh()`). An image without a set `width`/`height` attribute, or content
that arrives after that measurement (fetch, AJAX, a CMS-driven block), changes
the document's layout — and every trigger *below* the shifted content is now
measuring against stale positions.

```jsx
<img
  src={src}
  alt=""
  onLoad={() => ScrollTrigger.refresh()}
/>
```

Set `width`/`height` (or `aspect-ratio` in CSS) wherever possible to avoid the
shift entirely — `refresh()` is the fix for when the shift is unavoidable
(a genuinely dynamic image, a CMS block of unknown height), not a substitute
for reserving layout space.

**`scroll-behavior: smooth` in CSS conflicts with ScrollTrigger's own
positioning.** If a global reset or framework default sets it on `html` or
the scrolling element, the browser's native smooth-scroll and ScrollTrigger's
own measurement of scroll position can disagree during refresh, producing
triggers that fire at the wrong point. Override it explicitly:

```css
html {
  scroll-behavior: auto !important;
}
```

This is unrelated to Lenis/ScrollSmoother — it applies even on a page with no
smooth-scroll library at all, purely from CSS-level native smooth scrolling
(e.g., a plain anchor-link `scroll-behavior: smooth` reset) fighting
ScrollTrigger.

## Observer recipes

`Observer` is an input normalizer, not a scroll-position tool — see the main
`SKILL.md` Step 5 for when to reach for it over `ScrollTrigger`. Confirmed
real options from the shipped source: `wheelSpeed`, `preventDefault`,
`tolerance`, `dragMinimum`.

```jsx
useGSAP(() => {
  Observer.create({
    target: window,
    type: 'wheel,touch',
    wheelSpeed: -1,
    tolerance: 10, // minimum delta before an event counts as intentional input
    preventDefault: true,
    onUp: () => goToSlide(current + 1),
    onDown: () => goToSlide(current - 1),
  })
}, { scope: container })
```

`tolerance` matters more than it looks — without it, a fullpage-slide pattern
built on `Observer` fires on trackpad micro-scroll noise, advancing multiple
slides from what the user experienced as one scroll gesture.

## The GSAP showcase sample, full breakdown

Sampled 2026-08-29 from `gsap.com/showcase`'s curated gallery — 60 consecutive
entries, each tagged by the GSAP team with the plugins actually used. Counts
below are entries where a tag was **visibly listed**; the gallery truncates
longer tag lists behind a `+N`, so every number here is a floor, not an exact
usage rate.

| Plugin | Entries tagged (of 60) | Floor % |
|---|---|---|
| ScrollTrigger | 50 | ~83% |
| SplitText | 37 | ~62% |
| Observer | 17 | ~28% |
| ScrollSmoother | 7 | ~12% |

Read this as validation of scope, not as a style guide: it confirms
`ScrollTrigger` and `SplitText` are the two techniques worth building deep
familiarity with first (this skill and `showcase-motion`, respectively), and
that `Observer` is common enough to document properly rather than treat as a
footnote — which is why it has its own step and its own recipes here, unlike
in an earlier draft of this skill's scope.
