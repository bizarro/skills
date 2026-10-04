# WebGL and WebGPU

## Put WebGL on Every Page

Every marketing or landing page gets a WebGL layer. It impresses clients and visitors every time. When someone asks whether a page "needs" WebGL, the answer is yes. The engineering job is to make it load cleanly, not to argue it away for performance reasons.

## Pick the Library by What the Scene Needs

Ask what the scene has to look like before choosing a library. Don't pick by habit.

- **Three.js**: use it when the scene has to look realistic. The test is whether it needs physically based rendering: Filament-style PBR shading, `MeshStandardMaterial`/`MeshPhysicalMaterial`, environment lighting, tone mapping, glTF models. Three.js has that pipeline built in, and rebuilding it by hand is not worth it.
- **OGPU** ([oframe/ogpu](https://github.com/oframe/ogpu)): the default for light work such as image planes, distortions, full-screen shaders, particles and gradients. It is the WebGPU successor to OGL and more future-proof. It is not published to npm. Clone the repository into the project and grow it, rather than importing it.
- **OGL** ([oframe/ogl](https://github.com/oframe/ogl)): the same kind of light work on WebGL. OGPU is replacing it in new projects. Keep OGL where it already exists, or use it when the project has to target WebGL specifically.
- **React Three Fiber**: the last resort. Use it only when the project is already a React application **and** much of the UI state has to drive the scene interactively. In any other case it is overkill: it adds a reconciler, React's render cycle and a component model on top of a scene that one class and one render loop could run.

When the project is React and the scene has to react to UI state but R3F is not justified, keep the scene in plain Three.js or OGPU and share state through MobX. See `react.md`.

When using OGPU, check `navigator.gpu` before creating the renderer. Browsers without WebGPU keep the poster image described below instead of showing a blank or broken canvas. Vendor it under `vendor/ogpu`, exclude that folder from Biome, and on device loss or a shader compile error destroy the canvas and fall back to the DOM images.

Two more cases from recent work:

- **GPU compute at scale** (hundreds of thousands of particles driven by compute kernels) has been built with `three/webgpu` and TSL (`instancedArray` storage buffers, `RenderPipeline`), because the compute and post-processing nodes come with it. Fill rate, not the compute, is usually the cost.
- **Editors and tools** use `three/webgpu` without the WebGL fallback. Import everything from `three/webgpu`, never mix it with `three`.

## The First Frame Is the Only Performance Floor That Matters

Core Web Vitals scores are not the goal (see `performance.md`). The hard rule is that **the first thing the visitor sees never flickers, pops or swaps**.

The pattern (used on Apple product pages):

1. Render a **blurred poster** of the WebGL scene as a plain image in the HTML. It shows the same composition the canvas will draw, blurred so small differences don't show.
2. Mount the canvas behind the poster, or on top of it at `opacity: 0`.
3. Load, compile and upload everything the scene needs.
4. Wait for the first **real** rendered frame, then fade the poster out (or the canvas in) with a CSS transition.

The visitor sees an image that sharpens into a live scene and never sees a moment with no content.

Making the poster:

- Capture it from the scene itself, and capture it again whenever the scene changes. Seed every random layout so the first frame is the same on every load (see `rendering.md`), or the poster stops matching.
- Ship it sharp and blur it in CSS (`filter: blur(18px); transform: scale(1.06)`, the scale hides the blurred edges), with `<link rel="preload" as="image">` in the head.
- The canvas fades in with a CSS transition on `html.is-ready`, which the code adds in the `requestAnimationFrame` after the first real render.
- For CMS media, generate a 20px-wide base64 preview of every image (and of every video's first frame) at build time. It is the placeholder texture until the real one is uploaded, and the DOM blur under a hero video (see `sanity.md`).

The load sequence: `await document.fonts.ready`, update the scene once at time 0, `await renderer.compileAsync(scene, camera)`, `renderer.initTexture()` for each texture, render once, then add `is-ready` and start the loop in the next frame.

Before revealing the scene, make sure the GPU has everything, because first-use shader compiles and texture uploads show up as a stall:

- Wait for `document.fonts.ready` before creating the canvas. Layout, and any text measured for the scene, is final only after fonts load.
- Decode images before uploading them (`await image.decode()`). For videos, wait for a real frame with `requestVideoFrameCallback` before showing the plane.
- Three.js: `await renderer.compileAsync(scene, camera)` and `renderer.initTexture(texture)` for each texture.
- OGL/OGPU: create the programs and pipelines and upload the textures before starting the reveal.
- Each media plane fades in from an alpha uniform once its texture is ready. Never pop from empty to full.

## Loading Strategy Depends on the Thread

- **Rendering on the main thread** (the usual case): load **everything** during the initial page load, behind the preloader or poster. Don't stream assets in while the experience runs. Decoding images and uploading textures on the main thread drops frames in the scroll and in the animations the visitor is watching.
- **Rendering in a worker** (`OffscreenCanvas`): progressive loading is fine, because the loading work no longer competes with the main thread's frames.

The exception is media that belongs to individual items rather than the scene: a still for each of fifty projects, hover loops, slides further down a story. Load the scene up front, then stream those after the first frame, one at a time and in priority order (the current slide and its neighbors first, an item the pointer is on immediately). Assign each texture only after `await image.decode()`, retry a failed one when its item comes up again, and warm up decodes before recording a capture.

## DOM and Canvas

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

## Media Planes

**The markup declares the planes.** `data-gl-media="<url>"` marks an element the canvas takes over, with `data-gl-placeholder`, `data-gl-media-delay` and `data-gl-background` as options. Once a plane is showing, the DOM copy hides above the phone breakpoint (`[data-gl-media][data-gl-media-active] { opacity: 0 }`). On phones that skip the canvas, copy `data-gl-media` into `src` and let the HTML show.

**One class tree.** `Canvas` owns the renderer, the camera and the loop. It creates a `Scene` per page, which creates one `Media` per element. `onLoop` and `onResize` cascade down the tree, and the scene is destroyed and rebuilt on every page transition. The canvas loop runs after Lenis in the same frame.

- All planes share one plane geometry, and where they can, one program, with per-mesh uniforms written in `onBeforeRender`.
- Share time by reference: `uTime: this.canvas.time`, where `time = { value: 0 }` is updated once per frame.
- Each plane fades in from an alpha uniform once its texture is ready, never popping from empty.

**One texture loader** with a `Map` cache keyed by URL, so two planes with the same image upload it once. Images are decoded before upload. For video, upload only new frames with `requestVideoFrameCallback`, falling back to polling `readyState >= HAVE_ENOUGH_DATA` and updating while the video plays. Mipmaps off for video.

**Object-fit: cover in the shader.** Pass the plane size and the media aspect as one `vec4`, and crop the UVs:

```ts
const aspect = media.height / media.width // videoHeight / videoWidth for video
const [x, y] = height / width > aspect ? [(width / height) * aspect, 1] : [1, height / width / aspect]

this.program.uniforms.uResolution.value = [width, height, x, y]
```

```glsl
vec2 uv = (vUv - vec2(0.5)) * uResolution.zw + vec2(0.5);
uv = 0.5 + (uv - 0.5) * (1.0 - 0.05 * uHover); // zoom in slightly on hover
```

**Scroll velocity bends the planes.** Take the speed from the scroll delta (clamped, then eased), and push the vertices along z with a sine across the viewport:

```glsl
newPosition.z += sin(newPosition.y / uViewportSizes.y * PI + PI / 2.0) * uSpeed;
```

**Infinite galleries** wrap planes in world units with the same `extra` offset as DOM marquees (see `motion.md`): a plane past one edge of the visible area moves by the total width of the gallery.

**Clicks stay in the DOM.** Raycast the pointer against the planes for hover states and the pointer cursor, but on click call `.click()` on the plane's hidden `<a>`, so routing, history and accessibility stay with the links.

**Text in the scene** is drawn with Canvas 2D into textures (see `rendering.md`). Older OGL sites used MSDF text with a per-letter attribute to stagger glyph motion in the shader.
