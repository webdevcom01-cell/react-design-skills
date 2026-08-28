# react-design

Design-decision skills for React and Next.js projects that use Ant Design or DaisyUI. Each
skill is independently verified against live sources (npm, official docs, package source)
and is triggered automatically when its topic comes up in conversation — none of them need
to be invoked by name.

## Skills

| Skill | Purpose |
|---|---|
| `director` | Orchestrator. Routes a project brief to the right skills below, in the right order, and runs a QA gate before calling the work done. Start here for anything open-ended. |
| `component-library-advisor` | Chooses between Ant Design and DaisyUI, checks the Tailwind version gate, and builds the token layer (color, type, spacing, radius, density) that keeps a project from looking generic. |
| `storybook-workflow` | Sets up Storybook for a component library — CSF 3 stories, variant coverage, and a preview that renders the project's real theme. |
| `penpot-workflow` | Uses Penpot as the single source of truth for design tokens, exported and compiled into the Ant Design / DaisyUI theme. |
| `tldraw-workflow` | Establishes tldraw's licensing status (source-available, not open source) before any canvas/whiteboard feature is built. |
| `icon-resources` | Sources brand marks (Simple Icons) and UI glyphs correctly, with the trademark distinction most projects get wrong. |
| `motion-principles` | Adds animation with Motion (formerly Framer Motion) — a duration/easing token layer, `prefers-reduced-motion` handled correctly, and subtlety rules. |

## Verification

Every factual claim in these skills (package versions, license terms, API defaults,
pricing) was checked directly against live sources — npm registry, official documentation,
and in several cases the published package's own source code — rather than trusted from
the skill's own metadata. See `_research/` for the source material used, and each
`SKILL.md`'s `verified_against` frontmatter field for what was checked and when.

## Installing

Drop this directory into a Claude Code / Cowork plugin location, or install the packaged
`.plugin` file. No MCP servers or external credentials are required — every skill is pure
knowledge/instructions.
