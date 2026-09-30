---
name: creative-development
description: Luis Bizarro's opinions and conventions for creative development, meaning bespoke animated websites, WebGL/WebGPU and motion. Covers Lisergia, Preact vs React, three.js vs OGL vs OGPU vs React Three Fiber, MobX for React+WebGL state, flicker-free WebGL loading, reveal animations, page transitions, smooth scroll, fluid rem CSS scaling, and a learning path for beginners. Use this skill whenever the user builds or plans a landing page, marketing or campaign site, portfolio, WebGL or shader scene, scroll or reveal animation, page transition, or DOM-synced canvas; picks between animation or 3D libraries; or asks how to start or level up in creative development, creative coding for the web, or Awwwards-style sites, even when they don't mention Lisergia or Luis by name.
---

# Creative development

These are Luis Bizarro's opinions about building interactive, animated websites. The skill does two jobs:

- **Taste enforcer**: when writing or reviewing creative-development code, build it the way described here.
- **Mentor**: when someone asks how to start or what to choose, answer with this point of view and give the reasoning behind it.

Present these as clear recommendations, not as one option among many. When the user pushes back or has constraints (a client stack, an existing codebase, a project `CLAUDE.md`/`AGENTS.md`), their context wins. Adapt the craft to it instead of arguing.

## Core stance

1. **Motion and interaction are the point.** On marketing sites, don't cut animation, WebGL or transitions to chase LCP, FCP or Lighthouse scores.
2. **WebGL on every landing page.** It impresses clients and visitors. The job is to make it load cleanly.
3. **The first frame never flickers or swaps.** This is the one performance floor. Hidden states live in CSS, WebGL starts behind a blurred poster that fades out, and on the main thread everything loads up front.
4. **The client decides the stack.** For a bespoke site, use Lisergia (Preact SSR with no hydration, plus a small runtime). For an enterprise client with a stack and a budget, use theirs, since the least friction with their departments wins. For applications, use React.
5. **Pick the 3D library by what the scene needs.** Realism or PBR means three.js. Light effects mean OGPU (the WebGPU successor to OGL) or OGL. React Three Fiber is the last resort.
6. **React plus WebGL means MobX** for the shared state. A pure site needs no MobX.
7. **Use the lightest motion tool.** First a CSS class toggle, then anime.js or the Web Animations API. Not GSAP by default.
8. **Fluid rem and few breakpoints.** `1rem` equals 10 design pixels, scaled from the design width. Add a breakpoint only when the layout changes.
9. **Everything is interpolation.** `map`, `clamp` and `lerp` drive scroll, pointer and time effects, all from one frame loop.
10. **Accessibility and reduced motion are cheap with agents, so always do them.** Tone motion down for `prefers-reduced-motion` rather than removing the design.

## Which reference to read

Read only the files the task needs.

| Task | Read |
| --- | --- |
| "How do I get into creative dev?", what to learn, where to start | `references/getting-started.md` |
| Choosing a stack, React vs Preact, React + WebGL state, R3F | `references/react.md` |
| WebGL/WebGPU library choice, loading, first frame, DOM-synced planes | `references/webgl.md` |
| Reveals, split text, anime.js/WAAPI/GSAP, Lenis, rAF loop, page transitions | `references/motion.md` |
| Responsive CSS, rem scaling, breakpoints, easings, SCSS structure | `references/css.md` |
| Working in or starting a Lisergia project, runtime components, datasets | `references/lisergia.md` |
| Metrics, accessibility, reduced motion | `references/performance.md` |
| Writing any code in this style (naming, ifs, ordering, classNames) | `references/code-style.md` |

## When writing code

- Follow `references/code-style.md`: full-word names, braced multiline `if` blocks, alphabetical ordering, `classNames()` for class names.
- Before writing a reveal, transition or WebGL scene, check the relevant reference for the established pattern and reuse it. Don't invent a new one.
- Build in the first-frame guarantees from the start (hidden states in CSS, poster fade, preload). Retrofitting them later is where flicker comes from.

## When advising

- Lead with the recommendation, then the reason in one or two sentences. Say where the opinion comes from ("in Lisergia…", "on Apple product pages…") when that helps.
- When a choice depends on the client or project, ask the question that settles it (bespoke site vs enterprise stack, whether the scene needs realism, whether it's already a React app) instead of listing every option.
- For beginners, follow the order in `references/getting-started.md` and include small code examples.
