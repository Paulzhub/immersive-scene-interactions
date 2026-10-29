# Accessibility, responsiveness, i18n and resilience

Covers the accessibility baseline, phone and tablet layouts, reduced motion, graceful fallbacks, empty/offline/error states, internationalisation with right-to-left support, and print styles. These rules apply to every effect in the skill.

## Contents
- Accessibility baseline
- Phone and tablet layouts
- Reduced motion
- Graceful fallbacks
- Empty, offline and error states
- Internationalisation and RTL
- Print stylesheet

## Accessibility baseline

- **Skip link:** the first focusable element, visible when focused.
  ```html
  <a class="skip" href="#main">Skip to content</a>
  ```
  ```css
  .skip { position: fixed; top: 8px; inset-inline-start: 8px; z-index: var(--z-skip); transform: translateY(-200%); }
  .skip:focus { transform: none; }
  ```
- **Visible focus:** `:focus-visible { outline: 2px solid var(--focus); outline-offset: 3px; }` (3px in the high-contrast theme). Never remove an outline without a replacement, and make sure the focus ring isn't hidden by `overflow: hidden` or glass layers.
- **Targets:** at least 44×44px for touch controls, including rail dots, hotspot markers and icon buttons (pad the hit area if the visible element is smaller).
- **Announcements:** one polite live region for status messages, and one `role="alert"` for errors.
  ```js
  const live = document.querySelector('#live');                 // <div id="live" class="visually-hidden" aria-live="polite"></div>
  export function announce(msg) { live.textContent = ''; requestAnimationFrame(() => { live.textContent = msg; }); }
  ```
  Announce scene changes, tab changes, quiz results, theme and language changes, and errors. Don't announce every caption line while narration is playing (`tour-player.md`).
- **Alt text:** meaningful images get a description. Decorative layers (canvas background, particles, grain, ghost numeral, curtain, cursor) get `aria-hidden="true"` or `alt=""`.
- **Full keyboard use:** everything is reachable and operable, the tab order is logical, Esc closes the top layer, and there are no keyboard traps outside modal dialogs. See the key map in `architecture.md`.
- **Headings and landmarks:** one `<h1>`, sections with headings, plus `<header>`, `<nav>`, `<main id="main">` and `<footer>`. The 3D canvas is a labelled `role="application"` region with keyboard help.
- **Reading order** matches the visual order. Don't use CSS `order` or absolute positioning that scrambles the sequence.
- **Language:** `<html lang>` matches the content, and mixed-language fragments carry their own `lang` attribute.
- **Visually hidden helper:**
  ```css
  .visually-hidden { position: absolute !important; width: 1px; height: 1px; margin: -1px; overflow: hidden; clip-path: inset(50%); white-space: nowrap; }
  ```

## Phone and tablet layouts

The layout adapts down to small phones (320px wide).

```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

```css
.app { min-height: 100dvh;
  padding: max(16px, env(safe-area-inset-top)) max(16px, env(safe-area-inset-right))
           max(16px, env(safe-area-inset-bottom)) max(16px, env(safe-area-inset-left)); }

.tabs { display: flex; gap: 8px; overflow-x: auto; scroll-snap-type: x proximity; scrollbar-width: none;
  overscroll-behavior-x: contain; mask-image: linear-gradient(90deg, transparent, #000 16px, #000 calc(100% - 16px), transparent); }
.tabs::-webkit-scrollbar { display: none; }
.tabs [role="tab"] { scroll-snap-align: center; flex: none; min-height: 44px; }

@media (max-width: 768px) {
  .split, .story { grid-template-columns: 1fr; }
  .rail { inset: auto 16px max(16px, env(safe-area-inset-bottom)) 16px; flex-direction: row; }
  .player-controls { flex-wrap: wrap; }
  .hotspot-card { position: fixed; inset: auto 0 0 0; border-radius: 16px 16px 0 0; }   /* bottom sheet */
}
@media (max-width: 360px) { :root { font-size: 15px; } .ghost { display: none; } }
@media (orientation: landscape) and (max-height: 500px) { .player-controls { position: absolute; bottom: 0; } }
```

```js
// keep the active tab visible in the scrollable row
tabs.querySelector('[aria-selected="true"]')?.scrollIntoView({ inline: 'center', block: 'nearest', behavior: motion.reduced ? 'auto' : 'smooth' });
```

- Use `100dvh` instead of `100vh` so mobile browser toolbars don't crop content.
- Respect user font scaling. Use `rem` units and test at 200% zoom: no clipped text and no horizontal page scroll (WCAG 1.4.10 reflow).
- Use pointer media queries (`hover`, `pointer`), not screen widths, to decide whether hover effects and the custom cursor appear.

## Reduced motion

The live `motion.reduced` value and the in-page toggle are defined in `architecture.md`. The motion tokens drop to 0ms (`theming.md`). With reduce motion on:

| Effect | Behaviour |
|---|---|
| Autoplay | Off. The tour starts paused. |
| Fades, slides, reveals, tab transitions | Instant |
| Curtain / View Transition | Instant swap (or a 150ms cross-fade at most) |
| Magnetic buttons, parallax, gyroscope tilt, lens breath, auto-rotate, spline fly-through | Off; the camera cuts to presets |
| Particles | Frozen static field |
| Cursor | No trailing lag; label still shown |
| Confetti, quiz shake/pulse | Off; static badge and text instead |
| Marquee, grain animation | Static |
| Smooth scrolling (Lenis, `behavior: smooth`) | Off |
| Scroll-driven and pinned sections, image sequence | Plain layout or a poster image |

Safety net for anything missed:

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: .01ms !important; animation-iteration-count: 1 !important;
    transition-duration: .01ms !important; scroll-behavior: auto !important; }
}
```

JavaScript that waits for `transitionend` or `animationend` must also work when durations are close to zero; those events still fire. Code that awaits WAAPI `.finished` checks `motion.reduced` first and skips the animation.

## Graceful fallbacks

| Missing capability | Detection | Fallback |
|---|---|---|
| WebGL | `hasWebGL()` (`three.md`), `webglcontextlost` | Hide the canvases, use a static CSS gradient background and a poster image per scene. The tour still plays captions and voice. Show the note "The 3D view isn't available on this device. Captions and narration still play." |
| Audio / Kokoro | `error` on the audio element, `play()` rejected | Captions on, transcript offered, toast with Retry (`audio.md`) |
| Fullscreen API | `fullscreenEnabled` or `requestFullscreen` rejects | Pseudo-fullscreen overlay (`tour-player.md`) |
| Document PiP | `!('documentPictureInPicture' in window)` | In-page mini player |
| Web Share | `!navigator.canShare?.(data)` | Download the image and copy the link; show a visible link if the clipboard fails |
| Storage | `save()` returns false | In-memory state, with one toast saying progress won't be saved |
| Leaderboard | Network error or 5xx | Inline message with Retry; the quiz still works offline |
| `backdrop-filter` | `@supports` | Opaque `--surface` |
| CSS scroll-driven animations | `CSS.supports('animation-timeline: view()')` | IntersectionObserver or ScrollTrigger |
| View Transitions | `!document.startViewTransition` | Curtain |
| JavaScript | — | Server-rendered content, chapters as a list, transcript visible, poster images |

Every failure message uses plain language, is announced through the live region, offers the next step (Retry or an alternative), and never blocks the rest of the page.

## Empty, offline and error states

One component for all of these, so they look and read the same way everywhere.

```html
<div class="state glass" role="status">
  <svg class="state-icon" aria-hidden="true">…</svg>
  <h3>You're offline</h3>
  <p>The tour and quiz still work. The leaderboard will update when you reconnect.</p>
  <button>Try again</button>
</div>
```

The pattern is: an icon, a short title, one sentence explaining the cause and what still works, and one primary action. Empty states ("No badges yet") suggest the next step ("Finish the Basics quiz to earn your first badge").

```js
addEventListener('offline', () => { document.documentElement.dataset.network = 'offline'; toast("You're offline. Some features are paused."); });
addEventListener('online',  () => { delete document.documentElement.dataset.network; toast('Back online'); retryPending(); });
```

## Internationalisation and RTL

- All UI strings, captions, transcripts, quiz text and alt text live in per-locale dictionaries (`/i18n/en.json`, `/i18n/hi.json`, …) loaded on demand. No text is hard-coded in components.
- Use `Intl.NumberFormat`, `Intl.DateTimeFormat`, `Intl.PluralRules` and `Intl.RelativeTimeFormat` for numbers, dates and plurals.
- When the language switches:
  ```js
  export async function setLang(lang) {
    const dict = await loadDict(lang);
    document.documentElement.lang = lang;
    document.documentElement.dir = ['ar', 'he', 'fa', 'ur'].includes(lang) ? 'rtl' : 'ltr';
    applyDict(dict);                                     // data-i18n="key" attributes
    rebuildTimelineKeepingPosition(lang);                // tour-player.md
    store.set({ lang });
    announce(dict['lang.changed']);
  }
  ```
- Use CSS logical properties everywhere (`margin-inline-start`, `inset-inline-end`, `padding-block`, `text-align: start`) so layouts mirror automatically in RTL. Mirror directional icons (arrows, chevrons) with `[dir="rtl"] .icon-dir { scale: -1 1; }`. Don't mirror media controls (play), clocks, numerals or the 3D scene.
- Swipe direction and the marquee reverse in RTL (`navigation.md`, `motion.md`).
- Fonts: include scripts for every supported language (for example Noto Sans Devanagari for Hindi), and subset each with `unicode-range`. Line height and text effects (scramble, word split) must handle complex scripts. Disable the scramble for non-Latin text.
- If a language has no narration or captions yet, fall back to English with a notice ("Narration is in English for this section").
- The language choice is remembered and deep-linkable (`?lang=hi`).

## Print stylesheet

```css
@media print {
  canvas, .cursor-dot, .cursor-label, .grain, .curtain, .ghost, .rail, .player-controls, .toasts, .marquee, .coach { display: none !important; }
  :root { --bg: #fff; --text: #000; --glass: #fff; --glass-border: #999; }
  .glass { backdrop-filter: none !important; box-shadow: none; border: 1px solid #999; }
  .panel, .w, .wi { clip-path: none !important; transform: none !important; opacity: 1 !important; }
  [hidden].printable, .transcript, .quiz-results { display: block !important; }
  a[href^="http"]::after { content: " (" attr(href) ")"; font-size: .85em; }
  h2, h3 { break-after: avoid; } figure, .question, .state { break-inside: avoid; }
}
```

The transcript, chapter list and quiz results print as clean, readable documents. Add a "Print transcript" action to the transcript panel and the command palette.
