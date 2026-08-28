# Motion Tokens

Everything here is verified against `motiondivision/motion@1b037b0` (v13.1.1),
`ant-design/ant-design@6.6.1` and `saadeghi/daisyui@5.7.20`.

## Why a token file exists

Motion resolves a *different* default transition per animated value. From
`packages/motion-dom/src/animation/utils/default-transitions.ts`:

```ts
const underDampedSpring = { type: "spring", stiffness: 500, damping: 25, restSpeed: 10 }

const criticallyDampedSpring = (target) => ({
    type: "spring",
    stiffness: 550,
    damping: target === 0 ? 2 * Math.sqrt(550) : 30,
    restSpeed: 10,
})

const keyframesTransition = { type: "keyframes", duration: 0.8 }

const ease = { type: "keyframes", ease: [0.25, 0.1, 0.35, 1], duration: 0.3 }

export const getDefaultTransition = (valueKey, { keyframes }) => {
    if (keyframes.length > 2) return keyframesTransition
    if (transformProps.has(valueKey))
        return valueKey.startsWith("scale")
            ? criticallyDampedSpring(keyframes[1])
            : underDampedSpring
    return ease
}
```

The source comment on `ease` describes `[0.25, 0.1, 0.35, 1]` as *"a slightly shallower
version of the default browser easing curve."*

Consequence: `animate={{ opacity: 1, y: 0 }}` with no `transition` runs a 0.3s eased opacity
next to an underdamped spring on `y`. They neither share a duration nor a shape. Any project
that animates more than a handful of elements needs the token layer for this reason alone.

## The token file

```ts
// src/styles/motion.ts
import type { Transition } from "motion/react"

export const duration = {
  micro: 0.15,
  base: 0.25,
  page: 0.4,
} as const

export const easing = {
  enter: [0.215, 0.61, 0.355, 1],
  exit: [0.645, 0.045, 0.355, 1],
} as const

export const transition = {
  micro: { duration: duration.micro, ease: easing.enter },
  base: { duration: duration.base, ease: easing.enter },
  page: { duration: duration.page, ease: easing.enter },
  exit: { duration: duration.base * 0.75, ease: easing.exit },
} as const satisfies Record<string, Transition>

export const distance = {
  nudge: 4,
  slide: 8,
} as const
```

Exit is shorter than entry on purpose — the user has already decided to dismiss.

## Wiring `MotionConfig`

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

`MotionConfig` spreads the parent context before its own props
(`config = { ...parentConfig, ...config }`), so nesting a provider overrides only what it
names. Its `transition` is resolved against the parent's via `resolveTransition`.

**Memoisation caveat.** The context object is memoised on:

```ts
[JSON.stringify(config.transition), config.transformPagePoint,
 config.reducedMotion, config.skipAnimations, config.isValidProp]
```

`transition` is compared by serialised value, so an inline object literal is safe. But
`transformPagePoint` and `isValidProp` are compared **by reference** — pass module-level
constants, never inline arrow functions, or every `motion` component in the tree re-renders
on each parent render.

## Springs sized in time

Do not hand-tune `stiffness`/`damping` when the goal is coordination with time-based
tokens. From `packages/motion-dom/src/animation/types.ts`:

- `visualDuration` — seconds until the animation *visually appears* to reach its target;
  the bounce mostly happens after. **Overrides `duration`.**
- `bounce` — `0` (none) to `1` (extreme). Defaults to `0.25` when `duration` is set.
- Setting `stiffness`, `damping`, or `mass` **overrides both `bounce` and `duration`.** This
  is the usual reason a spring ignores the duration someone set next to it.

```ts
export const spring = {
  // Feels responsive, does not overshoot — the safe default for UI.
  crisp: { type: "spring", visualDuration: 0.25, bounce: 0 },
  // Deliberately playful. Use once, on one element, on purpose.
  playful: { type: "spring", visualDuration: 0.4, bounce: 0.35 },
} as const satisfies Record<string, Transition>
```

`bounce: 0` on a spring is not the same as an eased tween — it still carries velocity
through, which is what makes drag-release and layout animations feel physical. That is the
one place a spring beats a duration.

## Stagger

`stagger()` lives in `motion-dom/src/utils/stagger.ts` and is re-exported from
`motion/react`:

```ts
stagger(duration = 0.1, { startDelay = 0, from = 0, ease })
```

`from` accepts `"first" | "last" | "center" | number`. The **default of `0.1s` per item is
too slow for lists** — six items already costs half a second before the last one starts.

```jsx
const list = {
  visible: { transition: { delayChildren: stagger(0.04, { startDelay: 0.05 }) } },
}
const item = {
  hidden: { opacity: 0, y: distance.slide },
  visible: { opacity: 1, y: 0 },
}
```

Cap participation. Data tables and search results should not stagger at all — the user is
waiting for information, not choreography.

## Mapping to Ant Design (6.6.1)

AntD derives durations from two seed values in `components/theme/themes/seed.ts`, expanded
in `components/theme/themes/shared/genCommonMapToken.ts`:

```ts
motionUnit: 0.1,
motionBase: 0,

motionDurationFast: `${(motionBase + motionUnit).toFixed(1)}s`,      // "0.1s"
motionDurationMid:  `${(motionBase + motionUnit * 2).toFixed(1)}s`,  // "0.2s"
motionDurationSlow: `${(motionBase + motionUnit * 3).toFixed(1)}s`,  // "0.3s"
```

Easing seeds, verbatim:

| Seed token | Value |
|---|---|
| `motionEaseOut` | `cubic-bezier(0.215, 0.61, 0.355, 1)` |
| `motionEaseInOut` | `cubic-bezier(0.645, 0.045, 0.355, 1)` |
| `motionEaseOutCirc` | `cubic-bezier(0.08, 0.82, 0.17, 1)` |
| `motionEaseInOutCirc` | `cubic-bezier(0.78, 0.14, 0.15, 0.86)` |
| `motionEaseOutBack` | `cubic-bezier(0.12, 0.4, 0.29, 1.46)` |
| `motionEaseInBack` | `cubic-bezier(0.71, -0.46, 0.88, 0.6)` |
| `motionEaseInQuint` | `cubic-bezier(0.755, 0.05, 0.855, 0.06)` |
| `motionEaseOutQuint` | `cubic-bezier(0.23, 1, 0.32, 1)` |

Generate the Motion tokens from the same source so a seed change moves both:

```ts
// src/styles/motion.ts
import { theme } from "antd"

const { getDesignToken } = theme
const antd = getDesignToken()   // reflects the app's ConfigProvider seed

export const duration = {
  micro: parseFloat(antd.motionDurationFast),   // 0.1
  base: parseFloat(antd.motionDurationMid),     // 0.2
  page: parseFloat(antd.motionDurationSlow) + 0.1,
} as const
```

If that indirection is more machinery than the project wants, hardcode the same numbers and
leave a comment naming the AntD tokens they mirror. What is not acceptable is a second,
unrelated scale.

### Reduced motion with Ant Design

**AntD 6.6.1 has no `prefers-reduced-motion` handling** — zero files under `components/`
reference it, and there is no `motion: false` switch on `ConfigProvider` or in the token
interfaces. The token maths is the lever: `motionUnit: 0, motionBase: 0` makes all three
duration tokens `"0.0s"`.

```jsx
"use client"

import { ConfigProvider } from "antd"
import { MotionConfig, useReducedMotion } from "motion/react"

const STILL = { motionUnit: 0, motionBase: 0 }

export function AppProviders({ children }) {
  const shouldReduceMotion = useReducedMotion()

  return (
    <ConfigProvider
      theme={{ token: { ...seedTokens, ...(shouldReduceMotion ? STILL : {}) } }}
    >
      <MotionConfig transition={transition.base} reducedMotion="user">
        {children}
      </MotionConfig>
    </ConfigProvider>
  )
}
```

`useReducedMotion()` returns `null` during SSR, so the server renders the animated theme and
the client corrects on first render. That is a style-value difference, not a markup
difference — no hydration mismatch. It also does not update mid-session; see
`reduced-motion.md`.

## Mapping to DaisyUI (5.7.20)

DaisyUI is the mirror image of AntD here:

- **No duration or easing theme variable.** Its theme files define `--depth`, `--noise`,
  radius and size variables; timing is written inline in each component's CSS.
- **It does guard itself, partially.** 14 of the 61 component stylesheets under
  `packages/daisyui/src/components/` wrap animation in
  `@media (prefers-reduced-motion: no-preference)` — including `tooltip`, `collapse`,
  `progress`, `radio`, `skeleton`, `loading`, `carousel`, `toast`. The other 47 do not, so
  DaisyUI's coverage is real but not complete.

So with DaisyUI the Motion token file *is* the system, and Tailwind utilities read from it:

```css
/* app.css */
:root {
  --motion-micro: 150ms;
  --motion-base: 250ms;
  --motion-page: 400ms;
  --motion-ease-enter: cubic-bezier(0.215, 0.61, 0.355, 1);
  --motion-ease-exit: cubic-bezier(0.645, 0.045, 0.355, 1);
}
```

Then `duration-[var(--motion-base)] ease-[var(--motion-ease-enter)]` in class names, and the
same numbers in `src/styles/motion.ts`. Never a one-off `duration-[237ms]`.

Tailwind v4 also has an `--ease-*` theme namespace that turns `@theme { --ease-enter: … }`
into a real `ease-enter` utility, which is nicer to read. Check it against the installed
Tailwind version before relying on it — the arbitrary-value form above works on v3 and v4
either way.

Library selection and the rest of the theme layer belong to `component-library-advisor`.
