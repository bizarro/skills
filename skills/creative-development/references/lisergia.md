# Lisergia

[Lisergia](https://github.com/bizarro/lisergia) is Luis Bizarro's opinionated stack and front-end framework for **bespoke, animated marketing sites**. It is deliberately **not** for product engineering. It exists to make adding motion and interaction to a page simple instead of controversial.

## When to use it

- Use it for a bespoke landing page, campaign, portfolio or brand site where the creative direction leads.
- Don't use it when the client has its own stack and the budget to keep it (see `react.md`), or when the project is an application with heavy UI state.

## Stack

- **Rendering**: server-rendered TSX views. Preact renders the view tree to static HTML **and never hydrates**. A TypeScript client runtime enhances the generated HTML.
- **Server**: [Elysia](https://elysiajs.com/) as a deliberately thin HTTP layer on **Cloudflare Workers**. It validates routes and passes content to the view renderer. Workers Static Assets serves the compiled front end.
- **CMS**: [Sanity](https://www.sanity.io/).
- **Tooling**: Bun, Turborepo, Biome and TypeScript.
- **Client libraries**: [Lenis](https://lenis.darkroom.engineering/) (smooth scroll), [NanoEvents](https://github.com/ai/nanoevents) (`on`/`off`/`fire`), [auto-bind](https://github.com/sindresorhus/auto-bind) (no manual `.bind(this)`), [Tempus](https://github.com/darkroomengineering/tempus) (one coordinated rAF loop) and anime.js (split text and timelines).
- **No MobX.** It was removed to keep the bundle as small as possible. A pure site doesn't need it.

## Monorepo layout

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

## Runtime building blocks

**`Component`** (from `@lisergia/core`). It extends a NanoEvents emitter with auto-binding.

```ts
new Component({
  application,    // optional: auto-subscribes onScroll(scroll) and onResize()
  autoListeners,  // default true: calls addEventListeners() after create()
  autoMount,      // default true: calls create() in the constructor
  classes,        // { active: 'block--active' }
  element,        // selector or HTMLElement
  elements,       // { title: '.block__title' } → null, one element or NodeList
  id,
})
```

Lifecycle and hooks: `create()`, `addEventListeners()`, `removeEventListeners()`, `onScroll(scroll)`, `onResize()`, `addDisposer(fn)` for cleanup, and `destroy()`.

**`Page`**: owns Lenis for its wrapper and ticks it from Tempus. It fires `scroll` and `resize` on the application and creates the page's datasets.

**`Application`**: the singleton on `.app`. It registers routes, components and datasets, intercepts same-origin links, and runs fetch-and-swap navigation with the `Transition` component (see `motion.md`).

## Adding behavior

1. Put the markup in a TSX section under `templates/sections/`, with BEM classes and `data-*` hooks.
2. Put the behavior in a `Component` subclass under `src/app/datasets/`.
3. Register it in `src/app/index.ts` with its selector, wrapped in `createAsyncDataset(() => import(...))` so it loads only on pages that use it.
4. Style it in `src/styles/sections/<section>.scss`, extending the shared animation placeholders.

Keep the server in charge of HTML and the client in charge of motion. Don't render markup on the client.
