# Rendering Craft

How a WebGL piece holds its frame rate, looks finished, and gets tuned and recorded. `webgl.md` covers choosing a library and loading. This file covers what happens once the scene runs.

## Frame Budget

- Hold the display's refresh rate, not 60. On a 120 Hz screen the budget is 8.3 ms. Write the budget into the project's `AGENTS.md` and check frame rate at full-screen size before adding cost.
- Learn the refresh rate from the fastest frames (or the median `requestAnimationFrame` interval), instead of assuming it.
- Count draw calls for the whole frame, post-processing included: set `renderer.info.autoReset = false`, reset it at the start of the frame, and show `info.render.calls` in the debug panel.

## Frame-Rate Independence

`current = lerp(current, target, 0.1)` every frame runs twice as fast at 120 Hz as at 60. Damp by elapsed time instead, and clamp the delta so a hidden tab or a hitch can't make things jump:

```ts
// Frame rate independent smoothing factor for a value chasing its target.
export function damp(speed: number, delta: number) {
  return 1 - Math.exp(-speed * delta)
}

const delta = Math.min(this.clock.getDelta(), 0.1)

this.pointer.lerp(this.pointerTarget, damp(3, delta))
```

Simulations (fluids, cloth) step on a fixed accumulator (for example 120 steps per second) with the delta clamped to `1 / 30`, so the result doesn't depend on the display.

## Adaptive Resolution and Quality

Trade pixels for frames before the visitor notices. A `Resolution` class moves the pixel ratio in 0.25 steps between 0.75 and `min(devicePixelRatio, 2)`:

- Average a window of 30 frames, leaving out the 2 slowest, so one texture upload or tab switch doesn't count.
- Late frames (8% over the interval) drop a step. Three seconds of frames on time try a step up.
- Let a new ratio settle for a second before judging it, and don't retry a step that failed for 20 seconds, so it settles instead of oscillating.
- Turn it off while capturing and from the debug panel.

For particle counts and other quality steps, step down once while the median frame is slower than about 45 fps and never step back up, so it can't oscillate. Skip the first second, when compiles and uploads distort timings. Decide the starting tier from capabilities: a fine pointer, 8 or more cores and 8 GB or more of memory. Offer `?quality=low` for integrated GPUs and phones.

Follow pixel-ratio changes when a window moves between screens: re-arm a `matchMedia('(resolution: ${ratio}dppx)')` listener on every change.

## Render Only When Something Changes

- Stop the loop when the canvas leaves the viewport (an IntersectionObserver) and when the tab is hidden (`visibilitychange`). Pause the videos and suspend the audio with it.
- A scene that only reacts to input renders on demand: a layer reports `animating`, and the stage idles when none does.
- Under `prefers-reduced-motion`, keep the scene and scale its ambient motion down (multiply every sway, drift and parallax by about 0.35), or render one frame and stop.

## Post-Processing

Write the chain yourself as full-screen passes. Three.js's `EffectComposer` is fine for a sketch, not for a site.

1. Render the scene into a linear HDR target (`HalfFloatType`, 2× MSAA: 4× cost about 20% of a frame for little difference).
2. Optional motion blur and lens blur, with the shutter normalized to 60 fps (`shutter / Math.max(delta * 60, 0.25)`).
3. Bloom pyramid: a threshold prefilter, 13-tap downsamples and tent upsamples over about 6 levels.
4. One final pass: chromatic aberration, bloom plus a tinted halation, vignette, tone mapping (ACES), sRGB encoding, saturation and contrast in display space, then grain seeded per pixel and per frame.

```glsl
vec2 center = vUv - 0.5;
float distance = length(center);
vec2 shift = center * distance * uAberration;
vec3 color = vec3(texture(tFrame, vUv + shift).r, texture(tFrame, vUv).g, texture(tFrame, vUv - shift).b);

vec3 bloom = texture(tBloom, vUv).rgb;
color += mix(bloom, uBloomTint * dot(bloom, vec3(0.2126, 0.7152, 0.0722)), 0.35) * uBloom;
color *= 1.0 - smoothstep(0.25, 0.85, distance) * uVignette;
color = toSrgb(tonemap(color * uExposure));

float luminance = dot(color, vec3(0.2126, 0.7152, 0.0722));
color = mix(vec3(luminance), color, uSaturation);
color = (color - 0.5) * uContrast + 0.5;

float noise = hash12(floor(vUv * uResolution) + fract(uTime) * vec2(1931.0, 2713.0));
color += (noise - 0.5) * uGrain;
```

Each pass is one oversized triangle with a `RawShaderMaterial`. Shader files keep `#version 300 es` on their first line for tooling, and the pass strips it and sets `glslVersion`, because Three.js writes its own defines first:

```ts
function createTriangle() {
  const geometry = new BufferGeometry()

  geometry.setAttribute('position', new BufferAttribute(new Float32Array([-1, -1, 3, -1, -1, 3]), 2))

  geometry.setAttribute('uv', new BufferAttribute(new Float32Array([0, 0, 2, 0, 0, 2]), 2))

  geometry.boundingSphere = new Sphere(new Vector3(), 3)

  return geometry
}

function stripVersion(shader: string) {
  return shader.replace(/^\s*#version 300 es\s*/, '')
}
```

## Three.js Physically Based Scenes

- Use `MeshStandardMaterial`. Physical's clearcoat and sheen doubled the cost of every light.
- No `transmission`: it renders every opaque object again and builds mipmaps each frame. Fake glass in `onBeforeCompile` instead: turn diffuse off, blend with `ONE, ONE_MINUS_SRC_ALPHA`, take alpha from fresnel, and add an edge glow from `pow(1.0 - facing, 4.0)`.
- Materials lit only by the environment compile out the direct-light loop: replace `#include <lights_fragment_begin>` with a version without it.
- Snippets injected through `onBeforeCompile` live in `shaders/chunks/` without a version line.
- Use a prefiltered studio environment rather than many lights.

## Many Objects

- Merge the parts of an object that share a material into one geometry, then draw each material as one `InstancedMesh` across all objects. Dozens of objects become a handful of draw calls.
- Cull by hand when the layout is ordered: sort instances along the axis the camera travels, copy the visible run to the front of `instanceMatrix`, set `mesh.count` to its length, and skip the copy when the run hasn't changed. Set `frustumCulled = false` on the mesh.
- Per-instance data rides on instance attributes (an instance color can carry an index into a texture atlas).

## Text and Overlays

- Text inside the scene is drawn with Canvas 2D into a texture atlas (`CanvasTexture`, `SRGBColorSpace`, mipmaps and anisotropy 8), not MSDF. Canvas uploads cost a few milliseconds each, so keep them to two or three per scene and redraw only when the text changes.
- HUDs, reticles and labels that follow objects go on a second 2D canvas above the WebGL one, with its pixel ratio capped at 2 through `setTransform`.
- ASCII and pixel looks sample a small luminance grid of the media (around 96 × 54) and look glyphs up in an atlas drawn with `fillText`, with nearest filtering.

## Color

Author colors as sRGB hex in one palette, convert them to linear where they enter the scene, render linear HDR, and encode to sRGB once, in the final pass. Canvas 2D layers take the hex directly.

```ts
export const PALETTE = {
  background: '#000000',
  key: '#ffffff',
  tape: '#1a1a1a',
}
export const LINEAR = Object.fromEntries(Object.entries(PALETTE).map(([key, value]) => [key, hexToLinear(value)]))
```

Values above about 0.85 bloom, so place highlights deliberately.

## Determinism

Seed every random layout (Mulberry32) so the scene is the same on every load. The blurred poster keeps matching the first frame, and recordings are reproducible. Visuals never read `Math.random`, `Date.now` or `performance.now` directly. Jitter that has to change per frame is keyed to the frame index.

## Camera

A still camera looks dead. Let it breathe with slow sines on different axes (frequencies around 0.13, 0.09 and 0.07), add damped pointer parallax, roll it slightly through `camera.up`, and rack focus slowly. Scale all of it down under reduced motion.

## Shaders

- Hashes: Dave Hoskins's Hash without Sine, not `fract(sin(dot(...)) * 43758.5453)`, which breaks on some GPUs.
- Noise: simplex 2D and 3D, `fbm` and curl noise in a shared chunk. Pull in only what the shader calls.
- Includes are `#include ./chunks/name.glsl;` through `vite-plugin-glsl`.

## Tuning

- Every tunable value lives in one `settings` object in `utils/Settings.ts`, each with a one-line comment. Constants that never change at runtime sit above it in `SCREAMING_CASE`.
- `Controls.ts` exposes them in lil-gui, loaded with a dynamic `import()` only behind `#debug` or `?gui`, so the panel never ships in the main chunk. Group folders by concern (Motion, Materials, Lights, Lens, Post) and add read-only stats with `.disable().listen()`.
- When an effect is tuned in a panel, add a "Copy values" button that prints the values in the shape of the settings file, so they can be pasted back.
- Other URL flags for QA: `?quality=low`, `?capture`, `?fps` for stats.

## Recording

Pieces get posted as video, so build the capture into the project:

- `?capture` switches the page off the clock. Each call to `window.capture.frame()` advances the scene, the CSS animations, the playing videos and the sound by exactly one frame, however long it takes to draw, so a recording never drops or repeats a frame.
- A script (`scripts/record-video.ts`) drives the system Chrome with `puppeteer-core`, presses real keys on the frames they fall on, screenshots each frame at twice the size, and pipes them into ffmpeg, which scales down and adds the sound rendered offline in step:

```ts
const encoder = Bun.spawn(
  [
    'ffmpeg',
    '-v',
    'error',
    '-y',
    '-f',
    'image2pipe',
    '-framerate',
    `${FRAME_RATE}`,
    '-c:v',
    'mjpeg',
    '-i',
    '-',
    '-vf',
    `scale=${SIZE.width}:${SIZE.height}:flags=lanczos,format=yuv420p`,
    '-c:v',
    'libx264',
    '-preset',
    'slow',
    '-crf',
    '16',
    '-profile:v',
    'high',
    video,
  ],
  { stderr: 'inherit', stdin: 'pipe', stdout: 'ignore' },
)
```

- Long generative renders read frames with `readRenderTargetPixelsAsync` and send them over a WebSocket, with an acknowledgment per frame as backpressure, into ffmpeg's stdin.

## Media Preparation

Prepare media with scripts, not by hand:

- Images: ImageMagick with Lanczos, then `cwebp -m 6 -exact` (`-exact` keeps the color under transparent pixels, which shaders read). Masks go lossless. Ship two tiers (2048 and 1024) and pick 1024 when `min(screen.width, screen.height) * min(devicePixelRatio, 2) <= 900`.
- Videos and stills: ffmpeg into content-hashed WebP and MP4 files, marked immutable in `_headers` like `/assets/*`.

## Testing on a Phone

`getUserMedia` and `navigator.gpu` only exist in secure contexts, so a LAN address over plain HTTP reports them as unsupported. Run the dev server over HTTPS with `@vitejs/plugin-basic-ssl` and `server: { host: true }` when testing those on a real phone.
