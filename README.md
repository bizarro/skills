# Skills

[Luis Bizarro](https://bizar.ro/)'s brain dump on creative development, packaged as agent skills. Install it and your coding agent builds interactive, animated websites the way I would, and answers "how do I get into creative development?" the way I would.

> "Prioritizing animations, motion and interactions in a website shouldn't be controversial. Not adding interesting things to your web pages because of metrics will always be a downgrade."

## What's inside

| Skill | What it does |
| --- | --- |
| [`creative-development`](skills/creative-development/SKILL.md) | Opinions and conventions for creative development: [Lisergia](https://github.com/bizarro/lisergia), Preact vs React, three.js vs OGL vs OGPU vs React Three Fiber, WebGL loading, motion, CSS scaling, and a learning path for beginners. |

## Install

### Claude Code

```sh
claude plugin marketplace add bizarro/skills
claude plugin install bizarro@bizarro
```

Or from inside a session:

```
/plugin marketplace add bizarro/skills
/plugin install bizarro@bizarro
```

### Any agent (Codex, Cursor, Windsurf, Cline, Gemini CLI, Copilot, …)

Uses the open [skills](https://skills.sh) installer:

```sh
npx skills add bizarro/skills
```

Add `-g` to install for your user instead of the current project, or `-a <agent>` to target a specific agent (for example `-a codex`).

### Manually

Copy the skill folder into your agent's skills directory:

```sh
git clone https://github.com/bizarro/skills.git
cp -r skills/skills/creative-development ~/.claude/skills/
```

## Use

The skill loads on its own when you work on landing pages, WebGL scenes, page transitions, scroll or reveal animations, or ask where to start in creative development. You can also call it directly:

- Claude Code plugin: `/bizarro:creative-development`
- Manual install: `/creative-development`

Example prompts:

- "I'm a frontend developer and want to get into creative development. Where do I start?"
- "Client wants an animated landing page for a product launch. What stack should I use?"
- "Should I use React Three Fiber for this hero scene?"
- "Add a WebGL image distortion to the gallery without flicker on load."

## Structure

```
.claude-plugin/          Claude Code plugin and marketplace manifests
skills/
└── creative-development/
    ├── SKILL.md         Core stance and routing
    └── references/      One file per topic, loaded only when needed
```

## License

MIT
