# Scroll-driven motion

Covers controlled parallax, CSS scroll-driven animations, Lenis smooth scrolling, SVG line draw, pinned horizontal sections, sticky scrollytelling and image-sequence scrubbing.

Choose the navigation model first (`architecture.md`). In **step** mode, "scroll progress" below means `store.sceneProgress` and the scene index, not `scrollY`. Lenis, pinned sections and image sequences are for **scroll** or **hybrid** pages only.

## Contents
- Controlled parallax (pointer tilt + scroll drift)
- CSS scroll-driven animations (no JavaScript)
- Lenis smooth scroll + ScrollTrigger
- Scroll-triggered SVG line draw
- Pinned horizontal section
- Sticky scrollytelling (split screen)
- Image-sequence scrubbing

## Controlled parallax

Panels tilt with the pointer and drift against the scroll, within fixed limits. The loop only runs while it's needed, and it stops for reduced motion or a hidden tab.

```js
const panels = [...document.querySelectorAll('[data-parallax]')];
const TILT = 6, DRIFT = 60, LERP = .08, EPS = .001;
let tx = 0, ty = 0, cx = 0, cy = 0, sp = 0, cs = 0, raf = 0;

addEventListener('pointermove', e => {
  if (!pointer.fine) return;
  tx = (e.clientX / innerWidth - .5) * 2;
  ty = (e.clientY / innerHeight - .5) * 2;
  start();
}, { passive: true });

function readProgress() {
  if (store.get().mode === 'step') {
    const s = store.get();
    sp = scenes.length > 1 ? (s.scene + s.sceneProgress) / (scenes.length - 1) : 0;
  } else {
    const max = document.documentElement.scrollHeight - innerHeight;
    sp = max > 0 ? scrollY / max : 0;                        // no divide-by-zero on short pages
  }
  start();
}
addEventListener('scroll', readProgress, { passive: true });
addEventListener('resize', readProgress, { passive: true });
store.subscribe(readProgress);

function frame() {
  cx += (tx - cx) * LERP; cy += (ty - cy) * LERP; cs += (sp - cs) * LERP;
  panels.forEach((p, i) => {
    const depth = +(p.dataset.depth || 1 + i * .25);
    p.style.transform = `perspective(1000px) rotateY(${cx * TILT}deg) rotateX(${-cy * TILT}deg) translate3d(0, ${-cs * DRIFT * depth}px, 0)`;
  });
  const settled = Math.abs(tx - cx) < EPS && Math.abs(ty - cy) < EPS && Math.abs(sp - cs) < EPS;
  raf = settled ? 0 : requestAnimationFrame(frame);           // idle when nothing is moving
}
function start() { if (!raf && !motion.reduced && !document.hidden) raf = requestAnimationFrame(frame); }
function stop() { cancelAnimationFrame(raf); raf = 0; panels.forEach(p => { p.style.transform = ''; }); }

bus.on('motion:change', reduced => reduced ? stop() : start());
document.addEventListener('visibilitychange', () => document.hidden ? (cancelAnimationFrame(raf), raf = 0) : start());
readProgress();
```

- `[data-parallax]` is the outer wrapper and owns `transform` (see transform ownership in `architecture.md`).
- Clamp `DRIFT × depth` so panels never leave the viewport or cover text. Test at the shortest supported viewport height.
- A gyroscope on mobile can feed `tx` and `ty` within the same limits (`three.md`).

## CSS scroll-driven animations

Reveals and parallax with no JavaScript, running off the main thread. Guard them with `@supports` and keep the JavaScript versions as the fallback.

```css
@supports (animation-timeline: view()) {
  @media (prefers-reduced-motion: no-preference) {
    .reveal {
      animation: reveal linear both;
      animation-timeline: view();
      animation-range: entry 0% cover 30%;
    }
    .drift {
      animation: drift linear both;
      animation-timeline: scroll(root);
    }
  }
}
@keyframes reveal { from { opacity: 0; transform: translateY(24px); } to { opacity: 1; transform: none; } }
@keyframes drift  { to { transform: translateY(calc(var(--depth, 1) * -60px)); } }
```

```js
const nativeScrollAnim = CSS.supports('animation-timeline: view()');
if (!nativeScrollAnim) initJsReveals();                   // IntersectionObserver / ScrollTrigger fallback
```

These animations also own `transform`, so don't combine `.drift` with JavaScript parallax on the same element. A scroll progress bar is a single line: `.progress { animation: grow linear; animation-timeline: scroll(root); transform-origin: left; } @keyframes grow { from { transform: scaleX(0); } }`.

## Lenis smooth scroll + ScrollTrigger

Use only for scroll-mode pages, and only when the design truly needs inertia. Native scrolling is better for accessibility and battery life.

```js
import Lenis from 'lenis';
import 'lenis/dist/lenis.css';

let lenis;
function initLenis() {
  if (motion.reduced || store.get().mode === 'step') return;
  lenis = new Lenis({ autoRaf: false, anchors: true });     // touch stays native by default (syncTouch: false)
  lenis.on('scroll', ScrollTrigger.update);
  gsap.ticker.add(t => lenis.raf(t * 1000));
  gsap.ticker.lagSmoothing(0);
}
bus.on('motion:change', r => { if (r) { lenis?.destroy(); lenis = null; } else initLenis(); });
initLenis();
```

- Stop Lenis while a modal, command palette or mini-player drag is active (`lenis.stop()` / `lenis.start()`), and add `data-lenis-prevent` to scrollable panels inside the page.
- Keyboard scrolling, Find in page and anchor links must still work. Test all three.

## Scroll-triggered SVG line draw

```html
<svg viewBox="0 0 800 200" aria-hidden="true"><path class="draw" pathLength="1" d="…"/></svg>
```

```css
.draw { fill: none; stroke: var(--accent); stroke-width: 2; vector-effect: non-scaling-stroke;
        stroke-dasharray: 1; stroke-dashoffset: var(--off, 1); }
@supports (animation-timeline: view()) {
  @media (prefers-reduced-motion: no-preference) {
    .draw { animation: draw linear both; animation-timeline: view(); animation-range: entry 10% cover 60%; }
  }
}
@keyframes draw { from { stroke-dashoffset: 1; } to { stroke-dashoffset: 0; } }
@media (prefers-reduced-motion: reduce) { .draw { --off: 0; } }
```

`pathLength="1"` avoids measuring with `getTotalLength()`. The JavaScript fallback sets `--off` to `1 - progress` from ScrollTrigger (`scrub: true`). In step mode it uses `sceneProgress` instead.

## Pinned horizontal section

```js
const mm = gsap.matchMedia();
mm.add('(min-width: 769px) and (prefers-reduced-motion: no-preference)', () => {
  const section = document.querySelector('.hscroll'), track = section.querySelector('.hscroll-track');
  const distance = () => track.scrollWidth - innerWidth;
  const tween = gsap.to(track, {
    x: () => -distance(), ease: 'none',
    scrollTrigger: { trigger: section, pin: true, scrub: 1, end: () => '+=' + distance(), invalidateOnRefresh: true },
  });
  // keyboard users: focusing an off-screen card scrolls the page to the matching position
  const onFocus = e => {
    const card = e.target.closest('.hscroll-card'); if (!card) return;
    const st = tween.scrollTrigger, p = Math.min(1, card.offsetLeft / distance());
    scrollTo({ top: st.start + (st.end - st.start) * p, behavior: 'instant' });
  };
  section.addEventListener('focusin', onFocus);
  return () => section.removeEventListener('focusin', onFocus);
});
```

Under 769px or with reduced motion, the section is a plain vertical stack (`gsap.matchMedia` reverts automatically). In RTL layouts, invert the direction.

## Sticky scrollytelling (split screen)

A sticky graphic on one side and text steps on the other. The active step drives the graphic.

```css
.story { display: grid; grid-template-columns: 1fr 1fr; gap: 4vw; }
.story-graphic { position: sticky; top: 10vh; height: 80vh; }
.story-step { min-height: 80vh; display: grid; align-content: center; }
@media (max-width: 768px) { .story { grid-template-columns: 1fr; } .story-graphic { top: 0; height: 45vh; z-index: var(--z-content); } }
```

```js
const io = new IntersectionObserver(entries => entries.forEach(e => {
  if (e.isIntersecting) setStep(+e.target.dataset.step);      // update graphic, aria-current, 3D camera preset
}), { rootMargin: '-50% 0px -50% 0px' });                      // fires when a step crosses the middle
document.querySelectorAll('.story-step').forEach(s => io.observe(s));
```

The text steps are the accessible content. The graphic is decorative or has a text equivalent for each step.

## Image-sequence scrubbing

Apple-style canvas frames that play as you scroll. Use it for short, high-impact moments; a scrubbed `<video>` is often lighter.

```js
const COUNT = matchMedia('(max-width: 768px)').matches ? 60 : 120;
const frames = new Array(COUNT);
const canvas = document.querySelector('.seq'), ctx = canvas.getContext('2d');
const src = i => `/seq/${String(i + 1).padStart(3, '0')}.webp`;

// progressive preload: every 8th frame first, then fill the gaps
const order = [...Array(COUNT).keys()].sort((a, b) => (a % 8 !== 0) - (b % 8 !== 0) || a - b);
for (const i of order) { const img = new Image(); img.decoding = 'async'; img.src = src(i); img.onload = () => { frames[i] = img; }; }

function nearest(i) { for (let d = 0; d < COUNT; d++) { if (frames[i - d]) return frames[i - d]; if (frames[i + d]) return frames[i + d]; } }
let current = -1, pending = 0;
function draw(i) {
  if (i === current) return; current = i;
  cancelAnimationFrame(pending);
  pending = requestAnimationFrame(() => {
    const img = nearest(i); if (!img) return;
    const s = Math.max(canvas.width / img.width, canvas.height / img.height);   // cover
    ctx.drawImage(img, (canvas.width - img.width * s) / 2, (canvas.height - img.height * s) / 2, img.width * s, img.height * s);
  });
}
ScrollTrigger.create({ trigger: '.seq-wrap', start: 'top top', end: '+=300%', pin: true, scrub: true,
  onUpdate: st => draw(Math.round(st.progress * (COUNT - 1))) });
```

- Size the canvas to its CSS size × `devicePixelRatio` (capped at 2) and redraw on resize.
- Use AVIF/WebP and keep the total under about 1.5MB on mobile (fewer frames, smaller dimensions).
- With reduced motion, show a single poster image with alt text and no pin.
