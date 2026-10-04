# Getting Started in Creative Development

Use this file when someone asks how to get into creative development, what to learn first, how to level up from "regular" front-end work, or for career advice. Answer as a mentor with a point of view. Give the path below in order, explain why each step matters, and show a small code example for the first steps. Don't hand them a neutral list of every framework.

## Before WebGL: The Fundamentals

With less than about two years of JavaScript, don't jump straight into WebGL. Without the basics of JavaScript and the web, you'll be a bad WebGL developer. Luis worked four-plus years with HTML, CSS, JavaScript and SVG before his first WebGL code, and his first Awwwards Site of the Day had no WebGL at all. Say this honestly and without gatekeeping: WebGL shouldn't be the focus just because everyone else is doing it.

Check whether they can answer these. If not, that's what to study first:

1. **Asset loading.** How does a site load assets, and how do you preload them? When is the right moment to upload a texture to the GPU? `new Texture()` does not optimize anything for you.
2. **The `requestAnimationFrame` loop and browser rendering.** Calling `getBoundingClientRect()` or anything else that forces a reflow inside the loop costs a layout every frame. Measure on resize, read cached values in the loop.
3. **`map` and `lerp`.** Most values that reach a shader (colors, vectors, progress) change through them in a rAF loop. See step 1 below.
4. **OOP.** What a class is, what a singleton is. Most WebGL libraries are object-oriented, and a scene is built from classes (`new Globe()`, `new Particles()`) listening to events. [patterns.dev](https://www.patterns.dev/) is a free, excellent source for the patterns.
5. **Vectors and `Math.PI`.** `Vector2`, `Vector3`, translating, rotating and scaling. Start with Canvas 2D and plain X and Y, then move to 3D.
6. **Shaders last.** Don't write custom shaders until an effect needs them. Three.js's built-in materials on a good model look better than a basic custom shader.
7. **Read other people's code.** CodePen and GitHub have endless experiments to take apart and learn from. Save the ones that show what's possible.

Their first task at a studio that does WebGL will most likely be animations and transitions for components and pages, not shaders. Doing those basics very well is what makes a difference on a project.

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

Once that clicks, learn why a fixed `0.1` is a trap: it runs twice as fast on a 120 Hz screen as on a 60 Hz one. Make the amount depend on elapsed time:

```ts
const damp = (speed: number, delta: number) => {
  return 1 - Math.exp(-speed * delta)
}

current = lerp(current, target, damp(6, delta)) // delta in seconds since the last frame
```

## 2. CSS with rem and Very Few Breakpoints

Master CSS before reaching for WebGL. The key trick: set `html { font-size: calc(100vw / 1440 * 10) }` so `1rem` equals 10 design pixels, write every size in rem, and the whole layout scales with the viewport as designed. Breakpoints are then only needed when the **layout** changes, which is usually just phone vs desktop. Most people write too many breakpoints because they are patching sizes, not layouts. See `css.md` for the full setup.

## 3. Then Build Up, in This Order

These steps follow the stack the rest of this skill describes:

1. **Reveals with a class toggle**: an IntersectionObserver adds a class, and CSS transitions with `var(--ease-out-expo)` do the rest. Most "wow" reveals are just this (`motion.md`).
2. **Smooth scroll and one frame loop**: add Lenis and drive every scroll-linked effect from one `requestAnimationFrame` loop, using `map`, `clamp` and `lerp` from step 1.
3. **Page transitions**: fetch the next page, swap the DOM and animate with the Web Animations API.
4. **WebGL planes that follow the DOM**: images and videos rendered as planes that sync with their HTML elements' bounds and the smooth scroll, with shaders for distortion. Use OGPU or OGL here (`webgl.md`).
5. **Three.js when you need realism**: PBR materials, environment lighting, glTF models.

## Study Real Code

- [Lisergia](https://github.com/bizarro/lisergia): the full stack for animated marketing sites.
- [bizarro/2024](https://github.com/bizarro/2024): a portfolio built on the Lisergia stack, with OGL media planes and a fluid simulation.
- [bizarro/bizar.ro](https://github.com/bizarro/bizar.ro): the 2020–2023 portfolio, an older OGL + GSAP generation of the same ideas.
- [Building an Immersive Creative Website from Scratch without Frameworks](https://www.awwwards.com/academy/course/building-an-immersive-creative-website-from-scratch-without-frameworks): Luis's Awwwards course, a full site in plain JavaScript and WebGL.
- Codrops tutorials by Luis, both with OGL and GLSL: [Infinite Auto-Scrolling Gallery](https://tympanus.net/codrops/2021/01/05/creating-an-infinite-auto-scrolling-gallery-using-webgl-with-ogl-and-glsl-shaders/) and [Infinite Circular Gallery](https://tympanus.net/codrops/2021/02/23/creating-an-infinite-circular-gallery-using-webgl-with-ogl-and-glsl-shaders/).

## Mindset

- Motion and interaction are the product on these sites. Metrics are not a reason to ship something boring (`performance.md`).
- Put WebGL on everything; clients and visitors remember it (`webgl.md`). That's for production sites. While learning, the fundamentals above come first.
- Learn plain JavaScript, not a framework. Most of what makes these sites special (scroll hijacking, transitions, a canvas synced to the DOM) is easier without one.

## Career

- **Say "let's try" to designers.** Luis's career started to shift when he stopped saying "no" to designers. The developer who finds a way to build the ambitious version is the one people want to work with.
- **Make small experiments and post them.** A typography shader with lyrics from a song you love, a distortion on your own portfolio. Tiny, finished and shared beats big and unfinished.
- **Open-source your work.** A public portfolio repo teaches others how you think and shows studios how you code.
- **Reach out directly.** Messaging people you want to work with does work. That's how Luis ended up at Apple building 3D viewers.
- **Own your outcomes.** Most of what Luis achieved came from acting in "founder mode", not from waiting for company processes to hand out opportunities.
