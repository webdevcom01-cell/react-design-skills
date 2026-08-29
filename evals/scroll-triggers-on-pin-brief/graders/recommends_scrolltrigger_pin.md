---
type: llm
criteria: The response recommends GSAP ScrollTrigger with pin and scrub for this effect, explicitly states or implies that native CSS scroll-driven animations and Motion's useScroll do not support pinning, and does not recommend building this with only CSS animation-timeline or only Motion.
focus: technical correctness of the tool recommendation, not code style
---
This checks that the skill's Step 4 decision rule ("anything pinned ... → GSAP
ScrollTrigger, no contest") actually shapes the response, not just that the
skill was invoked.
