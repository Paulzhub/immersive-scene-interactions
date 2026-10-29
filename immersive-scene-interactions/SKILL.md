---
name: immersive-scene-interactions
description: Build cinematic, award-style scene-based websites and interactive 3D guides — theme tokens, glass UI, magnetic buttons, text reveals, parallax, curtain/View Transitions, custom cursors, three.js particles and post-processing, a 360° tour player with captions and Kokoro voiceover, quizzes, plus accessibility, reduced motion, performance and fallbacks. Use when the user asks for any of these effects, "award-style" motion, a multi-scene landing page, or an interactive tour/guide.
---

# Immersive Scene Interactions

A system of effects and rules for sites made of numbered **scenes** (panels), usually with a three.js canvas, a tour player and a quiz. The code is vanilla JS and CSS, with GSAP where it helps; it adapts to React, Vue or Svelte by keeping the same store, events and ownership rules.

## Step 0: decide scope before writing code

If the request doesn't already make these clear, ask, or state the default you're using:

1. **Tier.** Which tier(s) are wanted (table below)? Don't add every effect when only one was asked for.
2. **Navigation model.** Step (default for tours), scroll (long pages) or hybrid. See `references/architecture.md`.
3. **Stack.** Framework, and whether three.js, GSAP or a build step are allowed.

| Tier | What it includes |
|---|---|
| **Core** (always) | Tokens and themes, glass, text reveal, scene transition, tab transitions, accessibility baseline, reduced motion, fallbacks, responsive layout |
| **Polish** | Magnetic buttons, parallax, ghost numeral, cursors, selection colour, grain, preloader, progress rail, counters, scramble, line draw, marquee, variable fonts |
| **3D** | Particle background, lens breath, post-processing, hover distortion, hotspots, fly-through, auto-rotate, LOD loading, exploded view, lighting, snapshot, measuring, AR, gyroscope |
| **Tour player** | Timeline clock, WebVTT captions/chapters, controls, orbit/zoom, speed, fullscreen, mini player/PiP, Media Session, Wake Lock, caption options, transcript, voiceover, sound design |
| **Learning** | Quiz, feedback, explanations, spaced repetition, streaks/badges, confetti, haptics, share card, resume, secure leaderboard |
| **Platform** | Shortcuts overlay, gestures, deep links, command palette, comparison slider, toasts, coach marks, glossary, i18n/RTL, print, PWA, analytics, adaptive quality |

## Non-negotiable rules

1. **Tokens only.** Colours, durations, easings and z-index come from CSS variables (`references/theming.md`). Every text pair must reach 4.5:1 in every theme.
2. **One source of truth.** A single `store` and `bus`. Scene changes go only through `goToScene(n)`. In tours, the timeline clock drives everything (`references/architecture.md`).
3. **One owner per transform.** Nest wrappers when two effects need to move the same thing.
4. **Live preferences.** Read `motion.reduced` and `pointer.fine` when an effect runs, not at startup. Pointer effects are for fine pointers only.
5. **Everything is reachable without the effect.** Keyboard, screen reader, touch, no WebGL, no audio and no JavaScript all have a working path. The cursor, canvas and decorative layers are `aria-hidden`.
6. **Respect the keyboard map and the Esc layer stack.** Components that use arrow keys win over global shortcuts. Single-key shortcuts can be turned off.
7. **Budgets.** Initial JS of 200 KB or less, with 3D, Kokoro, AR, quiz and palette loaded lazily. Adaptive quality protects frame rate (`references/performance.md`).
8. **Clean up.** Every effect's `init()` returns a `destroy()` that removes listeners, stops loops and disposes GPU resources. Pause loops when the tab is hidden.
9. **Pin exact versions** (see `references/performance.md`). Never use `latest` or `^` ranges.

## Reference index

Read only the files the chosen tiers need. Read `architecture.md` first whenever more than two or three effects are combined.

| File | Read when building… |
|---|---|
| `references/architecture.md` | Anything multi-effect: store/bus, live preferences, navigation model, `goToScene`, transform ownership, keyboard map, Esc layers |
| `references/theming.md` | Colour/motion/z-index tokens, theme switcher (system/dark/light/high contrast), forced colours, glass, grain, selection colour |
| `references/motion.md` | Magnetic buttons, staggered text reveal, View Transitions/curtain, ghost numeral, tab transitions, preloader, progress rail, scramble, counters, card flip, variable fonts, marquee |
| `references/scroll.md` | Parallax, CSS scroll-driven animations, Lenis, SVG line draw, pinned horizontal sections, scrollytelling, image sequences |
| `references/cursor.md` | Contextual/sticky cursor, inverted blend-mode cursor, spotlight/mask reveal |
| `references/three.md` | three.js setup, ember particles, lens breath, post-processing, hover distortion, hotspots, spline camera, auto-rotate, LOD, exploded view, lighting, snapshot, measuring, AR, gyroscope |
| `references/tour-player.md` | Timeline clock, WebVTT captions/chapters, controls, orbit/zoom, speed, fullscreen, mini player + Document PiP, Media Session, Wake Lock, caption options, transcript |
| `references/audio.md` | Audio unlock, Kokoro voiceover (build-time EN/HI pipeline, optional in-browser), sync with timeline, sound design, spatial audio, mutes |
| `references/navigation.md` | Shortcuts overlay, swipe/wheel-snap, deep links, command palette, comparison slider, toasts, coach marks, glossary tooltips |
| `references/quiz.md` | Quiz markup, feedback, explanations, accessible timer, spaced repetition, streaks/badges, confetti, haptics, share card, resume, leaderboard security |
| `references/accessibility.md` | Accessibility baseline, phone/tablet layouts, reduced-motion table, fallbacks table, empty/offline/error states, i18n/RTL, print |
| `references/performance.md` | Budgets, versions, adaptive quality, lazy loading, PWA, privacy-safe analytics, Playwright + axe test recipe |

## Build order

1. Tokens, theme resolved before paint, glass (`theming.md`).
2. Store, bus, preferences, navigation model, `goToScene`, keyboard map (`architecture.md`).
3. Semantic HTML that works with no JavaScript: content, chapters, transcript, posters.
4. Core tier: transitions, reveals, tabs, the live region, reduced motion, fallbacks.
5. The chosen feature tiers, each as an `init()`/`destroy()` module, with heavy ones lazy-loaded.
6. Adaptive quality, then measure against the budgets.
7. Test with the recipe in `performance.md`: axe, a keyboard-only pass, rapid scene switching, no WebGL, reduced motion, phone width, light/dark/high contrast, audio blocked, offline.

## Definition of done

- Every effect asked for works, and nothing outside the chosen tiers was added.
- No horizontal scroll at 320px, no clipped text at 200% zoom, and every target is at least 44px.
- The axe scan is clean, the keyboard and screen-reader paths are complete, and contrast passes in every theme.
- Reduced motion, no WebGL, blocked audio, blocked storage and offline each give a working experience with clear messages.
- The budgets are met, and frame rate stays steady through scene changes on a mid-range phone.
