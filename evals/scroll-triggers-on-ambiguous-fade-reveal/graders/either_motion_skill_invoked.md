---
type: llm
criteria: The response either invoked the motion-principles skill, the scroll-choreography skill, or both, to answer this "fade in on scroll" request — it did not answer purely from general knowledge with neither skill's guidance reflected (e.g. no mention of Motion's viewport/whileInView prop, GSAP ScrollTrigger, or the once/no-re-trigger rule either skill states).
focus: whether skill-specific guidance shaped the answer, not which specific skill fired
---
This is a deliberately ambiguous brief — a plain, once-only fade-in-on-scroll
sits at the boundary between `motion-principles` (which covers this exact
case via Motion's own `viewport` prop when Motion is already the project's
library) and `scroll-choreography` (which frames the same effect as its "most
common case" for GSAP ScrollTrigger, no pin/scrub needed). This case does not
assert which one *should* win — that would be asserting a certainty the skill
authoring didn't actually establish — it only asserts that the request should
not fall through both skills' triggers and get answered as generic advice
with no reduced-motion handling, no "once" guidance, and no library-specific
correctness applied. If this case ever fails, it's a signal the two
descriptions' trigger phrases have drifted apart again, not that one specific
skill is "the" correct answer.
