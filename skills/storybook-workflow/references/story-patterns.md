# Story patterns: what to write, and what not to

Verified against `storybookjs/storybook` v10 docs (`docs/writing-stories/`,
`docs/writing-docs/autodocs.mdx`, `docs/api/csf/`).

## Contents
- CSF 3 anatomy
- Which stories to write
- Variants vs. states vs. content edges
- Args, controls, and the combinatorial trap
- Composition and multi-component stories
- Autodocs
- Naming and hierarchy
- CSF Next status

## CSF 3 anatomy

```tsx
import type { Meta, StoryObj } from '@storybook/react-vite';
import { Button } from './Button';

const meta = {
  component: Button,
  args: { children: 'Save changes' },   // defaults shared by every story below
  argTypes: { onClick: { action: 'clicked' } },
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = { args: { variant: 'primary' } };
export const Danger: Story  = { args: { variant: 'danger' } };
```

- `satisfies Meta<typeof Button>` is what ties `args` to the component's real props. Drop it
  and type errors in args stop being reported.
- `StoryObj<typeof meta>` (not `StoryObj<typeof Button>`) inherits the meta-level args, so
  required props set once in `meta.args` are not demanded again in every story.
- Story exports are `UpperCamelCase`; Storybook derives the displayed story name from the
  export by running it through a `startCase`-style transformation (so `WithIcon` displays as
  "With Icon"), not by using the raw identifier verbatim.
- Prefer a `component` property over a hardcoded `title` — Storybook computes the title, so
  moving the file does not break its sidebar position or URL.

## Which stories to write

A story is worth writing when someone would need to **see** it to make a decision. Three
categories qualify:

1. **Variants a consumer must choose between.** Every `variant`, `tone`, or `size` value the
   API exposes. If a story does not exist for it, consumers will not know it exists.
2. **States the component enters on its own.** Loading, disabled, error, empty, selected,
   read-only. These are where implementations actually break, and they are the ones
   routinely missing.
3. **Content edges that break layout.** A 60-character label in a button sized for eight, a
   table with one row and with a thousand, a card with no image, a name in a script that
   renders taller than Latin. These catch real bugs before they reach a screen.

The default story alone is close to worthless — it is the one case everyone already sees.

## Variants vs. states vs. content edges

Keep them distinguishable in the sidebar. A useful set for a button:

```
Primary        Secondary      Ghost          Danger        ← variants
Loading        Disabled                                    ← states
WithIcon       IconOnly                                    ← composition
LongLabel                                                  ← content edge
```

Someone scanning that list learns the component's whole surface in a few seconds. That is
the actual deliverable — not coverage numbers.

## Args, controls, and the combinatorial trap

Do not write a story per prop combination. Four variants × three sizes × two states is 24
stories that no one reads and everyone has to maintain.

Split the work: **stories** carry the cases worth naming, **controls** carry the rest. Args
are editable in the Controls panel, so a single `Primary` story lets a reviewer try every
size without a story existing for each.

When a genuine matrix view is wanted — all variants at once for visual comparison — write
**one** story with a `render` function that maps over the values:

```tsx
export const AllVariants: Story = {
  render: (args) => (
    <div style={{ display: 'flex', gap: 8 }}>
      {(['primary', 'secondary', 'ghost', 'danger'] as const).map((variant) => (
        <Button key={variant} {...args} variant={variant} />
      ))}
    </div>
  ),
};
```

This is also the story that makes visual regression testing pay off: one snapshot covers
the whole variant surface.

## Composition and multi-component stories

Components that only make sense together (a `List` and its `ListItem`, a form field and its
label and error) get one story file for the parent, with the children used inside it. Do not
create a story file whose only purpose is to render a subcomponent that is never used alone.

Decorators supply what a component needs but does not receive as props — a router, a query
client, a form context. Keep the story itself a plain rendering of the component; anything
extra belongs in a decorator, which also keeps the Source doc block readable.

## Autodocs

Tagging a story with `autodocs` generates a documentation page from the component's prop
types plus its stories. Set it once for the whole project:

```ts
// .storybook/preview.ts
const preview: Preview = { tags: ['autodocs'] };
```

A component with no story tagged `autodocs` and no hand-written MDX docs page produces
**no** docs page — a common surprise when the tag is applied per file and one file is
missed.

Two useful tag mechanics:

- Removing the `dev` tag from a story makes it docs-only: it appears on the docs page but
  not in the sidebar. Good for illustrative examples that would clutter navigation.
- A tag's default filter state can be set at the project level (e.g.
  `tags: { experimental: { defaultFilterSelection: 'exclude' } }` in `.storybook/main.ts`),
  so an `experimental` tag starts hidden from the sidebar filter by default. This changes
  what's shown by default, not what's generated — the docs page and the story still exist
  and can be filtered back in; it does not delete or suppress them outright.

Autodocs is generated from types, so the quality of the docs page is the quality of the
component's prop types. Vague prop types produce a useless docs page; that is a signal about
the component, not about Storybook.

## Naming and hierarchy

Group by what a consumer is looking for, not by how the code is organized. A flat list of 80
components is unusable; so is a five-level tree mirroring the folder structure.

A practical shape for a design system:

```
Foundations/   (color, type, spacing, icons — usually MDX docs, not components)
Primitives/    (Button, Input, Badge)
Patterns/      (DataTable, FormField, PageHeader)
```

Storybook hoists a single-story group into its parent when that one story's name exactly
matches the component's name — so a component with exactly one story, named to match, does
not add a pointless nesting level. A single story named anything else keeps its own level.

## CSF Next status

Storybook 10 ships **CSF Next** — factory functions `defineMain`, `definePreview`,
`preview.meta()`, `meta.story()` — which give full type inference including addon
parameters. The docs mark it a **preview feature** and state that the API may still change,
and it currently supports React, Vue, Angular, and Web Components.

Use CSF 3 for anything a team relies on. CSF Next is a reasonable choice when the user asks
for it with the stability caveat understood; an automigration exists for adopting it later,
so choosing CSF 3 now is not a trap.
