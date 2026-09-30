# Performance, accessibility and reduced motion

## What the metrics are for

Marketing landing pages follow an art director's creative vision. The goal is an elegant, surprising experience, and LCP, FCP or a Lighthouse score are not the target.

> "Prioritizing animations, motion and interactions in a website shouldn't be controversial. Not adding interesting things to your web pages because of metrics will always be a downgrade." (Luis Bizarro, from the Lisergia README)

So never remove motion, WebGL or a transition just to improve a score.

## Do the hygiene anyway, because agents make it cheap

With coding agents, the basic hygiene costs almost nothing, so do it every time **as long as it doesn't take anything away from the experience**:

- Semantic HTML, headings in order, `alt` text, focusable controls, visible focus states.
- Real content lives in server-rendered HTML, not only inside the canvas.
- Responsive images with correct sizes, and fonts preloaded without layout shift.
- Metadata, Open Graph tags and a sensible `<title>`.
- `prefers-reduced-motion`: tone motion down instead of removing the design. Shorten or skip scroll-driven and parallax motion, turn off smooth-scroll inertia, and replace large movement with opacity fades. The WebGL scene can stay, running calmer.

## The one hard floor

**The first rendered frame never flickers or visibly swaps.** No flash of unstyled content, no layout jump when fonts load, no blank canvas that suddenly fills in, and no reveal element visible before its animation starts (hide it in its initial state from CSS, not from JavaScript after load).

For WebGL, use the blurred-poster fade and the load-everything-up-front rule in `webgl.md`.
