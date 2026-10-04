# Working with Agents on a Project

How Luis sets up a repository so coding agents work in his style, and how they should verify and hand work back.

## Agent Files

- The rules live in `AGENTS.md`. `CLAUDE.md` is a symlink to it (`ln -s AGENTS.md CLAUDE.md`) or a one-line `@AGENTS.md`, so every agent reads the same file.
- `AGENTS.md` holds the commands, the conventions, the architecture decisions and their reasons ("No transmission: it re-renders every opaque object"), and the performance budget. Keep it short enough to read every session.
- Areas with their own pitfalls get a nested `AGENTS.md` (or `CLAUDE.md` pointing at it) in their folder: a shader directory, an engine package, a sections folder.
- Point at a canonical example instead of describing a pattern twice: "Follow `templates/sections/Hero.tsx` and `datasets/sections/Hero.ts`."
- A complex component (a lens, a cloth simulation, a cursor) gets a `README.md` in its folder explaining how it works.
- Generative and art pieces get a `docs/TREATMENT.md` (idea, tone, palette, camera, how sound and pointer map to the picture, references, and a **"Not this"** list) and a `docs/ENGINE.md` (how to run it, the file map, the rules, the budget).

## Verifying

- Verify with the project's own checks: type-check, lint (scripts and styles), tests and build. Write the exact command list into `AGENTS.md`, for example `bun run check && bun run typecheck && bun run test && bun run build`.
- **Don't open a browser to look at the result unless asked.** Luis checks visuals, motion and sound himself, on desktop and on a real phone. Screenshots and Lighthouse runs spend credits and time on judgments he makes better.
- **Don't drive the page with synthetic pointer or mouse events** to "check" an effect.
- After building an effect, describe the knobs worth tuning (which values in `Settings.ts`, which flag opens the panel) and wait for his feedback instead of iterating against screenshots.
- Exceptions, when the project sets them up: offline art pieces matched to a reference image check stills or contact sheets from a render script, and performance probes (a script that logs `requestAnimationFrame` gaps over 50 ms while pressing keys) run when a stall is being hunted.

## Tests

Test the math and the state, never the visuals.

- `bun test`, with the test next to the code (`sliderMath.ts` and `sliderMath.test.ts`): easing and damping at 60 and 120 Hz, wrap logic, physics steps, store behavior.
- Keep pure logic in `utils/` or `lib/` so it can be tested without a DOM or a GPU.
- Shaders can be checked headless: compile them in a test (Deno's WebGPU, or `naga` for WGSL), and test the binding contract between a pipeline and its shader. An engine with a golden image compares with a small tolerance (3/255 per channel, under 0.2% of pixels differing) and regenerates with an environment flag.

## Building Effects

- **Lab first.** Build a hard effect alone, in a lab route or a throwaway project, until the look is approved, then port it into the site. Note where it came from in a comment ("Ported from the OGL lab to OGPU/WGSL").
- **Credit sources.** Comments name the site, person or demo behind an effect ("after jesperlandberg.com's ribbon"). Licensed assets get a line in `CREDITS.md`.
- **Tuning panels behind a flag.** See `rendering.md`.

## Content and Design

- **No invented content.** No placeholder copy, made-up sections, guessed tokens or fixture fallbacks. If something isn't defined in the design or the CMS, leave it out and say so.
- **Design values come from the Figma file.** When the Figma MCP returns asset URLs, download each one into `public/images/<section>/` right away: the URLs expire.
- **Section checklist.** Write down the steps to add a section in `AGENTS.md` or the README (schema, template, styles, behavior, registration), so an agent never skips one.

## Several Agents at Once

When agents work in parallel, give each one an area of the repository and keep shared files (the section registry, the page-builder array, global styles, the stores) with whoever integrates. An agent that finds a problem in another agent's files reports it instead of fixing it.

## Git

- In his own repositories, Luis commits straight to `main` with the message style in `code-style.md`.
- In client repositories, follow the client's flow: feature branches named `luis/<topic>`, merge requests, conventional commits, and the language the team writes in.
