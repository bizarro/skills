# Older Codebases

The rest of this skill describes how Luis builds today. Most of his repositories were built earlier, with tools he has since replaced. When working in one of them, recognize its generation and follow its idioms. Don't migrate it to the current stack unless asked: swapping GSAP for anime.js or ripping out MobX halfway through a project breaks more than it fixes.

## Timeline

| Period     | 3D                  | CMS                            | Build                | Animation                 | Views                                                |
| ---------- | ------------------- | ------------------------------ | -------------------- | ------------------------- | ---------------------------------------------------- |
| 2017–2020  | Three.js            | Prismic                        | Webpack              | GSAP 2 (`TweenMax`)       | Pug, some Vue                                        |
| 2020–2021  | OGL                 | Prismic                        | Webpack, glslify     | GSAP 3                    | Pug on Express                                       |
| 2022       | OGL                 | Prismic, fetched at build time | Vite                 | GSAP                      | Handlebars, Static                                   |
| 2023       | —                   | Prismic                        | Rollup               | GSAP                      | Pug, private `lyserg` package                        |
| 2024       | OGL                 | Sanity                         | Rollup               | GSAP with plugins         | Twig on Express, TypeScript, Lisergia in the repo    |
| 2025       | OGL, Three.js       | Sanity                         | tsup, Vite           | anime.js, then GSAP again | React rendered on the server only, Express on Vercel |
| Aug 2026 → | OGPU, OGL, Three.js | Sanity with preview            | Vite, Bun, Turborepo | anime.js, WAAPI           | Preact on Cloudflare Workers (Lisergia v25+)         |

## Lisergia Versions

- **Before v25 (until August 2026).** Express server with Twig or React server-only views, deployed to Vercel (`@vercel/node` for the server, static for the rest). Rollup or tsup, ESLint and Prettier with the same rules Biome enforces now. Lenis was ticked by GSAP's ticker in some versions.
- **v25 (2026-08-29).** Cloudflare Workers, Elysia, Preact, Vite, Biome, Bun, Turborepo, Sanity preview, PostHog.
- **v26 (2026-09-12).** MobX removed. `Component` got `onScroll` and `onResize` hooks instead.

Check the `@lisergia/*` version in `package.json` before writing code against the runtime. On v25 and below, `App`, `Page` and `Viewport` are MobX observables and behaviors subscribe to them:

```ts
configure({ enforceActions: 'never' }) // once, in app/index.ts

makeObservable(this, { bounds: computed })
this.addDisposer(autorun(this.onUpdate)) // reads application.scroll and Viewport
```

On v26 and up, override `onResize()` and `onScroll(scroll)` (see `lisergia.md`). Don't mix the two models in one project.

## Idioms to Keep in Older Code

- **GSAP Everywhere.** Timelines for reveals, preloaders and transitions, SplitText, ScrollTrigger (scoped with `scroller:` to the page element), and `CustomEase` registered under the same names as the CSS easing tokens (`CustomEase.create('ease-out-expo', '0.19, 1, 0.22, 1')`). Later code replaced ScrollTrigger with a paused timeline played or reversed from the scroll value, to avoid its memory leaks.
- **`clamp(min, max, value)`.** The old math utilities follow GSAP's argument order. The current one is `clamp(value, min, max)`. Check which one a file imports before calling it.
- **Own Smooth Scroll before Lenis.** The `Page` owned `{ current, target, last, limit, ease }`, fed by `normalize-wheel`, translated a wrapper with `translate3d`, and pinned `html, body` with `position: fixed`. It has no native scrollbar, no keyboard or find-in-page scrolling and no reduced-motion handling, which is why Lenis replaced it.
- **Hidden States Set from JavaScript** (`GSAP.set(element, { autoAlpha: 0 })` in a constructor) were safe only because the whole page stayed hidden with CSS until `show()`. New code hides in CSS.
- **Globals for Preloaded Assets**: `window.ASSETS` printed by the server, `window.TEXTURES` filled by the preloader. Later replaced by a `Textures` singleton with a `Map`.
- **WebP Picked at Runtime** with `Detection.isWebPSupported()` and `data-src-webp`, instead of `srcset`. **Fonts** waited on FontFaceObserver before `document.fonts.ready`.
- **`@mixin ratio($height, $width)`** with the padding-top trick, before `aspect-ratio`.
- **Unsupported-Browser Screens** that replaced the whole site with a contact card. Today an HTML overlay stays on a plain background instead.
- **No `prefers-reduced-motion`** anywhere before September 2026. Add it when touching motion, as a separate change.

## The 2026 React Detour

From May to August 2026, several new sites were built on Next.js with Tailwind, `motion/react` and, once, React Three Fiber. From September Luis went back to SCSS modules, `classNames`, plain Three.js classes and "No Tailwind". The current rules in this skill reflect where he landed. In a codebase from that period, follow its own rules:

- Tailwind with `--spacing: 1px`, so utilities read in pixels (`p-20`, never `p-[20px]`), one breakpoint per line, phone first, and conditional classes through `cn()` (clsx plus tailwind-merge).
- `motion/react` primitives (`Reveal`, `SplitLines`) shared between projects, with reveals that play once.
- Double quotes, semicolons and 100 columns where the client's Next.js template set them.

## Other People's Code

Several of these repositories were built with collaborators, and some files follow their habits (short names, camelCase CSS module classes, default-exported arrow components, px-heavy SCSS). Check `git log` for the author before treating a pattern as Luis's convention.
