# Getting started in creative development

Use this file when someone asks how to get into creative development, what to learn first, or how to level up from "regular" front-end work. Answer as a mentor with a point of view. Give the path below in order, explain why each step matters, and show a small code example for the first steps. Don't hand them a neutral list of every framework.

## 1. Interpolation: `map`, `clamp`, `lerp`

Nearly everything in creative development is turning one range of numbers into another: scroll position into scale, mouse position into rotation, time into opacity. Learn these three functions until you can write them from memory:

```ts
const clamp = (value: number, min: number, max: number) => {
  return Math.min(Math.max(value, min), max)
}

const lerp = (start: number, end: number, amount: number) => {
  return start + (end - start) * amount
}

const map = (value: number, inMin: number, inMax: number, outMin: number, outMax: number) => {
  return outMin + ((value - inMin) / (inMax - inMin)) * (outMax - outMin)
}
```

Show how they combine:

- **`map`**: scroll progress through a section becomes `scale` from `1` to `1.2`.
- **`clamp`**: keep that value from overshooting past the section.
- **`lerp`**: `current = lerp(current, target, 0.1)` every frame. Mouse followers, smooth parallax and eased values in general come from this one line.

## 2. CSS with rem and very few breakpoints

Master CSS before reaching for WebGL. The key trick: set `html { font-size: calc(100vw / 1440 * 10) }` so `1rem` equals 10 design pixels, write every size in rem, and the whole layout scales with the viewport as designed. Breakpoints are then only needed when the **layout** changes, which is usually just phone vs desktop. Most people write too many breakpoints because they are patching sizes, not layouts. See `css.md` for the full setup.

## 3. Then build up, in this order

These steps follow the stack the rest of this skill describes:

1. **Reveals with a class toggle**: an IntersectionObserver adds a class, and CSS transitions with `var(--ease-out-expo)` do the rest. Most "wow" reveals are just this (`motion.md`).
2. **Smooth scroll and one frame loop**: add Lenis and drive every scroll-linked effect from one `requestAnimationFrame` loop, using `map`, `clamp` and `lerp` from step 1.
3. **Page transitions**: fetch the next page, swap the DOM and animate with the Web Animations API.
4. **WebGL planes that follow the DOM**: images and videos rendered as planes that sync with their HTML elements' bounds and the smooth scroll, with shaders for distortion. Use OGPU or OGL here (`webgl.md`).
5. **three.js when you need realism**: PBR materials, environment lighting, glTF models.

## Study real code

- [Lisergia](https://github.com/bizarro/lisergia): the full stack for animated marketing sites.
- [bizarro/2024](https://github.com/bizarro/2024): a portfolio built on the Lisergia stack, with OGL media planes and a fluid simulation.
- [bizarro/bizar.ro](https://github.com/bizarro/bizar.ro): the 2020–2023 portfolio, an older OGL + GSAP generation of the same ideas.

## Mindset

- Motion and interaction are the product on these sites. Metrics are not a reason to ship something boring (`performance.md`).
- Put WebGL on everything; clients and visitors remember it (`webgl.md`).
