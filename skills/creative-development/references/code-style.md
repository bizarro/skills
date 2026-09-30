# Code style

These conventions apply to any code written in this style. A project's own `CLAUDE.md` or `AGENTS.md` wins where it disagrees.

## Naming

Use descriptive, unabbreviated names: `index` not `idx`, `context` not `ctx`, `element` not `el`, `event` not `e`, `callback` not `cb`. Code is read far more often than it is typed, and full words cost an agent nothing to write.

## Control flow

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

## React components (when on React)

- One folder per component: `ComponentName/index.tsx` and `ComponentName/styles.module.scss`.
- Build `className` with `classNames()` from the `classnames` package. Never use template-literal concatenation or `.trim()`. Write conditional classes as `condition && styles.name`. Optional `className` props have no default value.

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
