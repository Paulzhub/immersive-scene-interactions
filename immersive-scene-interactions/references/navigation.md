# Navigation and interaction extras

Covers the keyboard shortcuts overlay, gesture navigation, deep links, the command palette, the before/after comparison slider, toasts, onboarding coach marks and glossary tooltips.

The key map and the Esc layer stack are defined in `architecture.md`; this file builds on them.

## Contents
- Keyboard shortcuts overlay
- Gesture navigation (swipe + wheel-snap)
- Deep links
- Command palette
- Before/after comparison slider
- Toasts and micro-feedback
- Onboarding coach marks
- Glossary tooltips

## Keyboard shortcuts overlay

Press `?` to open a dialog that lists every shortcut, grouped as in the key map, with a switch to turn single-key shortcuts off (WCAG 2.1.4).

```html
<dialog class="shortcuts glass" aria-labelledby="sc-h">
  <h2 id="sc-h">Keyboard shortcuts</h2>
  <dl>
    <dt><kbd>Space</kbd> / <kbd>K</kbd></dt><dd>Play or pause</dd>
    <dt><kbd>←</kbd> <kbd>→</kbd></dt><dd>Previous or next scene (<kbd>Shift</kbd>: seek 5 seconds)</dd>
    <dt><kbd>M</kbd> <kbd>F</kbd> <kbd>C</kbd> <kbd>T</kbd></dt><dd>Mute, full screen, captions, theme</dd>
    <dt><kbd>Ctrl</kbd>/<kbd>⌘</kbd> + <kbd>K</kbd></dt><dd>Command palette</dd>
    <dt>3D view focused</dt><dd>Arrows rotate, <kbd>+</kbd>/<kbd>−</kbd> zoom, <kbd>0</kbd> reset</dd>
  </dl>
  <label><input type="checkbox" class="sc-toggle" checked> Single-key shortcuts</label>
  <button class="close">Close</button>
</dialog>
```

```js
const dlg = document.querySelector('.shortcuts');
const layer = { close: () => dlg.close() };
export const shortcutsOverlay = {
  open() { dlg.showModal(); layers.push(layer); },
};
dlg.addEventListener('close', () => layers.remove(layer));
dlg.querySelector('.close').onclick = () => dlg.close();
dlg.querySelector('.sc-toggle').addEventListener('change', e => store.set({ shortcuts: e.target.checked }));
```

A native `<dialog>` with `showModal()` traps focus, closes on Esc and returns focus to the opener automatically. Show `⌘` on Apple platforms and `Ctrl` elsewhere (`/Mac|iPhone|iPad/.test(navigator.userAgent)`).

## Gesture navigation

Step mode only. Every gesture also has a visible button and a key.

```js
// swipe
let start = null;
stage.addEventListener('pointerdown', e => {
  if (e.pointerType === 'mouse' || e.target.closest('canvas.tour, .tabs, [data-lenis-prevent], input, .hscroll')) return;
  start = { x: e.clientX, y: e.clientY, t: performance.now() };
}, { passive: true });
stage.addEventListener('pointerup', e => {
  if (!start) return;
  const dx = e.clientX - start.x, dy = e.clientY - start.y, dt = performance.now() - start.t;
  start = null;
  if (Math.abs(dx) > 50 && Math.abs(dx) > Math.abs(dy) * 1.5 && Math.abs(dx) / dt > .3) {
    const dir = document.dir === 'rtl' ? 1 : -1;
    goToScene(store.get().scene + Math.sign(dx) * dir);
  }
}, { passive: true });

// wheel-snap with accumulator and cooldown, so trackpad inertia never skips scenes
let acc = 0, lockUntil = 0;
stage.addEventListener('wheel', e => {
  if (e.target.closest('[data-lenis-prevent], .scrollable, canvas.tour') || e.ctrlKey) return;   // ctrlKey = pinch-zoom on trackpads
  e.preventDefault();
  const now = performance.now();
  if (now < lockUntil) return;
  acc += e.deltaY;
  if (Math.abs(acc) > 60) {
    goToScene(store.get().scene + Math.sign(acc));
    acc = 0; lockUntil = now + 900;
  }
}, { passive: false });
```

Never wheel-snap a scroll-mode page (`architecture.md`). Set `touch-action: pan-y` on the stage, so vertical scrolling inside panels still works while horizontal swipes are captured.

## Deep links

Reflect the shareable state in the URL: `#scene-3`, `?t=42`, `?lang=hi`, `?tab=advanced`.

```js
const params = () => new URLSearchParams(location.search);
export function readURL() {
  const p = params(), patch = {};
  const m = location.hash.match(/^#scene-(\d+)$/);
  if (m) patch.scene = Math.min(scenes.length - 1, Math.max(0, +m[1] - 1));
  const t = parseFloat(p.get('t')); if (Number.isFinite(t) && t >= 0) patch.time = Math.min(t, total());
  if (['en', 'hi'].includes(p.get('lang'))) patch.lang = p.get('lang');
  if (['basics', 'advanced', 'leaderboard'].includes(p.get('tab'))) patch.tab = p.get('tab');
  return patch;                                                // validated and clamped; unknown values ignored
}

let writeTimer;
store.subscribe((s, prev) => {
  const navigational = s.scene !== prev.scene || s.tab !== prev.tab || s.lang !== prev.lang;
  clearTimeout(writeTimer);
  writeTimer = setTimeout(() => {
    const p = params();
    p.set('t', Math.floor(s.time)); p.set('lang', s.lang); p.set('tab', s.tab);
    const url = `${location.pathname}?${p}#scene-${s.scene + 1}`;
    navigational ? history.pushState(null, '', url) : history.replaceState(null, '', url);
  }, navigational ? 0 : 1000);                                  // continuous changes (seeking) only replace
});
addEventListener('popstate', () => { const p = readURL(); if (p.scene != null) goToScene(p.scene); store.set(p); });
```

On load, apply `readURL()` with playback **paused**, and wait for a gesture before playing. A "Copy link to this moment" button copies the current URL and shows a toast. Remove `t` from shared links when the user is at 0.

## Command palette

`Ctrl/Cmd + K` opens a searchable list of every action: go to a scene, switch tab, change theme, language or speed, toggle captions or sound, open the transcript or shortcuts, or take a snapshot.

```html
<dialog class="palette glass" aria-label="Command palette">
  <input role="combobox" aria-expanded="true" aria-controls="pal-list" aria-autocomplete="list" placeholder="Type a command or scene…">
  <ul id="pal-list" role="listbox"></ul>
</dialog>
```

```js
const ACTIONS = () => [
  ...scenes.map((s, i) => ({ label: `Go to scene ${i + 1}: ${s.title[store.get().lang]}`, run: () => goToScene(i) })),
  { label: 'Theme: System', run: () => applyTheme('system') }, { label: 'Theme: Dark', run: () => applyTheme('dark') },
  { label: 'Theme: Light', run: () => applyTheme('light') }, { label: 'Theme: High contrast', run: () => applyTheme('contrast') },
  { label: 'Language: English', run: () => setLang('en') }, { label: 'भाषा: हिन्दी', run: () => setLang('hi') },
  { label: 'Toggle captions', run: () => toggleCaptions() }, { label: 'Open transcript', run: () => transcript.open() },
  { label: 'Keyboard shortcuts', run: () => shortcutsOverlay.open() },
  ...[0.5, 1, 1.5, 2].map(v => ({ label: `Speed ${v}×`, run: () => store.set({ speed: v }) })),
];
const score = (q, s) => {                                      // simple fuzzy: all query chars in order, bonus for word starts
  let i = 0, pts = 0;
  for (let j = 0; j < s.length && i < q.length; j++) if (s[j] === q[i]) { pts += (j === 0 || s[j - 1] === ' ') ? 3 : 1; i++; }
  return i === q.length ? pts : -1;
};
let items = [], active = 0;
function render(q) {
  q = q.toLocaleLowerCase();
  items = ACTIONS().map(a => ({ ...a, s: q ? score(q, a.label.toLocaleLowerCase()) : 0 })).filter(a => a.s >= 0).sort((a, b) => b.s - a.s).slice(0, 12);
  active = 0;
  list.innerHTML = items.map((a, i) => `<li role="option" id="pal-${i}" aria-selected="${i === 0}"></li>`).join('');
  [...list.children].forEach((li, i) => { li.textContent = items[i].label; li.onclick = () => run(i); });
  input.setAttribute('aria-activedescendant', items.length ? 'pal-0' : '');
}
input.addEventListener('input', () => render(input.value));
input.addEventListener('keydown', e => {
  if (e.key === 'ArrowDown' || e.key === 'ArrowUp') {
    e.preventDefault();
    active = (active + (e.key === 'ArrowDown' ? 1 : -1) + items.length) % items.length;
    [...list.children].forEach((li, i) => li.setAttribute('aria-selected', i === active));
    input.setAttribute('aria-activedescendant', `pal-${active}`);
    list.children[active]?.scrollIntoView({ block: 'nearest' });
  } else if (e.key === 'Enter' && items[active]) { e.preventDefault(); run(active); }
});
function run(i) { palette.close(); items[i].run(); }
```

Open the palette with `showModal()` and register it in the layer stack. Show a one-line hint ("Press Ctrl K for commands") in the shortcuts overlay and the footer.

## Before/after comparison slider

```html
<div class="compare" style="--pos: 50%">
  <img src="after.webp" alt="After: cleaned engine bay">
  <img class="compare-before" src="before.webp" alt="Before: engine bay with oil residue">
  <div class="compare-handle" role="slider" tabindex="0" aria-label="Comparison position"
       aria-valuemin="0" aria-valuemax="100" aria-valuenow="50" aria-valuetext="50% before"></div>
  <span class="compare-tag before">Before</span><span class="compare-tag after">After</span>
</div>
```

```css
.compare { position: relative; touch-action: pan-y; user-select: none; }
.compare img { display: block; width: 100%; }
.compare-before { position: absolute; inset: 0; clip-path: inset(0 calc(100% - var(--pos)) 0 0); }
.compare-handle { position: absolute; top: 0; bottom: 0; left: var(--pos); width: 44px; translate: -50% 0; cursor: ew-resize; }
.compare-handle::before { content: ''; position: absolute; inset: 0 auto 0 50%; width: 2px; background: var(--text); }
```

```js
function set(p) {
  p = Math.max(0, Math.min(100, p));
  compare.style.setProperty('--pos', p + '%');
  handle.setAttribute('aria-valuenow', Math.round(p));
  handle.setAttribute('aria-valuetext', `${Math.round(p)}% before`);
}
handle.addEventListener('pointerdown', e => {
  handle.setPointerCapture(e.pointerId);
  const move = ev => { const r = compare.getBoundingClientRect(); set((ev.clientX - r.left) / r.width * 100); };
  handle.addEventListener('pointermove', move);
  handle.addEventListener('pointerup', () => handle.removeEventListener('pointermove', move), { once: true });
});
handle.addEventListener('keydown', e => {
  const cur = +handle.getAttribute('aria-valuenow'), step = e.shiftKey ? 10 : 2;
  const next = { ArrowLeft: cur - step, ArrowRight: cur + step, Home: 0, End: 100 }[e.key];
  if (next != null) { set(next); e.preventDefault(); }
});
```

Clicking anywhere on the image also moves the handle there. Both images keep meaningful alt text.

## Toasts and micro-feedback

Small glass toasts for copy, save, share, unlocks and errors.

```html
<div class="toasts" aria-live="polite"></div>
<div class="toasts-alert" role="alert"></div>
```

```js
export function toast(msg, { kind = 'info', ms = kind === 'error' ? 7000 : 3000, action } = {}) {
  const host = kind === 'error' ? alertHost : politeHost;
  const el = document.createElement('div');
  el.className = `toast glass toast-${kind}`;
  el.append(Object.assign(document.createElement('span'), { textContent: msg }));
  if (action) el.append(Object.assign(document.createElement('button'), { textContent: action.label, onclick: () => { action.run(); close(); } }));
  el.append(Object.assign(document.createElement('button'), { className: 'toast-x', ariaLabel: 'Dismiss', textContent: '×', onclick: () => close() }));
  host.append(el);
  while (host.children.length > 3) host.firstElementChild.remove();
  let timer = setTimeout(close, ms);
  el.addEventListener('pointerenter', () => clearTimeout(timer));
  el.addEventListener('focusin', () => clearTimeout(timer));
  el.addEventListener('pointerleave', () => { timer = setTimeout(close, ms); });
  function close() { clearTimeout(timer); el.remove(); }
}
toast.error = (msg, o) => toast(msg, { ...o, kind: 'error' });
```

```css
.toasts, .toasts-alert { position: fixed; inset-inline: 0; bottom: max(16px, env(safe-area-inset-bottom)); z-index: var(--z-toast);
  display: grid; justify-items: center; gap: 8px; pointer-events: none; }
.toast { pointer-events: auto; display: flex; gap: 12px; align-items: center; padding: 10px 14px; border-radius: 10px;
  animation: toast-in var(--dur-2) var(--ease-out); }
@keyframes toast-in { from { opacity: 0; transform: translateY(8px); } }
```

Micro-feedback on the triggering button: swap the icon for a check mark for 1.2 seconds and scale it briefly (`scale: .96` → `1`). Change the label too ("Copied"), so the feedback isn't only visual.

## Onboarding coach marks

Up to three hints on the first visit: "Drag to look around", "Tap the glowing points for tips" and "Press ? for shortcuts". They can be skipped, and they never appear again once dismissed.

```html
<div id="coach-1" popover="manual" class="coach glass" role="note">
  <p>Drag to look around, or use the arrow keys.</p>
  <button data-next>Next</button> <button data-skip>Skip tips</button>
</div>
```

```css
.coach { position-anchor: --coach-target; position-area: bottom; margin: 8px; position-try-fallbacks: flip-block, flip-inline; }
```

```js
async function runCoach(steps) {                          // steps: [{ popover, target }]
  try { if (localStorage.getItem('coachDone')) return; } catch {}
  for (const { popover, target } of steps) {
    target.style.anchorName = '--coach-target';
    popover.showPopover();
    const r = await new Promise(res => { popover.querySelector('[data-next]').onclick = () => res('next'); popover.querySelector('[data-skip]').onclick = () => res('skip'); });
    popover.hidePopover(); target.style.anchorName = '';
    if (r === 'skip') break;
  }
  try { localStorage.setItem('coachDone', '1'); } catch {}
}
```

Where CSS anchor positioning isn't supported, position the popover with `getBoundingClientRect()` and clamp it to the viewport. Start coach marks only after the preloader finishes, never on top of a modal. Hints are non-modal and don't steal focus.

## Glossary tooltips

Definitions for technical terms, using the toggletip pattern (a button that shows a definition), which works for touch and keyboard users too.

```html
<span class="term">
  <button class="term-btn" aria-expanded="false" aria-controls="def-injector" popovertarget="def-injector">fuel injector</button>
  <span id="def-injector" popover class="term-def glass" role="note">A valve that sprays fuel into the engine in precise amounts.</span>
</span>
```

```css
.term-btn { all: unset; text-decoration: underline dotted; text-underline-offset: .2em; cursor: help; }
.term-btn:focus-visible { outline: 2px solid var(--focus); outline-offset: 2px; }
```

- The popover API gives light dismiss and Esc for free. Update `aria-expanded` on the `toggle` event.
- On fine pointers, also open on hover after 300ms, and close when the pointer leaves both the term and the definition.
- Generate terms from `glossary.json`, linking only the first occurrence in each section, never inside headings or links. Add the full glossary as a page or panel too.
