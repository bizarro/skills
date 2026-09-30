# React, Preact and state

## The client decides, not the framework

Choose the rendering stack by **who the client is and what they want**, not by what looks best on a résumé.

- **Bespoke, animated, interactive landing page** (the usual creative job): use Lisergia, meaning Preact renders the TSX views to static HTML and does **not** hydrate. A small TypeScript runtime then enhances the page with transitions, reveals, smooth scroll and WebGL. See `lisergia.md`.
  - Why: this gives full control over rendering. No reconciler rewrites DOM nodes while they animate, there is no hydration pass or hydration mismatch to cause a flash on load, and the client bundle holds only the animation runtime. The DOM belongs to the animation code.
- **Enterprise client with its own stack and the budget for it**: adopt what they want (Next.js, React, their design system, their CMS). The deciding factor is **the least friction with the client's departments**: their engineering, IT, security review and marketing operations. A stack their team can keep maintaining beats a stack that is technically cleaner. Bring the craft (motion, WebGL, first-frame polish) into their stack instead of fighting it.
- **Applications** (editors, tools, dashboards, anything with heavy UI state): React is fine and expected.

Don't push the user toward a different stack than the one the client situation calls for. If the user is on the client's React stack, work inside it.

## React plus WebGL: use MobX

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
