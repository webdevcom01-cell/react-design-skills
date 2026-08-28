---
name: tldraw-workflow
description: Establish whether tldraw can legally and financially be used in a project before any code is written, then integrate it correctly. tldraw is source-available, not open source - free in development, but production requires a license key, and an unlicensed production build stops rendering the editor after five seconds. Use whenever tldraw, an infinite canvas, a whiteboard, a sketching or diagramming surface, a node canvas, or an embeddable drawing tool comes up; whenever a React or Next.js app needs a canvas feature; and whenever architecture sketching or wireframing before coding is being planned. Also covers Serbian phrasings - tldraw, tabla, vajtbord, beskonacno platno, crtanje, dijagram, skica arhitekture, licenca. Do NOT use for design-token or handoff workflows, which are penpot-workflow.
metadata:
  version: "0.1.0"
  owner: "buky <webdevcom01@gmail.com>"
  verified_against: "tldraw/tldraw @ v5.3.2; LICENSE.md, apps/docs/content/community/license.mdx, apps/docs/content/sdk-features/license-key.mdx (updated 2026-01-31), apps/docs/content/getting-started/installation.mdx, tldraw.dev/pricing (read from source, 2026-08-23); re-verified 2026-08-25 directly against the live GitHub repo — npm version/peer deps, all seven README watermark quotes at their exact cited line numbers, the main tldraw package's exception, the 30-day/no-grace-period wording, and the data-collection table all confirmed byte-for-byte with no corrections needed"
---

# tldraw Workflow

tldraw is an excellent infinite-canvas SDK and it is **not open source**. It is
source-available under a proprietary license that permits development use and forbids
production use without a license key. An unlicensed production deployment logs errors and
**stops rendering the editor after five seconds** — the failure is not subtle, but it
arrives at deploy time, after the work is done and promised.

That is why the license question comes before the technical one, always.

## Step 1 — Ask the commercial question first

Before recommending tldraw, before any setup instruction, establish two things:

1. **Is this a process tool or a product feature?** Is tldraw a whiteboard the team uses to
   think — sketching architecture, mapping flows, wireframing before code — or is it being
   embedded into an application that users will touch?
2. **If it is a product feature: is the product commercial?** Anything with revenue, a
   client, a company behind it, or a paid tier is commercial. Internal company tools count
   as production too.

If the answer is unclear, ask. Do not proceed on an assumption — the two paths lead to
completely different recommendations, and getting it wrong means either a needless licensing
conversation or a promise the project cannot keep.

## Step 2 — Path A: tldraw as a process tool

Sketching architecture on a whiteboard before writing code is genuinely valuable, and here
the licensing question mostly dissolves: using **tldraw.com**, or the tldraw desktop app, is
using their product, not embedding their SDK. No license key, no integration, no budget line.

What to actually do with it:

- Sketch the **screen flow** before building screens — which surfaces exist, what moves
  between them. Cheaper to redraw than to refactor.
- Map **component boundaries** and data flow when a feature spans several parts of a system.
- Wireframe at low fidelity, deliberately. A rough box-and-line sketch invites structural
  feedback; a polished mockup invites color feedback.
- Export as SVG or PNG into the repo or a ticket, so the reasoning survives the meeting.

Keep it a thinking tool. The moment a sketch is treated as a spec, it needs to move into the
design tool where tokens live — see `penpot-workflow`.

## Step 3 — Path B: tldraw as a product feature

Here the license is a real constraint and a real cost, and it must be raised **before**
anyone commits to the feature. Read `references/license-terms.md` for the full verified
terms; the decision-relevant facts:

- **Development is free.** The SDK treats an environment as development when the protocol is
  not HTTPS, the hostname is `localhost` or a loopback address, or `NODE_ENV` is not
  `production`. Build and demo freely.
- **Production requires a license key** of one of three kinds:

| Type | Watermark | Duration | For |
|---|---|---|---|
| Trial | No | **100 days** | Evaluation before purchase |
| Commercial | No | Annual | Commercial production use |
| Hobby | **Yes** | Varies | **Non-commercial projects only** |

- **The free trial is 100 days**, one per company or project, no credit card, issued
  immediately by email from a form on tldraw.dev. It has **no grace period** — it stops
  working the day it expires.
- **Commercial pricing is not published.** tldraw describes it as "value-based pricing",
  annual, arranged through their sales team, with discounted startup pricing available for
  new companies. You cannot give a client a number from the website; you can only tell them
  a sales conversation is required. Say exactly that rather than guessing a figure.
- **The hobby license is non-commercial only**, discretionary (they review each request),
  and keeps the "made with tldraw" watermark on the canvas.

**A warning about a contradiction in tldraw's own materials.** Seven README files in the
repository — including `@tldraw/editor`, `@tldraw/sync`, and the `create-tldraw` starter,
though notably **not** the main `tldraw` package — say the SDK may be used "in commercial or
non-commercial projects so long as you preserve the watermark". The license documentation
says the opposite: the hobby license is for non-commercial projects, and commercial licenses
"are required for any commercial use in production". The licence docs are the more recent
and more specific source, and the conservative reading is the safe one. Do not let a client build a commercial product on the
README's wording — get it in writing from tldraw if it matters.

### What this means in practice

When a client or user asks for a canvas feature, the honest sequence is:

1. Say up front that tldraw is commercially licensed and the cost is quoted by their sales
   team, not published.
2. Offer the 100-day trial as the way to build and validate the feature before any money is
   committed — this is what the trial is designed for and it is generous.
3. Make the renewal a known line item, not a discovery. An annual license that lapses takes
   the feature down after a 30-day grace period.
4. If the budget is not there, say so before building. The alternatives — Excalidraw
   (MIT-licensed), React Flow, or a purpose-built canvas — are worse in different ways, and
   choosing one knowingly beats discovering the licence at launch.

Never ship a canvas feature to production on a development build and hope. The enforcement
is client-side, deterministic, and visible to users.

## Step 4 — Integrate it

Only once the licensing path is settled. Current version is **tldraw 5.3.2**, requiring
React 18.2+ or 19.2.1+.

```bash
npm install tldraw
```

```tsx
import { Tldraw } from 'tldraw'
import 'tldraw/tldraw.css'

export default function Canvas() {
  return (
    <div style={{ position: 'fixed', inset: 0 }}>
      <Tldraw licenseKey={process.env.NEXT_PUBLIC_TLDRAW_LICENSE_KEY} />
    </div>
  )
}
```

Three things that trip up first integrations:

- **The parent needs an explicit size.** The component sets its own height and width to
  `100%`, so a parent without dimensions renders a zero-height canvas.
- **`tldraw/tldraw.css` must be imported.** Without it the editor renders as unstyled chaos.
- **The license key is public by design.** Keys are validated on the client, work offline,
  and are safe in frontend code. The SDK also picks up `NEXT_PUBLIC_TLDRAW_LICENSE_KEY`,
  `VITE_TLDRAW_LICENSE_KEY`, and several other conventional variables automatically, so the
  prop is often unnecessary.

Asset hosting, domain validation, Next.js specifics, and styling are in
`references/react-integration.md`.

## Step 5 — Verify before declaring done

- The commercial question was asked and answered **before** any tldraw code was written.
- For a product feature: the client or user knows tldraw is commercially licensed, knows
  pricing comes from a sales conversation, and has agreed to the path (trial → commercial,
  or a different library).
- A license key is present in the production build and its allowed domains cover every
  deployment target, including preview and staging domains on their own hostnames.
- The renewal date is recorded somewhere a human will see it before expiry.
- Nobody has been told the hobby license covers a commercial product.
- The canvas has been loaded in a real production build, not only on localhost — that is the
  only test that exercises the license check at all.
