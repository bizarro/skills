# CSS: fluid rem scaling and few breakpoints

This applies to **websites**. Application UI (editors, tools) uses plain `px`, because tool UI should stay the same size and not scale with the viewport.

## One rem equals ten design pixels

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

- Use the design widths from the file (1440 and 375 are common; past projects have also used 1920 and 320).
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

## Keep breakpoints to a minimum

Because everything scales proportionally, you **don't need breakpoints to fix sizes**. Use a breakpoint only when the **layout** changes (columns stacking, navigation turning into a menu), and in practice that is almost always just the phone breakpoint.

- Lisergia defines three breakpoints with [include-media](https://eduardoboucas.github.io/include-media/): `phone: 768px`, `tablet: 1024px`, `desktop: 1920px`. `desktop` is mostly the "stop scaling" cap. A single `phone` breakpoint has been enough for whole portfolios.
- Don't add intermediate breakpoints to fine-tune sizes. If something looks wrong at a middle width, fix the rem value.

## Supporting tokens

- **Easings as CSS custom properties**: `--ease-{in,out,in-out}-{sine,quad,cubic,quart,quint,expo,circ,back}`. Reveals mostly use `var(--ease-out-expo)`.
- **Viewport height**: set `--100vh` from JavaScript on resize and use `var(--100vh)` for full-height sections, so mobile browser toolbars don't cause jumps.
- **Hover only where hover exists**: a `hover` mixin wraps rules in `html.desktop &:hover`, so touch devices never get stuck hover states.
- **Z-index from a map**: `z('navigation')` looks up the value in one `$z-indexes` list instead of scattering magic numbers.
- `%cover`: an absolute fill placeholder for media layers.

## Naming on sites

Sites use BEM (`block__element--modifier`), with one SCSS file per section, and import `@lisergia/styles` first. Reveal states are `--active` modifiers (see `motion.md`).
