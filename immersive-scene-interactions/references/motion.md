# Motion: micro-interactions and transitions

Covers magnetic buttons, staggered text reveal, the curtain transition (with View Transitions), the rolling ghost numeral, tab transitions, the preloader, the scene progress rail, text scramble, number counters, 3D card flip, variable-font animation and the infinite marquee.

Scroll-driven motion (parallax, Lenis, pinned sections, line draw, image sequences) is in `scroll.md`. Everything here assumes `store`, `bus`, `motion`, `pointer` and `goToScene` from `architecture.md` and the tokens from `theming.md`.

## Contents
- Magnetic buttons
- Staggered text reveal
- Scene transition: View Transitions or curtain wipe
- Rolling ghost numeral
- Tab transitions (GSAP)
- Preloader with progress
- Scene progress rail
- Text scramble / decode
- Animated number counters
- 3D card flip
- Variable-font animation
- Infinite marquee

## Magnetic buttons

Buttons spring toward the cursor within a radius and bounce back on leave. The wrapper stays still and is measured; only the inner element moves, so the measurement never drifts.

```html
<span class="magnet" data-magnetic><button class="btn magnet-inner"><span class="magnet-label">Start tour</span></button></span>
```

```css
.magnet { display: inline-block; padding: 12px; margin: -12px; }  /* larger field without changing layout */
.magnet-inner, .magnet-label { display: inline-block; will-change: transform; }
```

```js
const STRENGTH = 0.3, LABEL = 0.5, MAX = 12;
const clamp = v => Math.max(-MAX, Math.min(MAX, v));

document.querySelectorAll('[data-magnetic]').forEach(wrap => {
  const inner = wrap.querySelector('.magnet-inner'), label = wrap.querySelector('.magnet-label');
  let r = null;
  const measure = () => { r = wrap.getBoundingClientRect(); };
  const release = () => {
    r = null;
    for (const el of [inner, label]) {
      el.style.transition = `transform 700ms var(--ease-spring)`;
      el.style.transform = '';
    }
  };
  wrap.addEventListener('pointerenter', e => { if (e.pointerType === 'mouse') measure(); });
  wrap.addEventListener('pointermove', e => {
    if (!r || !pointer.fine || motion.reduced) return;
    const dx = e.clientX - (r.left + r.width / 2), dy = e.clientY - (r.top + r.height / 2);
    inner.style.transition = label.style.transition = 'transform 150ms linear';
    inner.style.transform = `translate(${clamp(dx * STRENGTH)}px, ${clamp(dy * STRENGTH)}px)`;
    label.style.transform = `translate(${clamp(dx * (LABEL - STRENGTH))}px, ${clamp(dy * (LABEL - STRENGTH))}px)`;
  });
  wrap.addEventListener('pointerleave', release);
  addEventListener('scroll', () => r && measure(), { passive: true });
});
```

The label offset is relative to the button, which is already offset, so it uses `LABEL - STRENGTH`. Keyboard focus gets no magnetic movement; the visible focus ring is the feedback.

## Staggered text reveal

Each panel unmasks first, then its heading rises word by word. Split the heading at runtime and keep the real text available to screen readers.

```js
export function splitWords(h) {
  if (h.dataset.split) return;
  const text = h.textContent.trim().replace(/\s+/g, ' ');
  h.setAttribute('aria-label', text);
  h.textContent = '';
  text.split(' ').forEach((word, i, all) => {
    const outer = document.createElement('span');
    const inner = document.createElement('span');
    outer.className = 'w'; outer.setAttribute('aria-hidden', 'true');
    inner.className = 'wi'; inner.style.setProperty('--i', i); inner.textContent = word;
    outer.append(inner); h.append(outer);
    if (i < all.length - 1) h.append(' ');            // keep real spaces between words
  });
  h.dataset.split = '1';
}
```

Only split plain-text headings. Inline markup such as links or `<em>` would be lost.

```css
.panel[data-reveal] { clip-path: inset(0 0 100% 0); transition: clip-path var(--dur-4) var(--ease-out); }
.panel[data-reveal].is-active { clip-path: inset(0); }
.w  { display: inline-block; overflow: hidden; vertical-align: bottom; padding-bottom: .08em; }  /* room for descenders */
.wi { display: inline-block; transform: translateY(110%);
      transition: transform var(--dur-4) var(--ease-out);
      transition-delay: calc(var(--i) * var(--stagger) + var(--dur-3) * .75); }
.panel.is-active .wi { transform: none; }
```

Toggle the class on `scene:mid` so the reveal replays on every visit:

```js
bus.on('scene:mid', n => panels.forEach((p, i) => p.classList.toggle('is-active', i === n)));
```

In scroll mode, add `.is-active` from an `IntersectionObserver` instead, or use the CSS scroll-driven version in `scroll.md`. With reduced motion, the zeroed tokens make it appear instantly.

## Scene transition

`transition(n)` from `goToScene` covers the screen, calls `swapScene(n)` at the midpoint, then uncovers it. There are two implementations.

### Option A — View Transitions API (preferred where supported)

```js
const webglMustKeepRunning = !!document.querySelector('canvas.tour');   // see note below
async function transition(n, fromClock) {
  if (!document.startViewTransition || webglMustKeepRunning) return curtain(n, fromClock);
  document.documentElement.dataset.dir = n > store.get().scene ? 'fwd' : 'back';
  const vt = document.startViewTransition(() => swapScene(n, fromClock));
  await vt.finished;
}
```

```css
::view-transition-old(root) { animation: none; }
::view-transition-new(root) { animation: wipe var(--dur-4) var(--ease-in-out); }
[data-dir="back"]::view-transition-new(root) { animation-name: wipe-back; }
@keyframes wipe      { from { clip-path: inset(100% 0 0 0); } to { clip-path: inset(0); } }
@keyframes wipe-back { from { clip-path: inset(0 0 100% 0); } to { clip-path: inset(0); } }
```

View Transitions snapshot the WebGL canvas as a frozen image during the transition. If the 3D scene must keep animating through the change, use the curtain instead.

### Option B — Two-layer curtain wipe showing the scene number

```html
<div class="curtain" aria-hidden="true">
  <div class="curtain-layer a"></div><div class="curtain-layer b"></div>
  <span class="curtain-num"></span>
</div>
```

```css
.curtain { position: fixed; inset: 0; z-index: var(--z-curtain); pointer-events: none; display: grid; place-items: center; }
.curtain-layer { position: absolute; inset: 0; transform: scaleY(0); }
.curtain-layer.a { background: linear-gradient(160deg, var(--accent), var(--bg)); }
.curtain-layer.b { background: linear-gradient(160deg, var(--surface), var(--bg)); }
.curtain-num { position: relative; font-size: clamp(4rem, 14vw, 10rem); color: var(--text); opacity: 0; font-variant-numeric: tabular-nums; }
.curtain.is-active { pointer-events: auto; }          /* block clicks while covered */
```

```js
const curtainLayers = [...document.querySelectorAll('.curtain-layer')];
const curtainEl = document.querySelector('.curtain'), num = document.querySelector('.curtain-num');
const EASE = 'cubic-bezier(.76,0,.24,1)';

function wipe(dir) {
  const kf = dir === 'in' ? [{ transform: 'scaleY(0)' }, { transform: 'scaleY(1)' }]
                          : [{ transform: 'scaleY(1)' }, { transform: 'scaleY(0)' }];
  return Promise.all(curtainLayers.map((l, i) => {
    l.style.transformOrigin = dir === 'in' ? 'bottom' : 'top';
    const order = dir === 'in' ? i : curtainLayers.length - 1 - i;     // top layer leaves first
    return l.animate(kf, { duration: 700, delay: order * 120, easing: EASE, fill: 'forwards' }).finished;
  }));
}

async function curtain(n, fromClock) {
  num.textContent = String(n + 1).padStart(2, '0');
  curtainEl.classList.add('is-active');
  try {
    await wipe('in');
    await num.animate([{ opacity: 0, transform: 'translateY(20%)' }, { opacity: 1, transform: 'none' }],
                      { duration: 250, easing: 'ease-out', fill: 'forwards' }).finished;
    swapScene(n, fromClock);
    await num.animate([{ opacity: 1 }, { opacity: 0 }], { duration: 200, delay: 150, fill: 'forwards' }).finished;
    await wipe('out');
  } finally {
    curtainEl.classList.remove('is-active');
  }
}
```

Lens breath and the ghost numeral fire from `scene:mid`, so they're already moving as the curtain lifts. Sound design adds a whoosh on `scene:before` (`audio.md`).

## Rolling ghost numeral

A large outlined scene number in a corner. On each change the old digits roll out and the new ones roll in, in the direction of travel.

```html
<div class="ghost" aria-hidden="true"></div>
```

```css
.ghost { position: fixed; right: 4vw; bottom: 2vh; z-index: var(--z-content); pointer-events: none;
  display: grid; overflow: hidden; height: .82em; line-height: .82;
  font-size: clamp(8rem, 22vw, 20rem); font-variant-numeric: tabular-nums;
  color: transparent; -webkit-text-stroke: 1px color-mix(in srgb, var(--text) 25%, transparent); }
.ghost .roll { grid-area: 1 / 1; }
:root[data-theme="contrast"] .ghost { display: none; }
```

```js
const ghost = document.querySelector('.ghost');
let last = 0;

bus.on('scene:mid', n => {
  const dir = n >= last ? 1 : -1; last = n;
  ghost.querySelectorAll('.roll.leaving').forEach(el => el.remove());      // rapid changes: drop stale exits
  const prev = ghost.querySelector('.roll');
  const next = Object.assign(document.createElement('span'), { className: 'roll', textContent: String(n + 1).padStart(2, '0') });
  ghost.append(next);
  if (!prev) return;
  if (motion.reduced) { prev.remove(); return; }
  prev.classList.add('leaving');
  const o = { duration: 800, easing: 'cubic-bezier(.16,1,.3,1)' };
  prev.animate([{ transform: 'translateY(0)' }, { transform: `translateY(${-100 * dir}%)` }], o).finished.then(() => prev.remove());
  next.animate([{ transform: `translateY(${100 * dir}%)` }, { transform: 'translateY(0)' }], o);
});
```

## Tab transitions (GSAP)

The current section fades out and the next fades and slides in. The quiz's Basics, Advanced and Leaderboard tabs use the same helper. Before a new switch starts, the previous one is completed, so fast clicks never leave two panels visible.

```js
let tl;
export function switchPanel(from, to) {
  if (from === to) return;
  tl?.progress(1).kill();
  if (motion.reduced) { from.hidden = true; to.hidden = false; return; }
  tl = gsap.timeline({ defaults: { ease: 'power2.out' } })
    .to(from, { autoAlpha: 0, y: -12, duration: .25,
                onComplete() { from.hidden = true; gsap.set(from, { clearProps: 'opacity,visibility,transform' }); } })
    .fromTo(to, { autoAlpha: 0, y: 24 },
                { autoAlpha: 1, y: 0, duration: .45,
                  onStart() { to.hidden = false; },
                  onComplete() { gsap.set(to, { clearProps: 'opacity,visibility,transform' }); } });
}
```

Animate the `.panel-transition` wrapper, not the parallax wrapper (see transform ownership in `architecture.md`).

Accessibility markup: `role="tablist"`, `role="tab"` with `aria-selected` and `aria-controls`, and `role="tabpanel"` with `aria-labelledby`. Use a roving `tabindex` (←/→, Home, End). Focus stays on the tab; the panel has `tabindex="0"` if it contains no focusable content. With GSAP 3.13 and later, all plugins (ScrollTrigger, SplitText, MorphSVG and others) are free.

## Preloader with progress

A full-screen loader counts from 0 to 100% while 3D assets, fonts and the first audio clip load. It then hands off to the curtain and scene 1's reveal.

```js
const loader = document.querySelector('.preloader'), bar = loader.querySelector('[role="progressbar"]');
let target = 0, shown = 0;
const sources = { three: 0, fonts: 0, audio: 0 };
const WEIGHTS = { three: .7, fonts: .1, audio: .2 };
const update = () => { target = Object.entries(sources).reduce((a, [k, v]) => a + v * WEIGHTS[k], 0); };

manager.onProgress = (_, loaded, total) => { sources.three = loaded / total; update(); };   // THREE.LoadingManager
document.fonts.ready.then(() => { sources.fonts = 1; update(); });
firstClip.then(() => { sources.audio = 1; update(); }, () => { sources.audio = 1; update(); });

const t0 = performance.now(), MIN = 600, TIMEOUT = 15000;
(function tick() {
  shown += (target - shown) * .12;                       // smoothed; never goes backwards
  const pct = Math.round(shown * 100);
  bar.setAttribute('aria-valuenow', pct);
  loader.querySelector('.pct').textContent = pct + '%';
  const elapsed = performance.now() - t0;
  if ((target >= 1 && shown > .995 && elapsed > MIN) || elapsed > TIMEOUT) return finish();
  requestAnimationFrame(tick);
})();

async function finish() {
  document.querySelector('main').removeAttribute('aria-busy');
  if (motion.reduced) loader.remove();
  else await loader.animate([{ opacity: 1 }, { opacity: 0 }], { duration: 400, fill: 'forwards' }).finished.then(() => loader.remove());
  bus.emit('scene:mid', 0);                               // first reveal, ghost numeral, camera
}
```

Use `<div role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-label="Loading">`, and set `aria-busy="true"` on `<main>` until loading finishes. After the timeout, continue anyway and let the fallbacks handle whatever didn't load (`accessibility.md`).

## Scene progress rail

A thin progress bar with a clickable dot per scene. It sits on the right edge on desktop and along the bottom on mobile.

```html
<nav class="rail" aria-label="Scenes">
  <div class="rail-fill" aria-hidden="true"></div>
  <ol>
    <li><button aria-label="Scene 1: Introduction" data-scene="0"></button></li>
    …
  </ol>
</nav>
```

```js
rail.addEventListener('click', e => { const b = e.target.closest('[data-scene]'); if (b) goToScene(+b.dataset.scene); });
store.subscribe(s => {
  const p = (s.scene + s.sceneProgress) / scenes.length;        // step mode; scroll mode: page scroll progress
  fill.style.transform = `scaleY(${p})`;
  rail.querySelectorAll('[data-scene]').forEach((b, i) =>
    i === s.scene ? b.setAttribute('aria-current', 'step') : b.removeAttribute('aria-current'));
});
```

The active dot scales up and glows with `--accent`. On hover and focus, show the scene title as a visible tooltip in addition to the `aria-label`. Each dot's hit area must be at least 44×44px even though the visible dot is small.

## Text scramble / decode

Labels and counters resolve from random glyphs into the final text.

```js
export function scramble(el, final, dur = 700) {
  el.setAttribute('aria-label', final);                 // screen readers get the real text
  if (motion.reduced) { el.textContent = final; return; }
  const chars = '!<>-_\\/[]{}—=+*^?#', t0 = performance.now();
  (function tick(now) {
    const p = Math.min((now - t0) / dur, 1), n = Math.floor(p * final.length);
    el.textContent = final.slice(0, n) +
      [...final.slice(n)].map(c => c === ' ' ? ' ' : chars[(Math.random() * chars.length) | 0]).join('');
    if (p < 1) requestAnimationFrame(tick);
  })(t0);
}
```

Use a monospace font or `font-variant-numeric: tabular-nums` so the width doesn't jitter. Don't scramble long text or Devanagari/Arabic script; it reads as broken.

## Animated number counters

Stats count up the first time they enter view.

```js
const fmt = new Intl.NumberFormat(document.documentElement.lang);
const io = new IntersectionObserver(entries => entries.forEach(({ isIntersecting, target: el }) => {
  if (!isIntersecting) return;
  io.unobserve(el);
  const to = +el.dataset.to, dur = 1200, t0 = performance.now();
  if (motion.reduced) { el.textContent = fmt.format(to); return; }
  (function tick(now) {
    const p = Math.min((now - t0) / dur, 1), eased = 1 - Math.pow(1 - p, 3);
    el.textContent = fmt.format(Math.round(to * eased));
    if (p < 1) requestAnimationFrame(tick);
  })(t0);
}), { threshold: .4 });
document.querySelectorAll('[data-to]').forEach(el => { el.setAttribute('aria-label', fmt.format(+el.dataset.to)); io.observe(el); });
```

Add `font-variant-numeric: tabular-nums` to prevent jitter. The server-rendered text should already be the final value, so the page is correct without JavaScript.

## 3D card flip

Used for quiz answers and flashcards.

```html
<div class="card"><div class="card-inner">
  <div class="face front">…question…</div>
  <div class="face back" inert>…answer…</div>
</div></div>
```

```css
.card { perspective: 900px; }
.card-inner { display: grid; transform-style: preserve-3d; transition: transform var(--dur-3) var(--ease-out); }
.face { grid-area: 1 / 1; backface-visibility: hidden; -webkit-backface-visibility: hidden; }
.back { transform: rotateY(180deg); }
.card.is-flipped .card-inner { transform: rotateY(180deg); }
```

```js
function flip(card, on = !card.classList.contains('is-flipped')) {
  card.classList.toggle('is-flipped', on);
  card.querySelector('.front').inert = on;
  card.querySelector('.back').inert = !on;              // hidden face leaves the tab order and the accessibility tree
}
```

With reduced motion, the duration token is zero, so the card swaps instantly. Announce the revealed answer through the live region.

## Variable-font animation

Font weight or width responds to hover, pointer proximity or scroll. Prefer axes that don't change text width (`GRAD` in Roboto Flex, for example) so nearby layout doesn't reflow. Otherwise, reserve the width.

```css
.vf { font-variation-settings: 'wght' var(--w, 400); transition: font-variation-settings var(--dur-2) var(--ease-out); }
.vf:hover, .vf:focus-visible { --w: 800; }
```

Proximity effect: split letters (as in the text reveal) and set `--w` per letter from its distance to the pointer, inside the same rAF loop as the cursor. Use only fine pointers, not reduced motion, and only display headings. Load one variable font file and subset it to the scripts you need.

## Infinite marquee

```html
<div class="marquee" role="region" aria-label="Highlights">
  <div class="marquee-track">
    <ul class="marquee-items">…</ul>
    <ul class="marquee-items" aria-hidden="true">…same items…</ul>
  </div>
  <button class="marquee-toggle" aria-pressed="false">Pause</button>
</div>
```

```css
.marquee { overflow: hidden; }
.marquee-track { display: flex; width: max-content; animation: marquee 30s linear infinite; }
.marquee-items { display: flex; gap: 3rem; padding-inline-end: 3rem; }
@keyframes marquee { to { transform: translateX(-50%); } }
[dir="rtl"] .marquee-track { animation-direction: reverse; }
.marquee:hover .marquee-track, .marquee:focus-within .marquee-track, .marquee.is-paused .marquee-track { animation-play-state: paused; }
@media (prefers-reduced-motion: reduce) { .marquee-track { animation: none; flex-wrap: wrap; width: auto; } .marquee-items[aria-hidden] { display: none; } }
```

WCAG 2.2.2 requires a visible pause control for anything that moves for more than 5 seconds. The duplicate list is `aria-hidden`, and links inside it get `tabindex="-1"`.
