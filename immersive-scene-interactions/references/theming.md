# Theming, tokens and surfaces

Covers colour, motion and z-index tokens; the theme switcher (system, dark, light, high contrast); forced colours; frosted glass; film grain; and the text selection colour.

## Contents
- Rules
- Colour tokens (dark, light, high contrast, forced colours)
- Motion tokens
- Z-index tokens
- Reading tokens from JS
- Theme switcher (no flash; follows the OS while set to System)
- Frosted-glass panels
- Film grain overlay
- Text selection colour

## Rules
- Components use tokens only: no raw hex values, durations, easings or z-index numbers outside this file.
- Every text/background pair must reach at least 4.5:1 (3:1 for large text and UI borders) in every theme. Check new pairs before adding them.
- The accent colour is for large text, icons, focus rings and filled buttons. Body-size links in the light theme use `--accent-text`.

## Colour tokens

```css
:root, :root[data-theme="dark"] {
  color-scheme: dark;
  --bg: #0b0b0d;        --surface: #121216;   --text: #ecebe8;   --muted: #9a9a9f;
  --accent: #ff8a5c;    --accent-text: #ff8a5c; --on-accent: #111;
  --good: #4ade80;      --bad: #f87171;
  --code-bg: #16161a;   --code-text: #d6e2ff;
  --glass: rgba(20,20,24,.55); --glass-border: rgba(255,255,255,.08);
  --focus: #ff8a5c;
  --blush: #f6d9d6;     --on-blush: #111;
  --ember: #ff8a5c;     --grain-opacity: .06;
}
:root[data-theme="light"] {
  color-scheme: light;
  --bg: #f7f5f1;        --surface: #ffffff;   --text: #17171a;   --muted: #5d5d63;
  --accent: #b93d0c;    --accent-text: #b93d0c; --on-accent: #fff;   /* 5.16:1 on --bg, white on accent 5.62:1 */
  --good: #166534;      --bad: #b91c1c;
  --code-bg: #ece9e2;   --code-text: #1f2a44;
  --glass: rgba(255,255,255,.6); --glass-border: rgba(0,0,0,.08);
  --focus: #b93d0c;
  --blush: #f6d9d6;     --on-blush: #111;
  --ember: #c2410c;     --grain-opacity: .04;
}
:root[data-theme="contrast"] {
  color-scheme: dark;
  --bg: #000;           --surface: #000;      --text: #fff;      --muted: #fff;
  --accent: #ffd166;    --accent-text: #ffd166; --on-accent: #000;
  --good: #7CFC9A;      --bad: #ff8080;
  --code-bg: #000;      --code-text: #fff;
  --glass: #000;        --glass-border: #fff;  /* opaque; borders at full strength */
  --focus: #ffd166;
  --blush: #ffd166;     --on-blush: #000;
  --ember: #ffd166;     --grain-opacity: 0;
}
body { background: var(--bg); color: var(--text); }
```

High-contrast theme rules: no transparency, no backdrop blur, no grain, particles off or reduced to a few static points, thick focus rings (3px), borders on all controls, and 7:1 for body text.

**Forced colours** (Windows High Contrast): let the system palette win and only repair things that disappear.

```css
@media (forced-colors: active) {
  .glass { backdrop-filter: none; background: Canvas; border: 1px solid CanvasText; }
  .cursor-dot, .grain, .ghost, canvas.bg { display: none; }
  :focus-visible { outline: 2px solid Highlight; }
  .icon { forced-color-adjust: auto; fill: CanvasText; }
}
```

## Motion tokens

```css
:root {
  --dur-1: 150ms; --dur-2: 300ms; --dur-3: 600ms; --dur-4: 900ms;
  --stagger: 70ms;
  --ease-out: cubic-bezier(.16, 1, .3, 1);
  --ease-in-out: cubic-bezier(.76, 0, .24, 1);
  --ease-spring: cubic-bezier(.34, 1.56, .64, 1);
}
:root[data-motion="reduce"] { --dur-1: 0ms; --dur-2: 0ms; --dur-3: 0ms; --dur-4: 0ms; --stagger: 0ms; }
@media (prefers-reduced-motion: reduce) {
  :root { --dur-1: 0ms; --dur-2: 0ms; --dur-3: 0ms; --dur-4: 0ms; --stagger: 0ms; }
}
```

## Z-index tokens

```css
:root {
  --z-canvas: -1;   --z-content: 1;  --z-glass: 10;  --z-rail: 20;
  --z-hotspot: 30;  --z-mini: 50;    --z-menu: 90;   --z-modal: 100;
  --z-toast: 110;   --z-curtain: 200; --z-cursor: 300; --z-skip: 400;
}
```

The curtain sits above modals because it covers everything during a scene change. The cursor sits above the curtain. The skip link sits above everything so it's always visible when focused.

## Reading tokens from JS

```js
// config.js — read once at startup, and again on theme change for colours
const css = getComputedStyle(document.documentElement);
const ms = v => parseFloat(v) * (v.trim().endsWith('ms') ? 1 : 1000);
export const DUR = { 1: ms(css.getPropertyValue('--dur-1')), 2: ms(css.getPropertyValue('--dur-2')),
                     3: ms(css.getPropertyValue('--dur-3')), 4: ms(css.getPropertyValue('--dur-4')) };
export const color = name => getComputedStyle(document.documentElement).getPropertyValue(name).trim();
```

GSAP easing equivalents: `--ease-out` ≈ `expo.out`, `--ease-in-out` ≈ `power4.inOut`, `--ease-spring` ≈ `back.out(1.7)`.

## Theme switcher

Four choices: **System** (default), **Dark**, **Light**, **High contrast**. Use a native radio group, which gives arrow-key navigation and the correct roles without extra code. The `T` shortcut cycles through the same four options.

```html
<fieldset class="theme-switch">
  <legend class="visually-hidden">Theme</legend>
  <label><input type="radio" name="theme" value="system"> System</label>
  <label><input type="radio" name="theme" value="dark"> Dark</label>
  <label><input type="radio" name="theme" value="light"> Light</label>
  <label><input type="radio" name="theme" value="contrast"> High contrast</label>
</fieldset>
```

Resolve the theme before first paint with a small inline script in `<head>`, so there's no flash of the wrong theme:

```html
<script>
(function () {
  var c; try { c = localStorage.getItem('themeChoice'); } catch (e) {}
  if (['dark', 'light', 'contrast'].indexOf(c) < 0) c = 'system';
  var t = c !== 'system' ? c
        : matchMedia('(prefers-contrast: more)').matches ? 'contrast'
        : matchMedia('(prefers-color-scheme: light)').matches ? 'light' : 'dark';
  document.documentElement.dataset.theme = t;
  document.documentElement.dataset.themeChoice = c;
})();
</script>
```

```js
// prefs.js
const mqLight = matchMedia('(prefers-color-scheme: light)');
const mqMore = matchMedia('(prefers-contrast: more)');
const systemTheme = () => mqMore.matches ? 'contrast' : mqLight.matches ? 'light' : 'dark';
const CHOICES = ['system', 'dark', 'light', 'contrast'];

export function applyTheme(choice, { save = true } = {}) {
  const root = document.documentElement;
  const theme = choice === 'system' ? systemTheme() : choice;
  root.dataset.theme = theme;
  root.dataset.themeChoice = choice;
  if (save) try { choice === 'system' ? localStorage.removeItem('themeChoice') : localStorage.setItem('themeChoice', choice); } catch (e) {}
  document.querySelector(`.theme-switch input[value="${choice}"]`).checked = true;
  store.set({ themeChoice: choice, theme });
  bus.emit('theme:change', theme);             // particles, post-processing, PWA theme-color meta
}
export const cycleTheme = () => applyTheme(CHOICES[(CHOICES.indexOf(store.get().themeChoice) + 1) % CHOICES.length]);

document.querySelector('.theme-switch').addEventListener('change', e => applyTheme(e.target.value));
[mqLight, mqMore].forEach(mq => mq.addEventListener('change', () => {
  if (store.get().themeChoice === 'system') applyTheme('system', { save: false });   // follow the OS live only while on System
}));
applyTheme(document.documentElement.dataset.themeChoice, { save: false });
```

On theme change, also update `<meta name="theme-color">` and announce politely ("Light theme").

To animate a theme change without transitioning every property, use a View Transition when available:

```js
const swap = () => applyTheme(choice);
document.startViewTransition && !motion.reduced ? document.startViewTransition(swap) : swap();
```

## Frosted-glass panels

The top bar, tabs and panels blur the particles behind them.

```css
.glass {
  background: var(--glass);
  border: 1px solid var(--glass-border);
  -webkit-backdrop-filter: blur(14px) saturate(140%);
          backdrop-filter: blur(14px) saturate(140%);
  z-index: var(--z-glass);
}
@supports not ((backdrop-filter: blur(1px)) or (-webkit-backdrop-filter: blur(1px))) {
  .glass { background: var(--surface); }              /* opaque fallback keeps text legible */
}
:root[data-quality="low"] .glass { backdrop-filter: none; -webkit-backdrop-filter: none; background: var(--surface); }
```

- The canvas sits behind (`position: fixed; inset: 0; z-index: var(--z-canvas)`). Glass elements need no opaque ancestor between them and the canvas.
- Keep blur at or below about 16px on mobile, don't nest glass inside glass, and keep text contrast measured against the worst case (the brightest particles behind the panel).

## Film grain overlay

A subtle texture that adds depth to glass and the dark theme.

```css
.grain {
  position: fixed; inset: -50%; pointer-events: none; z-index: calc(var(--z-glass) + 1);
  opacity: var(--grain-opacity);
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='160' height='160'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='.9' numOctaves='2' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)'/%3E%3C/svg%3E");
  mix-blend-mode: overlay;
  animation: grain 1s steps(6) infinite;
}
@keyframes grain {
  0% { transform: translate(0, 0); } 20% { transform: translate(-5%, 3%); } 40% { transform: translate(4%, -6%); }
  60% { transform: translate(-3%, 5%); } 80% { transform: translate(6%, -2%); } 100% { transform: translate(0, 0); }
}
@media (prefers-reduced-motion: reduce) { .grain { animation: none; } }
:root[data-motion="reduce"] .grain, :root[data-quality="low"] .grain { animation: none; }
```

Add `aria-hidden="true"` to the grain element. It must not lower text contrast below the minimum; re-check pairs with grain enabled. It's disabled in the high-contrast theme (`--grain-opacity: 0`).

## Text selection colour

```css
::selection { background: var(--blush); color: var(--on-blush); }
```

`::-moz-selection` is no longer needed. Pale blush with dark text measures above 15:1. Scoped overrides are only needed if a section has its own unusual background.
