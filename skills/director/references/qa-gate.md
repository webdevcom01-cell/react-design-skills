# QA Gate

The end-of-task quality check, in both of its forms.

## Dependency status — read this first

The architecture specifies two skills at this stage:

- `design:design-critique`
- `design:accessibility-review`

Their availability is **not fixed by "local CLI vs. Cowork"** — it depends on which plugins
and marketplaces the current installation has enabled, which varies per user and per session.
Nothing in the `react-design` plugin provides them, and no sibling skill implements them, so
never assume either one is present or absent from a remembered answer — check the current
environment's actual skill list each time.

This is a documented external dependency, not a bug to work around silently. The rule:

1. **Check availability rather than assuming.** If the skills are listed in the current
   environment, use them — they are more thorough than the checklists below and they were
   written for this purpose.
2. **If they are unavailable, say so in the response**, then run the manual gate below.
   "Design and accessibility review: `design:design-critique` and
   `design:accessibility-review` are not available in this environment, so the manual
   checklist was applied instead" is a complete and honest statement.
3. **Never report a task complete with neither path having run.** The gate is the difference
   between "the code works" and "the work is done."

The manual checklists are deliberately a floor, not a replacement. They catch the recurring,
mechanical failures. They do not replace a human looking at the screen, and they should not
be presented as if they do.

## How to run the manual gate

Every item below has a **check** line: the specific action that produces a pass or fail.
Follow it rather than forming an impression — an impression is what
`design:design-critique`/`design:accessibility-review` are there to replace when available.

**Run the app first.** Most of these checks read the *rendered* UI, not the source. Source
grep is used only where the failure is genuinely a code artifact (hardcoded values, icon
package imports). This distinction matters: the most common design failure is a default that
was never overridden, and defaults live in `node_modules`, so they are invisible to any grep
of `src/`.

Two tools carry most of the work:

- **DevTools Elements → Computed** for resolved values on a selected element.
- **DevTools Elements → Accessibility** (Chrome) or the Accessibility Inspector (Firefox) for
  names, roles, and computed contrast.

Where a check is a one-liner, it is written as a console snippet you can paste.

## Manual design critique

### Color

- **No default primary survives.**
  Ant Design's `#1677ff` and DaisyUI's 35 stock themes are the two most recognizable
  "nobody chose this" signals in React. One hit here undoes the rest.
  **check:** Do not grep the source — a project that never wrote a custom theme has the
  default active on screen while `src/` contains no color at all. Read the rendered value.
  Select a primary button in DevTools → Computed → `background-color`.
  - **Ant Design fail:** `rgb(22, 119, 255)` — that is `#1677ff`, the untouched
    `colorPrimary` seed. Confirm the intent separately: the project should pass
    `theme={{ token: { colorPrimary: … } }}` to `ConfigProvider`.
  - **DaisyUI fail:** the resolved custom property matches any stock theme. Paste in the
    console:
    ```js
    getComputedStyle(document.documentElement).getPropertyValue("--color-primary").trim()
    ```
    `oklch(45% 0.24 277.023)` is stock `light`; `oklch(58% 0.233 277.117)` is stock `dark`.
    Any of the 35 stock values is a fail. DaisyUI 5 themes contain **zero hex values** — a
    hex result means an older DaisyUI or a hand-written override, both worth reading.
  - **The stronger check, either library:** confirm a custom theme was *authored at all* —
    `colorPrimary` in a `ConfigProvider` theme, or `@plugin "daisyui/theme"` with
    `--color-primary` in the project's CSS. Its absence means every default is live
    regardless of what any single element renders.

- **Neutrals carry a temperature.**
  **check:** Select the page background and a card surface; read Computed
  `background-color`. Convert to HSL in the color picker — hue `0` with saturation `0` on
  every neutral is untouched gray. A deliberate ramp holds one hue (roughly 20–40 for warm,
  200–240 for cool) at low saturation across all neutral steps.

- **The palette derives, it is not hand-picked.**
  **check:** List the palette steps and look at the spacing between them.
  ```js
  // AntD: read the generated scale off the document
  Array.from({length: 10}, (_, i) =>
    getComputedStyle(document.documentElement).getPropertyValue(`--ant-blue-${i+1}`).trim())
  ```
  Uneven perceptual jumps — two steps that look identical, one that jumps — mean the shades
  were typed by hand instead of derived from a seed.

- **Second accent color is justified or absent.**
  **check:** Count distinct saturated hues on one screen, excluding semantic status colors
  (success/warning/error). More than two means the palette is decorating rather than
  signalling.

### Typography

- **5–6 steps from one ratio.**
  **check:** Paste in the console on a representative screen:
  ```js
  [...new Set([...document.querySelectorAll("body *")]
    .map(el => getComputedStyle(el).fontSize))].sort((a,b) => parseFloat(a)-parseFloat(b))
  ```
  Fewer than 4 distinct sizes is a flat hierarchy; more than 8 is no scale at all. Then check
  the ratio between consecutive steps is roughly constant — an irregular sequence means the
  sizes were chosen per component.

- **Body size is constant** across the whole product.
  **check:** Run the snippet above on three different screens and compare the size that
  dominates by element count. It should be identical on all three.

- **Hierarchy uses size, weight, and color — but not all three at every level.**
  **check:** For an `h2` and its body text, read Computed `font-size`, `font-weight`, and
  `color`. If every heading level differs from its body on all three axes, everything is
  shouting; if it differs on none, there is no hierarchy.

- **Line length stays readable.**
  **check:** On the widest viewport the design supports, select a long-form paragraph and
  read Computed `max-width` (or measure the rendered box). Beyond roughly 75 characters per
  line, prose gets hard to track.

### Shape and depth

- **Radius differs by element role.**
  **check:** Select a checkbox, a button, and a card in turn; read Computed
  `border-radius`. Three identical values is the single clearest generated-UI tell. DaisyUI
  encodes the roles (`--radius-selector`, `--radius-field`, `--radius-box`); Ant Design
  derives `borderRadiusSM`/`borderRadius`/`borderRadiusLG` from the `borderRadius` seed.

- **Borders or shadows, not both everywhere.**
  **check:**
  ```js
  [...document.querySelectorAll("*")].filter(el => {
    const s = getComputedStyle(el)
    return s.boxShadow !== "none" && s.borderStyle !== "none"
  }).length
  ```
  A large count means both separators are doing the same job everywhere. Pick one; reserve
  the other for genuinely floating surfaces — dropdowns, modals, popovers.

- **No shadow on every card.**
  **check:** Count elements with a non-`none` `box-shadow` and compare against the number of
  visible surfaces. Near parity is the loudest slop signal in this list.

### Space

- **Everything on the 4px scale.**
  **check:** Source grep is correct here, because this failure *is* a code artifact:
  ```bash
  grep -rnE '\[[0-9]+px\]|(margin|padding|gap|top|left|right|bottom)[A-Za-z]*: *[0-9]+[,;]' src/ \
    | grep -vE '\b(0|4|8|12|16|20|24|32|40|48|56|64)\b'
  ```
  Then confirm on screen: select two adjacent elements and read Computed `margin`/`gap`.

- **Spacing carries grouping.**
  **check:** Pick one form or list. Read the gap between related fields and the gap between
  groups. If they are equal, the layout has the flattest possible hierarchy — the most common
  spacing failure, and invisible until measured.

- **Density is one dial set once.**
  **check:** Read Computed `height` on a button, an input, and a select. They should agree.
  Confirm the dial is set centrally — AntD `controlHeight` in the theme token, DaisyUI
  `--size-field` — rather than per component.

### Icons

- **One glyph family.**
  **check:** Source grep, since this is a dependency fact:
  ```bash
  grep -rnE "from ['\"](lucide-react|@heroicons/[a-z]+|react-icons/[a-z]+|@phosphor-icons/react|@tabler/icons-react|@ant-design/icons)" src/ \
    | sed -E "s/.*from ['\"]([^'\"/]+(\/[a-z-]+)?).*/\1/" | sort -u
  ```
  More than one glyph package (Simple Icons and other brand-mark sources excepted) is a fail.
  Then confirm visually: two sets never match on stroke weight, corner radius, or optical
  grid.

- **Brand marks come from a brand-mark source**, glyphs from a glyph source.
  **check:** For every logo on screen, confirm it comes from the brand-mark package, not
  redrawn in the glyph set — and that its use has passed `icon-resources` Step 3, which is a
  trademark question, not a license question.

- **No emoji as UI icons.**
  **check:**
  ```bash
  grep -rnP '[\x{1F300}-\x{1FAFF}\x{2600}-\x{27BF}]' src/ --include=*.tsx --include=*.jsx
  ```
  `-P` needs GNU grep, ripgrep, or ugrep — **stock macOS BSD grep does not have it**.
  Portable fallback, which catches any non-ASCII and needs a glance to separate emoji from
  accented text:
  ```bash
  LC_ALL=C grep -rn '[^[:print:][:space:]]' src/ --include=*.tsx --include=*.jsx
  ```
  Platform-dependent rendering, uncolorable, and the clearest tell of a generated interface.

- **Sizes on the scale.**
  **check:** Select an icon next to text; read Computed `width`/`height`. Expect 16/20/24px,
  optically matched to the adjacent text size.

### Motion

- **Every animation answers feedback, continuity, or attention hierarchy.**
  **check:** Walk the primary flow and name the purpose of each animation out loud. Any that
  cannot be named should have been deleted, not shortened. Entrance animations on content
  the user explicitly requested fail this by definition.

- **Timing comes from tokens.**
  **check:**
  ```bash
  grep -rnE "duration: *[0-9.]+|transition: *[0-9.]+s|duration-\[[0-9]+ms\]" src/ \
    | grep -v "styles/motion"
  ```
  Any hit outside the token file means the token layer is decorative.

- **Scroll reveals are `once: true`** and fire at section level.
  **check:** `grep -rn "whileInView" src/` and confirm every hit carries
  `viewport={{ once: true }}`. Then scroll the page down and back up — anything that
  re-animates is a fail.

- **No infinite loops** outside genuine loading indicators.
  **check:** `grep -rnE "repeat: *Infinity|animate-(pulse|spin|bounce|ping)" src/` and
  confirm each hit is a loading or progress state.

- **One thing moves per event.**
  **check:** Trigger the main state change and watch. If three regions animate at once, the
  hierarchy that was the reason to animate is gone.

- **Exit is shorter than entrance** (~0.7–0.8×).
  **check:** Compare the `exit` and `animate` transitions on the modal or drawer.

### Token integrity

- **No hex values, arbitrary pixel values, or inline durations in screens.**
  **check:**
  ```bash
  grep -rnE "#[0-9a-fA-F]{3,8}\b|\[[0-9]+px\]|duration: *[0-9.]+" src/ \
    | grep -vE "src/styles/|\.test\.|\.stories\."
  ```
  Any of the three means the token layer is not actually the source of truth, whatever the
  file structure suggests. Brand-mark `hex` values from a brand-mark package are the one
  legitimate exception — recoloring a logo is often a guideline violation.

## Manual accessibility review

### Contrast

- Body text ≥ 4.5:1; large text (≥ 24px, or ≥ 19px bold) ≥ 3:1; meaningful UI boundaries and
  informational icons ≥ 3:1.
  **check:** Select the text node in DevTools → Elements → Accessibility pane, which prints
  the computed contrast ratio against the actual rendered background. For a whole-page sweep,
  run Lighthouse → Accessibility, which flags every failing pair at once. Check **both**
  themes if the project ships light and dark, and check every surface color that was
  customized — that is where the regressions live.

### Keyboard

- Reachable, operable, ordered, visible, no unintended traps.
  **check:** Put the mouse down. Tab from the top of the page to the bottom, then
  `Shift+Tab` back. Note the first element where focus disappears, jumps out of visual
  order, or cannot be activated with Enter/Space. Open a modal, Tab through it, press
  Escape.
  ```bash
  grep -rn "outline: *none\|outline-none" src/
  ```
  Every hit needs a designed focus replacement on the same element.

### Names and roles

- Icon-only controls named; decorative icons hidden; inputs labelled; buttons are buttons.
  **check:** Paste in the console:
  ```js
  [...document.querySelectorAll("button, a, [role=button]")]
    .filter(el => !el.innerText.trim() && !el.getAttribute("aria-label")
                  && !el.getAttribute("aria-labelledby") && !el.title)
  ```
  Every returned element is an unnamed control. Then:
  ```js
  [...document.querySelectorAll("input, select, textarea")]
    .filter(el => !el.labels?.length && !el.getAttribute("aria-label"))
  ```
  A placeholder is not a label — it disappears on input and fails contrast in most designs.
  Finally `grep -rn "<div[^>]*onClick" src/` — a clickable `div` fails keyboard, focus, and
  screen-reader semantics at once.

### Reduced motion

- **check:** Turn the preference on — DevTools Command menu (`Cmd/Ctrl+Shift+P`) → "Emulate
  CSS prefers-reduced-motion: reduce", or macOS System Settings → Accessibility → Display →
  Reduce motion for a real end-to-end test. Then walk the primary flow and confirm: nothing
  translates, slides, or scales; no video autoplays; no marquee, ticker, or carousel
  advances; no parallax; nothing loops.
  Turn it **off** and walk the same flow: the transitions must still exist. Over-guarding is
  a real regression — a project where reduced motion accidentally applies to everyone has
  lost the design, not protected it.
  Confirm the root wiring is present, since without it none of the above can pass by
  accident: `grep -rn 'reducedMotion' src/` should find `reducedMotion="user"` on a
  root-level `MotionConfig`.
  Platform toggles and the full procedure are in
  `motion-principles/references/reduced-motion.md`. Remember that `MotionConfig` covers only
  Motion's own animations: hand-written CSS, video, and the component library's internal
  animations need their own guards, and Ant Design ships none.

### Structure

- One `<h1>`, ordered headings, landmarks, real lists, `<html lang>`.
  **check:**
  ```js
  [...document.querySelectorAll("h1,h2,h3,h4,h5,h6")].map(h => h.tagName + " " + h.innerText.slice(0,40))
  ```
  Read the result top to bottom: exactly one `H1`, and no level skipped on the way down.
  Then:
  ```js
  ["header","nav","main","footer"].filter(t => !document.querySelector(t))  // should be []
  document.documentElement.lang                                            // should be set
  ```

### State and feedback

- State is never color alone; errors are associated and announced; async changes reach AT.
  **check:** Submit a form with an invalid field. Confirm the error carries a second signal
  besides color — icon, text, shape, or position — and that the input points at the message
  via `aria-describedby`. Trigger a loading or success state and confirm it lands in a
  region with `aria-live` (or `role="status"` / `role="alert"`), rather than appearing
  silently.

### Zoom and reflow

- **check:** `Cmd/Ctrl +` to 200% and walk the primary flow. The `body` must not gain a
  horizontal scrollbar; no content may be cut off or overlap. Then confirm text is
  selectable — anything baked into an image cannot be resized or read aloud.

## Delegating the accessibility pass

If the environment offers an accessibility agent — this one lists an **Accessibility
Auditor** — delegating the second list is better than eyeballing it, because a screen-reader
pass finds what a checklist cannot. Two conditions: the user has to have asked for agent
delegation, and the response has to say that is what ran. An agent's report is input to your
judgment, not a substitute for it; verify anything it claims before repeating it.
