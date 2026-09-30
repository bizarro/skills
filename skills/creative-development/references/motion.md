# Motion

## Reach for the lightest tool that does the job

1. **A CSS class toggle** is enough for most reveals. JavaScript only decides *when* (an IntersectionObserver adds or removes a class) and CSS does the animating.
2. **Split text plus inline transitions** for titles and paragraphs. Split with anime.js `splitText`, set each word's or line's initial `transform` inline, then set the final transform with a staggered `transition-delay` when it enters the viewport.
3. **anime.js** (v4) timelines for choreographed sequences such as a hero intro. It can animate CSS custom properties as well as transforms.
4. **The Web Animations API** (`element.animate`) for one-off imperative animations such as page transitions. It is built in and costs no bundle size.
5. **GSAP only if the user or an existing codebase already uses it.** It is heavy for what these sites need, so don't add it by default.

## The reveal pattern

The value of the attribute is the class to toggle:

```tsx
<p className="intro__label" data-reveal="intro__label--active">
```

```ts
import { Component } from '@lisergia/core'

export default class Reveal extends Component {
  declare classes: { active: string }
  declare element: HTMLElement
  declare observer: IntersectionObserver

  constructor({ element }: { element: HTMLElement }) {
    super({
      classes: {
        active: element.dataset.reveal!,
      },
      element,
    })
  }

  addEventListeners() {
    this.observer = new IntersectionObserver((entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          this.element.classList.add(this.classes.active)
        } else {
          this.element.classList.remove(this.classes.active)
        }
      })
    })

    this.observer.observe(this.element)
  }

  removeEventListeners() {
    this.observer.disconnect()
  }
}
```

Shared placeholders hold the motion, and each section `@extend`s them:

```scss
%animation-letter-spacing {
  letter-spacing: 0.5rem;
  opacity: 0;
  transition: letter-spacing 1s var(--ease-out-expo), opacity 1s var(--ease-out-expo);

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
- The feel comes from **long, expo-out transitions**: around `1s`–`1.5s` with `var(--ease-out-expo)`, and staggers of about `0.1s` per word or line.
- Animate `transform` and `opacity`. Use `translate3d` for per-frame values.
- Reveals animate out when the element leaves the viewport and animate in again when it returns, unless the design says they play once.
- Useful behaviors come as small `data-*` attribute components: `data-reveal`, `data-title="left,top,bottom"` (direction per word), `data-paragraph`, `data-parallax`, `data-translate="<speed>"`, and `data-src` (a lazy image that sets `src` on intersect, then adds `.loaded` for an opacity fade). Optional modifiers are `data-animation-delay` and `data-animation-target` (observe an ancestor instead of the element).
- Load these behaviors lazily with a dynamic `import()` for each selector, so a page only downloads the code it uses.

## Scroll and the frame loop

- Smooth scroll comes from **[Lenis](https://lenis.darkroom.engineering/)**. Every scroll-linked effect (parallax, WebGL planes, marquees) reads Lenis's scroll value, not `window.scrollY`, so the DOM and the canvas move together.
- Run **one `requestAnimationFrame` loop** that ticks Lenis first and then everything else. Several independent loops drift out of sync and waste frames.
  - [Tempus](https://github.com/darkroomengineering/tempus) is how Lisergia coordinates that loop (`Tempus.add(callback)`). It is useful, but not always needed with today's browsers. A single loop owned by the application or canvas class does the same job.
- Scroll-linked values come from interpolation: map the element's position to the output range, clamp it, and ease with `lerp` where needed.

```ts
const scale = MathUtils.map(top - scroll, -height, Viewport.height, 1, 1.2, true)

this.elements.media.style.transform = `translate3d(0, ${translateY}px, 0) scale(${scale})`
```

## Page transitions

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
