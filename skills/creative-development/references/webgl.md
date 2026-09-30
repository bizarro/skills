# WebGL and WebGPU

## Put WebGL on every page

Every marketing or landing page gets a WebGL layer. It impresses clients and visitors every time. When someone asks whether a page "needs" WebGL, the answer is yes. The engineering job is to make it load cleanly, not to argue it away for performance reasons.

## Pick the library by what the scene needs

Ask what the scene has to look like before choosing a library. Don't pick by habit.

- **three.js**: use it when the scene has to look realistic. The test is whether it needs physically based rendering: Filament-style PBR shading, `MeshStandardMaterial`/`MeshPhysicalMaterial`, environment lighting, tone mapping, glTF models. three.js has that pipeline built in, and rebuilding it by hand is not worth it.
- **OGPU** ([oframe/ogpu](https://github.com/oframe/ogpu)): the default for light work such as image planes, distortions, full-screen shaders, particles and gradients. It is the WebGPU successor to OGL and more future-proof. It is not published to npm. Clone the repository into the project and grow it, rather than importing it.
- **OGL** ([oframe/ogl](https://github.com/oframe/ogl)): the same kind of light work on WebGL. OGPU is replacing it in new projects. Keep OGL where it already exists, or use it when the project has to target WebGL specifically.
- **React Three Fiber**: the last resort. Use it only when the project is already a React application **and** much of the UI state has to drive the scene interactively. In any other case it is overkill: it adds a reconciler, React's render cycle and a component model on top of a scene that one class and one render loop could run.

When the project is React and the scene has to react to UI state but R3F is not justified, keep the scene in plain three.js or OGPU and share state through MobX. See `react.md`.

When using OGPU, check `navigator.gpu` before creating the renderer. Browsers without WebGPU keep the poster image described below instead of showing a blank or broken canvas.

## The first frame is the only performance floor that matters

Core Web Vitals scores are not the goal (see `performance.md`). The hard rule is that **the first thing the visitor sees never flickers, pops or swaps**.

The pattern (used on Apple product pages):

1. Render a **blurred poster** of the WebGL scene as a plain image in the HTML. It shows the same composition the canvas will draw, blurred so small differences don't show.
2. Mount the canvas behind the poster, or on top of it at `opacity: 0`.
3. Load, compile and upload everything the scene needs.
4. Wait for the first **real** rendered frame, then fade the poster out (or the canvas in) with a CSS transition.

The visitor sees an image that sharpens into a live scene and never sees a moment with no content.

Before revealing the scene, make sure the GPU has everything, because first-use shader compiles and texture uploads show up as a stall:

- Wait for `document.fonts.ready` before creating the canvas. Layout, and any text measured for the scene, is final only after fonts load.
- Decode images before uploading them (`await image.decode()`). For videos, wait for a real frame with `requestVideoFrameCallback` before showing the plane.
- three.js: `await renderer.compileAsync(scene, camera)` and `renderer.initTexture(texture)` for each texture.
- OGL/OGPU: create the programs and pipelines and upload the textures before starting the reveal.
- Each media plane fades in from an alpha uniform once its texture is ready. Never pop from empty to full.

## Loading strategy depends on the thread

- **Rendering on the main thread** (the usual case): load **everything** during the initial page load, behind the preloader or poster. Don't stream assets in while the experience runs. Decoding images and uploading textures on the main thread drops frames in the scroll and in the animations the visitor is watching.
- **Rendering in a worker** (`OffscreenCanvas`): progressive loading is fine, because the loading work no longer competes with the main thread's frames.

## DOM and canvas

Content stays in the DOM, where it is readable and indexable. The canvas is a fixed, full-viewport layer behind or above it. WebGL planes follow the bounds of their DOM elements: measure on resize, then offset by the smooth-scroll value every frame from the same loop that drives Lenis.

```ts
// On resize: world-space size of the viewport at the camera's distance.
const height = 2 * Math.tan((camera.fov * Math.PI) / 180 / 2) * camera.position.z
const width = height * camera.aspect

this.sizes = { x: width, y: height }
this.viewport = { x: window.innerWidth, y: window.innerHeight }

// Every frame, per plane. `bounds` is measured on resize relative to the document
// (getBoundingClientRect().top + scroll), so subtracting the Lenis scroll places it.
mesh.scale.x = (this.sizes.x * bounds.width) / this.viewport.x
mesh.scale.y = (this.sizes.y * bounds.height) / this.viewport.y

mesh.position.x = -this.sizes.x / 2 + mesh.scale.x / 2 + (bounds.left / this.viewport.x) * this.sizes.x
mesh.position.y = this.sizes.y / 2 - mesh.scale.y / 2 - ((bounds.top - scroll) / this.viewport.y) * this.sizes.y
```

The HTML `<img>`/`<video>` stays in the page, hidden visually, as the source of layout, `alt` text and the fallback. Play and pause videos with an IntersectionObserver so off-screen planes cost nothing. For effects that only make sense with a cursor (fluid simulations, hover distortions), a phone can show the plain HTML media instead.
