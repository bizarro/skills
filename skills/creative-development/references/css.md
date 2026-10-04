# CSS: Fluid rem Scaling and Few Breakpoints

This applies to **websites with scrolling content**. Application UI (editors, tools) uses plain `px`, because tool UI should stay the same size and not scale with the viewport. So does a full-screen WebGL piece with a thin HTML overlay (a logo, an email, a sound toggle): the canvas fills the window anyway, and the overlay reads better at fixed sizes.

## One rem Equals Ten Design Pixels

Set the root font size so that `1rem` equals 10 pixels of the **design file** at any viewport width. Every value in the Figma file then becomes `value / 10` rem, and the whole layout scales with the window exactly as it was designed.

```scss
html {
  font-size: calc(100vw / 1440 * 10); // 1440 = desktop design width

  @include media('<phone') {
    font-size: calc(100vw / 375 * 10); // 375 = mobile design width
  }
}

body {
  font: 1.6rem/1.3 $font-body; // 16px in the design

  @include media('>=desktop') {
    font-size: 16px; // stop scaling on very large screens
  }
}
```

- Use the design widths from the file. Desktop has been 1920 (most 2020–2022 sites), 1440, 1512 and 1728; mobile 375, 390, 404 and 320.
- Write sizes in `rem`: spacing, type, radii, component dimensions. A 48px heading in Figma is `4.8rem`.
- Above the desktop breakpoint, cap text that would grow too large by switching to `px`:

```scss
%title-70 {
  font: 7rem/0.9 $font-display;

  @include media('>=desktop') {
    font-size: 70px;
  }
}
```

## Keep Breakpoints to a Minimum

Because everything scales proportionally, you **don't need breakpoints to fix sizes**. Use a breakpoint only when the **layout** changes (columns stacking, navigation turning into a menu), and in practice that is almost always just the phone breakpoint.

- Lisergia defines three breakpoints with [include-media](https://eduardoboucas.github.io/include-media/): `phone: 768px`, `tablet: 1024px`, `desktop: 1920px`. `desktop` is mostly the "stop scaling" cap. A single `phone` breakpoint has been enough for whole portfolios.
- Don't add intermediate breakpoints to fine-tune sizes. If something looks wrong at a middle width, fix the rem value.

Two alternatives to the px cap above the desktop width:

- **Cap at the content width.** Scale from `min(100vw, 1512px)` so everything stops growing together, with no px overrides:

```scss
:root {
  --content-max-width: 1512px;
  --content-width: min(100vw, var(--content-max-width));
}

html {
  font-size: calc(var(--content-width) / #{$design-width} * 10);
}
```

- **Fit the viewport height too.** A one-screen composition (a hero, a gallery that must not scroll) scales by whichever side runs out first: `font-size: min(calc(100vw / 1440 * 10), calc(100vh / 900 * 10))`, set on the section, with its children sized in `em`.

## Media Queries Live inside the Rule They Change

Nest every media query inside the selector it modifies, as the last thing in that rule. Never collect them in `@media` blocks at the bottom of the file or in a separate responsive file: whoever reads `.copy` should see everything `.copy` does at every size, in one place.

```scss
.copy {
  font-size: 1.4rem;
  max-width: 40rem;

  @include media('>=desktop') {
    font-size: 14px;
  }

  @include media('<phone') {
    font-size: 1.3rem;
    max-width: 32.5rem;
  }
}
```

- Order inside a rule: base declarations, nested states and modifiers (`&:hover`, `&--active`), then the `>=desktop` cap, then the `<phone` override.
- The same goes for every other at-rule that targets one element: `@media (prefers-reduced-motion: reduce)`, `@media (hover: none)`, `@supports`. Each goes inside the rule it changes, even when that means several small blocks instead of one big one.
- Projects without include-media nest plain queries the same way: `@media (width <= 640px) { … }` inside the rule.

## Supporting Tokens

- **Easings as CSS custom properties**: `--ease-{in,out,in-out}-{sine,quad,cubic,quart,quint,expo,circ,back}`. Reveals mostly use `var(--ease-out-expo)`.
- **Viewport height**: set `--100vh` from JavaScript on resize and use `var(--100vh)` for full-height sections, so mobile browser toolbars don't cause jumps.
- **Hover only where hover exists**: a `hover` mixin wraps rules in `html.desktop &:hover`, so touch devices never get stuck hover states.
- **Device class from the server**: the server parses the user agent (`ua-parser-js`) and renders `<html class="desktop|tablet|phone">`, which the `hover` mixin and `Detection.isMobile()` read. Cache rendered HTML per device and send `vary: user-agent`. Static sites without a server set it from an inline script in `<head>` before the stylesheet applies:

```js
var isCoarse = matchMedia('(pointer: coarse)').matches

document.documentElement.classList.add('js', isCoarse ? (innerWidth >= 768 ? 'tablet' : 'phone') : 'desktop')
```

- **Z-index from a map**: `z('navigation')` looks up the value in one `$z-indexes` list instead of scattering magic numbers. The list reads top to bottom: the first name gets the highest index (`'cursor', 'loader', 'menu', 'navigation', 'content', 'canvas'`), and an unknown name warns at compile time.
- **Colors as custom properties with evocative names** in `variables.scss`: `--color-pampas`, `--color-cod-gray`, `--color-ember`, not `--color-primary`. Fonts as `$font-<family>` variables.
- **JavaScript animates unitless custom properties and CSS does the math.** A timeline tweens `--inset` from 0 to 1, and the rule says `inset: calc(8rem * var(--inset, 1))`. A cursor writes lerped `--pointer-x` and `--pointer-y` on `body`. The CSS stays the single source of the layout.
- `%cover`: an absolute fill placeholder for media layers. A media wrapper using it needs a parent with `position: relative`, a defined height and `overflow: hidden`.
- **Global resets**: hide scrollbars (`::-webkit-scrollbar { display: none }` with `scrollbar-width: none`) since Lenis drives the scroll, fade images in with `img { opacity: 0; transition: opacity 1s } img.loaded { opacity: 1 }`, and hide `img:not([src])` and `video:not([src])`.
- **Boot gate**: `body` stays hidden until `html.loaded`, which the entry adds after `document.fonts.ready`. During a page transition, `html.transitioning * { pointer-events: none !important }`.

## Naming on Sites

Sites use BEM (`block__element--modifier`), with one SCSS file per section, and import `@lisergia/styles` first. Reveal states are `--active` modifiers (see `motion.md`).

- **Write each BEM selector out in full** as its own top-level rule. Don't build names with `&__`, so every class can be found with a search. Only modifiers and states nest (`&--active`, `&:hover`). Deeper elements chain: `.home__gallery__media`.
- **Sort declarations alphabetically**, enforced by Stylelint's `order/properties-alphabetical-order` (in `@lisergia/config-stylelint`). Run it with `lint:styles` and `lint:styles:fix` scripts.
- **`index.scss` order**: `@lisergia/styles`, then `utils/variables`, `base/` (fonts, reset), `shared/` (placeholders), `components/`, `sections/`, `pages/`.
- Lisergia projects still use `@import`, with Sass deprecations silenced in the Vite config (`silenceDeprecations: ['import', 'global-builtin', …]`). Don't convert a project to `@use` on the side.

## No Tailwind

Write SCSS, not Tailwind. The motion system lives in SCSS: shared placeholders that sections `@extend`, BEM `--active` states, easing tokens and rem values taken from the design. Utility classes in the markup don't express that well. If a client codebase already uses Tailwind, work inside it (see `history.md` for the rules those projects followed).
