# GSAP vs. Motion for showcase work

Verified against `gsap@3.15.0` — `package.json`, `README.md`,
`gsap.com/standard-license`; `framer-motion@13.1.1`; `split-type@0.3.4`.

## Contents

- The license, in full — what's actually restricted
- API-depth comparison, for the specific techniques this skill covers
- `SplitText` vs `split-type`, now that both are viable
- When the answer is genuinely "both"

## The license, in full — what's actually restricted

Read directly from `gsap.com/standard-license` and GSAP's own FAQ, not
inferred from "it's free now":

- **Commercial use is unrestricted.** The FAQ states it plainly: *"Can I
  really use GSAP in commercial projects without paying anything? Yes,
  really!"*
- **Every formerly Club-GreenSock-exclusive plugin is included** —
  `SplitText`, `MorphSVG`, `DrawSVG`, `ScrollSmoother`, `Draggable`, `Flip`,
  `Physics2D`, and the rest. Confirmed physically: all of these `.js` files
  are present in the base `gsap` npm package, not gated behind a separate
  paid install.
- **Proprietary notices can't be stripped.** The license grants a
  "non-exclusive, worldwide license to use, reproduce, display, and
  implement GSAP Products" but explicitly forbids removing or altering
  "proprietary notices or branding."
- **The one real carve-out: no-code visual animation tools that compete
  with Webflow.** GSAP's own FAQ names this directly — building "tools that
  allow users to build visual animations without code" that compete with
  Webflow's animation-authoring product is a Prohibited Use requiring prior
  written consent. The same FAQ clarifies the boundary is about *competing*
  products, not about using GSAP inside any tool: *"We want to encourage
  developers to build on top of GSAP, including visual tools that don't
  directly compete with Webflow's rich animation-building capabilities."*

For every case this skill is written for — building a site, not building an
animation-authoring SaaS — none of this is a live concern. It's worth
knowing the actual boundary rather than repeating "it's free" as an
unqualified claim, especially the notice-preservation term if GSAP output is
ever redistributed as part of a template or starter kit sold to others.

## API-depth comparison, for the specific techniques this skill covers

| Technique | GSAP | Motion |
|---|---|---|
| Character/word/line text splitting | `SplitText` — native, `Intl.Segmenter`-based, returns `chars`/`words`/`lines` arrays | No equivalent in the library itself; would need a third-party splitter plus manual `motion.span` wrapping per unit |
| Multi-element relative sequencing | `gsap.timeline()` with explicit position parameters (`"+=0.2"`, absolute time, labels) — built for this | Achievable via `variants` + `staggerChildren`/`delayChildren`, but no equivalent to a timeline's arbitrary relative-offset API across unrelated elements |
| High-frequency pointer tracking | `gsap.quickTo()` — purpose-built, reuses one tween via `resetTo()` instead of creating tweens per event | `useMotionValue` + `useSpring`/`animate()` on mousemove — comparable performance characteristics, more manual wiring |
| Declarative React state-driven animation | Possible via `useGSAP` + refs, always imperative underneath | Native strength — `animate`/`initial`/`exit` props map directly to component state |

Neither column is "better" in general — they're better at different halves
of what a showcase site needs, which is exactly why Step 2 of the main skill
frames this as a per-effect decision, not a standing library choice the way
`component-library-advisor` treats Ant Design vs. DaisyUI.

## `SplitText` vs. `split-type`, now that both are viable

Before GSAP's plugins went free, `split-type` (currently 0.3.4, ISC license)
was the default answer for text splitting without a GSAP dependency or cost.
That tradeoff has changed:

- `SplitText` splits using `Intl.Segmenter` where the environment supports
  it — confirmed reading the shipped source — which correctly handles
  grapheme clusters: emoji with modifiers, combined characters, and other
  multi-codepoint sequences that a plain string/regex splitter can visually
  or functionally break apart mid-character.
- `split-type` is a simpler, dependency-light splitter with no such
  Unicode-awareness guarantee, and it doesn't return the same rich object
  (`chars`/`words`/`lines` as live arrays, `mask` option) that `SplitText`
  does.

**Default to `SplitText` whenever GSAP is already a dependency** — which,
given Step 2's guidance to reach for GSAP whenever text is the centerpiece,
is most of the time this skill applies at all. Reach for `split-type` only
when the project is deliberately avoiding a GSAP install for one effect and
Motion (or CSS) is driving the actual animation on the split output.

## When the answer is genuinely "both"

A project can legitimately run GSAP for the showcase surfaces
(`SplitText`-driven hero, `ScrollTrigger`-pinned case studies) and Motion for
the product app behind the marketing site — different routes, different
bundles, no conflict. What doesn't work is loading both into the same
screen "just in case" — two animation libraries competing for the same
element's timing is a maintenance cost with no matching benefit. Pick per
surface, the same rule `component-library-advisor` states for mixing
component libraries.
