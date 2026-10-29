# Architecture

Read this first for any build that uses more than two or three effects. Every other reference assumes the pieces defined here: `store`, `bus`, `motion`, `goToScene`, the navigation model, transform ownership and the keyboard map.

## Contents
- Module layout
- State store and event bus
- Live user preferences (`motion`, `finePointer`)
- Navigation model: step, scroll or hybrid
- `goToScene` — the single scene entry point
- Transform ownership
- Keyboard map and Esc layer stack

## Module layout

```
src/
  config.js        // constants: strengths, limits, durations (reads CSS motion tokens once)
  store.js         // createStore + bus
  prefs.js         // motion / pointer / theme preference listeners
  scenes.js        // scene data: ids, titles, camera presets, caption/voice refs
  navigation.js    // goToScene, input mapping (wheel, swipe, keys, rail)
  keyboard.js      // global shortcuts + Esc layer stack
  fx/              // one file per effect; each exports init(store, bus) → destroy()
  three/           // lazy-loaded 3D (renderer, particles, hotspots, post-processing)
  player/          // timeline clock, captions, audio, controls
```

Each effect module exports `init()` returning a `destroy()` that removes listeners, cancels rAF loops and disposes GPU resources. Effects never talk to each other directly; they read the store and listen on the bus.

## State store and event bus

One store holds everything that more than one module needs. Transient happenings (a transition reaching its midpoint, a theme repaint) go on the bus.

```js
// store.js
export function createStore(initial) {
  let state = initial;
  const subs = new Set();
  return {
    get: () => state,
    set(patch) {
      const prev = state;
      state = { ...state, ...(typeof patch === 'function' ? patch(state) : patch) };
      subs.forEach(fn => fn(state, prev));
    },
    subscribe(fn) { subs.add(fn); return () => subs.delete(fn); },
  };
}

const target = new EventTarget();
export const bus = {
  emit: (type, detail) => target.dispatchEvent(new CustomEvent(type, { detail })),
  on(type, fn) {
    const h = e => fn(e.detail);
    target.addEventListener(type, h);
    return () => target.removeEventListener(type, h);
  },
};

export const store = createStore({
  mode: 'step',            // 'step' | 'scroll' | 'hybrid' (see Navigation model)
  scene: 0, sceneProgress: 0,
  time: 0, playing: false, speed: 1,
  lang: 'en', themeChoice: 'system', theme: 'dark',
  captions: { on: true, size: 'M', contrast: 'default', position: 'bottom' },
  mute: { voice: false, sfx: false }, haptics: true, shortcuts: true,
  tab: 'basics', quality: 'high', player: 'inline', // 'inline' | 'mini' | 'pip' | 'fullscreen' | 'pseudo-fullscreen'
  quiz: { answers: {}, streak: 0, badges: [] },
});
```

Bus events used across references: `scene:before`, `scene:mid` (screen covered — swap content here), `scene:after`, `theme:change`, `motion:change`, `quality:change`, `audio:unlocked`.

Persistence (resume) and the URL (deep links) are plain store subscribers. See `navigation.md` and `quiz.md`.

## Live user preferences

Read preferences as live values. Never cache them in a `const` at startup.

```js
// prefs.js
export const motion = { reduced: false };
export const pointer = { fine: false };

const mqMotion = matchMedia('(prefers-reduced-motion: reduce)');
const mqFine = matchMedia('(hover: hover) and (pointer: fine)');

function sync() {
  const was = motion.reduced;
  motion.reduced = mqMotion.matches || document.documentElement.dataset.motion === 'reduce';
  pointer.fine = mqFine.matches;
  if (was !== motion.reduced) bus.emit('motion:change', motion.reduced);
}
mqMotion.addEventListener('change', sync);
mqFine.addEventListener('change', sync);
sync();
```

Offer an in-page "Reduce motion" switch as well. It sets `data-motion="reduce"` on `<html>`, persists the choice and calls `sync()`. Every animated effect checks `motion.reduced` at the moment it runs and subscribes to `motion:change` to stop running loops.

## Navigation model

Decide this before building anything that responds to scroll. Mixing models causes skipped scenes, fights between wheel-snap and smooth scroll, and parallax driven by the wrong value.

| Model | Source of truth | Use for | Scene changes come from | Parallax/rail driven by |
|---|---|---|---|---|
| **Step** (default for cinematic tours) | `store.scene` + timeline clock | Full-viewport stage, one scene at a time | Buttons, rail dots, keys, swipe, wheel-snap, timeline | `sceneProgress` (0–1 within the current scene) and `scene / (count-1)` |
| **Scroll** | Scroll position | Long-form pages, articles, landing pages | Scrolling; `goToScene(n)` scrolls to the section | Scroll progress |
| **Hybrid** | Page scrolls; an embedded player is step mode inside its own box | A guide page with a tour player and a quiz | Page: scroll. Player: its own controls | Page: scroll. Player: its clock |

Rules:
- Step mode: the stage does not scroll (`overflow: hidden` on the stage; long text inside a panel scrolls within the panel). Wheel-snap and swipe call `goToScene`. Don't use Lenis.
- Scroll mode: never hijack the wheel. An `IntersectionObserver` or ScrollTrigger sets `store.scene` when a section crosses the viewport centre. `goToScene(n)` becomes `sections[n].scrollIntoView({ behavior: motion.reduced ? 'auto' : 'smooth' })` (or `lenis.scrollTo`). The curtain is used only for route or tab changes, not for scrolling.
- Hybrid: the wheel over the player scrolls the page unless the player is focused, fullscreen or in mini mode.

## goToScene

The single entry point for scene changes in step mode. It never drops the user's last request: if a transition is running, it remembers the latest target and runs it afterwards.

```js
// navigation.js
let busy = false, queued = null;

export async function goToScene(n, { instant = false, fromClock = false } = {}) {
  n = Math.max(0, Math.min(scenes.length - 1, n));
  if (busy) { queued = n; return; }
  if (n === store.get().scene) return;
  busy = true;
  bus.emit('scene:before', n);
  try {
    if (instant || motion.reduced) {
      swapScene(n, fromClock);
    } else {
      await transition(n, fromClock); // curtain or View Transition (motion.md); calls swapScene at the covered midpoint
    }
  } catch (err) {
    console.error(err);
    if (store.get().scene !== n) swapScene(n, fromClock);   // never strand the user mid-transition
  } finally {
    busy = false;
    bus.emit('scene:after', n);
    if (queued !== null && queued !== n) { const q = queued; queued = null; goToScene(q); }
    else queued = null;
  }
}

function swapScene(n, fromClock = false) {
  store.set({ scene: n, sceneProgress: 0 });
  if (!fromClock && timeline) store.set({ time: timeline[n].start });   // user navigation seeks the tour clock (tour-player.md)
  bus.emit('scene:mid', n);          // text reveal, ghost numeral, camera, particles, lens breath, captions
  announce(`Scene ${n + 1} of ${scenes.length}: ${scenes[n].title}`); // polite live region (accessibility.md)
}
```

The tour clock calls `goToScene(i, { fromClock: true })` when playback crosses a scene boundary, so it doesn't seek itself. Every other caller (buttons, rail, keys, swipe, deep links) seeks the clock to the scene's start.

Store subscribers run on every clock tick while the tour plays, so keep them cheap: compare against `prev` and return early when nothing they care about changed.

Listen on `scene:mid` for anything that should change while the screen is covered: text reveal, ghost numeral, camera target, particle palette, lens breath and captions. Scenes are announced through the live region, never through the decorative curtain number.

## Transform ownership

One element, one owner of `transform`. Two effects writing `transform` to the same node overwrite each other every frame.

Nest wrappers instead:

```html
<section class="panel-parallax" data-parallax>   <!-- parallax owns transform -->
  <div class="panel-transition">                 <!-- GSAP tab/scene transition owns transform + opacity -->
    <div class="panel" data-reveal>              <!-- reveal owns clip-path only -->
      …
    </div>
  </div>
</section>
```

The same applies to magnetic buttons: the wrapper stays still and is measured, and the inner element moves (see `motion.md`). The cursor dot and label are separate elements, each with its own transform (`cursor.md`).

## Keyboard map

Keys are handled in priority order. The first layer that claims a key handles it and calls `preventDefault()`. The global handler ignores any event that is already handled.

1. **Open layer** (dialog, command palette, menu, hotspot card): Esc closes the topmost layer, Tab is trapped inside modal layers, and arrows move within menus and lists.
2. **Focused component that claims arrows:**

   | Component | Keys |
   |---|---|
   | Tab list | ←/→ move, Home/End |
   | 3D canvas (`tabindex="0"`) | arrows rotate, `+`/`-` zoom, `0` reset |
   | Mini player handle | arrows move 16px (Shift = 48px) |
   | Sliders (seek bar, comparison, exploded view) | arrows step, Shift = big step, Home/End |
   | `<select>`, inputs, text areas, contenteditable | native behaviour |

3. **Global shortcuts.** These only run when focus is on the page body or a non-interactive element:

   | Key | Action |
   |---|---|
   | Space / `K` | play/pause (Space only when focus is not on a button or link, which use Space natively) |
   | ← / → | previous/next scene (Shift: seek ∓5s) |
   | `M` | mute voice · `F` fullscreen · `C` captions · `T` cycle theme |
   | `?` | shortcuts overlay |
   | Ctrl/Cmd + K | command palette (works everywhere, including inputs) |

```js
// keyboard.js
const OWNS_KEYS = 'input, textarea, select, [contenteditable], [role="slider"], [role="tab"], [role="menu"], [role="menuitem"], [role="listbox"], [role="option"], [data-owns-keys]';
const NATIVE_SPACE = 'button, a[href], summary, [role="button"], [role="checkbox"], [role="switch"]';

export const layers = {
  stack: [],
  push(layer) { this.stack.push(layer); },          // layer = { close(): void }
  remove(layer) { this.stack = this.stack.filter(l => l !== layer); },
  closeTop() { const l = this.stack.pop(); l?.close(); return !!l; },
};

addEventListener('keydown', e => {
  if (e.defaultPrevented || e.altKey) return;
  const mod = e.metaKey || e.ctrlKey;
  if (mod && e.key.toLowerCase() === 'k') { e.preventDefault(); palette.open(); return; }
  if (mod) return;
  if (e.key === 'Escape') { if (layers.closeTop()) e.preventDefault(); return; }

  const t = e.target;
  if (t.closest?.(OWNS_KEYS) || t.isContentEditable) return;
  if (!store.get().shortcuts) return;                // WCAG 2.1.4: single-key shortcuts can be turned off

  const s = store.get();
  switch (e.key) {
    case ' ': if (t.closest(NATIVE_SPACE)) return; // fall through
    case 'k': case 'K': player.toggle(); break;
    case 'ArrowLeft':  e.shiftKey ? player.seekBy(-5) : goToScene(s.scene - 1); break;
    case 'ArrowRight': e.shiftKey ? player.seekBy(5)  : goToScene(s.scene + 1); break;
    case 'm': case 'M': store.set({ mute: { ...s.mute, voice: !s.mute.voice } }); break;
    case 'f': case 'F': player.toggleFullscreen(); break;
    case 'c': case 'C': store.set({ captions: { ...s.captions, on: !s.captions.on } }); break;
    case 't': case 'T': cycleTheme(); break;
    case '?': shortcutsOverlay.open(); break;
    default: return;
  }
  e.preventDefault();
});
```

Every layer registers with `layers.push` when it opens and `layers.remove` when it closes some other way, so Esc always closes the right thing. Order from top to bottom: command palette, dialogs, menus, hotspot card, mini player (Esc brings it back), fullscreen (the browser handles Esc for true fullscreen, and the pseudo-fullscreen fallback registers itself as a layer).

Keyboard listeners attached to `window` do not receive events from a Document Picture-in-Picture window. Bind the same handler on `pipWindow` as well (`tour-player.md`).
