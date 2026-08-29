---
name: scroll-choreography
description: Wire scroll-linked animation and smooth scroll — choosing Lenis or GSAP ScrollSmoother, deciding between native CSS scroll-driven animations, GSAP ScrollTrigger, and Motion's useScroll, pinning/scrubbing sections, using Observer for cross-browser wheel/touch/drag input, and wiring the View Transitions API for route changes. Use whenever scroll animation, smooth scroll, Lenis, ScrollTrigger, ScrollSmoother, pin, scrub, snap, parallax, scroll reveal, fade in on scroll, animation-timeline, view-timeline, Observer, or View Transitions come up, or a showcase site needs scroll choreography, scroll jank, or scroll accessibility fixed. Also covers Serbian phrasings - scroll animacija, smooth scroll efekat, pinovanje sekcije, scroll koreografija, parallax efekat, prelaz izmedju stranica. Do NOT use for text-reveal or hero entrance choreography (showcase-motion), or product-UI feedback motion and MotionConfig root wiring (motion-principles).
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "gsap@3.15.0, @gsap/react@2.1.2, lenis@1.3.26 — read from published npm tarballs, source-verified 2026-08-29 (gsap-core.js for matchMedia; ScrollTrigger.js for pin/scrub/snap/pinSpacing/anticipatePin; ScrollSmoother.js and Observer.js for full API surface and confirming zero prefers-reduced-motion handling in ScrollSmoother; @gsap/react src/index.js for useGSAP's gsap.context()-based cleanup); lenis README.md and dist/lenis-react.mjs for the ReactLenis/useLenis API and the native respectReducedMotion default; official gsap.com/resources/st-mistakes/ page (fetched 2026-08-29) for the ScrollTrigger.refresh()-after-lazy-image and scroll-behavior:smooth gotchas; @mdn/browser-compat-data@8.0.13 (published 2026-08-27) for animation-timeline support (Firefox: preview, not shipped in stable), Document.startViewTransition support (Chrome 111+, Firefox 144+ shipped 2025-10-14, Safari 18+), and the dvh unit (Baseline since Safari 15.4/Firefox 101/Chrome 108); gsap.com/showcase curated gallery, 60 sampled entries tagged by the GSAP team (2026-08-29), for real-world plugin usage frequency; own Vite+React+TS reference build, browser-tested 2026-08-29, confirming window.scrollTo()/scrollIntoView() desyncs Lenis's internal animated-scroll state from the real scroll position and that keyboard Page Down scrolling works correctly through Lenis"
---

# Scroll Choreography

Scroll-linked motion has three independent layers that get tangled together by
habit: **whether the page scrolls smoothly at all** (a global decision), **what
drives an individual scroll-linked animation** (a per-effect decision), and
**how a route change transitions** (a separate API entirely). Treating them as
one decision is why most scroll-heavy sites end up with either a library
fighting itself or an animation system nobody can explain.

Work top-down: smooth-scroll engine first (Step 1–3, or explicitly none), then
the per-effect mechanism (Step 4–6), then page transitions (Step 8) — never the
reverse. A pinned section built before the smooth-scroll decision is made gets
rebuilt the moment that decision changes the coordinate system it scrolls in.

## Step 1 — Decide whether smooth scroll is wanted at all

Smooth scroll is not a default for "premium." It is a real, named cost:

- It intercepts native scroll and re-times it, which means every scroll-linked
  effect downstream must sync to the smoothing library's clock, not the
  browser's.
- It is capped at 60fps on Safari and 30fps in low-power mode — a hard ceiling
  from a WebKit bug, not a library limitation.
- Nested scroll containers (a modal, a sidebar, a code block) need explicit
  configuration or they inherit the outer smoothing and feel wrong.

Skip it for: internal tools, content-heavy sites where reading speed matters
more than motion, anything where `director`'s Ant Design lane already applies.
Reach for it for: single-narrative marketing sites, portfolios, case-study
pages — where the scroll *is* the interaction, not incidental to it.

## Step 2 — Choose the engine: Lenis or GSAP ScrollSmoother

Both are free. They are not interchangeable, and the difference that matters
most is one most comparisons miss:

| | Lenis (1.3.26) | ScrollSmoother (3.15.0) |
|---|---|---|
| Reduced motion | **Built in.** `respectReducedMotion` defaults `true` — `lerp` is forced to `1` (scroll tracks input 1:1) and programmatic scrolls jump instantly when the OS preference is set. | **None.** Zero references to `prefers-reduced-motion` anywhere in the source. Every reduction has to be hand-wired, same as the rest of GSAP (Step 9). |
| Dependency | Standalone, MIT, zero peer deps beyond the framework binding you use. | Requires `ScrollTrigger` registered alongside it — you are already inside the GSAP ecosystem or you're adding it just for this. |
| Touch behavior | Smooths touch scroll too, unless configured off. | `smoothTouch` is **off by default** — touch scrolls natively unless explicitly enabled (`true` → 0.8s, or a number). |
| Framework binding | First-party `lenis/react` subpath — `ReactLenis`, `useLenis()`. | None; it's a GSAP plugin, wire it in `useGSAP`. |
| Extras | `lenis/snap` for CSS-scroll-snap-like behavior; `respectReducedMotion` keeps running for WebGL/DOM sync, only smoothing is disabled. | `effects` (data-speed parallax shorthand), `normalizeScroll` (see below). |

**Default to Lenis** unless the project is already GSAP-only and the team
wants one fewer dependency to reason about — ScrollSmoother's missing
reduced-motion handling means it costs the same manual wiring either way, so
that's not a reason to prefer it. Pick ScrollSmoother when `normalizeScroll`'s
mobile-viewport fix or the `effects` shorthand outweighs the standalone
simplicity of Lenis.

`normalizeScroll` (`ScrollTrigger.normalizeScroll()` under the hood, which
ScrollSmoother's `normalizeScroll` option wires in for you) is a separate,
narrow fix worth knowing about even outside ScrollSmoother: mobile browsers
resize the viewport as the address bar shows/hides while scrolling, which
shifts every `vh`-based and scroll-position-based measurement mid-gesture.
`normalizeScroll` locks the viewport height for the duration of a scroll
gesture so that jank doesn't happen. It is not a general-purpose fix to reach
for by default — only pull it in when that specific mobile resize jank is
actually observed.

Real-world signal, not a rule: in a 60-entry sample of GSAP's own curated
showcase (tags are set by the GSAP team, not self-reported — see
`references/scrolltrigger-recipes.md` for the full breakdown), ScrollSmoother
was explicitly tagged on **~12%** of entries against **~83%** for ScrollTrigger
itself — most curated GSAP sites either use no smoothing library or use Lenis
alongside plain ScrollTrigger.

Full wiring for both — including the exact GSAP-ticker sync snippet Lenis's
own README ships, and the mistake of double-driving the raf loop — is in
`references/lenis-integration.md`.

## Step 3 — Never bypass the smoothing library's own scroll methods

This is not a style preference — it is a reproducible bug, and how bad it is
depends on Lenis's internal state when the bypass happens. Lenis only
resyncs its own `animatedScroll`/`targetScroll` to the real scroll position
from a native scroll event when `isScrolling` is `false` or `"native"` —
confirmed reading `onNativeScroll` in Lenis's own source. While Lenis is
mid-animation (`isScrolling === "smooth"`, e.g. right after a `scrollTo()`
call or during its own inertia), a `window.scrollTo()` call in that window
does **not** get picked up, and the page's visual content stops following the
real scroll position until the animation ends or user input resyncs it.
Verified directly: calling `window.scrollTo(0, 950)` on a Lenis page in that
state left `scrollY` correctly at `950` while the rendered content stayed
exactly where it was. On a fully idle page the same call is less likely to
visibly desync — but "usually fine, breaks unpredictably near any Lenis
animation" is not a state worth debugging into existence.

Always route programmatic scrolling through the library that owns the scroll:
`lenis.scrollTo(target, options)`, or ScrollSmoother's own `scrollTo()`. This
includes anchor-link handling, "back to top" buttons, and any router's
scroll-restoration logic — all of it has to go through the smoother, not
`window`.

Wheel-driven and keyboard-driven scrolling (`Page Down`, arrow keys) do **not**
have this problem — Lenis eases the *native* scroll position rather than
hijacking it with a transform, so the browser's own scroll machinery, the
scrollbar, and keyboard navigation stay live. Verified by driving a Lenis page
with `Page Down` and confirming both the reported `scrollY` and the rendered
content advanced together correctly.

## Step 4 — The per-effect decision: native CSS, GSAP ScrollTrigger, or Motion's useScroll

This is the decision that repeats for every individual scroll-linked effect,
and it is not the same decision as Step 2. Three real options:

**Native CSS (`animation-timeline: scroll()` / `view()`)** — zero JavaScript,
runs on the compositor thread, cannot jank the main thread. **Not safe as the
only mechanism today**: Chrome/Edge 115+ and Safari 26+ support it, but
Firefox's own support status is `preview` — not shipped in stable as of the
data checked (2026-08-27, two days before verification). Use it as a
progressive enhancement behind `@supports (animation-timeline: view())`, with
a JS fallback for Firefox, or accept plain motion there. Never use it alone
for an effect the design depends on.

**Motion's `useScroll()` / `useTransform()`** — the one genuinely surprising
finding here: Motion's `useScroll` **automatically detects and uses the native
`ScrollTimeline`/`ViewTimeline`** where the browser supports it, and falls
back to a `requestAnimationFrame`-driven listener where it doesn't — verified
directly in `use-scroll.mjs`, which calls `supportsScrollTimeline()` /
`supportsViewTimeline()` and switches an internal `accelerate` config
accordingly. This means Motion gets native-thread performance on Chrome/Safari
and a working fallback on Firefox from one API, with no `@supports` branching
required. The tradeoff: Motion's scroll-linked API is thinner than
ScrollTrigger's — no pinning, no snap, no scrub-with-timeline-labels.

**GSAP ScrollTrigger** — always JS-driven (no native-timeline acceleration
path), but by far the most complete feature set: `pin`, `scrub`,
`snap`, `pinSpacing`, `anticipatePin` are all real, verified options
(confirmed present in the shipped `ScrollTrigger.js`). This is the only one of
the three that handles **pinning** — freezing a section in the viewport while
its internal content animates against scroll — which is the signature
technique behind horizontal-scroll sections, sticky reveals, and most
scroll-scrubbed hero sequences. If the effect needs pinning, the decision is
already made.

**Rule of thumb:** simple opacity/transform-tied-to-scroll-progress with no
pin and Firefox not a hard requirement → native CSS as a progressive
enhancement. Already-Motion project, no pinning needed → `useScroll`. Anything
pinned, scrubbed against a multi-step timeline, snapped, or horizontal → GSAP
ScrollTrigger, no contest.

Full code for all three, plus the `@supports` fallback pattern, is in
`references/native-vs-js-scroll.md`.

## Step 5 — Observer, for input that isn't page-scroll

`Observer` normalizes wheel, touch, and pointer events into one cross-browser
interface — it is not a scroll-position tool, it's an *input-delta* tool.
Confirmed real, present options in the shipped source: `wheelSpeed`,
`preventDefault`, `tolerance`, `dragMinimum`.

This is what a fullpage slide deck, a drag-driven horizontal gallery, or a
"next section on scroll-intent, not scroll-position" pattern is actually built
on — not `ScrollTrigger` directly. In the same 60-entry GSAP showcase sample
from Step 2, `Observer` was tagged on **~28%** of entries, ahead of everything
except `ScrollTrigger` and `SplitText` — it is a mainstream part of the
toolkit here, not a niche plugin.

Reach for `ScrollTrigger` when the effect is tied to *scroll position*. Reach
for `Observer` when the effect is tied to *scroll or gesture intent* —
"the user tried to scroll down" rather than "the user is 40% through this
section." The two compose: `Observer` deciding when to advance a slide,
`ScrollTrigger`/GSAP tweens animating the transition itself.

## Step 6 — Pin and scrub, the core recipe

```jsx
useGSAP(
  () => {
    const panels = gsap.utils.toArray('.panel')
    gsap.to(panels, {
      xPercent: -100 * (panels.length - 1),
      ease: 'none',
      scrollTrigger: {
        trigger: container.current,
        pin: true,
        scrub: 1,
        end: () => '+=' + (container.current.offsetWidth * (panels.length - 1)),
        invalidateOnRefresh: true,
      },
    })
  },
  { scope: container },
)
```

`scrub: 1` ties the tween's progress to scroll position with 1 second of
catch-up easing — `scrub: true` ties it exactly, with no smoothing.
`invalidateOnRefresh: true` recalculates the tween's start values on refresh,
which matters the moment the trigger's dimensions depend on something that can
change (a resize, a font load). Cleanup is automatic: `useGSAP`'s
`gsap.context()` reverts every animation and kills every ScrollTrigger created
inside its scope when the component unmounts — verified from `@gsap/react`'s
own source, not assumed.

Full pin/scrub/snap recipes, horizontal-scroll setups, and the
`ScrollTrigger.refresh()` gotchas below are in
`references/scrolltrigger-recipes.md`.

## Step 7 — Two integration mistakes that are easy to ship

Both verified against GSAP's own official "mistakes" documentation, not
community folklore:

- **Lazy-loaded or dynamically-sized content shifts trigger positions.** If an
  image loads without a set `width`/`height`, or content arrives via
  fetch/AJAX after ScrollTrigger has already measured the page, every trigger
  below it is now positioned wrong. Fix: call `ScrollTrigger.refresh()` in the
  image's `load` handler (or the data-arrival callback), not just once on
  mount.
- **`scroll-behavior: smooth` in CSS fights ScrollTrigger's own
  positioning.** The browser's native smooth-scroll and ScrollTrigger's
  measurement of "where the user is" can disagree. If it's set anywhere in the
  project (a global reset, a framework default), override it to
  `scroll-behavior: auto !important` on the scrolling element.

One more, verified independently while building this skill's reference
project: for a full-bleed, viewport-height section (a pinned hero, a full
first panel), use **`100dvh`, never `100vh`.** `100vh` is the classic mobile
Safari bug — it measures the *maximum* viewport, address bar included, so the
section is taller than what's actually visible. `dvh` (dynamic viewport
height) has been Baseline-safe for years — Safari 15.4+, Firefox 101+, Chrome
108+ — there is no reason left to reach for `vh` on a section whose height
matters.

## Step 8 — View Transitions API, for route and page changes

A completely separate API from everything above — same-document (SPA) view
transitions via `document.startViewTransition()`, not to be confused with the
CSS scroll-driven `animation-timeline` from Step 4. Confirmed shipped in every
current evergreen engine: Chrome/Edge 111+, Safari 18+, and — the one worth
double-checking before assuming it's safe — **Firefox 144+, which shipped
2025-10-14**, well before current Firefox releases. This one *is* safe to rely
on today, unlike native scroll-driven animations.

```js
function navigate(url) {
  if (!document.startViewTransition) {
    location.href = url // no transition, just navigate
    return
  }
  document.startViewTransition(async () => {
    await router.push(url) // swap the DOM inside the callback
  })
}
```

Styling the transition is CSS, via the `::view-transition-old()` /
`::view-transition-new()` pseudo-elements — that part is genuinely simple once
the DOM-swap timing above is right.

**Framework router integration (Next.js App Router, React Router) is
explicitly not verified here.** Wrapping a framework router's navigation in
`startViewTransition` correctly — timing the DOM commit inside the callback —
is framework-specific and version-sensitive enough that it needs its own
source check before being written as fact, the same way `icon-resources`
flags its UI-glyph recommendation as unverified rather than asserting it.
Check the framework's own current router API before committing to a specific
wrapper package.

## Step 9 — Reduced motion, across every mechanism in this file, wired to one signal

Three different defaults, three different fixes, and they must land on the
**same** on/off signal or a user gets a half-reduced page:

| Mechanism | Default | Fix |
|---|---|---|
| Lenis | Respects it automatically (`respectReducedMotion: true`) | Nothing to do — verify, don't re-implement |
| ScrollSmoother | Ignores it entirely | Wrap smoothing setup in `gsap.matchMedia()`, branch on `(prefers-reduced-motion: reduce)` |
| One-shot GSAP tweens (Step 6's plain reveal) | Ignores it entirely | Same `gsap.matchMedia()` block, reduce distances/durations to near-zero in the `reduce` branch |
| Scrub-driven pin/scroll (Step 6's pin+scrub recipe) | Ignores it entirely | **Not a duration to shrink — see below** |
| Native `animation-timeline` | Ignores it entirely | `@media (prefers-reduced-motion: reduce) { animation-timeline: none; }` |
| Motion (`useScroll`, transforms) | Ignores it entirely — see `motion-principles` | The root `MotionConfig reducedMotion="user"` from `motion-principles` covers this; do not re-wire it here |

**Scrub-driven effects don't have a duration to shrink** — with `scrub: 1`
or `scrub: true`, scroll position *is* the timeline's playhead, so "reduce the
duration" doesn't map onto anything real. Two honest options instead:

- **Skip creating the ScrollTrigger entirely** in the `reduceMotion` branch of
  `gsap.matchMedia()`, and let the underlying content render in its natural
  document flow — a horizontal-pin gallery becomes a plain vertical stack, a
  scrubbed reveal starts at its end state. This is usually the right call: it
  removes the pin, not just the motion.
- **Keep the pin but drop `scrub`**, replacing it with a `once: true`
  `ScrollTrigger` (`start`/`toggleActions` instead of `scrub`) that jumps the
  content to its end state the moment the section enters view — for the rare
  case where the pinned layout itself needs to survive but the scroll-tied
  choreography doesn't.

Do not ship a `reduceMotion` branch that only tweaks numbers on a
scrub-driven trigger and calls it handled — verify by testing with the OS
preference actually set, not by reading the branch and assuming it applies.

The pattern that works for the one-shot case, verified end-to-end in this
skill's own reference build (mount, no scroll-trigger needed for this one):

```jsx
gsap.matchMedia().add(
  {
    reduceMotion: '(prefers-reduced-motion: reduce)',
    noPreference: '(prefers-reduced-motion: no-preference)',
  },
  (context) => {
    const { reduceMotion } = context.conditions
    // build the ScrollSmoother / ScrollTrigger animation here,
    // branching duration/distance/scrub on `reduceMotion`
  },
)
```

`gsap.matchMedia()` itself is core, not a plugin — confirmed present in
`gsap-core.js` — so it's available with a bare `gsap` install.

## Step 10 — Verify before declaring done

- The smooth-scroll decision (Step 1–2) was made deliberately and stated, not
  defaulted into.
- No code calls `window.scrollTo()` or `.scrollIntoView()` while a smoothing
  library owns the scroll; all programmatic scrolling goes through
  `lenis.scrollTo()` / the smoother's own method.
- Every pinned/scrubbed effect uses GSAP ScrollTrigger, not native CSS or
  Motion — those two do not support pinning.
- Native `animation-timeline` usage, if any, sits behind
  `@supports (animation-timeline: view())` with a stated fallback for Firefox.
- `ScrollTrigger.refresh()` is called after any lazy-loaded or dynamically
  arriving content that affects layout.
- `scroll-behavior: smooth` is not set anywhere in the project alongside
  ScrollTrigger.
- Full-bleed, viewport-height sections use `100dvh`, not `100vh`.
- Every scroll mechanism in use (Lenis, ScrollSmoother, ScrollTrigger, native
  CSS) is reduced under the same `prefers-reduced-motion` signal — check each
  row of the Step 9 table, not just the one that was top of mind.
- If any effect uses `scrub`, its `reduceMotion` branch removes the pin/scrub
  entirely (or replaces it with a one-shot jump-to-end-state) rather than
  just shrinking a duration that scrub-driven motion doesn't have — verified
  by actually toggling the OS preference, not by reading the branch.
- If a route/page transition uses the View Transitions API, framework router
  integration was checked against the framework's current docs, not assumed
  from a remembered pattern.
