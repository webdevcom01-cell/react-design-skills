# React Integration

Verified against `motiondivision/motion@1b037b0` (v13.1.1).

## Server Components and the client boundary

Every React entry point is a client module. 66 files under `packages/framer-motion/src`
carry `"use client"`, including `motion/index.tsx` where `createMotionComponent` lives, and
`context/MotionConfigContext.tsx`.

Two consequences in a Next.js App Router project:

**1. `MotionConfig` needs its own small client component.** Do not put `"use client"` on the
root layout to get one provider — that pulls the whole layout out of the server tree.

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

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <MotionProvider>{children}</MotionProvider>
      </body>
    </html>
  )
}
```

**2. Use `motion/react-client` for motion elements inside Server Components.**
`motion/react` builds `motion.div` through a `Proxy` (`render/components/create-proxy.ts`),
which the RSC boundary cannot serialise. `motion/react-client` re-exports
`framer-motion/client`, which is a flat list of **pre-created** components
(`render/components/motion/namespace.ts` — `a`, `div`, `button`, `svg`, …, plus `create`):

```jsx
// A Server Component
import * as motion from "motion/react-client"

export default function Hero() {
  return <motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }}>…</motion.div>
}
```

This works for declarative props only. Event handlers (`onAnimationComplete`,
`onViewportEnter`), variant functions, hooks, and `useAnimate` still require a client
component — functions cannot cross the boundary regardless of entry point.

## Bundle size and `LazyMotion`

`packages/framer-motion/package.json` declares these CI budgets (values as written there —
they are enforced ceilings, not measured output):

| Bundle | Budget |
|---|---|
| `size-rollup-motion` — the full `motion` component | 34.9 kB |
| `size-rollup-m` — the `m` component | 6 kB |
| `size-rollup-dom-animation` — `domAnimation` features | 17.85 kB |
| `size-rollup-dom-max` — `domMax` features | 29.8 kB |
| `size-rollup-animate` — vanilla `animate` | 19.1 kB |
| `size-rollup-scroll` — vanilla `scroll` | 5.2 kB |
| `size-rollup-waapi-animate` — mini WAAPI animate | 2.26 kB |

The ratio is what matters: `m` is roughly a fifth of the full component. `m` renders and
animates nothing on its own — features arrive via `LazyMotion`, synchronously or as a
dynamic import.

```jsx
"use client"

import { LazyMotion, m } from "motion/react"

// Lazy — features land in a separate chunk after first paint.
const loadFeatures = () => import("motion/react").then((mod) => mod.domAnimation)

export function Marketing({ children }) {
  return (
    <LazyMotion features={loadFeatures} strict>
      <m.div initial={{ opacity: 0 }} animate={{ opacity: 1 }}>{children}</m.div>
    </LazyMotion>
  )
}
```

Feature bundles, from `render/dom/features-animation.ts` and `features-max.ts`:

- `domAnimation` — `animations` + `gestureAnimations` (`whileHover`, `whileTap`,
  `whileFocus`, `whileInView`). Covers most product UI.
- `domMax` — `domAnimation` + `drag` + `layout`. Only when the page actually drags things or
  runs layout animations.

`strict` is worth turning on: with it, rendering a full `motion` component inside
`LazyMotion` throws an `invariant` instead of silently defeating the split (`motion/index.tsx`
downgrades to a `warning` when strict is off).

**When not to bother.** In an app shell that animates on nearly every screen, `domAnimation`
loads immediately anyway and `LazyMotion` is pure indirection. The split earns its keep on
marketing and content pages where animation is decorative and below the fold.

## `AnimatePresence`

Modes, from `components/AnimatePresence/types.ts`:

| `mode` | Use for |
|---|---|
| `"sync"` (default) | Enter and exit overlap. Correct when they occupy different space |
| `"wait"` | Route and page swaps, tab panels — the old child fully exits first |
| `"popLayout"` | Lists where removal reflows siblings; the exiting child is popped out of flow |

`popLayout` also takes `anchorX` (`"left" | "right"`) and `anchorY` for positioning the
popped element. `initial={false}` suppresses the entrance animation on first mount — use it
whenever the content is present on page load and its arrival is not news.

```jsx
<AnimatePresence mode="wait" initial={false}>
  <motion.main
    key={pathname}
    initial={{ opacity: 0, y: distance.slide }}
    animate={{ opacity: 1, y: 0 }}
    exit={{ opacity: 0 }}
    transition={transition.page}
  >
    {children}
  </motion.main>
</AnimatePresence>
```

Exit animations need a stable `key` and the `AnimatePresence` must not itself unmount — the
most common reason an exit animation "does nothing" is that its parent was removed in the
same render.

## Scroll reveals

`whileInView` takes a `viewport` object (`motion-dom/src/node/types.ts`):

```ts
interface ViewportOptions {
    root?: { current: Element | null }
    once?: boolean
    margin?: string
    amount?: "some" | "all" | number
}
```

```jsx
<motion.section
  initial={{ opacity: 0, y: distance.slide }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, amount: 0.3 }}
  transition={transition.base}
/>
```

`once: true` is not optional in a reviewed codebase. Re-animating on every scroll-back is
distracting and is one of the clearest signals of unconsidered motion. `amount: 0.3` fires
when 30% of the element is visible, which avoids the "it animated while off-screen" problem
on tall sections.

Reveal *sections*, not every element in them. A page where each paragraph fades in
individually reads as a slideshow.

## Deterministic tests and Storybook

Two switches, both real API:

```jsx
// Provider form — scopes to a tree.
<MotionConfig skipAnimations>{children}</MotionConfig>
```

```ts
// Global form — no provider needed. From motion-utils/src/global-config.ts
import { MotionGlobalConfig } from "motion/react"

MotionGlobalConfig.skipAnimations = true
```

`MotionGlobalConfig` also carries `instantAnimations` and `useManualTiming`. In Storybook,
put the provider in `preview.tsx` decorators so every story renders at its final state and
Chromatic snapshots are stable. Do not gate this on `NODE_ENV` inside application code — the
test harness owns the switch.

`storybook-workflow` covers decorator wiring and the Chromatic setup.

## Hooks worth knowing, and when they are the wrong tool

| Hook | Use |
|---|---|
| `useReducedMotion` | Branch animation *values*. See `reduced-motion.md` for its two caveats |
| `useAnimate` | Imperative sequences that declarative props cannot express. Respects `skipAnimations` |
| `useInView` | Needing the boolean without animating — analytics, lazy loading |
| `useScroll` + `useTransform` | Scroll-linked values. Hold this to a very high bar — it is the highest-risk pattern for reduced motion |
| `useSpring` | Smoothing a `MotionValue`, e.g. a cursor follower |

Reaching for `useAnimate` for something `animate` + `variants` already does is the usual
sign of fighting the library. Start declarative; go imperative only when the sequence has
real branching.
