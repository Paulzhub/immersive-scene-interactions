# 360° tour player

A timeline-driven 3D tour with captions, chapters and voiceover. A single clock (`store.time`) is the source of truth; the scene, captions, camera, voice, rail and URL all derive from it.

Covers the timeline clock, WebVTT captions and chapters, playback controls, orbit and zoom, the speed menu, full screen, the mini player and Document Picture-in-Picture, the Media Session API, Screen Wake Lock, caption styling options and the transcript panel. Voice, sound effects and spatial audio are in `audio.md`.

## Contents
- Scene and timeline data
- Timeline clock
- WebVTT captions and chapters
- Playback controls (play/pause, restart, seek bar, time display, chapters)
- 360° orbit and zoom
- Speed menu
- Full screen with fallback
- Mini player and Document Picture-in-Picture
- Media Session (lock-screen and headphone controls)
- Screen Wake Lock
- Caption styling options
- Transcript panel

## Scene and timeline data

Build the timeline from the narration durations, so voice and scenes never drift apart. Each language has its own timeline, because Hindi and English narration differ in length.

```js
// scenes.js — durations come from the pre-generated audio manifest (audio.md)
export const scenes = [
  { id: 'intro', title: { en: 'Introduction', hi: 'परिचय' }, minDuration: 6,
    camera: { position: [0, 2, 9], target: [0, 0, 0] }, particles: { color: '#ff8a5c', speed: 1 } },
  // …one entry per scene
];

export function buildTimeline(lang, manifest) {
  let t = 0;
  return scenes.map(s => {
    const voice = manifest[lang][s.id];                     // { src, duration }
    const duration = Math.max(s.minDuration, (voice?.duration ?? 0) + .6);   // breathing room after the line
    const seg = { ...s, start: t, end: t + duration, voice };
    t += duration;
    return seg;
  });
}
```

When the language changes, keep the position: find the current scene and the fraction through it, rebuild the timeline, then seek to `newScene.start + fraction × newScene.duration`.

## Timeline clock

```js
let timeline = buildTimeline(store.get().lang, manifest);
const total = () => timeline.at(-1).end;
let last = 0;

function tick(now) {
  const dt = Math.min((now - last) / 1000, .1); last = now;  // clamp after tab switches or long frames
  const s = store.get();
  if (s.playing) {
    const time = Math.min(s.time + dt * s.speed, total());
    store.set({ time, playing: time < total() });
  }
  requestAnimationFrame(tick);
}
requestAnimationFrame(t => { last = t; tick(t); });

const sceneAt = t => Math.max(0, timeline.findIndex(seg => t < seg.end)) ;

store.subscribe((s, prev) => {
  if (s.time === prev.time) return;
  const i = s.time >= total() ? timeline.length - 1 : sceneAt(s.time);
  const seg = timeline[i];
  const sceneProgress = Math.min(1, (s.time - seg.start) / (seg.end - seg.start));
  if (i !== s.scene) goToScene(i, { fromClock: true });       // fires scene:mid → camera, captions, voice
  else store.set({ sceneProgress });
  placeCamera(i + easeCameraMove(sceneProgress));             // three.md spline (move happens early in the scene)
});
```

Rules:
- `goToScene(n)` from a chapter button, the rail or a key seeks the clock to `timeline[n].start`. The clock then drives everything else. When the clock itself crosses a boundary, it calls `goToScene(i, { fromClock: true })`, which swaps without seeking again.
- Seeking while dragging the seek bar updates visuals and captions live. The voice only restarts when the drag is released (`audio.md`).
- When playback ends: `playing: false`, show "Replay", and announce "Tour finished".
- Reduced motion: no autoplay. The tour starts paused on scene 1, and the camera cuts between presets instead of flying.

## WebVTT captions and chapters

Store captions and chapters as standard WebVTT files. The captions, the transcript and the chapter buttons all read the same parsed data, and the files work with other tools too.

```
WEBVTT

intro-1
00:00:00.500 --> 00:00:04.200
Welcome to the engine tour.
```

```js
const toSec = t => t.split(':').reverse().reduce((a, v, i) => a + parseFloat(v) * 60 ** i, 0);
export function parseVTT(text) {
  return text.replace(/\r/g, '').split(/\n{2,}/).map(b => b.split('\n')).flatMap(lines => {
    const i = lines.findIndex(l => l.includes('-->'));
    if (i < 0) return [];
    const [a, b] = lines[i].split('-->').map(x => toSec(x.trim().split(/\s+/)[0]));
    return [{ id: i > 0 ? lines[0].trim() : null, start: a, end: b, text: lines.slice(i + 1).join('\n') }];
  });
}
const cues = { en: parseVTT(await (await fetch('/captions/en.vtt')).text()) };   // load hi lazily on switch
```

The caption region updates only when the active cue changes, never on every frame:

```html
<div class="captions" aria-live="polite" aria-atomic="true" lang="en"></div>
```

```js
let activeCue = null;
store.subscribe(s => {
  const cue = s.captions.on ? cues[s.lang]?.find(c => s.time >= c.start && s.time < c.end) ?? null : null;
  if (cue === activeCue) return;
  activeCue = cue;
  captionsEl.textContent = cue?.text ?? '';
  captionsEl.lang = s.lang;                                     // correct pronunciation for Hindi text
});
```

If the voiceover is playing, consider `aria-live="off"` on captions, so screen readers don't read the narration aloud at the same time as the audio. The transcript (below) remains the full text alternative.

Chapters use a `chapters.vtt` file with the same parser; each cue's text is the chapter title.

## Playback controls

```html
<div class="player-controls glass" role="group" aria-label="Tour controls">
  <button class="play" aria-label="Play">…</button>
  <button class="restart" aria-label="Restart">…</button>
  <input class="seek" type="range" min="0" step="0.1" aria-label="Seek" aria-valuetext="0:00 of 3:20">
  <span class="time"><span class="cur">0:00</span> / <span class="dur">3:20</span></span>
  <select class="speed" aria-label="Playback speed">…</select>
  <button class="cc" aria-pressed="true">Captions</button>
  <button class="mini" aria-label="Pop out player">…</button>
  <button class="fs" aria-label="Full screen">…</button>
</div>
<ol class="chapters" aria-label="Chapters">
  <li><button data-scene="0" aria-current="step">1. Introduction</button></li>
  …
</ol>
```

```js
const fmt = t => `${Math.floor(t / 60)}:${String(Math.floor(t % 60)).padStart(2, '0')}`;
let dragging = false;

play.addEventListener('click', () => player.toggle());
restart.addEventListener('click', () => { store.set({ time: 0 }); player.play(); });
seek.addEventListener('input', () => { dragging = true; store.set({ time: +seek.value }); });
seek.addEventListener('change', () => { dragging = false; bus.emit('seek:end'); });
chapters.addEventListener('click', e => { const b = e.target.closest('[data-scene]'); if (b) player.seekTo(timeline[+b.dataset.scene].start); });

store.subscribe(s => {
  play.setAttribute('aria-label', s.playing ? 'Pause' : 'Play');
  if (!dragging) seek.value = s.time;
  seek.max = total();
  seek.setAttribute('aria-valuetext', `${fmt(s.time)} of ${fmt(total())}`);
  cur.textContent = fmt(s.time);
});
```

`player` exposes `play()`, `pause()`, `toggle()`, `seekTo(t)`, `seekBy(dt)` and `toggleFullscreen()`. The first `play()` also unlocks audio (`audio.md`). All controls are at least 44×44px. On phones, the controls wrap into two rows and the chapter list becomes a horizontally scrollable strip.

## 360° orbit and zoom

```js
const controls = new OrbitControls(camera, tourCanvas);
Object.assign(controls, { enableDamping: true, dampingFactor: .08, enablePan: false,
  minDistance: 3, maxDistance: 18, minPolarAngle: .2, maxPolarAngle: Math.PI / 2 + .1 });

const zoomBy = f => {
  const d = camera.position.distanceTo(controls.target);
  const next = THREE.MathUtils.clamp(d * f, controls.minDistance, controls.maxDistance);
  gsap.to(camera.position, { ...camera.position.clone().sub(controls.target).setLength(next).add(controls.target), duration: motion.reduced ? 0 : .4 });
};
zoomIn.onclick = () => zoomBy(.8);
zoomOut.onclick = () => zoomBy(1.25);
reset.onclick = () => resetToScenePreset(store.get().scene);

tourCanvas.addEventListener('keydown', e => {                  // canvas has tabindex="0" and data-owns-keys
  const R = .08;
  const map = { ArrowLeft: () => rotate(-R, 0), ArrowRight: () => rotate(R, 0), ArrowUp: () => rotate(0, -R), ArrowDown: () => rotate(0, R),
                '+': () => zoomBy(.8), '=': () => zoomBy(.8), '-': () => zoomBy(1.25), '0': () => reset.onclick() };
  if (map[e.key]) { map[e.key](); e.preventDefault(); }
});
```

`rotate(dTheta, dPhi)` adjusts the spherical coordinates of the camera around `controls.target` and clamps them to the same polar limits.

- The zoom in, zoom out and reset buttons are stacked on the left edge of the scene, inside a glass group, each 44px with an `aria-label`. In RTL layouts they move to the right edge.
- The canvas has `role="application"`, `aria-label="3D view of the engine"`, `aria-describedby` pointing to visible help ("Drag or use arrow keys to rotate; scroll, pinch or +/− to zoom; 0 resets"), `touch-action: none` and `data-cursor="Drag 360°"`.
- User orbiting never stops the timeline. The offset decays on the next scene change (the spline section in `three.md`).

## Speed menu

A native `<select>` gives the correct semantics, keyboard support and the mobile picker at no cost.

```js
const SPEEDS = [0.25, 0.5, 0.75, 1, 1.25, 1.5, 1.75, 2];
speed.innerHTML = SPEEDS.map(v => `<option value="${v}"${v === 1 ? ' selected' : ''}>${v}×</option>`).join('');
speed.addEventListener('change', () => store.set({ speed: +speed.value }));   // clock, voice playbackRate (audio.md)
```

If the design needs a custom-styled menu, use a menu button that opens a `role="listbox"` with `role="option"` items: `aria-haspopup="listbox"`, `aria-expanded`, ↑/↓, Home and End, Enter to select, and Esc to close and return focus. Don't mix that with the `menu` pattern.

## Full screen with fallback

```js
const playerEl = document.querySelector('.player');
const fsLayer = { close: () => exitPseudo() };

export async function enterFullscreen() {
  const req = playerEl.requestFullscreen ?? playerEl.webkitRequestFullscreen;
  try {
    if (!req || !document.fullscreenEnabled && !document.webkitFullscreenEnabled) throw new Error('unsupported');
    await req.call(playerEl);                     // rejects inside restricted iframes; older Safari returns undefined
    store.set({ player: 'fullscreen' });
  } catch {
    enterPseudo();
  }
}
function enterPseudo() {
  playerEl.classList.add('is-pseudo-fs');
  document.documentElement.classList.add('no-scroll');
  layers.push(fsLayer);                           // Esc exits (architecture.md)
  store.set({ player: 'pseudo-fullscreen' });
}
function exitPseudo() {
  playerEl.classList.remove('is-pseudo-fs');
  document.documentElement.classList.remove('no-scroll');
  layers.remove(fsLayer);
  store.set({ player: 'inline' });
}
export function exitFullscreen() {
  if (document.fullscreenElement || document.webkitFullscreenElement) (document.exitFullscreen ?? document.webkitExitFullscreen).call(document);
  else exitPseudo();
}
['fullscreenchange', 'webkitfullscreenchange'].forEach(t => document.addEventListener(t, () => {
  const on = !!(document.fullscreenElement || document.webkitFullscreenElement);
  if (!on && store.get().player === 'fullscreen') store.set({ player: 'inline' });
  fsButton.setAttribute('aria-label', on ? 'Exit full screen' : 'Full screen');
  dispatchEvent(new Event('resize'));             // renderer resizes
}));
```

```css
.player.is-pseudo-fs { position: fixed; inset: 0; z-index: var(--z-modal); width: 100vw; height: 100dvh;
  padding: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left); background: var(--bg); }
html.no-scroll { overflow: hidden; }
```

On iPhone, Safari only allows full screen for `<video>` elements, so the pseudo-fullscreen fallback is what iPhone users get.

## Mini player and Document Picture-in-Picture

The player pops out as a small window. Where Document Picture-in-Picture is supported (Chromium desktop), it becomes a real floating window that stays on top of other apps. Elsewhere it becomes a draggable in-page mini player. "Bring it back" or Esc returns it to the page.

```js
const placeholder = document.createElement('div');

export async function popOut() {
  placeholder.style.height = playerEl.offsetHeight + 'px';       // no layout jump
  playerEl.before(placeholder);
  if ('documentPictureInPicture' in window && store.get().pipPreferred !== false) {
    try { return await openPiP(); } catch { /* fall back to in-page */ }
  }
  enterMini();
}

async function openPiP() {
  const pip = await documentPictureInPicture.requestWindow({ width: 400, height: 260 });   // must follow a user gesture
  for (const sheet of document.styleSheets) {
    try { const st = pip.document.createElement('style'); st.textContent = [...sheet.cssRules].map(r => r.cssText).join('\n'); pip.document.head.append(st); }
    catch { const l = pip.document.createElement('link'); l.rel = 'stylesheet'; l.href = sheet.href; pip.document.head.append(l); }
  }
  pip.document.documentElement.dataset.theme = document.documentElement.dataset.theme;
  pip.document.body.append(playerEl);
  pip.addEventListener('keydown', onGlobalKeydown);              // window listeners in the main page don't receive PiP keys
  pip.addEventListener('resize', () => dispatchEvent(new Event('resize')));
  pip.addEventListener('pagehide', bringBack, { once: true });
  store.set({ player: 'pip' });
}
```

Test that the WebGL canvas survives the move into the PiP document in your target browsers. If the context is lost (`webglcontextlost`), recreate the renderer in the new document, or show the controls and captions in the PiP window with the 3D view paused.

In-page mini player:

```css
.player.is-mini { position: fixed; width: min(320px, 90vw); inset-inline-end: max(16px, env(safe-area-inset-right));
  bottom: max(16px, env(safe-area-inset-bottom)); z-index: var(--z-mini); border-radius: 12px; overflow: hidden;
  box-shadow: 0 12px 40px rgb(0 0 0 / .35); }
```

```js
const miniLayer = { close: () => bringBack() };
let pos = null;

function enterMini() {
  playerEl.classList.add('is-mini');
  layers.push(miniLayer);
  store.set({ player: 'mini' });
  bus.emit('quality:hint', 'reduce');                            // pause background particles, lower pixel ratio
}

const handle = playerEl.querySelector('.mini-handle');           // <div role="button" tabindex="0" aria-label="Move mini player. Use arrow keys." data-owns-keys>
function moveTo(x, y) {
  const r = playerEl.getBoundingClientRect(), m = 16;
  x = Math.max(m, Math.min(innerWidth - r.width - m, x));
  y = Math.max(m, Math.min(innerHeight - r.height - m, y));
  pos = { x, y };
  Object.assign(playerEl.style, { left: x + 'px', top: y + 'px', right: 'auto', bottom: 'auto' });
}
handle.addEventListener('pointerdown', e => {
  const r = playerEl.getBoundingClientRect(), ox = e.clientX - r.left, oy = e.clientY - r.top;
  handle.setPointerCapture(e.pointerId);
  const move = ev => moveTo(ev.clientX - ox, ev.clientY - oy);
  handle.addEventListener('pointermove', move);
  handle.addEventListener('pointerup', () => handle.removeEventListener('pointermove', move), { once: true });
});
handle.addEventListener('keydown', e => {
  const step = e.shiftKey ? 48 : 16, r = playerEl.getBoundingClientRect();
  const d = { ArrowLeft: [-step, 0], ArrowRight: [step, 0], ArrowUp: [0, -step], ArrowDown: [0, step] }[e.key];
  if (d) { moveTo(r.left + d[0], r.top + d[1]); e.preventDefault(); }
});
addEventListener('resize', () => pos && moveTo(pos.x, pos.y));

export function bringBack() {
  if (store.get().player === 'pip') documentPictureInPicture.window?.close();
  playerEl.classList.remove('is-mini');
  playerEl.removeAttribute('style');
  placeholder.replaceWith(playerEl);
  layers.remove(miniLayer);
  store.set({ player: 'inline' });
  bus.emit('quality:hint', 'restore');
  popOutButton.focus();                                          // return focus to the control that opened it
}
```

`pagehide` fires when the PiP window is closed, so `bringBack` must cope with a window that's already closed. The "Bring it back" button inside the mini player has the visible text "Bring it back".

## Media Session

Lock-screen, notification and headphone controls for the narration. This only works while an `<audio>` element is actually playing, which is how the voiceover plays (`audio.md`).

```js
if ('mediaSession' in navigator) {
  const ms = navigator.mediaSession;
  const handlers = {
    play: () => player.play(), pause: () => player.pause(),
    previoustrack: () => goToScene(store.get().scene - 1), nexttrack: () => goToScene(store.get().scene + 1),
    seekbackward: d => player.seekBy(-(d.seekOffset || 10)), seekforward: d => player.seekBy(d.seekOffset || 10),
    seekto: d => player.seekTo(d.seekTime),
  };
  for (const [action, fn] of Object.entries(handlers)) { try { ms.setActionHandler(action, fn); } catch {} }

  store.subscribe((s, p) => {
    if (s.scene !== p.scene || s.lang !== p.lang)
      ms.metadata = new MediaMetadata({ title: timeline[s.scene].title[s.lang], artist: SITE_NAME, album: 'Interactive tour',
        artwork: [{ src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png' }] });
    ms.playbackState = s.playing ? 'playing' : 'paused';
    if (Math.abs(s.time - (p.time ?? 0)) > 1 || s.speed !== p.speed || s.playing !== p.playing)   // throttle updates
      try { ms.setPositionState({ duration: total(), playbackRate: s.speed, position: Math.min(s.time, total()) }); } catch {}
  });
}
```

## Screen Wake Lock

Keeps the screen awake while the tour plays.

```js
let lock = null;
async function keepAwake(on) {
  try {
    if (on && !lock && 'wakeLock' in navigator) {
      lock = await navigator.wakeLock.request('screen');
      lock.addEventListener('release', () => { lock = null; });
    } else if (!on && lock) { await lock.release(); lock = null; }
  } catch { /* denied (battery saver, permissions policy): silently continue */ }
}
store.subscribe((s, p) => { if (s.playing !== p.playing) keepAwake(s.playing); });
document.addEventListener('visibilitychange', () => {           // the browser releases the lock when the tab is hidden
  if (document.visibilityState === 'visible' && store.get().playing) keepAwake(true);
});
```

## Caption styling options

A settings panel for text size (S, M, L, XL), contrast (default, high, solid background), background opacity, position (bottom, top, or above the mini player) and font (default or a dyslexia-friendly option). Settings are applied as CSS variables on the player, persisted, previewed live, and respected in full screen, the mini player and PiP.

```css
.captions { font-size: var(--cap-size, 1.25rem); font-family: var(--cap-font, inherit);
  color: var(--cap-fg, #fff); background: rgb(0 0 0 / var(--cap-bg, .6));
  padding: .25em .6em; border-radius: 6px; max-width: min(60ch, 90%); margin-inline: auto; text-wrap: balance; }
.player[data-cap-pos="top"] .captions { top: 1rem; bottom: auto; }
.player[data-cap-contrast="high"] { --cap-fg: #ffd166; --cap-bg: .9; }
.player[data-cap-contrast="solid"] { --cap-bg: 1; }
```

```js
const SIZES = { S: '1rem', M: '1.25rem', L: '1.6rem', XL: '2rem' };
store.subscribe(({ captions: c }) => {
  playerEl.style.setProperty('--cap-size', SIZES[c.size]);
  playerEl.style.setProperty('--cap-bg', c.bgOpacity ?? .6);
  playerEl.style.setProperty('--cap-font', c.font === 'readable' ? '"Atkinson Hyperlegible", system-ui, sans-serif' : 'inherit');
  playerEl.dataset.capPos = c.position; playerEl.dataset.capContrast = c.contrast;
});
```

Captions always meet at least 4.5:1 against their own background, whatever the scene behind them.

## Transcript panel

A full, searchable text version of the narration in the current language. It's the main fallback when the voice or 3D view fails.

```html
<aside class="transcript glass" aria-labelledby="tr-h">
  <h2 id="tr-h">Transcript</h2>
  <label>Search transcript <input type="search" class="tr-search"></label>
  <p class="tr-count" role="status"></p>
  <ol class="tr-list"></ol>
  <button class="tr-copy">Copy</button> <button class="tr-download">Download .txt</button>
</aside>
```

```js
function renderTranscript(lang) {
  list.lang = lang;
  list.innerHTML = '';
  for (const c of cues[lang]) {
    const li = document.createElement('li');
    const b = Object.assign(document.createElement('button'), { className: 'tr-line' });
    b.dataset.start = c.start;
    b.innerHTML = `<time>${fmt(c.start)}</time> `;
    b.append(c.text);                                          // text node: never inject caption text as HTML
    b.addEventListener('click', () => player.seekTo(c.start));
    li.append(b); list.append(li); c.el = b;
  }
}
let userScrolledAt = 0;
list.addEventListener('wheel', () => { userScrolledAt = Date.now(); }, { passive: true });
store.subscribe(s => {
  const c = cues[s.lang]?.find(c => s.time >= c.start && s.time < c.end);
  list.querySelector('[aria-current]')?.removeAttribute('aria-current');
  if (!c?.el) return;
  c.el.setAttribute('aria-current', 'true');
  if (Date.now() - userScrolledAt > 4000) c.el.scrollIntoView({ block: 'nearest', behavior: motion.reduced ? 'auto' : 'smooth' });
});
search.addEventListener('input', () => {
  const q = search.value.trim().toLocaleLowerCase();
  let n = 0;
  for (const c of cues[store.get().lang]) { const hit = !q || c.text.toLocaleLowerCase().includes(q); c.el.parentElement.hidden = !hit; n += hit; }
  count.textContent = q ? `${n} matching line${n === 1 ? '' : 's'}` : '';
});
```

On desktop the transcript is a collapsible side panel; on phones it's a bottom sheet. It works without WebGL or audio, opens from the command palette, prints cleanly (`accessibility.md`), and highlights search matches with `<mark>` if you render matching text in pieces.
