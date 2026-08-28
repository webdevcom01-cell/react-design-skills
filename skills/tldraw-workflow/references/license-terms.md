# tldraw license terms

Verified against `tldraw/tldraw` @ v5.3.2 — `LICENSE.md`,
`apps/docs/content/community/license.mdx`,
`apps/docs/content/sdk-features/license-key.mdx` (page updated 2026-01-31; sole source for
the grace-period and domain-validation sections),
`packages/editor/api-report.api.md`, the seven README files listed under "The README
contradiction", and tldraw.dev/pricing.

These terms change. Re-read the two docs pages before quoting anything to a client.

## Contents
- What the license permits and forbids
- The three license types
- Pricing
- How enforcement works
- Development vs production detection
- Domain validation
- Grace periods and expiry
- Data collection
- Open source and trademark
- The README contradiction

## What the license permits and forbids

From `LICENSE.md`, the base terms. Permitted:

- Use in **Development Environments** — internal hosting for development, testing, or
  staging, not accessible to end users, customers, or the public.
- Modifying the software.
- Bundling it with your own projects.

Forbidden:

- Use in **Production Environments** — any deployment on servers, cloud platforms, or web
  applications that provides functionality to end users, customers, or the public.
- Disabling, changing, or interfering with License Key enforcement.
- Removing copyright notices.
- Sublicensing under terms that supersede this license.
- Distributing the software as a standalone product (only as part of another application).

Also required: include a verbatim copy of the license in any distribution, and comply with
tldraw's trademark policy.

The license terminates automatically on breach, or if you initiate a copyright, trade
secret, or patent claim against tldraw or any user of the software. Governed by Delaware
law with exclusive Delaware jurisdiction.

Note the definition boundary: **internal company tools are production**. "Development
Environment" means not accessible to end users; an internal tool used by employees is
serving end users.

## The three license types

| Type | Watermark | Duration | Purpose |
|---|---|---|---|
| Trial | No | 100 days | Evaluate before purchasing |
| Commercial | No | Annual | Production use in commercial apps |
| Hobby | Yes | Varies | Non-commercial projects |

**Trial.** Free, 100 days, requested through a form on tldraw.dev; the key arrives
immediately by email. One trial per company or project ("one trial per commercial unit"
in the license text) unless tldraw approves otherwise in writing. No credit card. When it
ends, the key stops working — see grace periods below.

**Commercial.** Requested through tldraw's plans form; their sales team discusses
requirements and pricing. Removes the watermark. Required for **any** commercial production
use.

**Hobby.** Discretionary — tldraw reviews each request and may reach out about the project.
**Non-commercial projects only.** The "made with tldraw" watermark must remain visible on
the canvas.

**Perpetual.** No longer sold except in exceptional cases, still honored for existing
customers. Tied to a version rather than a date: works with any patch release indefinitely,
but major or minor versions released after the license expiration date (plus grace) require
renewal.

## Pricing

Not published. The pricing page describes **value-based pricing** on an annual commercial
license, arranged through sales, with:

- a 100-day free trial, no credit card required,
- discounted startup pricing available for new companies (separate application),
- a free hobby license for non-commercial projects.

There is no public figure or range to quote. Telling a client "roughly $X" is fabrication —
the correct answer is that pricing is quoted per customer and requires contacting sales.

## How enforcement works

The SDK carries technical measures that verify key validity, detect the deployment
environment, enforce restrictions by license type, and ensure watermark display.

The editor's internal license states are visible in the public API report:

```
type LicenseState =
  | 'expired'
  | 'licensed-with-watermark'
  | 'licensed'
  | 'pending'
  | 'unlicensed-production'
  | 'unlicensed'
```

Keys are **validated on the client**. The SDK decodes and verifies the key's signature
locally with no network call to a license server, so keys work offline and are safe to
include in frontend code — they are public by design.

Each key encodes: the **allowed hosts**, the **license type**, and the **expiration date**.

Verification requires `crypto.subtle`. On plain HTTP in development the browser does not
expose it, so the SDK skips verification and logs a note — meaning a key that looks fine
locally over HTTP has not actually been checked. Test in a real production build.

## Development vs production detection

The SDK treats the environment as **development** — no key needed — if **any** of these
hold:

- the protocol is not HTTPS,
- the hostname is `localhost` or a loopback address (`127.x.x.x`, `::1`),
- `NODE_ENV` is not `'production'`.

Otherwise it is production: HTTPS, non-loopback domain, `NODE_ENV=production`. Without a
valid key there, the SDK logs errors and **stops rendering the editor after five seconds**.

## Domain validation

Keys specify which domains they work on, and the SDK checks the current hostname against
them.

- An exact host like `example.com` matches `example.com` and `www.example.com`.
- A wildcard like `*.example.com` matches any subdomain.
- Some enterprise licenses allow `*` for any domain.

Deploying to a domain not covered by the key is treated as unlicensed. **Preview and
staging deployments on their own hostnames need to be covered** — Vercel-style per-branch
preview URLs are a common gap. Console message: "License key is not valid for this domain";
the fix is to have tldraw update the allowed domains.

## Grace periods and expiry

Source: `apps/docs/content/sdk-features/license-key.mdx:105-109`. Note this is **not** on
the `community/license` page, which mentions only that trials expire — checking that page
alone will not find it.

- **Annual and perpetual licenses**: 30-day grace period after expiration. Quoting the docs:
  "Annual and perpetual licenses have a 30-day grace period after expiration. During this
  period, the SDK continues working but logs a message to the console. This gives you time
  to renew without service interruption. After the grace period the SDK stops rendering the
  editor."
- **Trial (evaluation) licenses**: **no grace period**. "Evaluation (trial) licenses have no
  grace period. They stop working immediately on expiration."

These are documentation statements, not contract text — `LICENSE.md` itself says nothing
about grace periods. For anything a renewal budget depends on, confirm with tldraw.

Record the renewal date somewhere a human sees it. The failure mode is a canvas that
disappears from a live product.

## Data collection

The two official pages disagree, and the more recent, more specific one is the license-key
page (updated 2026-01-31):

| License type | Data sent (per license-key docs) |
|---|---|
| Commercial | None |
| Hobby | License ID, license type, SDK version, page URL |
| Trial | License ID, license type, SDK version, page URL |
| Unlicensed | SDK version and page URL (production only) |

The older license page states that "under a commercial or hobby license, no information is
sent to tldraw", which conflicts on the hobby row. In either reading: no user data, canvas
content, or PII is collected, and nothing is sent from development environments. If a client
has a data-governance requirement, get the answer from tldraw in writing rather than from
either page.

## Open source and trademark

tldraw is **source available**, not Open Source by the OSI definition. Many examples and
demos are MIT-licensed and can be used freely; the SDK packages are not.

Including tldraw in an open source project is permitted, but the SDK stays under its own
license — which means **every downstream user needs their own trial, commercial, or hobby
license to run it in production**. That is a significant obligation to pass to users of an
open source project, and it should be stated in that project's README, not discovered.

Trademark guidelines separately limit use of the tldraw name and branding.

## The README contradiction

Seven README files in the repo carry the same sentence. Note that the main `tldraw`
package README is **not** among them — it contains no license text at all, only a link to
`LICENSE.md`. The files that do carry it are:

- `packages/editor/README.md:17`  ← the one most likely to be read, `@tldraw/editor`
- `packages/sync/README.md:17`
- `packages/sync-core/README.md:17`
- `packages/create-tldraw/README.md:69`
- `packages/namespaced-tldraw/README.md:15`
- `templates/image-pipeline/README.md:94`
- `templates/agent/README.md:772`

All seven say, verbatim:

> You can use the tldraw SDK in commercial or non-commercial projects so long as you
> preserve the "Made with tldraw" watermark on the canvas.

The license documentation says the hobby license is for **non-commercial** projects and that
commercial licenses "are required for any commercial use in production".

These cannot both be right. The documentation pages are more recent and more specific, and
the license text itself grants production rights only through a trial or commercial
agreement. Treat the README as outdated marketing copy.

If a project's viability depends on watermarked commercial use being free, that is a
question for sales@tldraw.com with the answer in writing — not something to infer from an
npm README.
