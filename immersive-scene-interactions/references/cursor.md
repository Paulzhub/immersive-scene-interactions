# Cursor effects

Covers the contextual cursor with sticky behaviour, the inverted (blend-mode) cursor and the spotlight/mask reveal. All three share one rAF loop and one pointer position.

## Contents
- Rules
- Markup and CSS (dot + label as separate elements)
- Contextual + sticky cursor loop
- Inverted cursor (why the structure matters)
- Spotlight / mask reveal

## Rules
- Only use these on fine pointers: `(hover: hover) and (pointer: fine)`. Touch devices get no custom cursor.
- Hide the native cursor only through a class on `<html>`, and never over text inputs, text areas, or selectable or editable text.
- The custom cursor is decorative (`aria-hidden="true"`). Every contextual label ("Drag 360°", "Open tip") must also be available another way: a visible hint, an `aria-label`, or `aria-describedby`.
- Write `transform` only, once per frame. Size and shape change only on enter and leave, and CSS transitions animate them.

## Markup and CSS

The dot and the label are **separate direct children of `<body>`**. The blend mode only inverts the page if the blended element isn't inside another stacking context (see the inverted cursor section below). A label inside the inverted dot would be inverted too, and become unreadable.

```html
<div class="cursor-dot" aria-hidden="true"></div>
<div class="cursor-label" aria-hidden="true"></div>
```

```css
html.has-cursor, html.has-cursor a, html.has-cursor button { cursor: none; }
html.has-cursor input, html.has-cursor textarea, html.has-cursor [contenteditable], html.has-cursor .selectable { cursor: auto; }

.cursor-dot, .cursor-label { position: fixed; top: 0; left: 0; pointer-events: none; z-index: var(--z-cursor);
  will-change: transform; opacity: 0; transition: opacity var(--dur-2); }
html.has-cursor.cursor-visible .cursor-dot, html.has-cursor.cursor-visible .cursor-label.on { opacity: 1; }

.cursor-dot { width: 40px; height: 40px; border-radius: 50%; background: #fff; mix-blend-mode: difference;
  transition: width var(--dur-2) var(--ease-out), height var(--dur-2) var(--ease-out),
              border-radius var(--dur-2) var(--ease-out), opacity var(--dur-2); }
.cursor-label { padding: .35em .7em; border-radius: 999px; font-size: .75rem; letter-spacing: .04em;
  background: var(--text); color: var(--bg); white-space: nowrap; }

:root[data-theme="contrast"] .cursor-dot { mix-blend-mode: normal; background: transparent; border: 3px solid var(--accent); }
@media (forced-colors: active) { .cursor-dot, .cursor-label { display: none; } html.has-cursor * { cursor: auto !important; } }
```

## Contextual + sticky cursor loop

```js
const dot = document.querySelector('.cursor-dot'), label = document.querySelector('.cursor-label');
const root = document.documentElement;
const BASE = 40, LABEL_SIZE = 96, PAD = 16, FOLLOW = .18, PULL = .15;
let mx = innerWidth / 2, my = innerHeight / 2, x = mx, y = my, w = BASE, h = BASE;
let stuck = null, box = null, raf = 0;

function setShape(width, height, radius) {
  w = width; h = height;
  dot.style.width = width + 'px'; dot.style.height = height + 'px'; dot.style.borderRadius = radius;
}
const resetSize = () => setShape(BASE, BASE, '50%');

function measure(el) {
  const r = el.getBoundingClientRect(), cs = getComputedStyle(el);
  box = { cx: r.left + r.width / 2, cy: r.top + r.height / 2 };
  setShape(r.width + PAD, r.height + PAD, cs.borderRadius === '0px' ? '12px' : cs.borderRadius);
}

function enable(on) {
  root.classList.toggle('has-cursor', on);
  if (on && !raf) raf = requestAnimationFrame(loop);
  if (!on) { cancelAnimationFrame(raf); raf = 0; }
}
const mqFine = matchMedia('(hover: hover) and (pointer: fine)');
mqFine.addEventListener('change', () => enable(mqFine.matches));
enable(mqFine.matches);

addEventListener('pointermove', e => {
  if (e.pointerType !== 'mouse') return;
  mx = e.clientX; my = e.clientY;
  root.classList.add('cursor-visible');
}, { passive: true });
document.addEventListener('pointerleave', () => root.classList.remove('cursor-visible'));
addEventListener('scroll', () => stuck && measure(stuck), { passive: true });

// sticky: wrap around buttons and other [data-sticky] targets
document.addEventListener('pointerover', e => {
  const el = e.target.closest('[data-sticky]');
  if (el && el !== stuck) { stuck = el; measure(el); }
  const ctx = e.target.closest('[data-cursor]');
  if (ctx) { label.textContent = ctx.dataset.cursor; label.classList.add('on'); if (!stuck) setShape(LABEL_SIZE, LABEL_SIZE, '50%'); }
});
document.addEventListener('pointerout', e => {
  const to = e.relatedTarget;
  if (stuck && !stuck.contains(to)) { stuck = null; box = null; resetSize(); }
  const ctx = e.target.closest('[data-cursor]');
  if (ctx && !ctx.contains(to)) { label.classList.remove('on'); if (!stuck) resetSize(); }
});

function loop() {
  let tx = mx, ty = my;
  if (box) {                                   // centred on the target, with a slight pull toward the pointer
    tx = box.cx + (mx - box.cx) * PULL;
    ty = box.cy + (my - box.cy) * PULL;
  }
  const k = motion.reduced ? 1 : FOLLOW;       // reduced motion: no trailing lag
  x += (tx - x) * k; y += (ty - y) * k;
  dot.style.transform = `translate3d(${x - w / 2}px, ${y - h / 2}px, 0)`;
  label.style.transform = `translate3d(${x + 14}px, ${y + 14}px, 0)`;
  raf = requestAnimationFrame(loop);
}
```

Usage:
- `data-cursor="Drag 360°"` on the 3D canvas, `data-cursor="Open tip"` on hotspots, `data-cursor="Play"` on a video poster.
- `data-sticky` on buttons. It combines with magnetic buttons: put `data-sticky` on the **wrapper** (`.magnet`), because that element doesn't move and its measurement stays valid.
- While a `[data-cursor]` target is also sticky, the label still shows and the dot keeps the target's shape.

## Inverted cursor (blend mode)

A white dot with `mix-blend-mode: difference` shows whatever is under it in the opposite colour.

Why the structure matters: `mix-blend-mode` blends only with content in the same stacking context. If any ancestor creates a stacking context (through `transform`, `filter`, `opacity < 1`, `isolation: isolate`, `will-change`, or a positioned element with a `z-index`), the dot blends only with that ancestor's contents and the page behind stays un-inverted. That's why the dot is a direct child of `<body>`, carrying its own transform.

- Over the WebGL canvas: it works when the canvas sits in the root stacking context (`position: fixed; z-index: var(--z-canvas)` directly under `<body>`). Test it in Chrome, Safari and Firefox.
- Over glass panels: the backdrop blur and the difference blend combine correctly. Check that text stays legible where they meet.
- Light theme: white with difference still inverts. For a softer look, use `mix-blend-mode: exclusion`.
- High-contrast theme and forced colours: the dot becomes an outline ring or is hidden (see the CSS above).

## Spotlight / mask reveal

A circle that follows the cursor and reveals a hidden layer: an X-ray view, a "before" image, or an annotated diagram.

```html
<div class="spot">
  <img src="base.webp" alt="Engine, exterior view">
  <img class="spot-reveal" src="xray.webp" alt="" aria-hidden="true">
</div>
```

```css
.spot { position: relative; --x: 50%; --y: 50%; --r: 0px; }
.spot-reveal { position: absolute; inset: 0;
  -webkit-mask-image: radial-gradient(circle var(--r) at var(--x) var(--y), #000 98%, transparent 100%);
          mask-image: radial-gradient(circle var(--r) at var(--x) var(--y), #000 98%, transparent 100%);
  transition: --r var(--dur-3) var(--ease-out); }
.spot:hover { --r: 120px; }
@property --r { syntax: '<length>'; inherits: true; initial-value: 0px; }
```

```js
spot.addEventListener('pointermove', e => {
  const r = spot.getBoundingClientRect();
  spot.style.setProperty('--x', `${e.clientX - r.left}px`);
  spot.style.setProperty('--y', `${e.clientY - r.top}px`);
}, { passive: true });
```

Touch and keyboard users need a different way in: a "Show X-ray" toggle button that reveals the full layer (`--r: 150vmax`). Hidden information must never be reachable only through the spotlight. If the revealed layer carries meaning, give it a text description.
