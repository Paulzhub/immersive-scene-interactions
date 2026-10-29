# Immersive Scene Interactions — an Agent Skill

A skill for Claude (Claude Code, the Claude apps, and other agents that read `SKILL.md`)
for building cinematic, award-style websites made of numbered **scenes**, and interactive
3D guides and tours.

Ask your agent for a multi-scene landing page, a 360° product tour or "award-style" motion,
and the skill gives it a tested set of patterns and rules for the job: tokens, accessibility,
reduced motion, performance budgets and fallbacks are built in from the first line,
not added at the end.

## What it covers

Effects are grouped into tiers, so the agent adds only what you asked for:

| Tier | Includes |
|---|---|
| **Core** (always) | Theme tokens, glass UI, text reveals, scene and tab transitions, accessibility baseline, reduced motion, fallbacks, responsive layout |
| **Polish** | Magnetic buttons, parallax, ghost numerals, custom cursors, grain, preloader, progress rail, counters, text scramble, line draw, marquee, variable fonts |
| **3D** | three.js particle backgrounds, post-processing, hover distortion, hotspots, camera fly-through, auto-rotate, LOD loading, exploded view, snapshot, measuring, AR, gyroscope |
| **Tour player** | Timeline clock, WebVTT captions and chapters, orbit/zoom, speed, fullscreen, mini player / Picture-in-Picture, Media Session, Wake Lock, transcript, Kokoro voiceover, sound design |
| **Learning** | Quizzes, feedback and explanations, spaced repetition, streaks and badges, confetti, haptics, share cards, resume, leaderboard security |
| **Platform** | Shortcuts overlay, gestures, deep links, command palette, comparison slider, toasts, coach marks, glossary, i18n/RTL, print, PWA, analytics, adaptive quality |

The code it writes is vanilla JS and CSS (with GSAP where it helps) and adapts to React,
Vue or Svelte.

## What's inside

```
immersive-scene-interactions/
├── SKILL.md              ← entry point: scope questions, rules, build order, definition of done
└── references/           ← loaded only when a tier needs them
    ├── architecture.md   store/bus, navigation models, goToScene, keyboard map, Esc layers
    ├── theming.md        colour/motion/z-index tokens, theme switcher, high contrast, glass
    ├── motion.md         reveals, View Transitions, magnetic buttons, preloader, counters…
    ├── scroll.md         parallax, scroll-driven animations, Lenis, pinned sections
    ├── cursor.md         contextual, blend-mode and spotlight cursors
    ├── three.md          three.js scenes, particles, post-processing, hotspots, AR
    ├── tour-player.md    timeline clock, captions, controls, PiP, Media Session
    ├── audio.md          audio unlock, Kokoro voiceover, sync, sound design
    ├── navigation.md     shortcuts, gestures, deep links, command palette, toasts
    ├── quiz.md           quiz markup, feedback, spaced repetition, badges
    ├── accessibility.md  baseline, layouts, reduced-motion and fallback tables, i18n, print
    └── performance.md    budgets, pinned versions, lazy loading, PWA, Playwright + axe tests
```

The skill is plain Markdown. It contains no scripts and runs nothing on your machine.

## Install

The skill lives in the `immersive-scene-interactions/` folder of this repo. Every method
below puts that **folder** (with `SKILL.md` directly inside it) where your agent looks for skills.

### Claude Code — with the Skills CLI (quickest)

```bash
npx skills add Paulzhub/immersive-screen-interactions
```

Add `-g` to install it for every project rather than just the current one.

### Claude Code — by hand

Clone the repo and copy the skill folder into your personal skills folder
(available in every project):

macOS / Linux:

```bash
git clone https://github.com/Paulzhub/immersive-screen-interactions.git
mkdir -p ~/.claude/skills
cp -r immersive-screen-interactions/immersive-scene-interactions ~/.claude/skills/
```

Windows (PowerShell):

```powershell
git clone https://github.com/Paulzhub/immersive-screen-interactions.git
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse immersive-screen-interactions\immersive-scene-interactions "$env:USERPROFILE\.claude\skills\"
```

To share it with a team through one project instead, copy the folder to
`.claude/skills/immersive-scene-interactions/` inside that project and commit it.

No git? Use **Code → Download ZIP** on this page, unzip it, and copy the inner
`immersive-scene-interactions` folder to the same place.

Start a new Claude Code session afterwards. You should see the skill when you type
`/immersive-scene-interactions`.

### Claude.ai and the Claude desktop app

1. Download this repo (**Code → Download ZIP**) and unzip it.
2. Zip **only** the inner `immersive-scene-interactions` folder, so the zip contains
   `immersive-scene-interactions/SKILL.md`.
3. In Claude, open **Settings → Capabilities → Skills**, choose **Upload skill**, and select that zip.

### Other agents

Any agent that supports the open `SKILL.md` format can use the folder as it is; put it in
that agent's skills directory.

## Use

You don't need to call it by name; Claude loads it when a request matches, for example:

- "Build a five-scene landing page for our product with award-style motion."
- "Make an interactive 3D tour of this model with captions and a voiceover."
- "Add magnetic buttons and a staggered text reveal to this page."
- "Turn this guide into scenes with a quiz at the end."

Or call it directly in Claude Code: `/immersive-scene-interactions build a 360° tour of …`

Before writing code, the agent settles three things (or tells you the defaults it chose):
which **tiers** you want, the **navigation model** (step, scroll or hybrid), and your
**stack** (framework, and whether three.js, GSAP or a build step are allowed).

## Update

- Skills CLI: `npx skills update`
- By hand: `git pull` in your clone and copy the folder again.

## Contributing

Issues and pull requests are welcome. Keep `SKILL.md` short. Put detail in a
`references/` file and add it to the reference index in `SKILL.md`.
