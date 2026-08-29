# Lenis integration reference

Verified against `darkroomengineering/lenis@1.3.26` — `README.md`,
`dist/lenis.js`, `dist/lenis-react.mjs`, `dist/lenis.d.ts`; GSAP
`gsap@3.15.0` `ScrollSmoother.js` for the alternative comparison.

## Contents

- Install and root wiring (React)
- The GSAP ticker sync, verbatim from source
- Reduced motion — what it actually does
- Nested scroll, anchors, and `lenis/snap`
- Limitations, stated plainly
- ScrollSmoother as the alternative

## Install and root wiring

```bash
npm install lenis
```

Import CSS once — it is not optional decoration, it fixes a handful of
cross-browser scroll quirks:

```js
import 'lenis/dist/lenis.css'
```

The React binding lives at the `lenis/react` subpath — confirmed exported
from `dist/lenis-react.mjs`: `ReactLenis` (also exported as `Lenis` and the
default), `useLenis`, `LenisContext`.

```jsx
// src/app/providers.tsx — small client component, same pattern as motion-principles' MotionProvider
'use client'
import { ReactLenis } from 'lenis/react'

export function LenisProvider({ children }: { children: React.ReactNode }) {
  return (
    <ReactLenis root options={{ autoRaf: false, anchors: true }}>
      {children}
    </ReactLenis>
  )
}
```

`anchors: true` is not optional boilerplate — it's the opposite of Lenis's own
default. Confirmed in the README: **"By default, Lenis will prevent anchor
links from working while scrolling. To enable them, you must set
`anchors: true`."** Any `<a href="#section">` on a Lenis page is broken until
this is set explicitly; nothing about it is automatic.

`root` tells `ReactLenis` to smooth the whole document (`window` /
`document.documentElement`) rather than creating its own wrapper/content div
pair — read directly from the component's source: when `root` is set, no
extra wrapper element is rendered, and Lenis is constructed without an
explicit `wrapper`/`content` override, so it defaults internally to smoothing
the real page scroll. This is the correct mode for a full-page smooth-scroll
site; the non-root mode (an explicit `wrapper`/`content` pair) is for
smoothing one scrollable region inside an otherwise normal page.

`autoRaf: false` is deliberate here, not a default to copy blindly — it's
needed the moment Lenis has to share a single animation clock with GSAP
(next section). If nothing else drives the raf loop, leave `autoRaf: true`
(or omit it) and Lenis runs its own loop.

## The GSAP ticker sync, verbatim from source

This exact pattern is Lenis's own README, not a community workaround:

```js
const lenis = new Lenis() // or read the instance out of useLenis()

lenis.on('scroll', ScrollTrigger.update)

gsap.ticker.add((time) => {
  lenis.raf(time * 1000) // gsap.ticker's time is in seconds; Lenis wants ms
})

gsap.ticker.lagSmoothing(0)
```

`lagSmoothing(0)` disables GSAP's tab-backgrounding compensation — without it,
GSAP tries to "catch up" after a throttled/backgrounded tab regains focus,
which fights Lenis's own easing and produces a visible jump-then-settle.

In React, wire this once at the root, keyed off `useLenis()`:

```jsx
function LenisTicker() {
  const lenis = useLenis()

  useEffect(() => {
    if (!lenis) return
    lenis.on('scroll', ScrollTrigger.update)

    const tick = (time: number) => lenis.raf(time * 1000)
    gsap.ticker.add(tick)
    gsap.ticker.lagSmoothing(0)

    return () => gsap.ticker.remove(tick)
  }, [lenis])

  return null
}
```

**Do not also set `autoRaf: true` when doing this.** Two independent raf loops
both calling `lenis.raf()` double-drives the scroll clock — the symptom is
scroll that feels too fast or stutters unpredictably, and it is easy to miss
because each loop looks correct in isolation.

## Reduced motion — what it actually does

Verified from the README, not inferred from the option name: with
`respectReducedMotion: true` (the default) and the OS preference set to
`reduce`:

- `lerp` is forced to `1` — scroll tracks the input device 1:1, `duration` and
  `easing` options are ignored.
- Programmatic scrolls (`scrollTo()`, anchor-link navigation) jump instantly
  to their target instead of animating.
- Lenis **keeps running** rather than fully disabling itself — this matters if
  anything (a WebGL scene, a DOM sync) depends on Lenis's scroll events
  continuing to fire.
- The preference is picked up live, without a reload — check
  `lenis.prefersReducedMotion` if a project needs to branch its own code on
  the same signal Lenis is already tracking.

This is the opposite default of Motion (`reducedMotion: "never"`, covered in
`motion-principles`) and of GSAP/ScrollSmoother (Step 9 of the main skill).
Do not add a redundant manual reduced-motion check around Lenis itself — it is
handled. Do make sure whatever content a smooth-scrolled page reveals (GSAP
tweens, Motion animations) is *separately* wired to the same OS signal, since
Lenis reducing its own smoothing does not reduce anyone else's animations.

## Nested scroll, anchors, and `lenis/snap`

- Nested scroll containers (a modal's internal scroll, a code block, a
  sidebar) need explicit configuration — either an HTML attribute
  (`data-lenis-prevent`) or the JS API — or they inherit the outer page's
  smoothing and feel wrong. Read the "Nested scroll" section of Lenis's own
  README before shipping any modal or overflow panel on a Lenis page.
- **Anchor links are broken by default, not handled automatically.** Lenis's
  own README states it plainly: "By default, Lenis will prevent anchor links
  from working while scrolling. To enable them, you must set
  `anchors: true`." Without it, `<a href="#section">` does nothing while the
  page is mid-scroll. This is why the root-wiring example above sets it
  explicitly — it is required setup, not a nice-to-have, and it is also why
  Step 3 of the main skill insists on routing all programmatic scrolling
  through `lenis.scrollTo()`: an anchor click bypassing Lenis risks the same
  desync problem as a manual `window.scrollTo()` call.
- **Lenis does not support native CSS `scroll-snap`.** If snap behavior is
  needed on a Lenis page, use the separate `lenis/snap` subpath package, not
  CSS `scroll-snap-*` properties — they are a documented, explicit
  limitation, not a bug to work around.

## Limitations, stated plainly

Read directly from Lenis's own README "Limitations" section — these are not
edge cases to discover in production:

- No native CSS scroll-snap support (use `lenis/snap`).
- Capped at 60fps on Safari (a WebKit bug, not fixable from userland) and
  30fps in low-power mode.
- Smooth scroll stops working inside an `<iframe>` — iframes don't forward
  wheel events to the parent.
- `position: fixed` can lag on pre-M1 Macs running Safari.
- Touch events with `syncTouch` enabled can behave unexpectedly on iOS < 16.

## ScrollSmoother as the alternative

If the project is GSAP-only and the `effects`/`normalizeScroll` conveniences
matter more than a standalone dependency, use ScrollSmoother instead — see the
comparison table in the main `SKILL.md` Step 2. The one fact worth repeating
here because it changes the reduced-motion work: ScrollSmoother's shipped
source (`ScrollSmoother.js`, 3.15.0) has **zero** references to
`prefers-reduced-motion` — every reduction has to be hand-wired through
`gsap.matchMedia()`, the same as any other GSAP animation (main skill,
Step 9). Do not assume ScrollSmoother "just handles it" the way Lenis does.
