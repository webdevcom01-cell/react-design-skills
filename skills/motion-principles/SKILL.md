---
name: motion-principles
description: Add animation to a React project with Motion (formerly Framer Motion) — correct import path, a duration/easing token layer instead of magic numbers, prefers-reduced-motion handled properly, and subtlety rules that keep an interface from reading as a toy. Use whenever animation, transitions, motion, Framer Motion, motion.dev, AnimatePresence, layout animations, scroll reveals via Motion's viewport prop, hover/tap feedback, page transitions, stagger, springs, or easing come up; whenever a UI feels static, abrupt, janky, or over-animated; and whenever prefers-reduced-motion or animation accessibility is raised. Also covers Serbian phrasings - animacija, tranzicija, prelaz izmedju stranica, pokret, easing, trajanje animacije, deluje staticno. Do NOT use for a component library (component-library-advisor), Storybook (storybook-workflow), canvas/whiteboard work (tldraw-workflow), scroll mechanics (scroll-choreography), or a showcase hero's bigger entrance choreography (showcase-motion).
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "motiondivision/motion@1b037b0 (v13.1.1, 2026-08-20) — packages/motion, packages/framer-motion, packages/motion-dom, packages/motion-utils, README.md, AGENTS.md, CHANGELOG.md, LICENSE.md; npm registry motion@13.1.1 and framer-motion@13.1.1; ant-design/ant-design@6.6.1 theme seeds; saadeghi/daisyui@5.7.20 component CSS (read from source, 2026-08-23); re-verified 2026-08-25 directly from the published framer-motion@13.1.1 npm tarball — bundle size budgets (34.9/6/17.85/29.8 kB), the `reducedMotion: \"never\"` context default, and the non-reactive `useReducedMotion` implementation (useState + TODO comment) all confirmed byte-for-byte; DaisyUI's 14-of-61 component stylesheets with prefers-reduced-motion guards confirmed by direct count against the published package"
---

# Motion Principles

Animation is a communication channel, not a finish. It answers three questions the user
would otherwise have to guess at: *did my action register*, *where did this thing come
from*, and *what should I look at*. An animation that answers none of those is costing
frames and attention for nothing.

There are two failure modes and they are equally common. A product with no motion feels
abrupt — state changes teleport and the user loses their place. A product with motion
everywhere feels like a toy — every card floats in, every button bounces, and nothing reads
as important because everything is moving. The work below is mostly about the second one,
because it is the one that generated UI produces by default.

**Motion is the animation library for this system.** Not a decision to relitigate per
project; the point of standardising is that timing tokens transfer between projects.

## Step 1 — Get the package name right, from source

```bash
npm install motion
```

```jsx
import { motion, AnimatePresence, MotionConfig } from "motion/react"
```

The naming history causes real confusion, so here is what is actually true as of
**13.1.1**:

- `motion` and `framer-motion` **both publish version 13.1.1**, from the same monorepo, on
  the same day. Neither is deprecated — `npm view framer-motion deprecated` returns empty.
- `framer-motion` is not a legacy stub. It is **the React implementation package**.
  `motion` declares `"framer-motion": "^13.1.1"` as a dependency, and
  `packages/motion/src/react.ts` is literally `export * from "framer-motion"` with `motion`
  and `m` re-bound explicitly. The repo's own `AGENTS.md` states it plainly:
  *"`motion`: Re-export of `framer-motion`."*
- The README is equally plain about which name to use: *"Framer Motion is now Motion.
  Import from `motion/react` instead of `framer-motion`."*

So: **new code installs `motion` and imports from `motion/react`.** Existing
`framer-motion` code is not broken and does not need a migration sprint — changing the
import is a rename, not an upgrade. Do not install both packages directly; `motion` already
pulls `framer-motion` in transitively, and seeing it in `npm ls` is expected, not a
leftover.

The subpaths matter more than they look:

| Import | What it is |
|---|---|
| `motion/react` | The React API — `motion`, `AnimatePresence`, `MotionConfig`, hooks |
| `motion/react-client` | Pre-created components (no `Proxy`), the RSC-safe entry — Step 6 |
| `motion/react-m` | The `m` component only, for `LazyMotion` bundle splitting |
| `motion` (root) | Vanilla JS/DOM API — `animate`, `scroll`, `inView`. Not React |
| `motion/mini`, `motion/react-mini` | Minimal WAAPI-only animate, smallest possible |

Peer deps are `react ^18.0.0 || ^19.0.0` and `react-dom` at the same range, both marked
optional. The library is **MIT**. Note that **Motion+** is a paid membership: the core
library stays MIT, but some APIs documented on motion.dev (`Cursor`, `Ticker`) sit behind
it. Check an API exists in the installed package before building on it.

## Step 2 — The purpose test, applied before writing the animation

Every animation must answer one of three questions. If it answers none, delete it.

| Purpose | What it does | Typical form |
|---|---|---|
| **Feedback** | Confirms the system received an action | Button press scale, toggle slide, input focus ring |
| **Continuity** | Preserves spatial context across a state change | Modal growing from its trigger, list item reordering, route transition |
| **Attention hierarchy** | Directs the eye to what changed | A single new row highlighting, an error field shaking once |

"It looks nice" is not on that list. Neither is "the page felt empty." A hero section that
fades in on load answers nothing — the user asked for the page, and the page arriving is not
news. Scroll-triggered reveals on every section are the most common example of decoration
sold as design: they delay content the user is actively trying to read.

**This purpose test is calibrated for product UI**, where the interface supports a task the
user is already trying to do. A marketing or showcase site's hero is a different brief — the
entrance itself is part of what's being communicated, not an obstacle in front of a task —
and runs under `showcase-motion`'s rules instead, not this section's. Confirm which brief is
actually in front of you (`director` Step 1, or `showcase-motion` Step 1) before applying
this rule to a hero section; applying it unconditionally is itself a mistake.

The reverse is also a bug. If a modal appears with no transition, continuity is broken and
the user has to re-orient. Motion earns its place exactly where a state change would
otherwise be discontinuous.

## Step 3 — Know the defaults you are overriding

This is the step people skip, and it is why inconsistency appears without anyone choosing
it. Motion has **no single default transition**. It picks one per animated value
(`packages/motion-dom/src/animation/utils/default-transitions.ts`):

| Animated value | Default transition |
|---|---|
| `opacity`, `backgroundColor`, any non-transform | keyframes, `duration: 0.3`, `ease: [0.25, 0.1, 0.35, 1]` |
| `x`, `y`, `rotate`, other transforms | spring, `stiffness: 500`, `damping: 25` |
| `scale`, `scaleX`, `scaleY` | critically damped spring, `stiffness: 550`, `damping: 30` (or `2√550 ≈ 46.9` when the target is `0`) |
| Any value with more than 2 keyframes | keyframes, `duration: 0.8` |

Read that table again with a normal card entrance in mind — `{ opacity: 1, y: 0 }` with no
`transition`. The opacity runs a 0.3s eased curve. The `y` runs an underdamped spring that
does not finish at the same moment and slightly overshoots. **The two halves of one
animation disagree.** Nobody chose that; it is what "just don't set a transition" buys.

Springs are also where the bounce comes from. Motion's transform defaults are springy on
purpose — that is the library's house style, and it is a fine house style for a product that
wants to feel playful. It is the wrong default for a data-dense admin tool, and it is
never something to inherit by accident.

## Step 4 — Define the token layer

Same rule as color and spacing: the values live in one file, and no screen invents its own.

```ts
// src/styles/motion.ts
import type { Transition } from "motion/react"

export const duration = {
  micro: 0.15,   // hover, press, focus — feedback the user should barely notice
  base: 0.25,    // dropdowns, tooltips, accordions, toasts
  page: 0.4,     // route changes, full-screen overlays
} as const

export const easing = {
  enter: [0.215, 0.61, 0.355, 1],   // ease-out: fast start, settles
  exit: [0.645, 0.045, 0.355, 1],   // ease-in-out
} as const

export const transition = {
  micro: { duration: duration.micro, ease: easing.enter },
  base: { duration: duration.base, ease: easing.enter },
  page: { duration: duration.page, ease: easing.enter },
} as const satisfies Record<string, Transition>
```

Then set the tree default once, at the root:

```jsx
<MotionConfig transition={transition.base} reducedMotion="user">
  {children}
</MotionConfig>
```

`MotionConfig` merges with any parent `MotionConfig`, so a subtree can override the default
without losing the rest of the config.

Rules that come with the token layer:

- **Two easing curves, maximum.** One for things arriving, one for things leaving. Five
  curves is indistinguishable from no curve system at all.
- **Nothing exceeds `duration.page`** unless it is a deliberate, once-per-session hero
  moment that someone signed off on.
- **Distances stay small: 4–8px of translate.** A card sliding 40px is not more expressive,
  it is slower. Scale entrances start at `0.98`, not `0.8`.
- **If you want a spring, size it in time.** Use `visualDuration` and `bounce` rather than
  `stiffness`/`damping`, because `visualDuration` is expressed in seconds and coordinates
  with the duration tokens. `bounce: 0` is no bounce, `1` is extreme; when `duration` is
  set, `bounce` defaults to `0.25`. Setting `stiffness`, `damping` or `mass` **overrides**
  `bounce` and `duration`, which is the usual reason a "tuned" spring ignores its duration.
- **Stagger tightly.** `stagger()` defaults to `0.1s` per item — visibly slow past about
  six items. Use `0.03–0.05` and cap how many elements participate; a 40-row table staggering
  at any value is a two-second wait for data.

Full config shape, spring recipes and the stagger helper are in `references/motion-tokens.md`.

## Step 5 — `prefers-reduced-motion` is OFF by default. This is the important one.

Motion does **not** respect the OS reduced-motion setting unless told to. The context
default in `packages/framer-motion/src/context/MotionConfigContext.tsx` is:

```ts
export const MotionConfigContext = createContext<MotionConfigContext>({
    transformPagePoint: (p) => p,
    isStatic: false,
    reducedMotion: "never",   // ← this
})
```

`"never"` means *never reduce*. Read that as the plain-language claim it is: **Motion does
not honour `prefers-reduced-motion` automatically, and it never will unless a developer
writes the opt-in.** A project that installs Motion, animates everything, and ships has
built an interface that ignores a documented accessibility preference — and nothing in the
library warns about it in production. This is not a preference the library gets right by
default and you occasionally tune; it is off until you turn it on.

Which makes the opt-in **required boilerplate, not a recommendation.** Every application
using Motion sets `reducedMotion="user"` on a root-level `MotionConfig`, in the same commit
that installs the package. Treat a project without it as incomplete setup, the same way a
missing `<html lang>` is incomplete setup — the token layer in Step 4 and this prop are the
same one-time root wiring, so they go in together.

`"user"` follows the OS setting. `"always"` forces reduction regardless — that is the value
to bind to an in-app "reduce animation" toggle, layered on top of the OS preference. Never
ship `"never"`; it is the default only because it is the backwards-compatible one.

### The root wiring, in full

**Next.js App Router** — the provider is a small client component so the layout stays a
Server Component:

```jsx
// src/app/providers.tsx
"use client"

import { MotionConfig } from "motion/react"
import { transition } from "@/styles/motion"

export function MotionProvider({ children }: { children: React.ReactNode }) {
  return (
    <MotionConfig transition={transition.base} reducedMotion="user">
      {children}
    </MotionConfig>
  )
}
```

```jsx
// src/app/layout.tsx — stays a Server Component
import { MotionProvider } from "./providers"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <MotionProvider>{children}</MotionProvider>
      </body>
    </html>
  )
}
```

**Vite / CRA / Remix** — same provider, no client boundary to worry about:

```jsx
// src/App.tsx
import { MotionConfig } from "motion/react"
import { transition } from "./styles/motion"

export default function App() {
  return (
    <MotionConfig transition={transition.base} reducedMotion="user">
      <Router />
    </MotionConfig>
  )
}
```

It goes **above the router**, not inside a page. A `MotionConfig` mounted per route only
covers that route's subtree, and every animation outside it silently falls back to
`"never"`.

What `"user"` (or `"always"`) actually does, verified in
`motion-dom/src/animation/interfaces/visual-element-target.ts`: values in `positionalKeys`
are given `{ type: false }` — set instantly, no animation. That set is every transform prop
plus `width`, `height`, `top`, `left`, `right`, `bottom`. **Opacity and color still
animate.** That is correct behaviour, not an oversight: the preference is about motion, and
a cross-fade is the standard substitute for a slide.

It also means the library only covers what it controls. Autoplaying background video,
parallax, looping marquees, and CSS animations written by hand are still yours to guard.

Two implementation caveats worth knowing before relying on `useReducedMotion()`:

- **It does not update after mount.** The JSDoc claims *"It will actively respond to changes
  and re-render your components"*, but the implementation is
  `const [shouldReduceMotion] = useState(prefersReducedMotion.current)` with a `TODO` in the
  source about exactly this. Trust the `MotionConfig` path over the hook for anything that
  must react to a mid-session change.
- **It returns `null` on the server.** Do not branch *rendered markup* on it — that is a
  hydration mismatch. Branch *animation values* instead, which is what the library's own
  example does: `const closedX = shouldReduceMotion ? 0 : "-100%"`.

Full patterns, the CSS-side guard, and what the library does not cover are in
`references/reduced-motion.md`.

## Step 6 — React Server Components and bundle cost

Every React entry point in the library is a client module — 66 source files carry
`"use client"`, including `motion/index.tsx` where components are created. So in a Next.js
App Router project:

- `<MotionConfig>` goes in a **small client provider component** imported by the root
  layout. Do not mark the whole layout `"use client"` to get one provider.
- `motion/react` uses a `Proxy` to create `motion.div` on demand. When you want motion
  elements directly inside a Server Component, import from **`motion/react-client`**, which
  exports pre-created components (`a`, `div`, `button`, …) with no proxy. Event handlers and
  variant functions still cannot cross the server/client boundary — those need a client
  component regardless of entry point.

On size, the repo enforces CI budgets in `packages/framer-motion/package.json` (values as
declared there): the full `motion` component bundle is capped at **34.9 kB** and the `m`
component at **6 kB**, with feature bundles at **17.85 kB** (`domAnimation`) and **29.8 kB**
(`domMax`). The ratio is the point — `m` plus `LazyMotion` is roughly a fifth of the full
component, and on a marketing page where animation is not needed for first paint it is worth
the extra wiring. In an app shell that animates constantly, it is not.

Setup for both, plus `AnimatePresence` mode selection, is in `references/react-integration.md`.

## Step 7 — Align with the component library's timing, do not compete with it

The component library is already animating things — dropdowns, collapses, ripples — on its
own timings. Two timing systems in one product is visible.

**Ant Design (6.6.1)** exposes motion in its theme seed. Verified in
`components/theme/themes/seed.ts` and `shared/genCommonMapToken.ts`:

| Token | Default |
|---|---|
| `motionUnit` / `motionBase` | `0.1` / `0` |
| `motionDurationFast` | `0.1s` (`motionBase + motionUnit`) |
| `motionDurationMid` | `0.2s` |
| `motionDurationSlow` | `0.3s` |
| `motionEaseOut` | `cubic-bezier(0.215, 0.61, 0.355, 1)` |
| `motionEaseInOut` | `cubic-bezier(0.645, 0.045, 0.355, 1)` |

Those are the numbers in the Step 4 token file, and that is deliberate — **derive the Motion
tokens from the AntD seed rather than inventing a parallel scale.** If the seed is
customised, the Motion tokens change with it.

**Ant Design has no reduced-motion handling at all** — zero files under `components/`
reference `prefers-reduced-motion`. But the token maths gives a clean lever: setting
`motionUnit: 0, motionBase: 0` makes all three duration tokens `"0.0s"`, disabling AntD's
own animation. Wire that to the same signal driving `MotionConfig`.

**DaisyUI (5.7.20)** is the opposite: it exposes no duration or easing theme variable, but
it does guard its own animations — **14 of its 61 component stylesheets** wrap animation in
`@media (prefers-reduced-motion: no-preference)`. So DaisyUI partially handles itself and
gives you nothing to configure. The Motion token file is the single source of truth, and
Tailwind `duration-*` / `ease-*` utilities should map to it rather than being picked per
element.

Both mappings, with the reduced-motion-aware `ConfigProvider`, are in
`references/motion-tokens.md`. Library choice and theming itself belong to
`component-library-advisor`.

## Step 8 — Subtlety rules that survive review

- **Animate `transform` and `opacity`.** Not `width`, `height`, `top`, `left` — they force
  layout on every frame, and they are exactly the properties reduced motion switches off, so
  you get the worst of both.
- **Entrance animations run once.** Scroll reveals use `viewport={{ once: true }}`. Content
  that re-animates every time it scrolls back into view is a reliable slop signal.
- **No infinite loops except genuine loading indicators.** A pulsing card is not "alive", it
  is a distraction the user cannot dismiss.
- **One thing moves at a time.** If three regions animate on the same event, the eye has no
  hierarchy to follow, which defeats the only reason to animate.
- **Exit animations are shorter than entrances.** Roughly 0.7–0.8× — the user has already
  decided to leave; making them wait for it reads as sluggish.
- **`AnimatePresence` mode is a real decision**: `"wait"` for route and page swaps so the old
  screen leaves before the new one arrives; `"popLayout"` for lists where removal would
  otherwise cause a jump; `"sync"` (the default) only when overlap is intended.
- **Hover-only motion needs a non-hover equivalent.** Touch has no hover; anything conveying
  state must also be visible without it.

## Step 9 — Make animation deterministic in tests and Storybook

Animation breaks visual regression testing unless it is switched off. Motion ships the
switch: `<MotionConfig skipAnimations>` sets values instantly for the whole tree, and
`MotionGlobalConfig.skipAnimations` does it globally without a provider. Put it in the
Storybook preview decorator and the visual test setup — not in application code behind an
env check.

`storybook-workflow` covers the decorator wiring and Chromatic setup.

## Step 10 — Verify before declaring done

- Imports come from `motion/react`; no direct `framer-motion` dependency was added
  alongside `motion`.
- Every animation in the diff answers feedback, continuity, or attention hierarchy — and the
  ones that answer nothing were removed, not shortened.
- Durations and easings come from the token file. No `duration: 0.3` typed inline, no
  fourth easing curve.
- `MotionConfig reducedMotion="user"` is present at the application root — above the
  router, not inside a page. This is required, not optional; its absence means the app
  ignores the OS preference entirely.
- The non-Motion motion sources (video autoplay, parallax, CSS keyframes, the component
  library) are guarded too — `MotionConfig` does not reach them.
- Nothing branches server-rendered markup on `useReducedMotion()`.
- Timing matches the component library's own tokens rather than running a second scale.
- Animated properties are `transform` and `opacity`; no layout-triggering property is
  animated without a stated reason.
- Scroll reveals are `once: true`; no infinite loop animates outside a loading state.
- Visual regression runs with animations skipped.
