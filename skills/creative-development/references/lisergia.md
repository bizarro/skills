# Lisergia

[Lisergia](https://github.com/bizarro/lisergia) is Luis Bizarro's opinionated stack and front-end framework for **bespoke, animated marketing sites**. It is deliberately **not** for product engineering. It exists to make adding motion and interaction to a page simple instead of controversial.

This file describes Lisergia v26 and up (September 2026). Projects pinned to an older `@lisergia/*` version run on Express, MobX and often GSAP. Read `history.md` before changing one.

## When to Use It

- Use it for a bespoke landing page, campaign, portfolio or brand site where the creative direction leads.
- Don't use it when the client has its own stack and the budget to keep it (see `react.md`), or when the project is an application with heavy UI state.

## Stack

- **Rendering**: server-rendered TSX views. Preact renders the view tree to static HTML **and never hydrates**. A TypeScript client runtime enhances the generated HTML.
- **Server**: [Elysia](https://elysiajs.com/) as a deliberately thin HTTP layer on **Cloudflare Workers**. It validates routes and passes content to the view renderer. Workers Static Assets serves the compiled front end.
- **CMS**: [Sanity](https://www.sanity.io/).
- **Tooling**: Bun, Turborepo, Biome and TypeScript.
- **Client libraries**: [Lenis](https://lenis.darkroom.engineering/) (smooth scroll), [NanoEvents](https://github.com/ai/nanoevents) (`on`/`off`/`fire`), [auto-bind](https://github.com/sindresorhus/auto-bind) (no manual `.bind(this)`), [Tempus](https://github.com/darkroomengineering/tempus) (one coordinated rAF loop) and anime.js (split text and timelines).
- **No MobX.** It was removed to keep the bundle as small as possible. A pure site doesn't need it.
- **Content** is a build-time `content.json` snapshot from Sanity, with live queries only in preview (see `sanity.md`).
- **Fonts** are self-hosted woff2 with `font-display: swap` and latin and latin-ext `unicode-range` subsets, listed once in `templates/manifest.ts`, preloaded with `<link rel="preload" as="font" crossorigin>` and sent in a `Link` header. Typekit is the alternative, through a `TYPEKIT` environment variable.
- **Analytics**: PostHog, loaded on `requestIdleCallback` after `load`, with web vitals on.
- **Hosting**: Cloudflare Workers for Luis's own projects. Client projects have often gone to Vercel instead; the static variant below covers that.

## Monorepo Layout

```
apps/
  backend/                  Sanity Studio
  frontend/
    worker.ts               Elysia worker entry
    router/ controllers/    routes → content → views
    templates/              Preact SSR views (TSX)
      pages/  layout/  shared/  components/  sections/
    src/app/                client runtime
      index.ts              registers components, datasets, templates
      classes/              shared bases (Animation)
      components/           global singletons (Menu, Navigation, Transition)
      datasets/             [data-*] behaviors (Reveal, Title, Paragraph, Parallax, …)
        sections/           behaviors for a whole section (Hero, Marquee, …)
      templates/            Page subclasses per html[data-template]
    src/styles/
      base/ shared/ components/ sections/ pages/
packages/
  core/        Application, Component, Page, Link, Links, EventEmitter
  managers/    Viewport, Pointer
  styles/      shared SCSS: include-media, easings, mixins, z-index
  utilities/   MathUtils (clamp, lerp, map, random), DOMUtils, Tempus helpers
```

## Runtime Building Blocks

**`Component`** (from `@lisergia/core`). It extends a NanoEvents emitter with auto-binding.

```ts
new Component({
  application, // optional: auto-subscribes onScroll(scroll) and onResize()
  autoListeners, // default true: calls addEventListeners() after create()
  autoMount, // default true: calls create() in the constructor
  classes, // { active: 'block--active' }
  element, // selector or HTMLElement
  elements, // { title: '.block__title' } → null, one element or NodeList
  id,
})
```

Lifecycle and hooks: `create()`, `addEventListeners()`, `removeEventListeners()`, `onScroll(scroll)`, `onResize()`, `onVisibilityChange(isInView)`, `addDisposer(fn)` for cleanup, and `destroy()`.

- A component subscribes only to the hooks it overrides. `AutoBind` runs in the `EventEmitter` base, so subclasses don't call it.
- **`onResize()` only measures, `onScroll(scroll)` only writes.** The application merges resize events into one frame and fires `resize` on every component before `scroll`, so reads never interleave with writes. A component that needs bounds at creation calls `this.onResize()` in its constructor and asks the application for a flush.
- `onScroll` only runs while the component is in view. A shared IntersectionObserver with a `25%` margin culls offscreen components, and `onVisibilityChange` is where videos pause and speed state resets.

**`Page`**: owns Lenis for its wrapper and ticks it from Tempus. It fires `scroll` and `resize` on the application and creates the page's datasets.

**`Application`**: the singleton on `.app`. It registers routes, components and datasets, intercepts same-origin links, and runs fetch-and-swap navigation with the `Transition` component (see `motion.md`).

## Adding Behavior

1. Put the markup in a TSX section under `templates/sections/`, with BEM classes and `data-*` hooks.
2. Put the behavior in a `Component` subclass under `src/app/datasets/`.
3. Register it in `src/app/index.ts` with its selector, wrapped in `createAsyncDataset(() => import(...))` so it loads only on pages that use it.
4. Style it in `src/styles/sections/<section>.scss`, extending the shared animation placeholders.

Keep the server in charge of HTML and the client in charge of motion. Don't render markup on the client.

The section is also a Sanity module of the same name, rendered by a `_type` switch in `templates/sections/Sections.tsx` (see `sanity.md`). The first section renders with `priority`, so its media loads eagerly. Name everything after the module: `hero` → `Hero.tsx` → `.hero` → `hero.scss` → `datasets/sections/Hero.ts`.

## Boot

`src/app/index.ts` waits for `document.fonts.ready` (split text measures wrong before it), registers routes, the sprite sheet, datasets, the page and the global components, then adds `html.loaded`, which un-hides `body`.

Icons are SVG files in `src/sprites/`, built into one `bundle.svg` symbol sprite. `Application.initSprites()` fetches it and injects it off-screen, and markup uses `<svg><use href="#icon-name" /></svg>`.

## Delivery on Workers

- **HTML** is rendered once per isolate, slug and device class, with a SHA-1 `ETag`, `304` responses and `cache-control: public, max-age=0, must-revalidate`.
- **Module preloads**: the Worker adds `modulepreload` hints for the async dataset chunks a page uses, found by naming convention (`datasets/Parallax` ↔ `data-parallax`, `datasets/sections/Hero` ↔ `class="hero"`).
- **Headers**: the CLI emits hashed `bundle-[hash].js` files at the build root, so `_headers` marks `/bundle-*` as `immutable`. Fonts, favicon and the sprite get `max-age=604800, stale-while-revalidate`. `.assetsignore` keeps the `.vite/` manifest, which the Worker imports, out of the public assets.

## Single-Page Sites and Teasers

A teaser, splash or one-off WebGL piece doesn't need the monorepo, the Worker or Sanity. It still uses Lisergia's layout and tooling, so it can grow into a full site without being moved around:

```
src/
  index.html             the page (Vite entry)
  app/
    index.ts             entry: imports the styles, waits for fonts, starts the Canvas
    classes/             Canvas, Pointer, Sound, simulations
    shaders/             .glsl files, chunks/ for shared code
    utils/               Constants, GL helpers, Math, shapes
    types.d.ts           declare module '*.glsl'
  styles/
    index.scss           @use of base/ and components/
    base/ components/ utils/
  shared/                copied as is: favicon, OG image, _headers
build/                   output, the only folder deployed
```

- `vite.config.ts` uses the same paths as `@lisergia/cli`: `root: 'src'`, `publicDir: 'shared'`, `outDir: '../build'`, and `vite-plugin-glsl` with `minify` on build. The CLI itself expects a Worker that renders the HTML from its manifest, so a static page uses a local config instead.
- `tsconfig.json` extends `@lisergia/config-tsconfig/base.json`. Biome uses the Lisergia config (single quotes, no semicolons, trailing commas, 120 columns, imports organized), plus the shared `.editorconfig`. `package.json` pins `packageManager` and `engines` to Bun.
- Scripts: `dev`, `build`, `preview` (build, then `wrangler dev`), `check-types`, `lint`, `format`, and `deploy` (build, then `wrangler deploy`).
- Deploy to Cloudflare Workers Static Assets with no Worker script: `wrangler.jsonc` has only `assets.directory: './build'`, the routes and the custom domain. `src/shared/_headers` marks `/assets/*` as `public, max-age=31536000, immutable`, because Vite hashes those file names.
- `.gitignore` is split into commented sections (`# Build`, `# Node`, `# Environment`, `# TypeScript`, `# Wrangler`).
- The README follows the same shape as other Lisergia projects: what the piece is, the stack in one paragraph, a commented Quick Start (`# Install dependencies.`, `bun install`, …) and a short file map. A `CLAUDE.md` holds the commands, the conventions and the agent rules.
- When WebGL2 or float render targets are missing, catch the error from the `Canvas` constructor, add `html.is-fallback` and let the HTML overlay stand on a plain background.

## Static Sites with Many Pages

When the host is not Cloudflare (a client on Vercel, for example) and the site doesn't need per-request rendering, keep the Lisergia runtime and templates but prerender them:

- `scripts/prerender.ts` renders every page of `content.json` to static HTML at build time, and copies the `not-found` page to `404.html`.
- A Vite plugin renders the same templates on request in development, so there is no separate dev server.
- The device class comes from an inline head script instead of the user agent (see `css.md`), and `vercel.json` sets `framework: null`.
