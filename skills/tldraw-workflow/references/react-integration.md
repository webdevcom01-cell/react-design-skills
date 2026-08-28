# tldraw React integration

Verified against `tldraw/tldraw` @ v5.3.2 —
`apps/docs/content/getting-started/installation.mdx`,
`apps/docs/content/sdk-features/license-key.mdx`, `packages/tldraw/README.md`,
npm `tldraw@5.3.2` (peer deps `react ^18.2.0 || ^19.2.1`).

## Contents
- Minimal setup
- The two mistakes everyone makes first
- License key placement
- Accessing the editor
- Static assets
- Fonts and styling
- Viewport meta
- Next.js notes
- Version and docs in node_modules

## Minimal setup

```bash
npm install tldraw
```

```tsx
import { Tldraw } from 'tldraw'
import 'tldraw/tldraw.css'

export default function Canvas() {
  return (
    <div style={{ position: 'fixed', inset: 0 }}>
      <Tldraw />
    </div>
  )
}
```

Requires React 18 or 19 (peer range `^18.2.0 || ^19.2.1`).

## The two mistakes everyone makes first

**No size on the parent.** The `Tldraw` component sets its own height and width to `100%`,
so it fills its parent container. A parent with no explicit dimensions gives a canvas with
zero height, which looks like the component failed to render.

**Missing CSS import.** `import 'tldraw/tldraw.css'` is required. Without it the editor
renders as unstyled markup. It can also be pulled in from another stylesheet:

```css
@import url('tldraw/tldraw.css');
```

To restyle deeply, copy `tldraw.css` into your own file, edit it, and import that instead.

## License key placement

Keys are validated client-side and are safe to expose, so they belong in a public
environment variable rather than a secret store.

```tsx
<Tldraw licenseKey={process.env.NEXT_PUBLIC_TLDRAW_LICENSE_KEY} />
```

The SDK also auto-detects a key from any of these, checking both `process.env` and
`import.meta.env`:

- `TLDRAW_LICENSE_KEY`
- `NEXT_PUBLIC_TLDRAW_LICENSE_KEY` (Next.js)
- `REACT_APP_TLDRAW_LICENSE_KEY` (Create React App)
- `GATSBY_TLDRAW_LICENSE_KEY` (Gatsby)
- `VITE_TLDRAW_LICENSE_KEY` (Vite)
- `PUBLIC_TLDRAW_LICENSE_KEY` (SvelteKit and others)

Setting the environment variable is usually cleaner than threading the prop through
components. The prop also exists on `TldrawEditor` and `TldrawImage`.

Remember the key encodes allowed domains. Preview and staging deployments on their own
hostnames must be covered by the key or they are treated as unlicensed — see
`license-terms.md`.

## Accessing the editor

`onMount` hands over the `Editor` instance once it is ready. The callback may return a
cleanup function, which runs on unmount.

```tsx
<Tldraw
  onMount={(editor) => {
    editor.selectAll()
    return () => {
      // cleanup
    }
  }}
/>
```

This is the entry point for programmatic control — loading a snapshot, restricting tools,
reacting to changes. Treat the editor instance as the API surface; do not reach into DOM
nodes.

## Static assets

The component needs asset folders — `embed-icons`, `fonts`, `icons`, `translations`. Three
strategies:

**Public CDN (default).** Works with no configuration. Good starting point; means a runtime
dependency on tldraw's CDN, which matters for offline or air-gapped deployments.

**Bundler imports.** Three helpers, pick by bundler:

```tsx
// Standards-based, works in most modern bundlers
import { getAssetUrlsByMetaUrl } from '@tldraw/assets/urls'
const assetUrls = getAssetUrlsByMetaUrl()

<Tldraw assetUrls={assetUrls} />
```

- `@tldraw/assets/imports` — plain import statements; the bundler must be configured to
  treat `.svg`, `.png`, `.json`, and `.woff2` as external assets.
- `@tldraw/assets/imports.vite` — appends `?url`; works with Vite with no extra config.
- `@tldraw/assets/urls` — `new URL(path, import.meta.url)`, standards-based.

**Self-hosting.** Copy the four folders from the repository's `assets/` directory into the
public path and use `getAssetUrls` from `@tldraw/assets/selfHosted`. This is the option for
deployments that cannot call out to a CDN.

Individual assets can be overridden without replacing the set:

```tsx
const assetUrls = { icons: { 'tool-hand': './custom-tool-hand.svg' } }
<Tldraw assetUrls={assetUrls} />
```

## Fonts and styling

The SDK bundles its own shape fonts — IBM Plex (Sans, Serif, Mono) and Shantell Sans for
draw-style text — loaded from the CDN or from self-hosted assets.

The **UI** inherits its font family from `.tl-container`, so it can be aligned with the
product's typography:

```css
.tl-container {
  font-family: 'Inter', sans-serif;
}
```

This is worth doing. A canvas whose chrome uses a different typeface from the surrounding
app reads as a third-party widget bolted on, which is exactly the impression to avoid. Match
the font and the surface colors to the project's token layer.

Note the distinction: the UI font follows your CSS, the **shape** fonts are the bundled ones
and are part of the drawing content.

## Viewport meta

For a full-screen canvas, the viewport meta needs `viewport-fit=cover` or features like safe
area positioning misbehave on mobile:

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
```

## Next.js notes

- Use a **client component**. The editor is browser-only; render it under `'use client'`.
- If hydration warnings or SSR errors appear, load it with `next/dynamic` and
  `{ ssr: false }`.
- `NEXT_PUBLIC_TLDRAW_LICENSE_KEY` is picked up automatically — the `NEXT_PUBLIC_` prefix is
  required for the value to reach the browser, and that is correct here since the key is
  public by design.
- The canvas is heavy. Route-level code splitting keeps it out of the initial bundle of
  pages that do not use it.

## Version and docs in node_modules

From 5.1.x onward the `tldraw` package ships a `DOCS.md` with the docs-site content for that
release and a generated `RELEASE_NOTES.md`. When behavior does not match the website, read
those files inside `node_modules/tldraw` — they match the installed version, which the
website does not necessarily do.

`tldraw.dev/llms.txt` provides the docs in an agent-friendly form.
