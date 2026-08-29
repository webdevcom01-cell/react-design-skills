# Eval cases — schema verification status

The `case.yaml` / `prompt.md` + `graders/*.md` format used below is **not
independently verified** against a second source. `claude plugin eval init
--bare` — the direct way to generate a real template from this account — is
gated as early access and failed to run. The schema here comes from a
`claude-code-guide` agent's research, which itself reported it as sourced from
an "embedded offline reference" rather than a fetched public doc — there is no
publicly documented schema to cite as of 2026-08-29.

Treat every file below as a best-effort draft, not a confirmed-working eval
suite. Before relying on these results: run `claude plugin eval` against this
directory once early access is available, or once official docs are
published, and fix whatever the real CLI rejects. Do not assume a clean run
here means the schema was right — it may mean the case never actually
executed.

## What these cases check

- `scroll-triggers-on-pin-brief` — a brief that explicitly asks for pinned,
  scrubbed scroll behavior should invoke the `scroll-choreography` skill.
- `scroll-does-not-trigger-on-text-reveal` — a brief that is purely about
  hero text entrance (no scroll mechanics at all) should **not** invoke
  `scroll-choreography` — it belongs to `showcase-motion` instead.
- `scroll-triggers-on-ambiguous-fade-reveal` — added after an independent
  cold review caught a real one-way routing gap: `motion-principles`
  advertised "scroll-triggered reveals" as its own trigger phrase with no
  awareness that `scroll-choreography` existed, while `scroll-choreography`
  itself never used the word "reveal." A plain "fade in as the user scrolls
  to it" brief — arguably the single most common scroll-choreography-shaped
  request — risked falling into neither skill's confident trigger set. Both
  descriptions were edited to close this (see the two skills' `SKILL.md`
  frontmatter and git history), and this case exists to catch a regression if
  the two drift apart again. It deliberately does not assert which of the two
  skills *should* fire — the case description explains why.
