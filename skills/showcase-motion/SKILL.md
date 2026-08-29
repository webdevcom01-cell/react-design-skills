---
name: showcase-motion
description: Choreograph hero entrances, text reveals, and hover/cursor interactions for marketing and showcase sites — GSAP SplitText character/word/line reveals, sequencing multiple elements, magnetic buttons and cursor-follow via gsap.quickTo(), and the CSS prerequisites SplitText masking silently depends on. Use whenever a hero entrance, text reveal, split text, staggered reveal, magnetic button, cursor-follow effect, hover interaction, or premium/brutal/awwwards-style animation comes up; whenever a marketing or portfolio site's motion needs to read as considered rather than generic. Also covers Serbian phrasings - hero animacija, text reveal, otkrivanje teksta, magnetic dugme, cursor efekat, premium animacije, brutalne animacije. Do NOT use for scroll-linked mechanics - smooth scroll, ScrollTrigger, pinning, native scroll-driven animation (scroll-choreography), product-UI feedback motion (motion-principles), or presentation slides (frontend-slides).
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "gsap@3.15.0 — read from the published npm tarball, source-verified 2026-08-29 (SplitText.js confirming Intl.Segmenter-based splitting and the free-since-2025 bundling of every former Club GreenSock plugin; gsap-core.js confirming quickTo() and matchMedia() are both core, not plugins); gsap.com/standard-license (fetched 2026-08-29) for the exact commercial-use and Prohibited-Uses terms; npm split-type@0.3.4 vs the bundled SplitText for the redundancy comparison; own Vite+React+TS reference build, browser-tested 2026-08-29 across desktop and mobile/touch viewports — reproduced and fixed two real defects (a SplitText mask clipped by an inherited percentage line-height, and SplitText-masked text rendered near-invisible from an inherited boilerplate heading color that only looked correct under a coincidental dark color-scheme) and confirmed a CSS `@media (hover: hover)` guard correctly prevents a hover style from sticking after a touch tap while `:active` still fires"
---

# Showcase Motion

`motion-principles` optimizes for restraint: small distances, one thing
moving at a time, animation that never outstays its purpose. Those rules are
correct for product UI, and they are the wrong rules here. A hero section
whose entire job is to make a first impression is not "over-animated" for
committing to a real entrance — it's under-animated if it doesn't. This skill
exists to give that different brief its own, equally deliberate rule set,
rather than let it inherit restraint rules that were never written for it.

The techniques below — text splitting, staggered reveals, magnetic
interactions — are genuinely powerful and genuinely easy to make look cheap.
Two of the three most consequential rules in this file (Step 4) were found by
building and literally watching a broken reveal in a browser, not by reading
documentation — that is the standard the rest of this skill tries to hold to.

## Step 1 — Confirm this is the right brief before applying any rule here

Ask what's being built, the same question `director` Step 1 asks: a
marketing/landing page, a portfolio, a case-study or campaign site — where
the motion itself is part of what's being sold — or a product surface where
motion supports a task. If it's the latter, stop and use `motion-principles`
instead; its 4–8px distance ceiling and "one thing moves at a time" rule are
correct there and would be actively wrong advice here.

If a project is genuinely both (a marketing site plus an app), that's two
passes with two different motion rule sets applied to two different route
trees — not a blend. Say so rather than splitting the difference.

## Step 2 — GSAP or Motion, for this specific job

Both are legitimate; the deciding factor is what the effect actually needs,
not a standing preference.

**Reach for GSAP** when the centerpiece is text — `SplitText` is GSAP's, full
stop, there is no equivalent depth in Motion — or when several elements need
to be sequenced against each other with explicit relative offsets
(`timeline.to(a, {...}, 0).to(b, {...}, 0.2)`), which GSAP's timeline API is
built for and Motion's isn't.

**Reach for Motion** when the project's product UI already runs on it (per
`motion-principles`) and the showcase surface is simple enough — a handful of
entrance animations, no character-level text work — that adding a second
animation library costs more than it buys.

GSAP is genuinely free for this, not "free with an asterisk that bites
later": as of the Webflow acquisition, every formerly Club-GreenSock-only
plugin — `SplitText`, `MorphSVG`, `DrawSVG`, `Flip`, `Draggable`,
`ScrollSmoother`, and the rest — ships in the base `gsap` package, confirmed
by the plugin files' physical presence in the published npm tarball. The one
real restriction, read directly from `gsap.com/standard-license`: it may not
be used to build a **no-code visual animation tool that competes with
Webflow's own animation builder** — irrelevant to building a site with it,
relevant only if the deliverable is itself an animation-authoring product.

## Step 3 — Wire it through `useGSAP`, the same pattern as `scroll-choreography`

Every recipe below assumes `@gsap/react`'s `useGSAP` hook — its
`gsap.context()`-based automatic cleanup, the missing `"use client"`
directive that the component itself must add, and the general pattern are
covered in full in `scroll-choreography`'s
`references/scrolltrigger-recipes.md`; this skill doesn't repeat it. Two
additions specific to `SplitText`:

- **Register it explicitly**: `gsap.registerPlugin(SplitText, useGSAP)`.
  Read directly from source: `SplitText.create()` only wires itself into
  GSAP's own context system if it hasn't already been registered, and its
  fallback path looks for `window.gsap` — which a normal ESM
  `import gsap from 'gsap'` never sets. Skip the explicit `registerPlugin`
  call and `SplitText` still visually splits and animates, but its
  integration with `gsap.context()` isn't reliable — which matters for the
  next point.
- **Its cleanup is not automatic the way a plain tween's is** — always call
  `split.revert()` yourself, in `useGSAP`'s own cleanup function, rather
  than assuming `gsap.context()`'s auto-revert covers it. Skipping this
  leaves the DOM permanently restructured into per-character `<div>`s even
  after the component using them is gone.

## Step 4 — Text reveal: the technique, and the two prerequisites nobody tells you about

```jsx
gsap.registerPlugin(SplitText, useGSAP)

useGSAP(
  () => {
    const split = SplitText.create('.hero-title', {
      type: 'chars,words',
      mask: 'chars',
    })

    gsap.from(split.chars, {
      yPercent: 120,
      opacity: 0,
      stagger: 0.03,
      duration: 0.8,
      ease: 'expo.out',
    })

    return () => split.revert()
  },
  { scope: container },
)
```

`SplitText` splits by grapheme cluster via `Intl.Segmenter` when available —
confirmed reading the shipped source — which means it handles emoji and
multi-codepoint characters correctly where a naive string-splitting
alternative (`split-type` or a hand-rolled splitter) doesn't. Prefer
`SplitText` over `split-type` whenever GSAP is already a dependency; reach
for `split-type` only when avoiding a GSAP install entirely for one effect.
`mask: 'chars'` wraps every character in an `overflow: clip` container sized
to the line box — this is what makes a "slide up from below the mask" reveal
actually clip at the mask edge instead of visibly sliding in from outside the
text's bounding box.

**Two CSS properties must be explicit on the masked element before any of
this looks right, and both failures are invisible in the DOM — the animation
completes perfectly, `opacity: 1`, `transform` exactly correct, and the
result is still broken:**

- **`line-height` must be explicit, never inherited.** Reproduced directly:
  a boilerplate reset with `font: 18px/145%` on `:root` computes that
  percentage line-height to a **fixed pixel value at the root's own
  font-size** (18px × 145% ≈ 26px) — and that fixed pixel value is what
  inherits down, not the percentage. A hero heading at `font-size: 5rem`
  (90px) inheriting that 26px line-height gets a mask wrapper sized to a line
  box far shorter than its own glyphs — every character renders visibly
  clipped top and bottom. Set `line-height` explicitly (e.g. `1.1`) on any
  element being `SplitText`-masked; never trust an inherited value, no matter
  how standard the boilerplate looks.
- **`color` must be explicit, never inherited from a theme-variable
  system.** Reproduced directly: a boilerplate `h1, h2 { color: var(--text-h)
  }` rule, where `--text-h` flips between a near-black light-mode value and a
  near-white dark-mode value via `@media (prefers-color-scheme: dark)`, can
  make text *look* correctly colored during development purely by
  coincidence — if the browser or OS happens to be in dark mode, a heading
  that was never explicitly colored renders light-on-dark and passes a quick
  visual check. Switch to light mode (or hand the same code to a user in
  light mode) and the identical markup renders **near-invisible**, because
  the color was never actually set — it was borrowed from a mode-dependent
  default that happened to agree with the design once. Test masked/reveal
  text in both color schemes, or better, set the color explicitly and remove
  the dependency on which scheme happens to be active.

Full recipes for word-by-word and line-by-line reveals, mixing reveal types
within one heading, and the mask-wrapper DOM shape `SplitText` actually
produces (no `.char`/`.word` CSS classes in the current version — target the
`split.chars`/`split.words`/`split.lines` arrays it returns, not a selector)
are in `references/text-reveal-patterns.md`.

## Step 5 — Entrance choreography: bigger than product UI, still not decoration

`motion-principles`' 4–8px distance ceiling doesn't apply to a hero moment —
but "bigger" still needs its own ceiling, or this becomes the thing that
makes generated UI look like a toy, just at showcase scale instead of app
scale.

- **Sequence matters more than any individual element's motion.** What
  enters first sets what the viewer reads as most important. A logo, then a
  headline, then a subhead, then a CTA — in that order, with a real,
  deliberate offset between each (0.1–0.15s is usually enough; more than
  that and it reads as sluggish, not stately) — communicates hierarchy for
  free. Firing everything at once communicates nothing.
- **Stagger tightly even at showcase scale.** The `stagger: 0.03–0.05` rule
  from `motion-principles` still holds for character-level staggers; it's
  the *distance* per element and the *scale* of the whole sequence that
  differ here, not the stagger interval itself. A 40-character headline
  staggering at `0.1` is a 4-second wait to finish reading a hero.
- **One hero moment per page, not one per section.** The showcase license to
  go bigger is for the entrance the visitor sees once, not a standing
  permission slip for every subsequent section. Sections below the fold
  revealing on scroll belong to `scroll-choreography`'s more restrained,
  "answers a question" scroll-reveal pattern, not a second hero-scale
  performance each time.
- **Exit is still faster than entrance**, same ratio `motion-principles`
  states (~0.7–0.8×) — a showcase site is not exempt from this because
  nothing about "the user is leaving" changes at a bigger scale.

## Step 6 — Hover and cursor interactions

```jsx
useGSAP(() => {
  const btn = btnRef.current
  if (!btn) return

  const strength = 0.35 // how strongly the button chases the cursor — 0.2-0.4 is typical
  const xTo = gsap.quickTo(btn, 'x', { duration: 0.4, ease: 'power3' })
  const yTo = gsap.quickTo(btn, 'y', { duration: 0.4, ease: 'power3' })

  const handleMove = (e: MouseEvent) => {
    const rect = btn.getBoundingClientRect()
    xTo((e.clientX - rect.left - rect.width / 2) * strength)
    yTo((e.clientY - rect.top - rect.height / 2) * strength)
  }
  const handleLeave = () => {
    xTo(0)
    yTo(0)
  }

  btn.addEventListener('mousemove', handleMove)
  btn.addEventListener('mouseleave', handleLeave)
  return () => {
    btn.removeEventListener('mousemove', handleMove)
    btn.removeEventListener('mouseleave', handleLeave)
  }
}, { scope: container })
```

`gsap.quickTo()` — confirmed core, not a plugin — exists exactly for this:
repeated, high-frequency updates to the same tween (every `mousemove`)
without the cost of creating a new tween per call. Creating a fresh
`gsap.to()` on every `mousemove` event is the version of this that looks
fine in a demo and drops frames in review.

**Every hover-only effect needs a non-hover equivalent, and this is not a
theoretical concern — it's directly verified.** Guard hover styles behind
`@media (hover: hover)`, and give the element a real `:active` (or
JS-driven tap) state for touch. Tested directly on a touch-emulated
viewport: with the guard in place, tapping the element correctly triggered
only the `:active` styling — the `:hover` style never applied and never got
stuck the way it does on an unguarded `:hover` rule tapped on a touchscreen.
Full pattern, plus a tilt-on-hover recipe and the touch-test method, is in
`references/hover-interactions.md`.

## Step 7 — Reduced motion, the showcase-specific note

The mechanism is identical to `scroll-choreography` Step 9 —
`gsap.matchMedia()`, confirmed core, branching on
`(prefers-reduced-motion: reduce)` — so wire it the same way and don't
re-derive it here. The showcase-specific judgment call: a hero entrance under
reduced motion should usually still *exist*, just without the movement —
cut `yPercent`/stagger to near-zero and keep the opacity fade, rather than
deleting the reveal outright. The content still needs to appear; only the
motion is the part being asked to go away.

## Step 8 — Where this skill stops

- Anything **scroll-linked** — a reveal tied to scroll position, pinning,
  parallax, smooth scroll — is `scroll-choreography`, not this skill. This
  skill's reveals fire on mount or on a discrete trigger (a button, a route
  change), never on scroll progress.
- **Product UI feedback motion** — button press states, toast transitions,
  the `MotionConfig` root wiring — is `motion-principles`, regardless of how
  visually rich the product's marketing pages are elsewhere.
- **Slide decks and presentations** are `frontend-slides`, a different
  medium with its own pacing and audience-control concerns, not a showcase
  website read continuously by one visitor.

## Step 9 — Verify before declaring done

- The brief was confirmed as showcase/marketing, not product UI, before any
  rule in this file was applied — if it's product UI, this was the wrong
  skill.
- Every `SplitText`-masked element has an **explicit** `line-height` and an
  **explicit** `color` — neither inherited from a boilerplate reset or a
  theme-variable system. Checked in more than one color scheme if the
  project supports both.
- `split.revert()` (or an equivalent context-scoped cleanup) runs on
  unmount; the DOM isn't left permanently split.
- The entrance sequence has a legible order — logo/headline/subhead/CTA or
  equivalent — not everything firing at once.
- Exactly one hero-scale entrance moment exists per page; sections below the
  fold use `scroll-choreography`'s more restrained reveal pattern, not a
  repeat performance.
- Hover-only interactions are guarded behind `@media (hover: hover)` and
  have a real `:active`/touch equivalent — verified on an actual touch
  viewport, not assumed from the CSS alone.
- `gsap.quickTo()`, not a fresh `gsap.to()` per event, drives any
  high-frequency pointer-tracked animation.
- Reduced motion keeps the content visible and removes the movement,
  wired through the same `gsap.matchMedia()` signal `scroll-choreography`
  uses elsewhere in the project.
