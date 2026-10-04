# React, Preact and State

## The Client Decides, Not the Framework

Choose the rendering stack by **who the client is and what they want**, not by what looks best on a résumé.

- **Bespoke, animated, interactive landing page** (the usual creative job): use Lisergia, meaning Preact renders the TSX views to static HTML and does **not** hydrate. A small TypeScript runtime then enhances the page with transitions, reveals, smooth scroll and WebGL. See `lisergia.md`.
  - Why: this gives full control over rendering. No reconciler rewrites DOM nodes while they animate, there is no hydration pass or hydration mismatch to cause a flash on load, and the client bundle holds only the animation runtime. The DOM belongs to the animation code.
- **Enterprise client with its own stack and the budget for it**: adopt what they want (Next.js, React, their design system, their CMS). The deciding factor is **the least friction with the client's departments**: their engineering, IT, security review and marketing operations. A stack their team can keep maintaining beats a stack that is technically cleaner. Bring the craft (motion, WebGL, first-frame polish) into their stack instead of fighting it.
- **Applications** (editors, tools, dashboards, anything with heavy UI state): React is fine and expected.

Don't push the user toward a different stack than the one the client situation calls for. If the user is on the client's React stack, work inside it.

## Inside a Client's Next.js

When the client's stack is Next.js, keep the craft and fight the framework as little as possible:

- Server components by default. Put `"use client"` on the smallest child that needs it.
- Plain `<img>` and `<video>`, not `next/image`, so the media stays under the animation code's control.
- WebGL lives in plain classes (`renderer.ts`) that a small client component imports dynamically. WebGPU code never loads on the server or in the main bundle, and the static image underneath stays as the fallback.
- Write the page transition yourself instead of adding a transition router: a fade out, `router.push`, and a fade in when `usePathname` changes. Keeping it in one file also lets it stop and restart Lenis.
- Render a safe default on the server and upgrade after mount (see `performance.md`).
- Next.js 16 changed enough that agents should read the guide in `node_modules/next/dist/docs/` before writing framework code. Say so in the project's `AGENTS.md`.

## Applications

Editors, tools and anything with heavy UI state are React apps, built like this:

- **MobX for all app state**, not only what the scene shares. Stores are classes with an exported singleton (`export const tabsStore = new TabsStore()`). A component takes its store as a prop with the singleton as default (`ToastStack({ store = notificationsStore })`), which keeps it testable.
- **Components** are named exports wrapped in `observer`: `export const Panel = observer(function Panel(...) { … })`. No default exports.
- **Hooks** are `use-kebab-case.ts` files (shortcuts, gestures). Pure logic sits in `lib/`, tests in `tests/`.
- **The engine is a separate package** (`packages/core`), plain classes with no React, that the UI drives. One explicit file per type rather than generic tables. Every `autorun` and `reaction` it creates is disposed in its `dispose()`.
- **UI in px**: type at 12, 11 and 10px, Lucide icons at `size={14} strokeWidth={1.75}`, 36px panel headers, 100–150ms transitions, panels that fold to their header.
- **Theme** from two base colors plus a `color-mix(in oklab, var(--color-foreground) N%, transparent)` ramp for every tint, switched with `data-theme` and following `prefers-color-scheme`.
- **Desktop apps** use electron-vite and electron-builder. The desktop app depends on the web app package and runs the same bundle. A typed `window.desktop` bridge is declared in `src/shared/desktop-api.ts`, IPC handlers are split by domain in `main/ipc/`, and the web build checks for the bridge (`window.desktop?.…`) so it runs without it. Releases build from a GitHub workflow on `v<version>` tags.

## React plus WebGL: Use MobX

When a project has **both** React UI and a WebGL/WebGPU scene, use **MobX** for the shared state. This is close to a requirement.

Why: the render loop is not React. It runs every frame, outside the component tree. MobX lets both sides share one observable store:

- React components wrap in `observer()` and re-render only when the values they read change.
- The scene reads the same store directly in its render loop, or subscribes with `reaction()`/`autorun()` for event-style updates (for example, rebuilding a geometry when a setting changes).
- No prop drilling into the canvas, no context re-render cascades, and no mirroring state into refs by hand.

```ts
import { makeAutoObservable, reaction } from 'mobx'

class SceneStore {
  color = '#ff3300'
  hovered: string | null = null

  constructor() {
    makeAutoObservable(this)
  }

  setColor(color: string) {
    this.color = color
  }
}

export const sceneStore = new SceneStore()

// WebGL side: react to changes without React.
reaction(
  () => sceneStore.color,
  (color) => {
    material.color.set(color)
  },
)
```

**Pure sites with no React don't need MobX.** Lisergia removed it only to keep the bundle as small as possible. Don't add it to a vanilla Lisergia site.

## React Three Fiber

R3F is the last resort. See `webgl.md`. It is justified only in a React application where a lot of UI state drives the scene interactively. Even then, keep heavy per-frame work out of React state: mutate refs in `useFrame` and keep shared app state in MobX.

Luis has tried it for a marketing-site scene and for a quick webcam experiment, and went back to plain Three.js classes for the sites that followed. A throwaway sketch is the one place its convenience wins.
