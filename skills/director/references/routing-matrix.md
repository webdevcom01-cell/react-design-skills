# Routing Matrix

The full decision logic behind SKILL.md Step 3, plus the cases where the simple table is
wrong. Every rule here is derived from a sibling skill's own Step 1 decision criteria — the
authority for any given rule is that skill, not this file.

## The brief template

Paste this when the request is vague enough that four questions is faster than three rounds
of guessing:

```
1. What is being built?     internal tool | customer app | marketing/content | component library
2. Brand?                   exists (files/tokens) | exists (informal) | blank sheet
3. Shared library output?   yes, a package | no, one app
4. Phase?                   nothing written | extending a codebase | reviewing built UI
```

Plus the repo read, which is not optional:

```bash
grep -E '"(react|next|vite|antd|daisyui|tailwindcss|@tailwindcss/[a-z]+|motion|framer-motion|storybook|tldraw)"' package.json
ls tailwind.config.* postcss.config.* .storybook 2>/dev/null
```

## Signal → lane

What the words in a brief actually indicate.

| Signal in the brief | Lane |
|---|---|
| "admin", "dashboard", "internal tool", "CRUD", "back office", "panel" | Ant Design |
| "table with sorting/filtering", "date range", "tree select", "multi-step form" | Ant Design — these are the behaviors you are buying |
| "i18n", "RTL", "enterprise" | Ant Design |
| "landing", "marketing", "docs", "blog", "portfolio", "content site" | DaisyUI |
| "strong brand", "custom look", "it must not look like a template" | DaisyUI **if Tailwind**, else Tailwind + headless |
| "design system", "@company/ui", "shared components", "component library" | Storybook lane; base library depends on consumers |
| "a few custom screens", "mostly bespoke" | Neither library — Tailwind + headless primitives |
| "canvas", "whiteboard", "infinite canvas", "diagram editor", "node editor", "drawing" | **tldraw blocking gate first** |
| "we have a Penpot file", "the designer sent tokens", "Tokens Studio", "handoff" | Penpot lane for the token layer |
| "sketch the architecture", "wireframe first", "map the flows" | tldraw Path A (process tool, free) |
| "review this UI", "does this look right", "audit" | QA gate only, then targeted skills |

Serbian briefs carry the same signals under different words. These map to the identical
lanes — the routing does not change with the language of the request:

| Signal in the brief | Lane |
|---|---|
| "admin panel", "interni alat", "za internu upotrebu", "kontrolna tabla", "tabela sa filterima" | Ant Design |
| "landing", "marketing sajt", "prezentacioni sajt", "jak brend", "ne sme da lici na template" | DaisyUI **if Tailwind**, else Tailwind + headless |
| "biblioteka komponenti", "komponentna biblioteka", "dizajn sistem", "zajednicke komponente", "za tim" | Storybook lane; base library depends on consumers |
| "platno", "tabla", "vajtbord", "beskonacno platno", "crtanje", "dijagram", "canvas editor" | **tldraw blocking gate first** |
| "imamo Penpot fajl", "dizajner je poslao tokene", "handoff", "dizajn tokeni" | Penpot lane for the token layer |
| "skica arhitekture", "da skiciramo prvo", "vajrfrejm pre koda" | tldraw Path A (process tool, free) |
| "pregledaj dizajn", "da li ovo izgleda dobro", "zavrsni pregled" | QA gate only, then targeted skills |

## Where the simple table is wrong

**A brief is often two projects.** "Marketing site plus the app behind it" is two lanes:
DaisyUI on the marketing routes, Ant Design in the app, as separate builds with separate CSS
entries. `component-library-advisor` Step 2 is explicit that mixing is by *surface*, never by
component — loading both into one screen collides on resets and on semantic color names, and
doubles the bundle for nothing. Route them separately and say that is what you are doing.

**"Component library" is sometimes not a library.** If the answer to "who consumes it" is
"one app, eventually maybe another," it is not a package yet. Storybook's own Step 1 test
needs two of: multiple consumers, a package output, real variant surface, non-frontend
viewers. One of four means say "not yet" and revisit when the second consumer appears.

**An existing dependency usually wins.** A project already on Ant Design does not switch
because the brief sounds marketing-ish. Switching needs a stated reason and a migration
budget; absent both, the recommendation is to keep what is installed and move the effort into
the token layer, where it will show.

**A blank sheet is not a free choice if Tailwind is absent.** DaisyUI requires Tailwind at a
locked major version (v4 → DaisyUI 5, v3 → DaisyUI 4). Introducing Tailwind into a project
that does not have it is real work with real conventions attached. Name it as its own line
item; do not slip it in under a library recommendation.

**"Early phase" does not mean "skip the library decision forever."** It means defer it past
the structural sketching (tldraw Path A) and the token work (Penpot), then make it before the
first screen. Deferring it past the first screen is the failure this whole skill exists to
prevent.

## Sequencing rationale

Why the Step 4 order is the order, stage by stage.

| Stage | Depends on | Breaks if run early |
|---|---|---|
| Constraints | Nothing | A feature gets committed to before its licensing is known |
| Library decision | Constraints | Recommendation contradicts the installed stack |
| Token layer | Library decision | Tokens are written in a shape the library cannot consume |
| Penpot export | A design file existing | The theme gets hand-written, then diverges from the design file |
| Motion root wiring | Token layer | Duration tokens have nowhere to live; `reducedMotion` gets deferred and forgotten |
| Icons | Nothing, but before screens | Two glyph families arrive by accident, one per developer |
| Storybook | Real theme existing | Stories render Storybook defaults, so they document a UI nobody ships |
| Screens | Everything above | Rebuilt when the token layer lands |
| Per-component motion | Screens + motion root | Inline durations proliferate before there is a token to reference |
| QA gate | Screens | Nothing to check |

The three cheap-now-expensive-later stages are the token layer, the motion root wiring, and
the icon family decision. All three are one-time, all three are invisible while they are
correct, and all three are painful to retrofit across a built UI.

## Skills that are frequently *not* needed

Naming an omission is part of the routing output. Silence reads as an oversight.

- **storybook-workflow** — wrong for landing pages, single admin screens, prototypes, and
  any project whose components are each used once.
- **penpot-workflow** — nothing to do when there is no designer and no design file. A
  hand-authored token file in the repo is a legitimate single source of truth.
- **tldraw-workflow** — only for canvas surfaces or deliberate pre-code sketching. Not a
  general diagramming recommendation.
- **icon-resources** Simple Icons half — no brand marks means no trademark exposure. The UI
  glyph half still applies to every project.
- **motion-principles** — never fully omitted. Even a project with no animation needs the
  root wiring and the decision *not* to animate recorded, because "no motion" is a choice
  and "we never got to it" is not.

## Cross-skill contracts

Places where two skills must agree, and this skill is what makes them agree.

- **Token source of truth.** If `penpot-workflow` is in the route, the Penpot token export is
  the source and `component-library-advisor`'s theme config is *generated*, not authored. If
  Penpot is not in the route, the hand-authored token file is the source. Never both.
- **Motion timing vs library timing.** `motion-principles` Step 7 derives its duration and
  easing tokens from the component library's own tokens — Ant Design's `motionUnit`/
  `motionBase` seed, or the project's CSS custom properties under DaisyUI. Two timing scales
  in one product is visible.
- **Motion root provider in a library package.** A shared component library cannot install
  its own `MotionConfig` — the consuming app owns the root. The library ships motion *tokens*
  and documents the provider requirement in its README. Check for that documentation, because
  a library whose animations silently ignore `prefers-reduced-motion` in every consumer is a
  worse outcome than one that never animated.
- **Storybook renders the real theme.** `storybook-workflow` Step 4 wires the project's actual
  theme into the preview. That is only possible after the token layer exists, which is why the
  sequence puts stories after tokens.
- **Icon family is one-way in a library.** The glyph set a shared library picks becomes every
  consumer's glyph set. Decide it at library scaffolding time, not per component.
