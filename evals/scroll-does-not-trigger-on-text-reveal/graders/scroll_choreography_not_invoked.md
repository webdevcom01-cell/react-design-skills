---
type: tool_used
tool: Skill
input_match: '"skill"\s*:\s*"(?:[\w-]+:)?scroll-choreography"'
min: 0
max: 0
---
The brief is an on-load hero text entrance with explicitly no scrolling
involved ("No scrolling involved, this happens the moment the page appears").
This is `showcase-motion` territory (text-reveal / hero entrance choreography)
or `motion-principles` territory, not `scroll-choreography` — the two skills'
descriptions both carry an explicit "Do NOT use for" line pointing at each
other for exactly this boundary. `scroll-choreography` firing here would be
the routing failure this case exists to catch.
