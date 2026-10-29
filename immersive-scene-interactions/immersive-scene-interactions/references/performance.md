# Performance, platform and testing

Covers budgets, version pinning, adaptive quality, lazy loading and code-splitting, offline support (PWA), privacy-safe analytics, and a test recipe (Playwright, axe, reduced motion, no WebGL).

## Contents
- Budgets
- Versions
- Adaptive quality
- Lazy loading and code-splitting
- Offline support (PWA)
- Privacy-safe analytics hooks
- Testing recipe

## Budgets

Check these on a mid-range Android phone over a throttled 4G connection, not just on a development laptop.

| Metric | Budget |
|---|---|
| Initial JS before 3D (gzip) | ≤ 200 KB |
| 3D chunk (three.js + scene code, gzip) | ≤ 250 KB, loaded lazily |
| Largest Contentful Paint | < 2.5 s |
| Interaction to Next Paint | < 200 ms |
| Cumulative Layout Shift | < 0.1 |
| Frame rate during scene changes | 60 fps desktop, ≥ 45 fps mid-range phone |
| First model (placeholder) | ≤ 150 KB; full model ≤ 3 MB compressed |
| Voice clip per scene | ≤ 150 KB (Opus, 48 kbps) |
| Image sequence (mobile, total) | ≤ 1.5 MB |

Enforce them in CI: use Lighthouse CI with a `budget.json` file and a bundle-size check.

## Versions

Pin exact versions and record them in the project, never a range such as `^` or `latest`. Known-good at the time of writing:

| Package | Version |
|---|---|
| `three` | 0.186.1 |
| `gsap` | 3.15.0 (all plugins free since 3.13) |
| `lenis` | 1.3.27 |
| `kokoro-js` | 1.2.1 |
| `@google/model-viewer` | 4.4.0 |
| `@playwright/test` | 1.64.1 |
| `@axe-core/playwright` | 4.14.0 |

When upgrading three.js, read the migration notes, because add-on import paths and loader names change between releases (for example `RGBELoader` → `HDRLoader`). From a CDN, use exact-version URLs only.

## Adaptive quality

Measure frame time, step quality down when frames are slow, and step back up only after a sustained period of headroom.

```js
const TIERS = ['low', 'mid', 'high'];
const SETTINGS = {
  high: { pixelRatio: Math.min(devicePixelRatio, 2),    particles: 1, post: true,  blur: true,  grain: true },
  mid:  { pixelRatio: Math.min(devicePixelRatio, 1.25), particles: .6, post: false, blur: true,  grain: true },
  low:  { pixelRatio: 1,                                particles: .3, post: false, blur: false, grain: false },
};

function initialTier() {
  const c = navigator.connection, mem = navigator.deviceMemory ?? 8, cores = navigator.hardwareConcurrency ?? 8;
  if (c?.saveData || mem <= 2 || cores <= 2) return 'low';
  if (mem <= 4 || cores <= 4 || matchMedia('(max-width: 768px)').matches) return 'mid';
  return 'high';
}

let tier = initialTier(), samples = [], last = performance.now(), cooldownUntil = 0, goodSince = 0;
function apply(t) {
  tier = t;
  const s = SETTINGS[t];
  renderer.setPixelRatio(s.pixelRatio);
  geo.setDrawRange(0, Math.floor(COUNT * s.particles));
  setPostProcessing(s.post);
  document.documentElement.dataset.quality = t;            // CSS: glass blur and grain off on low
  store.set({ quality: t });
  bus.emit('quality:change', t);
}

export function sampleFrame(now) {                          // call from the render loop
  const dt = now - last; last = now;
  if (dt > 250 || document.hidden) return;                  // ignore tab switches and pauses
  samples.push(dt); if (samples.length < 60) return;
  const avg = samples.reduce((a, b) => a + b) / samples.length; samples = [];
  if (now < cooldownUntil) return;
  const i = TIERS.indexOf(tier);
  if (avg > 22 && i > 0) { apply(TIERS[i - 1]); cooldownUntil = now + 3000; goodSince = 0; }     // under ~45 fps
  else if (avg < 14 && i < TIERS.length - 1) {                                                     // over ~70 fps
    goodSince ||= now;
    if (now - goodSince > 10000) { apply(TIERS[i + 1]); cooldownUntil = now + 5000; goodSince = 0; }
  } else goodSince = 0;
}
apply(tier);
bus.on('quality:hint', h => { if (h === 'reduce' && tier !== 'low') apply('low'); if (h === 'restore') apply(initialTier()); });
```

The thresholds are based on frame time, so they work on 60Hz and 120Hz screens. iOS Low Power Mode limits pages to 30fps; the `low` tier is correct there. Pause rendering entirely when the tab is hidden or the canvas is off screen (use an `IntersectionObserver` on the canvas).

## Lazy loading and code-splitting

```js
// load three.js and the scene only when the canvas is about to be seen and WebGL exists
const io = new IntersectionObserver(async ([e]) => {
  if (!e.isIntersecting) return;
  io.disconnect();
  if (!hasWebGL()) return showNoWebGLFallback();
  const { initScene } = await import('./three/scene.js');
  initScene(store, bus);
}, { rootMargin: '200px' });
io.observe(document.querySelector('canvas.tour, canvas.bg'));

quizTab.addEventListener('click', () => import('./quiz/index.js'), { once: true });
paletteTrigger.addEventListener('pointerenter', () => import('./navigation/palette.js'), { once: true });   // prefetch on intent
```

- What loads lazily: three.js and post-processing (canvas near the viewport), ScrollTrigger and Lenis (only on scroll-mode pages), Kokoro (explicit action only, `audio.md`), `@google/model-viewer` (AR button), quiz and leaderboard code (quiz tab), and the command palette (first `Ctrl/Cmd+K` or hover).
- Only the first scene's assets are preloaded with `<link rel="modulepreload">` and `<link rel="preload">`. Every image has `loading="lazy"`, `decoding="async"` and explicit `width`/`height` (no layout shift).
- Show skeletons, or the poster image, while modules load. The core text content, chapter list and transcript render before any heavy JavaScript.
- Fonts: at most two families, subset with `unicode-range`, `font-display: swap`, and the display face preloaded. Variable fonts replace multiple weight files.

## Offline support (PWA)

```json
// manifest.webmanifest
{ "name": "Engine Tour", "short_name": "Tour", "start_url": "/?source=pwa", "display": "standalone",
  "background_color": "#0b0b0d", "theme_color": "#0b0b0d",
  "icons": [{ "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
            { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
            { "src": "/icons/maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }] }
```

```js
// sw.js — versioned caches
const V = 'v12', SHELL = `shell-${V}`, ASSETS = `assets-${V}`;
const SHELL_FILES = ['/', '/index.html', '/app.js', '/app.css', '/i18n/en.json', '/captions/en.vtt', '/voice/manifest.json', '/poster/1.webp'];

self.addEventListener('install', e => e.waitUntil(caches.open(SHELL).then(c => c.addAll(SHELL_FILES))));
self.addEventListener('activate', e => e.waitUntil(caches.keys().then(keys =>
  Promise.all(keys.filter(k => !k.endsWith(V)).map(k => caches.delete(k))))));

self.addEventListener('fetch', e => {
  const url = new URL(e.request.url);
  if (e.request.method !== 'GET' || url.origin !== location.origin) return;
  if (url.pathname.startsWith('/api/leaderboard')) {                          // network first, cached fallback
    e.respondWith(fetch(e.request).catch(() => caches.match(e.request).then(r => r ?? new Response('{"offline":true}', { headers: { 'Content-Type': 'application/json' } }))));
    return;
  }
  if (/\.(glb|opus|m4a|webp|avif|hdr|vtt|json)$/.test(url.pathname)) {          // cache first for heavy assets
    e.respondWith(caches.match(e.request).then(hit => hit ?? fetch(e.request).then(res => {
      if (res.ok && res.status === 200) caches.open(ASSETS).then(c => c.put(e.request, res.clone()));
      return res;
    })));
    return;
  }
  e.respondWith(caches.match(e.request).then(hit => hit ?? fetch(e.request)));   // shell
});
self.addEventListener('message', e => { if (e.data === 'skipWaiting') self.skipWaiting(); });
```

- Register the service worker after the `load` event. When a new worker is waiting, show the toast "Update available" with a Reload action, which sends `skipWaiting` and reloads on `controllerchange`.
- Large files (full model, all voice clips, the Hindi pack) are cached only after an explicit "Save for offline" action. Show the download size first, show progress, and offer "Remove offline data".
- Update `theme_color` per theme with `<meta name="theme-color">`. Offer installation (the `beforeinstallprompt` event) as an optional menu item, never as a pop-up.

## Privacy-safe analytics hooks

```js
// analytics.js — one wrapper; event names and properties are allow-listed
const EVENTS = {
  scene_view: ['scene', 'lang'], tour_complete: ['duration'], drop_off: ['scene', 'time'],
  quiz_complete: ['level', 'score'], error: ['kind'], feature_used: ['name'],
};
const consent = () => load('analyticsConsent', false) && navigator.doNotTrack !== '1' && !navigator.globalPrivacyControl;
let queue = [];
export function track(name, props = {}) {
  if (!EVENTS[name] || !consent()) return;
  queue.push({ e: name, ...Object.fromEntries(EVENTS[name].filter(k => k in props).map(k => [k, props[k]])), t: Date.now() });
  if (queue.length >= 10) flush();
}
function flush() {
  if (!queue.length) return;
  try { navigator.sendBeacon('/api/events', JSON.stringify(queue)); } catch {}
  queue = [];
}
document.addEventListener('visibilitychange', () => { if (document.hidden) { track('drop_off', { scene: store.get().scene, time: Math.round(store.get().time) }); flush(); } });
bus.on('scene:after', n => track('scene_view', { scene: n, lang: store.get().lang }));
```

There are no user ids, cookies, fingerprinting, IP storage, free text or URLs with personal data. Consent is opt-in, and changeable in settings. If the endpoint is blocked, every call silently does nothing.

## Testing recipe

```js
// playwright.config.js
export default {
  projects: [
    { name: 'desktop', use: { viewport: { width: 1440, height: 900 } } },
    { name: 'phone', use: { viewport: { width: 360, height: 780 }, hasTouch: true, isMobile: true } },
    { name: 'reduced-motion', use: { reducedMotion: 'reduce' } },
    { name: 'no-webgl', use: { launchOptions: { args: ['--disable-3d-apis'] } } },
    { name: 'light', use: { colorScheme: 'light' } },
  ],
};
```

```js
// tests/smoke.spec.js
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('no automatically detectable accessibility violations', async ({ page }) => {
  await page.goto('/');
  await page.waitForSelector('.preloader', { state: 'detached' });
  const { violations } = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa', 'wcag21aa', 'wcag22aa']).analyze();
  expect(violations).toEqual([]);
});

test('keyboard-only run-through', async ({ page }) => {
  await page.goto('/');
  await page.keyboard.press('Tab');
  await expect(page.locator('.skip')).toBeFocused();
  await page.keyboard.press('ArrowRight');
  await expect(page.locator('#live')).toContainText('Scene 2');
  await page.keyboard.press('?');
  await expect(page.locator('dialog.shortcuts')).toBeVisible();
  await page.keyboard.press('Escape');
  await expect(page.locator('dialog.shortcuts')).toBeHidden();
});

test('rapid scene switching never leaves a stuck state', async ({ page }) => {
  await page.goto('/');
  for (let i = 0; i < 8; i++) await page.keyboard.press('ArrowRight');
  await page.waitForTimeout(3000);
  await expect(page.locator('.curtain')).not.toHaveClass(/is-active/);
  await expect(page.locator('.panel.is-active')).toHaveCount(1);
});

test('tour works without WebGL', async ({ page }, info) => {
  test.skip(info.project.name !== 'no-webgl');
  await page.goto('/');
  await expect(page.getByText("The 3D view isn't available")).toBeVisible();
  await page.getByRole('button', { name: 'Play' }).click();
  await expect(page.locator('.captions')).not.toBeEmpty();
});
```

Manual checks that tests can't fully cover:
- A screen-reader pass with VoiceOver and NVDA.
- 200% zoom and 320px width.
- Forced colours (Windows High Contrast).
- A real iPhone (audio unlock, pseudo-fullscreen, gyroscope permission).
- A mid-range Android phone (frame rate, haptics).
- The theme on first visit matches the OS.
- Audio blocked by browser settings.
- An offline reload after installation.
