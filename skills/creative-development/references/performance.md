# Performance, Accessibility and Reduced Motion

## What the Metrics Are For

Marketing landing pages follow an art director's creative vision. The goal is an elegant, surprising experience, and LCP, FCP or a Lighthouse score are not the target.

> "Prioritizing animations, motion and interactions in a website shouldn't be controversial. Not adding interesting things to your web pages because of metrics will always be a downgrade." (Luis Bizarro, from the Lisergia README)

So never remove motion, WebGL or a transition just to improve a score.

A Lighthouse 100 can also be faked with bad practices, so it proves little. For a creative developer, the real targets are **60fps** while scrolling and animating (the display's full refresh rate, 120 Hz included, for full-screen WebGL pieces), and **transitions that never block**. Profile frames during scroll, reveals and page transitions, not only the initial load. `rendering.md` has the frame budget, adaptive resolution and render-on-demand patterns.

When a client sends Lighthouse reports anyway, do a hygiene pass (font preloads, image sizes, priority hints, deferred analytics) and stop there.

Good scores and motion aren't opposites either: a WebGL portfolio full of video-texture shaders has scored 100 on mobile and desktop. Take the score when it comes for free, never trade the experience for it.

## Do the Hygiene Anyway, Because Agents Make It Cheap

With coding agents, the basic hygiene costs almost nothing, so do it every time **as long as it doesn't take anything away from the experience**:

- Semantic HTML, headings in order, `alt` text, focusable controls, visible focus states.
- Real content lives in server-rendered HTML, not only inside the canvas.
- Responsive images with correct sizes, and fonts preloaded without layout shift. The CMS image field requires `alt` and the frontend builds a `srcset` from it (see `sanity.md`). The first section's media loads with `fetchPriority="high"`, everything else lazily.
- Metadata, Open Graph tags and a sensible `<title>`, all from the document's `social` object. A favicon that follows the color scheme: two `<link rel="icon">` SVGs, light and dark, swapped on `matchMedia('(prefers-color-scheme: dark)')` changes, plus `apple-touch-icon` and `theme-color`.
- **A canvas reachable without a pointer.** Put the real content in plain HTML links: a visually hidden list that appears when it takes keyboard focus. Focusing a link fires an event that moves the scene to that item, the canvas is `aria-hidden`, and the page's `<h1>` stays in the DOM even when it's visually hidden.
- `prefers-reduced-motion`: tone motion down instead of removing the design. Shorten or skip scroll-driven and parallax motion, turn off smooth-scroll inertia, and replace large movement with opacity fades. The WebGL scene can stay, running calmer.
- **Server-rendered React**: decide reduced motion and device-dependent upgrades after mount. Render the safe default on the server (the plain line, the tinted pill without the WebGL glass) and upgrade on the client. React doesn't patch markup that differs at hydration, so a server-rendered mask can stay stuck.

## The One Hard Floor

**The first rendered frame never flickers or visibly swaps.** No flash of unstyled content, no layout jump when fonts load, no blank canvas that suddenly fills in, and no reveal element visible before its animation starts (hide it in its initial state from CSS, not from JavaScript after load).

For WebGL, use the blurred-poster fade and the load-everything-up-front rule in `webgl.md`.
