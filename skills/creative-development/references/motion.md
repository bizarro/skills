# Motion

## Reach for the Lightest Tool That Does the Job

1. **A CSS class toggle** is enough for most reveals. JavaScript only decides _when_ (an IntersectionObserver adds or removes a class) and CSS does the animating.
2. **Split text plus inline transitions** for titles and paragraphs. Split with anime.js `splitText`, set each word's or line's initial `transform` inline, then set the final transform with a staggered `transition-delay` when it enters the viewport.
3. **anime.js** (v4) timelines for choreographed sequences such as a hero intro. It can animate CSS custom properties as well as transforms.
4. **The Web Animations API** (`element.animate`) for one-off imperative animations such as page transitions. It is built in and costs no bundle size.
5. **GSAP only if the user or an existing codebase already uses it.** It is heavy for what these sites need, so don't add it by default. Most of Luis's sites before August 2026 are built on GSAP; keep it there (see `history.md`).
6. **`motion/react`** inside a client's React or Next.js stack, with a few shared primitives (`Reveal`, `SplitLines`) instead of ad hoc animations per component.

The same goes for other UI libraries. For a simple slider, native CSS Scroll Snap looks better than loading a JavaScript slider library.

## The Reveal Pattern

The value of the attribute is the class to toggle:

```tsx
<p className="intro__label" data-reveal="intro__label--active">
```

```ts
import { Component } from '@lisergia/core'

import { observeIntersection } from '../utilities/intersection'

export default class Reveal extends Component {
  declare classes: {
    active: string
  }

  declare element: HTMLElement

  constructor({ element }: { element: HTMLElement }) {
    super({
      classes: {
        active: element.dataset.reveal!,
      },
      element,
    })
  }

  declare unobserve?: () => void

  createObserver() {
    this.unobserve = observeIntersection(this.element, (isIntersecting) => {
      if (isIntersecting) {
        this.animateIn()
      } else {
        this.animateOut()
      }
    })
  }

  destroyObserver() {
    this.unobserve?.()
    this.unobserve = undefined
  }

  animateIn() {
    this.element.classList.add(this.classes.active)
  }

  animateOut() {
    this.element.classList.remove(this.classes.active)
  }

  addEventListeners() {
    this.createObserver()
  }

  removeEventListeners() {
    this.destroyObserver()
  }
}
```

`observeIntersection` shares one `IntersectionObserver` per scroll root and margin across every target, and returns a function that stops observing. Its root is `element.closest('.page')`, not the viewport: Lenis scrolls the `.page` element, which clips its children, so a viewport-rooted margin could never reach content inside it.

Shared placeholders hold the motion, and each section `@extend`s them:

```scss
%animation-letter-spacing {
  letter-spacing: 0.5rem;
  opacity: 0;
  transition:
    letter-spacing 1s var(--ease-out-expo),
    opacity 1s var(--ease-out-expo);

  &--active {
    letter-spacing: 0.05rem;
    opacity: 1;
  }
}

.intro__label {
  @extend %animation-letter-spacing;

  &--active {
    @extend %animation-letter-spacing--active;
  }
}
```

Rules that keep it clean:

- **The hidden state lives in CSS**, so the element is never visible before JavaScript runs. That prevents first-frame flashes.
- The feel comes from **long, expo-out transitions**: around `1s`–`1.5s` with `var(--ease-out-expo)` (`cubic-bezier(0.19, 1, 0.22, 1)`), and staggers of about `0.1s` per word or line. In React client stacks the shared curve has been `cubic-bezier(0.22, 1, 0.36, 1)`, fast out of the gate with a long settle, with reveals that combine opacity, translate and a little blur over about `0.9s`. Keep one curve per project and name it once.
- **Exits are quicker than entrances.** Leaving should never be something the visitor waits through: a page fades out in about 180 ms and in over 240 ms, a menu opens in 0.8 s and closes in 0.5 s. Ease out for entrances, ease in for exits.
- Animate `transform` and `opacity`. Use `translate3d` for per-frame values. Menus and panels reveal with `clip-path` and `transform`, never `top` or `height`.
- Reveals animate out when the element leaves the viewport and animate in again when it returns, unless the design says they play once. React client stacks have defaulted to once, triggered when the element reaches the middle of the viewport.
- Useful behaviors come as small `data-*` attribute components: `data-reveal`, `data-title="left,top,bottom"` (direction per word), `data-paragraph`, `data-parallax`, `data-translate="<speed>"`, and `data-src` (a lazy image that sets `src` on intersect, then adds `.loaded` for an opacity fade). Optional modifiers are `data-animation-delay` and `data-animation-target` (observe an ancestor instead of the element).
- Load these behaviors lazily with a dynamic `import()` for each selector, so a page only downloads the code it uses.

## Scroll and the Frame Loop

- Smooth scroll comes from **[Lenis](https://lenis.darkroom.engineering/)**. Every scroll-linked effect (parallax, WebGL planes, marquees) reads Lenis's scroll value, not `window.scrollY`, so the DOM and the canvas move together.
- Run **one `requestAnimationFrame` loop** that ticks Lenis first and then everything else. Several independent loops drift out of sync and waste frames.
  - [Tempus](https://github.com/darkroomengineering/tempus) is how Lisergia coordinates that loop (`Tempus.add(callback)`). It is useful, but not always needed with today's browsers. A single loop owned by the application or canvas class does the same job.
- Never read layout inside the loop. `getBoundingClientRect()`, `offsetTop` and friends force a reflow every frame. Measure on resize, cache the bounds, and combine them with the scroll value in the loop.
- Lenis scrolls the page element (`wrapper: .page`, `content: .page__wrapper`), not the window, and a ResizeObserver on the wrapper calls `lenis.resize()`. Anything that observes or measures scroll uses `.page` as its root.
- Scroll-linked values come from interpolation: map the element's position to the output range, clamp it, and ease where needed. Ease with a time-based `damp`, not a fixed `lerp` factor, so motion feels the same at 60 and 120 Hz (see `rendering.md`).

In Lisergia, `onResize()` only measures and `onScroll(scroll)` only writes. Resize events are merged into one frame, which fires `resize` for every component and then `scroll`, so all reads happen before any write. `onScroll` only runs while the component is in view:

```ts
onResize() {
  this.bounds = DOMUtils.getBounds(this.element, this.application!.scroll)
}

onScroll(scroll: number) {
  const scale = MathUtils.map(this.bounds.top - scroll, -this.bounds.height, Viewport.height, 1, 1.2, true)

  this.elements.media.style.transform = `scale(${scale})`
}
```

## Page Transitions

On bespoke sites, navigate without full reloads:

1. Intercept same-origin link clicks.
2. `fetch` the next URL and parse it with `DOMParser`.
3. Read its template name (`html[data-template]`) and its page element.
4. Animate the current page out with WAAPI, destroy it, append the new page, create it, animate it in, and call `history.pushState`. Handle `popstate`.

```ts
const animateOpacity = (element: HTMLElement, from: number, to: number) => {
  return element.animate({ opacity: [from, to] }, { duration: 1000, fill: 'forwards' }).finished
}
```

Under `prefers-reduced-motion`, skip the fades and swap instantly.

Scroll can drive navigation too. Reaching the end of a case study can take the visitor to the next project. Owning the scroll like this is far easier in plain JavaScript than inside a framework's router.

The details that make it hold up:

- **Prefetch** on `pointerover`, `focusin` and `touchstart`, into a session cache keyed by URL. Skip it when Save-Data is on and in CMS preview.
- **Guard** against a second click while loading and against navigating to the current URL. Set `html.transitioning`, which kills pointer events until the new page is in.
- **Both pages live in the DOM during the transition.** Page classes select `.page:last-child`, so the new page can be appended and animated in while the old one animates out (an overlap: the old page scales to 0.8 and fades, the new one slides up from `100%`). Remove the old node afterwards.
- Copy `document.title` from the fetched document, set `history.scrollRestoration = 'manual'`, and after the swap scroll to `location.hash` if there is one.
- **Tell every component.** The application calls `onTransitionStart()` and `onTransitionEnd()` on each global component, so the canvas can rebuild its scene and the cursor can rebind to the new links.
- With WAAPI, write the final value inline and call `animation.cancel()` instead of keeping `fill: 'forwards'` animations alive.
- **WebGL shared element.** When a gallery opens a case study, the clicked plane stays on the canvas and interpolates between its bounds on the old page and the media bounds on the new one. The DOM curtain is skipped for that route pair, because the canvas does the transition.
- **Per-page colors.** The page root carries `data-background` and `data-color`. On navigation, tween them on `<html>` and on the WebGL clear color together, so canvas and page never mismatch.

## Preloader

A preloader is part of the intro, not a progress bar to get past.

- Count real assets: images that are `complete`, videos with `readyState >= 3`, fonts, the sprite sheet, and the textures uploaded to the GPU.
- Ease the number toward the real percentage every frame, so it never jumps: `percent = lerp(percent, loaded / total * 100, 0.1)`.
- Wait for both the minimum intro time and the assets, with a ceiling so a slow asset can't hold the site hostage: `Promise.all([wait(MINIMUM), Promise.race([assets, wait(CEILING)])])`.
- Let a looping intro finish its cycle before the exit plays, so it is never cut mid-motion.
- Stop Lenis while it shows, add `html.preloaded` when it ends, then resize everything and create the WebGL meshes from the already uploaded textures before the page shows. The first page's reveals can start during the preloader's exit so they overlap.
- A logo or media can fly from the preloader to its place in the layout (measure the target's bounds, animate to them), so the hand-off reads as one motion.

## Marquees, Galleries and Infinite Lists

Wrap items with an `extra` offset instead of cloning on the fly. The markup repeats the item enough times to cover the widest screen, each item tracks its own `extra`, and only the item that leaves one edge jumps by the total width:

```ts
this.scroll.target += this.multiplier

this.scroll.current = MathUtils.lerp(this.scroll.current, this.scroll.target, this.scroll.ease)

this.direction = this.scroll.current < this.scroll.last ? 'down' : 'up'

this.items.forEach((item) => {
  item.position = -this.scroll.current - item.extra

  const bounds = item.position + item.offset + item.width

  item.isBefore = bounds < 0
  item.isAfter = bounds > this.widthTotal

  if (this.direction === 'up' && item.isBefore) {
    item.extra -= this.widthTotal
  }

  if (this.direction === 'down' && item.isAfter) {
    item.extra += this.widthTotal
  }

  item.element.style.transform = `translate3d(${Math.floor(item.position)}px, 0, 0)`
})

this.scroll.last = this.scroll.current
```

- Reset every transform to 0 before measuring, and re-measure only when the list width actually changes.
- Pause it offscreen, and flip the constant drift with the wheel direction.
- The same algorithm wraps WebGL planes in world units (see `webgl.md`).
- **Drag** multiplies the distance (about 1.5 on desktop, 2.5 on touch), locks to an axis only past about 30 px so vertical scrolling still works, and snaps to the nearest item on release. A press that moved less than a few pixels is a click: call `.click()` on the item's real `<a>`, so routing stays in the DOM.

## Split Text

- Keep it readable: the heading gets an `aria-label` with the full text, and every split word gets `aria-hidden`. With anime.js, `splitText(element, { accessible: false, words: '<div><div data-word="{i}">{value}</div></div>' })` gives each word a clipping wrapper.
- Re-split lines on resize (`split.addEffect(this.refreshLines)`) and re-apply the current in or out state.
- Hide text with `.js [data-paragraph] { visibility: hidden }` until it has been split, so it never flashes unsplit.
- Scramble effects use anime.js `scrambleText` per character with a short stagger. Lock each word's width while it scrambles so the line doesn't jitter, and replay on `pointerenter` and `focus`.

## Navigation, Footer and Cursor

- **Menu state lives on `<html>`** (`navigation--open`), and the menu closes on every route change. The page behind it shrinks with `clip-path: inset(... round 1rem)` and a transform.
- **Hide the header on scroll down**: accumulate scroll travel, reset it when the direction reverses, and toggle past a small threshold (about 8 px).
- **Footer reveal**: the footer is fixed behind the content and a spacer at the end of the page takes its height (written to `--height` only when it changes). The content scales from 1 to about 0.95 as the footer appears, with `will-change` set only while it scales.
- **Cursor**: create it only on `html.desktop`. Lerp it toward the pointer, write the position to custom properties, and let `data-cursor="play|plus|view"` on any element pick its state or label. Rebind its listeners after every page transition. A preloader can mirror its percentage into the cursor.
