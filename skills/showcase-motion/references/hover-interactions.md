# Hover and cursor interactions

Verified against `gsap@3.15.0` `gsap-core.js` (`quickTo`, confirmed core);
own Vite+React+TS reference build, touch-emulation-tested 2026-08-29 (375×812
mobile viewport, touch input, both light and dark color scheme).

## Contents

- Why `quickTo()`, not a fresh tween per event
- Magnetic button, full recipe
- The touch guard, verified by actual test
- Cursor-follow (a magnetic effect with no bounded target)
- Tilt-on-hover
- What still needs a project-specific check

## Why `quickTo()`, not a fresh tween per event

`mousemove` fires far more often than an animation needs new instructions —
calling `gsap.to()` on every event creates and immediately supersedes a new
tween dozens of times per second. `gsap.quickTo(target, property, vars)` —
confirmed a core method, not a plugin, reading `gsap-core.js` directly —
creates **one** paused tween up front and returns a function that calls the
tween's own `resetTo()` internally on every invocation, updating the same
tween's target value instead of creating new ones:

```js
const xTo = gsap.quickTo(el, 'x', { duration: 0.4, ease: 'power3' })
// later, as often as needed:
xTo(newValue)
```

This is the correct primitive for anything driven by a high-frequency event
— `mousemove`, `scroll` (outside `ScrollTrigger`'s own optimized path),
pointer tracking of any kind.

## Magnetic button, full recipe

```jsx
function MagneticButton({ children }: { children: React.ReactNode }) {
  const btnRef = useRef<HTMLButtonElement>(null)
  const container = useRef<HTMLDivElement>(null)

  useGSAP(
    () => {
      const btn = btnRef.current
      if (!btn) return

      const xTo = gsap.quickTo(btn, 'x', { duration: 0.4, ease: 'power3' })
      const yTo = gsap.quickTo(btn, 'y', { duration: 0.4, ease: 'power3' })

      const strength = 0.35 // how strongly the button chases the cursor — 0.2-0.4 is typical

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
    },
    { scope: container },
  )

  return (
    <div ref={container}>
      <button ref={btnRef} className="magnetic-btn">
        {children}
      </button>
    </div>
  )
}
```

`strength` below `1` means the button moves *less* than the cursor's
distance from center — it "chases" without fully tracking, which reads as
magnetic rather than as the button simply following the mouse 1:1 (which
looks like a bug, not an effect).

## The touch guard, verified by actual test

Every hover-driven interaction needs a non-hover fallback — this isn't a
hedge, it's tested behavior, not received wisdom:

```css
.magnetic-btn { background: #e63946; } /* the resting state, always applied */

@media (hover: hover) {
  .magnetic-btn:hover {
    background: #0a0a0a;
    color: #e63946;
    transform: scale(1.08);
  }
}

.magnetic-btn:active {
  background: #ffb703; /* the touch-accessible feedback state */
}
```

`@media (hover: hover)` only matches devices that can genuinely hover — a
mouse or trackpad — and excludes touch-only devices. **Verified directly**:
on a touch-emulated viewport, tapping this button correctly triggered only
the `:active` state (the amber background) — the guarded `:hover` rule (the
black/red state) never applied and never got stuck the way an *unguarded*
`:hover` rule does on touch devices (where a tap can trigger `:hover`
without a corresponding "un-hover" event, leaving the hover style applied
until the next unrelated tap clears it — the classic "sticky hover" mobile
bug this guard exists to prevent).

The JS-driven `mousemove` listener in the magnetic-button recipe above has
the same requirement implicitly: `mousemove` simply doesn't fire from touch
input, so the button's resting position is what touch users see and
interact with — no separate touch-specific code is needed there, but it's
worth confirming the resting (non-magnetic) state is itself a complete,
usable button, not a half-finished visual that assumes the magnetic motion
will always be present.

## Cursor-follow (a magnetic effect with no bounded target)

Structurally identical to the magnetic button, tracked against the viewport
instead of one element's bounds:

```jsx
useGSAP(() => {
  const cursor = cursorRef.current
  const xTo = gsap.quickTo(cursor, 'x', { duration: 0.3, ease: 'power3' })
  const yTo = gsap.quickTo(cursor, 'y', { duration: 0.3, ease: 'power3' })

  const handleMove = (e: MouseEvent) => {
    xTo(e.clientX)
    yTo(e.clientY)
  }

  window.addEventListener('mousemove', handleMove)
  return () => window.removeEventListener('mousemove', handleMove)
}, { scope: container })
```

A custom cursor is a hover-only device concept by definition — hide it
entirely outside `@media (hover: hover)` rather than rendering a dead
element on touch devices, and never remove or hide the browser's native
cursor without confirming the custom one is actually rendering (a common
failure mode: the native cursor gets hidden via `cursor: none` globally, and
if the custom cursor's mount or event wiring fails silently, the page is
left with no visible cursor at all).

## Tilt-on-hover

A CSS-transform technique, not a GSAP-specific one — perspective tilt based
on cursor position relative to the element:

```jsx
const handleMove = (e: MouseEvent) => {
  const rect = el.getBoundingClientRect()
  const px = (e.clientX - rect.left) / rect.width - 0.5 // -0.5 to 0.5
  const py = (e.clientY - rect.top) / rect.height - 0.5

  gsap.to(el, {
    rotateY: px * 12, // degrees — keep this small, 8-15° reads as "responsive," more reads as gimmicky
    rotateX: -py * 12,
    transformPerspective: 800,
    duration: 0.4,
    ease: 'power2.out',
  })
}
```

Requires `transform-style: preserve-3d` on the element (or its immediate
parent, depending on whether child content should tilt with it) for the
perspective to render correctly — a common reason this recipe "does
nothing visually" despite the tween running without error.

## What still needs a project-specific check

None of the recipes above have been checked against a design system's
existing interaction states (e.g. a component library's own `:hover`/`:focus`
styles) — layering a magnetic or tilt effect on top of `component-library-advisor`'s
chosen library's own interactive states needs its own look before shipping,
the same way `motion-principles` Step 7 says to align timing with the
component library rather than run a second, competing system.
