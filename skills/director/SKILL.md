---
name: director
description: Route a React design task through the right skills in the right order — read the project brief, pick the component library, sequence the token layer before screens, pull in Storybook, Penpot, tldraw, and motion only when they actually apply, and run a design and accessibility gate before calling the work done. Use at the start of any React or Next.js design or UI task where the approach is not already decided; whenever someone describes a project and asks how to build the UI, where to start, what stack to use, or what the plan is; whenever several design concerns collide in one request (library plus theme plus animation plus canvas); and whenever a UI task needs a final quality check before shipping. Also covers Serbian phrasings - odakle da krenem, koji je plan, sta mi treba za ovaj projekat, napravi UI za, dizajniraj aplikaciju, vodi me kroz proces, zavrsni pregled dizajna. Do NOT use when the user has already named the specific concern - go straight to that skill instead.
metadata:
  version: "0.1.1"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "the eight sibling skills in this plugin at their stated versions — component-library-advisor@0.1.0, penpot-workflow@0.1.0, storybook-workflow@0.1.0, tldraw-workflow@0.1.0, icon-resources@0.1.0, motion-principles@0.1.0, scroll-choreography@0.1.0, showcase-motion@0.1.0. No external source; routing logic is composed from those skills' own Step 1 decision rules (2026-08-23, extended 2026-08-29 for the two new siblings); spot-checked 2026-08-25 — AntD motion duration tokens (0.1/0.2/0.3s) and DaisyUI's 35 stock themes with exact light/dark primary oklch values in qa-gate.md all confirmed against live package source; 2026-08-29 — corrected the Step 6 QA-gate dependency claim, which had asserted design:design-critique/design:accessibility-review are permanently absent from any local CLI install, a claim independently found false in the session that added the two new skills (both were present as loaded plugin skills)"
---

# Director

This skill routes. It does not decide colors, write themes, or configure Storybook — six
sibling skills do that, each verified against its own sources. The failure this one prevents
is **order**: a project that picks a component library after building three screens, or
discovers a licensing constraint after committing to a feature, has to redo work that was
never wrong, only premature.

Two rules govern everything below.

**Constraints before preferences.** A licensing question, a Tailwind major version, or an
existing dependency can eliminate options. Resolve those first; taste only matters among
options that remain.

**Foundations before surfaces.** The token layer is authored once and every screen reads
from it. Screens built before tokens exist get rebuilt.

## Step 1 — Get the brief. Do not infer it.

Four questions. If the user's request already answers one, do not ask it again; if it
answers none, ask all four in a single message rather than interrogating across turns.

1. **What is being built?** An internal tool, a customer-facing app, a marketing or content
   site, or a reusable component library. If it is more than one thing, they are separate
   routes — say so.
2. **Is there a brand?** Existing colors, type, logo, a design file — or a blank sheet. This
   decides whether the token layer is a translation job or a design job.
3. **Is a shared component library part of the deliverable?** Not "will components be
   reused eventually" — is a package or design system an actual output.
4. **What phase is this?** Nothing written yet, an existing codebase being extended, or a
   built UI being reviewed.

Then read the repository before recommending anything, exactly as
`component-library-advisor` Step 1 requires:

```bash
grep -E '"(react|next|vite|antd|daisyui|tailwindcss|@tailwindcss/[a-z]+|motion|framer-motion|storybook|tldraw)"' package.json
ls tailwind.config.* postcss.config.* .storybook 2>/dev/null
```

An installed dependency outranks a stated preference. Report what was found before
proposing a plan — a plan that contradicts the codebase without acknowledging it reads as
guesswork.

## Step 2 — Answer the blocking question, if there is one

One question can invalidate a plan, so it runs before the plan exists.

**Does the project need a canvas, whiteboard, infinite canvas, drawing surface, diagram
editor, or node editor?**

If yes → **go to `tldraw-workflow` Step 1 immediately, before any other recommendation.**
tldraw is source-available, not open source: free in development, but a production build
without a license key stops rendering the editor after five seconds. That skill's mandatory
first step establishes whether tldraw is a *process tool* (free — the team sketching on
tldraw.com) or a *product feature* (licensed, with a real budget line), and whether the
product is commercial. Internal company tools count as production.

Do not soften this into "we should check the license later." A canvas feature whose
licensing was never raised is a promise the project may not be able to keep, and the answer
changes the architecture, not just the paperwork.

Nothing else in this system has a blocking gate. Everything below is ordering.

## Step 3 — Route

Match the brief to a lane. Most projects are one lane; a project that is genuinely two
things gets two passes, not a blend.

| Brief | Library lane | Also load |
|---|---|---|
| Admin panel, internal tool, CRUD dashboard, data-dense app | **Ant Design** | motion-principles, icon-resources |
| Marketing site, landing page, docs, content product | **DaisyUI** (requires Tailwind) | motion-principles, icon-resources |
| Premium/showcase marketing site, portfolio, case-study or campaign site — the motion itself is part of the pitch | **Neither** — Tailwind + headless primitives, not DaisyUI | **scroll-choreography**, **showcase-motion**, icon-resources |
| Reusable component library or design system package | Depends on consumers — see below | **storybook-workflow**, motion-principles, icon-resources |
| Small set of highly custom surfaces | **Neither** — Tailwind + headless primitives | motion-principles, icon-resources |
| Nothing written yet, structure still unclear | Defer the library | **tldraw-workflow** (Path A), **penpot-workflow** |
| Existing design file, designer in the loop, brand tokens exist | Follow the lane above | **penpot-workflow** (tokens are the source) |
| Canvas / whiteboard / diagram feature | Follow the lane above | **tldraw-workflow** — after Step 2 |
| Any brief with pinned/scrubbed sections, smooth scroll, or a hero/text-reveal centerpiece | Follow the library lane above | **scroll-choreography** and/or **showcase-motion**, in place of or alongside motion-principles — see the rule below |
| Reviewing a UI that already exists | No install | Step 6 QA gate, then targeted skills for what it finds |

Three routing rules that are easy to get wrong:

- **A component library package does not automatically mean Ant Design or DaisyUI.** Ask who
  consumes it. If the consuming apps are data-dense internal tools, a thin wrapper layer over
  Ant Design is the honest answer. If they are varied product surfaces, Tailwind plus
  headless primitives with your own components is better, because a shared library that
  fights its base library's opinions is worse than no shared library. Either way,
  `storybook-workflow` applies.
- **DaisyUI is gated on Tailwind, and the major versions are locked together.** Tailwind v4
  → DaisyUI 5; Tailwind v3 → DaisyUI 4. If the project has no Tailwind, adding it is a real
  cost to name out loud, not a footnote. `component-library-advisor` Step 3 has the gate.
- **"Marketing site" is not automatically the DaisyUI lane.** If the brief signals heavy,
  considered motion as part of the product itself — "premium," "brutal animations,"
  awwwards-style, a portfolio or campaign site where the entrance and scroll experience *are*
  the pitch, not decoration on top of it — DaisyUI's ready-made components fight
  `showcase-motion`'s custom magnetic buttons, tilt states, and `SplitText`-masked headings at
  every turn, the same way a shared component library fights a base library's opinions above.
  Route that brief to Tailwind plus headless primitives instead, and load
  `scroll-choreography` and `showcase-motion` in place of `motion-principles` — both skills
  state clearly in their own Step 1 that they apply to this brief and not to product UI. A
  standard marketing or content site without that signal stays on the DaisyUI lane with
  `motion-principles`' lighter reveal rules; don't over-route a business site into GSAP
  choreography it doesn't need.

Never load `storybook-workflow` for a landing page or a single admin screen. That skill's own
Step 1 says the useful answer there is "you don't need this yet," and routing it in anyway
manufactures work.

## Step 4 — Sequence the work

The order is the product of this skill. Run it top to bottom; do not start a stage before its
inputs exist.

1. **Constraints** — repo read, versions checked, tldraw licensing answered (Step 2).
2. **Library decision** — `component-library-advisor` Steps 2–3. Present the recommendation
   with its tradeoff named, and confirm before installing.
3. **Token layer** — `component-library-advisor` Step 4. If a design file exists or a
   designer is in the loop, `penpot-workflow` runs *first* and the theme is generated from
   the token export rather than hand-written. One token file, one source of truth.
4. **Motion root wiring** — `motion-principles` Steps 4–5. The duration/easing tokens and the
   root `<MotionConfig transition={…} reducedMotion="user">` land in the **same commit as the
   token layer**, not after the screens. This is required setup, not a later enhancement —
   see Step 5 below.
5. **Icons** — `icon-resources`. Pick one UI glyph family and, separately, a brand-mark
   source. Doing this before screens prevents the two-icon-set tell.
6. **Component library scaffolding** — `storybook-workflow`, if Step 3 routed it in. Stories
   are written against the real theme from stage 3, which is why it cannot come earlier.
7. **Screens** — now, and not before.
8. **Per-component motion** — `motion-principles` Steps 2, 3, 8. The purpose test and the
   subtlety rules apply per animation; the root wiring from stage 4 is already there.
9. **QA gate** — Step 6 below. Nothing is "done" before it runs.

Stages 3, 4 and 5 are the one-time foundation. They are cheap now and expensive later, which
is the entire argument for this skill existing.

## Step 5 — Non-negotiable boilerplate to carry into every project

Two items are not recommendations that survive a judgment call. Include them by default and
flag their absence as incomplete setup.

- **`<MotionConfig reducedMotion="user">` at the application root**, above the router, in the
  same provider that carries the transition tokens. Motion's context default is
  `reducedMotion: "never"` — the OS accessibility preference is ignored until a developer
  writes the opt-in, and nothing warns about it in production. `motion-principles` Step 5 has
  the full Next.js App Router and Vite wiring.
- **One token file as the only source of colors, spacing, radii, type steps, and motion
  timing.** Screens contain no hex values, no arbitrary pixel values, and no inline
  durations.

## Step 6 — The QA gate, and its optional dependency

**Read this before relying on the gate.**

The architecture calls for two skills at the end: `design:design-critique` and
`design:accessibility-review`. Their availability is **not a fixed property of "local CLI vs.
Cowork"** — it depends on which plugins and marketplaces are enabled in the current
installation, which varies per user and per session. Neither is bundled by this plugin, so
never assume either one is present; check the actually-available skill list every time rather
than trusting a remembered answer from a previous session, this file's own past revisions, or
general assumptions about what "a local CLI install" has. A claim about tool availability that
isn't re-checked against the current environment is exactly the kind of stale fact this whole
plugin's `verified_against` discipline exists to prevent.

So the gate has two forms, and you must state which one ran.

**If `design:design-critique` and `design:accessibility-review` are listed as available**:
invoke both, in that order, and resolve what they raise before declaring the task complete.

**If they are not listed as available**: say so explicitly rather than skipping silently, then
run the manual gate below. A task that reports "done" without one of these two paths having
run has not been checked.

### Manual design critique — fallback criteria

Each item below is a tell; `references/qa-gate.md` carries the **check procedure** for every
one of them — the exact DevTools step, console snippet, or grep that produces a pass or fail.
Run those rather than forming an impression, and run the app first: most of these read the
rendered UI, not the source.

- **Default primary color survives anywhere.** AntD's `#1677ff` and DaisyUI's 35 stock
  themes are the two most recognizable "nobody chose this" signals in React. **Do not grep
  the source for this one** — a project that never wrote a custom theme has the default live
  on screen while `src/` contains no color at all, because the default lives in
  `node_modules`. Read the computed value off a rendered primary button, and separately
  confirm a custom theme was authored at all.
- **One radius on everything.** A checkbox, a button, and a card sharing a radius is the
  tell; real systems differentiate by element role.
- **Shadow on every card.** Pick borders or shadows as the primary separator; reserve the
  other for genuinely floating surfaces.
- **Flat type hierarchy.** 5–6 steps from one ratio, body size constant across the product,
  hierarchy from size *plus* weight *plus* color — never all three at once.
- **Even spacing everywhere.** Related elements close, unrelated far. Uniform gaps are the
  flattest possible hierarchy.
- **Two icon families in one interface.** Stroke weight and optical grid never match between
  sets.
- **Animation that answers nothing.** Every animation must serve feedback, continuity, or
  attention hierarchy. Scroll reveals that re-fire, entrance animations on content the user
  requested, and infinite loops outside loading states all fail this.
- **Hardcoded values in screens.** Any hex, any arbitrary pixel value, any inline duration
  means the token layer is not actually the source of truth.

### Manual accessibility review — fallback criteria

Same rule: `references/qa-gate.md` has the procedure for each — which DevTools pane prints
the number, which snippet lists the failures, which keystroke reproduces the state.

- **Contrast** holds for text on every surface color that was changed — 4.5:1 for body,
  3:1 for large text and meaningful UI boundaries.
- **Keyboard**: every interactive element is reachable and operable, focus order follows
  visual order, focus is always visible, and nothing traps focus except a modal that is
  meant to.
- **Names**: icon-only controls have accessible names; decorative icons are
  `aria-hidden="true"`; form inputs have real labels, not placeholders as labels.
- **Reduced motion**: with the OS setting on, nothing translates or scales, no video
  autoplays, no marquee runs, nothing loops. With it off, the transitions still exist.
  `motion-principles` `references/reduced-motion.md` has the test procedure.
- **Semantics**: one `h1` per page, headings in order, landmarks present, lists are lists,
  buttons are `<button>` and links are `<a>`.
- **State is not color alone**: errors, required fields, and selection carry a second signal.
- **Zoom**: the layout survives 200% browser zoom without horizontal scrolling.

If the local installation has an accessibility agent available (this environment lists an
**Accessibility Auditor** agent), delegate the second list to it rather than eyeballing —
but only when the user has asked for agent delegation, and say that is what ran.

Full checklists, each item paired with its executable check procedure, are in
`references/qa-gate.md`. Treat that file as the gate; this section is the index.

## Step 7 — Worked routing

Four briefs, resolved end to end. `references/routing-matrix.md` carries the reasoning and
the edge cases.

**"Admin dashboard za internu upotrebu"**
→ Data-dense, widgets are the product: tables with sorting and filtering, forms, date
ranges. **Ant Design.** Storybook: **usually no** — run `storybook-workflow` Step 1's
two-of-four test rather than assuming. One application's screens with components used once
each fails it, and adding Storybook there manufactures work. A dashboard with a genuinely
shared widget layer (a chart wrapper, a filter bar, a data table used across a dozen screens)
can pass it — in which case Storybook covers that layer only, never the page compositions.
Penpot: only if a designer is involved. Motion: tokens derived from the AntD seed
(`motionDurationFast/Mid/Slow` = 0.1/0.2/0.3s) so the two timing systems agree, plus the
reduced-motion lever AntD does not ship — it has zero
`prefers-reduced-motion` handling. Icons: one glyph family; brand marks only if third-party
integrations are shown. Sequence: library → token layer → motion root → icons → screens →
gate.

**"Marketing landing sa jakim brendom"**
→ Visual identity dominates, interactive surface is small. **DaisyUI**, gated on Tailwind
being present at the right major version — if Tailwind is absent, name that cost before
recommending. Strong brand means the token layer is a *translation* job: if brand tokens
live in a design file, `penpot-workflow` runs first and the DaisyUI theme is generated from
the export. Storybook: **no** — explicitly wrong for marketing pages. Motion: the place
where `LazyMotion` + `m` earns its keep, since animation is decorative and below the fold;
scroll reveals `once: true`, at section level, never per paragraph. Icons: brand marks are
likely (payment, social, "trusted by") — the trademark rules in `icon-resources` Step 3
apply, and a "trusted by" wall is exactly where they bite.

**"Interna biblioteka komponenti za tim"**
→ A package with multiple consumers. **`storybook-workflow` is in.** The library choice
depends on who consumes it: data-dense internal apps → thin wrappers over Ant Design;
varied product surfaces → Tailwind plus headless primitives and your own components. Token
layer comes first and Storybook renders that real theme, not Storybook defaults — which is
why stories cannot be written before stage 3. Motion tokens ship *as part of the library*,
and the consuming app supplies the root `MotionConfig`; document that requirement in the
package README, because a library cannot install its own provider. Icons: the library picks
the glyph family for every consumer — that is a one-way decision, so make it deliberately.

**"App sa canvas editor feature-om"**
→ **Stop. `tldraw-workflow` Step 1, before anything else.** Process tool or product
feature? If a product feature: is the product commercial — revenue, a client, a company, a
paid tier, or an internal company tool? That answer decides whether there is a license cost
and whether the feature is viable at all, and it must be settled before any library, theme,
or screen work. Only once it is answered does normal routing resume: library lane by app
type, token layer, motion root, then the canvas integration from `tldraw-workflow` Step 4.
Note that the canvas surface is its own interaction model — the motion tokens apply to the
app chrome around it, not to canvas gestures, which tldraw owns.

## Step 8 — Verify before declaring done

- The brief's four questions were answered, not assumed, and the repo was read before any
  recommendation.
- If a canvas feature is in scope, the tldraw commercial question was answered **first**, in
  writing.
- The library recommendation named its tradeoff and was confirmed before installation.
- The token layer existed before the first screen was built; nothing was rebuilt because of
  ordering.
- `<MotionConfig reducedMotion="user">` is at the application root, above the router.
- Storybook was included only where the two-consumer test passes, and excluded — out loud —
  where it does not.
- Skills that were *not* routed in were named, with the reason, so the omission is a decision
  rather than an oversight.
- The QA gate ran in one of its two forms, and the response states which — including an
  explicit note when `design:design-critique` and `design:accessibility-review` were
  unavailable and the manual checklists were used instead.
