# Code Style

These conventions apply to any code written in this style. A project's own `CLAUDE.md` or `AGENTS.md` wins where it disagrees.

## Naming

Use descriptive, unabbreviated names: `index` not `idx`, `context` not `ctx`, `element` not `el`, `event` not `e`, `callback` not `cb`, `error` not `err`, `parameters` not `params`, `options` not `opts`, `reference` not `ref`, `source` not `src`, `temporary` not `tmp`, `configuration` not `cfg`, `message` not `msg`. Code is read far more often than it is typed, and full words cost an agent nothing to write.

- No single-letter parameters, arrow functions included: `(first, second) => first - second`, `items.map((item) => …)`.
- Booleans in state pairs read as questions: `[isCollapsed, setIsCollapsed]`.
- Lerped values are `{ current, target }` objects (plus `last` when direction matters).
- `elements` keys are the camelCased BEM path: `{ heroMedia: '.hero__media', menuLinks: '.menu__link' }`.
- Constants that never change at runtime are `SCREAMING_CASE` at the top of the module, each with a one-line comment.
- File names mirror their primary export (`Resolution.ts` exports `Resolution`). After renaming a file on macOS, check the import casing: Linux CI will fail on it.

## Control Flow

Write every `if` as a multiline block with braces. No one-line or brace-less `if`.

```ts
if (condition) {
  performAction()
}
```

## Ordering

In most cases, sort object properties, destructured names, imports, JSX props and HTML attributes alphabetically. Stable ordering makes diffs smaller and makes it obvious where to add the next key.

```tsx
const { height, left, top, width } = element.getBoundingClientRect()

<img alt={alt} className={styles.media} height={height} src={src} width={width} />
```

## Imports

Group imports with a blank line between groups, and sort alphabetically inside each group:

1. Side-effect imports such as styles (`import '../styles/index.scss'`).
2. Packages.
3. Local imports, split into groups by kind: shaders, then utilities, then sibling classes or components.

When a module gives both values and types, import them in one statement with inline `type` instead of a second `import type` line. Use `import type` only when everything imported is a type.

```ts
import AutoBind from 'auto-bind'

import fragment from '../shaders/media-fragment.glsl'
import vertex from '../shaders/media-vertex.glsl'

import { createTarget, drawFullscreen, type Target } from '../utils/GL'
import { lookAt, perspective, type Vector3 } from '../utils/Math'

import { Fluid } from './Fluid'
import { Pointer } from './Pointer'
```

Biome's import sorting keeps these groups, because it treats blank lines as boundaries between groups.

## Classes

Runtime code is classes, not factory functions returning objects of closures.

- Call `AutoBind(this)` first in the constructor, so methods can be passed as listeners with no `.bind(this)`. Lisergia components inherit it from their base class.
- Name methods by role: `create*` for setup, `destroy*` for its teardown, `addEventListeners()` for wiring, `on*` for event handlers (`onResize`, `onPointerDown`), `update(...)` for per-frame work called by an owner, and `onLoop` for the class that owns `requestAnimationFrame`. Older Lisergia code calls it `onRAF`; keep whichever the project uses.
- Prefix booleans with `is` or `has` (`isEnabled`, `isGhost`, `hasTarget`).
- Short classes list their fields at the top, alphabetically. A long class with several concerns (a canvas with camera, scene, post and pointer) groups them with banner comments instead, and declares each field right above the `create*` method that sets it, so a concern reads top to bottom in one place:

```ts
//
// Camera.
//
declare camera: PerspectiveCamera

createCamera() {
  this.camera = new PerspectiveCamera(35, this.aspect, 1, 500)
}
```

- Async setup exposes a `ready` promise. Disposers returned by observers or reactions are stored and called in `destroy*`.
- Name classes, even default exports: `export default class Reveal extends Component`, not an anonymous `class extends`.
- In hot paths, never allocate: cache math temporaries (vectors, matrices, quaternions) as underscore-prefixed fields with descriptive names (`_targetPosition`, not `_temp`).
- Pure helpers (matrices, WebGL target creation, shape sampling) live as functions in `utils/`. Tunables live in `utils/Settings.ts` (see `rendering.md`); older projects called it `Constants.ts`.

## Comments

Comments earn their place: a reason, a pitfall or an invariant, in full sentences. Put one above every class saying what it is for, and one above any number that isn't obvious. Don't narrate what the code already says.

```ts
// The slowest frames of each window are left out, so a single hitch (a texture upload, a tab switch) doesn't count.
const OUTLIERS = 2
```

## Formatting

Biome formats everything, so follow the project's config rather than habit. Luis's own projects use single quotes, no semicolons and 120 columns. Trailing commas are `all` in Lisergia and `none` in some newer standalone projects. Client Next.js projects have used double quotes, semicolons and 100 columns.

## Shaders

- One `.glsl` file per program stage, named `<pass>-vertex.glsl` / `<pass>-fragment.glsl`, imported as strings through `vite-plugin-glsl`. Group a subsystem's shaders in a folder (`shaders/fluid/`).
- Put shared code (hashes, noise, the fullscreen `in`/`out` header) in `shaders/chunks/` and pull it in with `#include ./chunks/hash.glsl`. Keep `#version 300 es` on the first line of each top-level shader, not inside a chunk.
- A value the CPU sets is a uniform. Don't interpolate JavaScript values into GLSL template strings.
- Delete chunks nobody calls. An unused noise function is still code the next reader has to understand.

## React Components (when on React)

- One folder per component: `ComponentName/index.tsx` and `ComponentName/styles.module.scss`.
- Build `className` with `classNames()` from the `classnames` package. Never use template-literal concatenation or `.trim()`. Write conditional classes as `condition && styles.name`. Optional `className` props have no default value. (Preact templates in Lisergia render on the server only and don't ship `classnames`; `[...].filter(Boolean).join(' ')` is fine there.)

```tsx
<header className={classNames(styles.element, minimal && styles.minimal, className)}>
```

- Prefer MobX over hooks-based state for anything shared (see `react.md`).

## SCSS

**Applications (React + CSS modules):**

- The first rule of every `.scss` file is `.element`, which styles the component's root.
- Other class names are a single lowercase word: `.title`, `.media`, `.minimal`. Avoid camelCase class names.
- Sizes in `px` only. Don't mix in `rem`.

**Sites (Lisergia):**

- Use BEM (`block__element--modifier`) with one SCSS file per section.
- Sizes in `rem` on the fluid 10-design-pixel scale (see `css.md`).

**Both:**

- Media queries nest inside the rule they change, never in a block at the end of the file (see `css.md`).
- Declarations are sorted alphabetically, enforced by Stylelint.

## Commits

Commit messages are written in Luis's voice, with no AI co-author or "generated with" lines.

- One change: a short imperative sentence ending in a period. `Fix hashchange and popstate conflict.`
- Several changes: one line of dash-separated sentences, each ending in a period. `- Remove MobX. - Add onScroll and onResize hooks to Component. - Fix active links on Navigation.`
- Name a new project's first commit `Initial commit.` When a prototype is cleaned up before it goes public, squash its history into that one commit.
- In a client's repository, follow the client's conventions instead (see `workflow.md`).
