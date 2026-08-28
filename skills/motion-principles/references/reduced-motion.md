# Reduced Motion

Verified against `motiondivision/motion@1b037b0` (v13.1.1).

## The default is "never"

`packages/framer-motion/src/context/MotionConfigContext.tsx`:

```ts
export const MotionConfigContext = createContext<MotionConfigContext>({
    transformPagePoint: (p) => p,
    isStatic: false,
    reducedMotion: "never",
})
```

`ReducedMotionConfig` is `"always" | "never" | "user"`. The default `"never"` means *never
reduce* — the OS setting is ignored until someone opts in. There is no production warning;
the only signal is a dev-only `warnOnce` that fires when reduced motion **is** active,
telling the developer their animations may look wrong.

**The opt-in is one prop on the root provider, and it is mandatory setup — not a
recommendation.** Without it the application ignores `prefers-reduced-motion` completely, no
matter how careful the individual animations are:

```jsx
<MotionConfig reducedMotion="user">
```

It belongs above the router, in the same root provider that carries the transition tokens.
A `MotionConfig` mounted per route covers only that subtree; everything outside it falls
back to `"never"`. SKILL.md Step 5 has the full Next.js App Router and Vite wiring.

`"always"` forces reduction regardless of OS setting — that is the value to bind to a
user-facing "reduce animation" toggle in app settings, layered on top of the OS preference.
`"never"` should never appear in application code; it exists as the default only for
backwards compatibility.

## What reduction actually does

`packages/motion-dom/src/animation/interfaces/visual-element-target.ts`:

```ts
const shouldReduceMotion = reduceMotion ?? visualElement.shouldReduceMotion

value.start(
    animateMotionValue(
        key,
        value,
        valueTarget,
        shouldReduceMotion && positionalKeys.has(key)
            ? { type: false }          // set instantly, no animation
            : valueTransition,
        visualElement,
        isHandoff
    )
)
```

`positionalKeys` (`motion-dom/src/render/utils/keys-position.ts`) is:

```ts
new Set(["width", "height", "top", "left", "right", "bottom", ...transformPropOrder])
```

So under reduced motion:

| Property | Behaviour |
|---|---|
| `x`, `y`, `scale`, `rotate`, `skew`, all transforms | **Set instantly** |
| `width`, `height`, `top`, `left`, `right`, `bottom` | **Set instantly** |
| `opacity` | Still animates |
| `backgroundColor`, `color`, `borderColor`, filters | Still animate |

This is the correct interpretation of the preference, not a gap. WCAG 2.3.3 (Animation from
Interactions) is about motion, and a cross-fade is the standard substitute for a slide. It
does mean the visual language should be designed so that **removing movement still leaves a
usable transition** — which is another argument for small distances: an 8px slide that
becomes a plain fade loses nothing, a 200px slide loses the whole idea.

Layout animations are covered too: `create-projection-node.ts` checks
`visualElement.shouldReduceMotion` before running a projection animation.

## Resolution order

`useReducedMotionConfig()` (`framer-motion/src/utils/reduced-motion/use-reduced-motion-config.ts`):

```ts
const reducedMotionPreference = useReducedMotion()
const { reducedMotion } = useContext(MotionConfigContext)

if (reducedMotion === "never") return false
else if (reducedMotion === "always") return true
else return reducedMotionPreference
```

`VisualElement.mount()` runs the same logic and only initialises the `matchMedia` listener
when the config is `"user"` — so `"never"` and `"always"` never touch `matchMedia` at all.

The media query used is `window.matchMedia("(prefers-reduced-motion)")`
(`motion-dom/src/render/utils/reduced-motion/index.ts`) — the boolean-context form, which
matches when the value is anything other than `no-preference`.

## Two caveats before relying on `useReducedMotion()`

**1. It does not update after mount.** The JSDoc says *"It will actively respond to changes
and re-render your components with the latest setting"* — the implementation does not:

```ts
export function useReducedMotion() {
    !hasReducedMotionListener.current && initPrefersReducedMotion()
    const [shouldReduceMotion] = useState(prefersReducedMotion.current)
    // ...
    /**
     * TODO See if people miss automatically updating shouldReduceMotion setting
     */
    return shouldReduceMotion
}
```

`useState` captures the value once; there is no subscription and no setter. The module-level
`matchMedia` listener keeps `prefersReducedMotion.current` fresh, so newly-mounted components
see the change — already-mounted ones do not. Treat the JSDoc as stale and the `TODO` as the
truth.

For UI that must react to a mid-session OS change, drive it from your own
`matchMedia` subscription and feed `MotionConfig` with `reducedMotion={"always" | "user"}`.

**2. It returns `null` on the server.** `prefersReducedMotion` initialises to
`{ current: null }` and `initPrefersReducedMotion` early-returns when `window` is undefined.

```ts
// Wrong — server renders one tree, client renders another.
{shouldReduceMotion ? <StaticHero /> : <AnimatedHero />}

// Right — same markup, different animation values.
const closedX = shouldReduceMotion ? 0 : "-100%"
<motion.div animate={{ opacity: isOpen ? 1 : 0, x: isOpen ? 0 : closedX }} />
```

The second form is the library's own documented example.

## What Motion does not cover

`MotionConfig` governs Motion animations only. These stay your responsibility:

- **Autoplaying video and GIFs.** Guard with `matchMedia` and drop `autoPlay`, or swap to a
  poster frame.
- **Parallax and scroll-linked effects.** These are the highest-risk category for
  vestibular disorders. Disable them, do not merely shorten them.
- **Looping marquees, tickers, and carousels with autoplay.** Pause them.
- **Hand-written CSS animations and transitions.** Guard them in CSS:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

  A blanket reset like this is a floor, not a design. It flattens intentional cross-fades
  along with everything else. Prefer per-component guards where the component is worth the
  attention, and keep the blanket rule as the catch-all for third-party CSS.

- **The component library.** Ant Design 6.6.1 ships no `prefers-reduced-motion` handling at
  all; DaisyUI 5.7.20 guards 14 of its 61 component stylesheets. See `motion-tokens.md` for
  the AntD `motionUnit: 0` lever.

## Testing it

Toggle the OS setting rather than trusting the code:

- **macOS** — System Settings → Accessibility → Display → Reduce motion
- **Chrome DevTools** — Command menu → "Emulate CSS prefers-reduced-motion: reduce"
- **Playwright** — `test.use({ reducedMotion: "reduce" })`

Then check, with the setting on: no element translates or scales; dialogs and drawers still
cross-fade so state changes stay legible; no video autoplays; no marquee runs; nothing
loops. And with the setting off, confirm the animations still exist — a common regression is
guarding so aggressively that the default experience loses its transitions too.

For deterministic snapshots, use `skipAnimations` rather than reduced motion — it is a
different switch with a different intent. See `react-integration.md`.
