# 3D with three.js

Covers renderer setup, the ember particle background, lens breath, post-processing, shader hover distortion, hotspots, the spline fly-through, idle auto-rotate, low-poly placeholder loading, exploded view, lighting presets, snapshots, the measurement tool, AR view and gyroscope tilt.

Load all of this lazily (`performance.md`). The page must work without it (`accessibility.md`, "Graceful fallbacks").

## Contents
- Setup (versions, imports, renderer, WebGPU option, teardown)
- Ember particle background
- Lens breath (FOV punch)
- Post-processing (bloom, chromatic aberration, vignette, depth of field)
- Shader distortion on hover
- Hotspots / annotations
- Camera path (spline fly-through)
- Auto-rotate on idle
- Low-poly placeholder, then full model
- Exploded view / part isolation
- Lighting presets
- Snapshot of the 3D view
- Measurement tool
- AR view (`<model-viewer>`)
- Gyroscope tilt on mobile

## Setup

Pin exact versions. Known-good at the time of writing: `three@0.186.1` and `gsap@3.15.0`. Check the three.js changelog when upgrading, because add-on paths and loader names change.

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

export function hasWebGL() {
  try { const c = document.createElement('canvas'); return !!(c.getContext('webgl2') || c.getContext('webgl')); }
  catch { return false; }
}

const canvas = document.querySelector('canvas.bg');
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: true, powerPreference: 'high-performance' });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));        // adaptive quality lowers this further
renderer.setSize(innerWidth, innerHeight, false);
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.toneMapping = THREE.ACESFilmicToneMapping;

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(45, innerWidth / innerHeight, .1, 200);
const BASE_FOV = 45;

canvas.addEventListener('webglcontextlost', e => { e.preventDefault(); bus.emit('webgl:lost'); });   // switch to fallback
addEventListener('resize', () => {
  camera.aspect = innerWidth / innerHeight; camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight, false); composer?.setSize(innerWidth, innerHeight);
});

renderer.setAnimationLoop(t => {           // one loop for everything 3D
  if (document.hidden) return;
  tickParticles(t); tickHotspots(); controls.update();
  composer ? composer.render() : renderer.render(scene, camera);
});

export function destroy() {
  renderer.setAnimationLoop(null);
  scene.traverse(o => { o.geometry?.dispose(); [].concat(o.material || []).forEach(m => { Object.values(m).forEach(v => v?.isTexture && v.dispose()); m.dispose(); }); });
  composer?.dispose(); renderer.dispose();
}
```

**WebGPU option:** `import { WebGPURenderer } from 'three/webgpu'` falls back to WebGL 2 automatically. Shader materials must then be written in TSL (three's node shader language) instead of GLSL. Stay on `WebGLRenderer` unless you need compute shaders, for example for more than 50,000 particles.

Canvas markup: `<canvas class="bg" aria-hidden="true" tabindex="-1">`. The interactive tour canvas is a separate, focusable element with `role="application"`, `aria-label` and `aria-describedby` pointing to the keyboard help (`tour-player.md`).

## Ember particle background

About 1,900 glowing "lamp ember" particles (900 on small screens) drift upward in 3D. They react to the cursor or touch, shift with scrolling, change palette per section, and switch blending with the theme. All movement happens in the vertex shader.

```js
const small = matchMedia('(max-width: 768px)').matches;
let COUNT = small ? 900 : 1900;
const geo = new THREE.BufferGeometry();
const pos = new Float32Array(COUNT * 3), seed = new Float32Array(COUNT);
for (let i = 0; i < COUNT; i++) {
  pos[i * 3] = (Math.random() - .5) * 20;
  pos[i * 3 + 1] = (Math.random() - .5) * 12;
  pos[i * 3 + 2] = (Math.random() - .5) * 14;
  seed[i] = Math.random();
}
geo.setAttribute('position', new THREE.BufferAttribute(pos, 3));
geo.setAttribute('aSeed', new THREE.BufferAttribute(seed, 1));

const uniforms = {
  uTime: { value: 0 }, uSpeed: { value: 1 }, uScroll: { value: 0 },
  uPointer: { value: new THREE.Vector2(9, 9) },          // off-screen until the pointer moves
  uColor: { value: new THREE.Color(color('--ember')) },
  uSize: { value: 28 * renderer.getPixelRatio() },
};
const mat = new THREE.ShaderMaterial({
  uniforms, transparent: true, depthWrite: false, blending: THREE.AdditiveBlending,
  vertexShader: /* glsl */`
    uniform float uTime, uSpeed, uScroll, uSize; uniform vec2 uPointer;
    attribute float aSeed; varying float vAlpha;
    void main() {
      vec3 p = position;
      float t = uTime * uSpeed * (.2 + aSeed * .3);
      p.x += sin(t + aSeed * 6.2831) * .6;
      p.y = mod(position.y + 6. + t * .4 + uScroll * 3., 12.) - 6.;     // rise and wrap
      vec4 mv = modelViewMatrix * vec4(p, 1.);
      vec4 clip = projectionMatrix * mv;
      vec2 d = clip.xy / clip.w - uPointer;
      float f = smoothstep(.35, 0., length(d));
      clip.xy += normalize(d + 1e-4) * f * .08 * clip.w;                // push away from the pointer
      gl_Position = clip;
      gl_PointSize = uSize * (.4 + aSeed) / -mv.z;
      vAlpha = (.35 + .65 * aSeed) * (.7 + .3 * sin(uTime * 2. + aSeed * 40.));   // flicker
    }`,
  fragmentShader: /* glsl */`
    uniform vec3 uColor; varying float vAlpha;
    void main() {
      float a = smoothstep(.5, 0., length(gl_PointCoord - .5));
      gl_FragColor = vec4(uColor, a * a * vAlpha);
    }`,
});
const points = new THREE.Points(geo, mat);
points.frustumCulled = false;                             // positions move in the shader
scene.add(points);

let scrollTarget = 0;
addEventListener('pointermove', e => uniforms.uPointer.value.set(e.clientX / innerWidth * 2 - 1, -(e.clientY / innerHeight * 2 - 1)), { passive: true });
addEventListener('pointerleave', () => uniforms.uPointer.value.set(9, 9));
store.subscribe(s => { scrollTarget = s.mode === 'step' ? s.scene + s.sceneProgress : scrollY / innerHeight; });

export function tickParticles(t) {
  if (!motion.reduced) uniforms.uTime.value = t / 1000;   // reduced motion: frozen, static field
  uniforms.uScroll.value += (scrollTarget - uniforms.uScroll.value) * .05;
}

bus.on('scene:mid', n => {                                // per-section look
  const s = scenes[n].particles || {};
  gsap.to(uniforms.uSpeed, { value: s.speed ?? 1, duration: 1.2 });
  const c = new THREE.Color(s.color ?? color('--ember'));
  gsap.to(uniforms.uColor.value, { r: c.r, g: c.g, b: c.b, duration: 1.2 });
});
bus.on('theme:change', theme => {
  mat.blending = theme === 'light' ? THREE.NormalBlending : THREE.AdditiveBlending;  // additive glow vanishes on light backgrounds
  uniforms.uColor.value.set(color('--ember'));
  points.visible = theme !== 'contrast';
  mat.needsUpdate = true;
});
```

Pointer events cover mouse, pen and touch. Give the background canvas `touch-action: auto` so it never blocks page scrolling; only the interactive tour canvas uses `touch-action: none`. Adaptive quality halves `COUNT` by drawing fewer points with `geo.setDrawRange(0, COUNT / 2)`, without reallocating.

## Lens breath (FOV punch)

When the scene changes, the camera briefly widens (zooms out) and the depth of field blurs, then settles. Fire it on `scene:mid`. Before a new breath starts, the previous one is cancelled, so rapid changes never stack.

```js
let breath;
export function lensBreath() {
  if (motion.reduced) return;
  breath?.kill();
  const o = { k: 0 };
  const apply = () => {
    camera.fov = BASE_FOV + 14 * o.k; camera.updateProjectionMatrix();
    if (fx.bokeh) fx.bokeh.uniforms.aperture.value = BOKEH_BASE + .004 * o.k;      // DOF blur via post-processing
    if (fx.chroma) fx.chroma.uniforms.amount.value = .002 + .006 * o.k;           // chromatic aberration spike
  };
  breath = gsap.timeline({ onComplete() { o.k = 0; apply(); } })
    .to(o, { k: 1, duration: .35, ease: 'power2.out', onUpdate: apply })
    .to(o, { k: 0, duration: 1.1, ease: 'power3.out', onUpdate: apply });
}
bus.on('scene:mid', lensBreath);
```

Don't blur the full-screen canvas with CSS `filter: blur()`, which is expensive on large canvases. Without post-processing (low quality tier), the breath is the FOV change only.

## Post-processing

Bloom on the embers, chromatic aberration during the lens breath, a subtle vignette, and optional depth of field. Enable it only on the high quality tier (`performance.md`), and render at reduced resolution on mobile.

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { BokehPass } from 'three/addons/postprocessing/BokehPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';

const ChromaVignette = {
  uniforms: { tDiffuse: { value: null }, amount: { value: .002 }, vignette: { value: .35 } },
  vertexShader: `varying vec2 vUv; void main(){ vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position,1.); }`,
  fragmentShader: `uniform sampler2D tDiffuse; uniform float amount, vignette; varying vec2 vUv;
    void main(){
      vec2 dir = vUv - .5;
      vec3 c = vec3(texture2D(tDiffuse, vUv + dir * amount).r, texture2D(tDiffuse, vUv).g, texture2D(tDiffuse, vUv - dir * amount).b);
      c *= 1. - vignette * dot(dir, dir) * 2.;
      gl_FragColor = vec4(c, 1.);
    }`,
};

export const fx = {};
let composer = null;
export function setPostProcessing(on) {
  composer?.dispose(); composer = null; fx.bokeh = fx.chroma = null;
  if (!on) return;
  composer = new EffectComposer(renderer);
  composer.setPixelRatio(Math.min(renderer.getPixelRatio(), small ? 1 : 1.5));
  composer.addPass(new RenderPass(scene, camera));
  const bloom = new UnrealBloomPass(new THREE.Vector2(innerWidth, innerHeight), store.get().theme === 'light' ? .35 : .8, .6, .2);
  composer.addPass(bloom);
  fx.bokeh = new BokehPass(scene, camera, { focus: 6, aperture: BOKEH_BASE, maxblur: .01 });
  composer.addPass(fx.bokeh);
  fx.chroma = new ShaderPass(ChromaVignette);
  composer.addPass(fx.chroma);
  composer.addPass(new OutputPass());
}
bus.on('quality:change', q => setPostProcessing(q === 'high'));
```

`BokehPass` renders the scene a second time for depth, which is expensive. Keep its aperture at the base value except during the lens breath, or drop the pass on mid-range devices.

## Shader distortion on hover

An image or heading ripples, or splits into RGB channels, around the pointer. Render the image as a textured plane in its own small canvas, or in the main scene aligned to the DOM rectangle.

```glsl
// fragment shader
uniform sampler2D uTex; uniform vec2 uMouse; uniform float uHover, uTime; varying vec2 vUv;
void main() {
  float d = distance(vUv, uMouse);
  float wave = sin(d * 30. - uTime * 4.) * .01 * uHover * smoothstep(.4, 0., d);
  vec2 uv = vUv + normalize(vUv - uMouse + 1e-4) * wave;
  float s = .006 * uHover;
  gl_FragColor = vec4(texture2D(uTex, uv + s).r, texture2D(uTex, uv).g, texture2D(uTex, uv - s).b, 1.);
}
```

On enter and leave, animate `uHover` between 0 and 1 (`gsap.to(u.uHover, { value: 1 })`). Keep the real `<img alt>` in the DOM and make the canvas `aria-hidden`. Use only fine pointers, not reduced motion, and only the high quality tier. A CSS fallback is two pseudo-element copies with `mix-blend-mode: screen` and small opposite translations on hover.

## Hotspots / annotations

Clickable points on the model open the "Open tip" cards.

```js
// hotspots: [{ id, position: Vector3, normal?: Vector3, title, body, scene?, camera? }]
const v = new THREE.Vector3(), toCam = new THREE.Vector3();
export function tickHotspots() {
  const { width, height } = canvas.getBoundingClientRect();
  for (const h of hotspots) {
    v.copy(h.position).project(camera);
    const inView = v.z < 1 && Math.abs(v.x) <= 1.05 && Math.abs(v.y) <= 1.05;
    const facing = !h.normal || toCam.copy(camera.position).sub(h.position).dot(h.normal) > 0;
    const inScene = h.scene == null || h.scene === store.get().scene;
    const show = (inView && facing && inScene) || h.el === document.activeElement;   // never hide the focused marker
    if (show !== h.shown) { h.el.classList.toggle('is-hidden', !show); h.el.tabIndex = show ? 0 : -1; h.shown = show; }
    if (show) h.el.style.transform = `translate3d(${(v.x * .5 + .5) * width}px, ${(-v.y * .5 + .5) * height}px, 0) translate(-50%, -50%)`;
  }
}
```

- Markers are `<button class="hotspot" data-cursor="Open tip" aria-expanded="false" aria-controls="tip-…" aria-label="Tip: Fuel injector">` with a pulsing ring (static when motion is reduced). Change visibility by class, not the `hidden` attribute, so a focused marker isn't removed. Toggle classes only when visibility changes, never every frame.
- The card is a glass panel anchored near its marker. It flips to stay inside the viewport and becomes a bottom sheet on phones. It registers in the Esc layer stack. Focus moves into the card on open and returns to the marker on close.
- Optionally, opening a tip pauses the timeline and flies the camera to `h.camera`, using the spline below.
- Always provide a list view of all tips ("Tips in this scene"), so keyboard and screen-reader users don't have to find markers in 3D space.

## Camera path (spline fly-through)

Smooth moves between scenes instead of straight tweens.

```js
const posCurve = new THREE.CatmullRomCurve3(scenes.map(s => new THREE.Vector3(...s.camera.position)), false, 'centripetal');
const lookCurve = new THREE.CatmullRomCurve3(scenes.map(s => new THREE.Vector3(...s.camera.target)), false, 'centripetal');
const ease = t => t < .5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;
const userOffset = new THREE.Vector3();                   // from orbit input; decays on each scene change

export function placeCamera(scenePos) {                   // scenePos = scene index + progress through the move, e.g. 2.4
  const k = Math.floor(scenePos), f = scenePos - k;
  const eu = scenes.length > 1 ? (k + ease(f)) / (scenes.length - 1) : 0;   // ease each segment
  camera.position.copy(posCurve.getPointAt(Math.min(1, eu))).add(userOffset);
  controls.target.copy(lookCurve.getPointAt(Math.min(1, eu)));
}
bus.on('scene:mid', () => gsap.to(userOffset, { x: 0, y: 0, z: 0, duration: 1.2 }));
```

Drive `scenePos` from the timeline clock (`tour-player.md`), so seeking, speed and chapters move the camera correctly. Combine with lens breath. With reduced motion, cut directly to each scene's preset instead.

## Auto-rotate on idle

```js
let idleTimer;
const IDLE_MS = 6000;
function armIdle() {
  controls.autoRotate = false;
  clearTimeout(idleTimer);
  idleTimer = setTimeout(() => {
    const s = store.get();
    controls.autoRotate = !motion.reduced && !document.hidden && s.player !== 'mini' && !openTip;
  }, IDLE_MS);
}
controls.autoRotateSpeed = .5;
controls.addEventListener('start', armIdle);
['pointerdown', 'keydown', 'wheel', 'touchstart'].forEach(t => tourCanvas.addEventListener(t, armIdle, { passive: true }));
armIdle();
```

## Low-poly placeholder, then full model

```js
const draco = new DRACOLoader().setDecoderPath('/draco/');
const loader = new GLTFLoader(manager).setDRACOLoader(draco);   // manager feeds the preloader (motion.md)

const proxy = (await loader.loadAsync('/models/engine-lod0.glb')).scene;   // under ~150 KB
scene.add(proxy);
loader.loadAsync('/models/engine.glb').then(({ scene: full }) => {
  full.traverse(o => { if (o.material) { o.material.transparent = true; o.material.opacity = 0; } });
  scene.add(full);
  gsap.to({ k: 0 }, { k: 1, duration: motion.reduced ? 0 : .6, onUpdate() {
    const k = this.targets()[0].k;
    full.traverse(o => { if (o.material) o.material.opacity = k; });
    proxy.traverse(o => { if (o.material) { o.material.transparent = true; o.material.opacity = 1 - k; } });
  }, onComplete() {
    full.traverse(o => { if (o.material) o.material.transparent = false; });
    scene.remove(proxy); disposeTree(proxy);
  }});
}, err => toast.error('The detailed model could not load. Showing a simplified version.'));
```

Compress models with `gltf-transform` (Draco or Meshopt geometry, KTX2 textures) before shipping.

## Exploded view / part isolation

```js
// each part: { mesh, dir: Vector3 (unit), dist, origin: Vector3 }
export function setExplode(amount) {                // 0–1, from a slider or a timeline scene
  for (const p of parts) p.mesh.position.copy(p.origin).addScaledVector(p.dir, p.dist * amount);
}
export function isolate(id) {
  for (const p of parts) {
    const on = !id || p.id === id;
    p.mesh.material.transparent = !on;
    gsap.to(p.mesh.material, { opacity: on ? 1 : .15, duration: motion.reduced ? 0 : .3 });
  }
  announce(id ? `${partName(id)} isolated` : 'All parts shown');
}
```

- Controls: an explode slider (`<input type="range">`, labelled), a parts list (buttons that isolate each part, also the keyboard and screen-reader route), and a reset button that restores the original transforms and opacity.
- Clone each material before changing its opacity, if meshes share materials.

## Lighting presets

Day, night and studio looks, each with its own environment map, exposure and key light.

```js
import { HDRLoader } from 'three/addons/loaders/HDRLoader.js';   // named RGBELoader in older releases
const pmrem = new THREE.PMREMGenerator(renderer);
const PRESETS = {
  day:    { hdr: '/env/day.hdr',    exposure: 1.0, envIntensity: 1.0, key: 2.5 },
  night:  { hdr: '/env/night.hdr',  exposure: .6,  envIntensity: .4,  key: .8 },
  studio: { hdr: '/env/studio.hdr', exposure: .9,  envIntensity: 1.2, key: 1.5 },
};
const cache = {};
export async function setLighting(name) {
  const p = PRESETS[name];
  cache[name] ??= pmrem.fromEquirectangular(await new HDRLoader().loadAsync(p.hdr)).texture;
  scene.environment = cache[name];
  const d = motion.reduced ? 0 : 1;
  gsap.to(renderer, { toneMappingExposure: p.exposure, duration: d });
  gsap.to(scene, { environmentIntensity: p.envIntensity, duration: d });
  gsap.to(keyLight, { intensity: p.key, duration: d });
}
```

Use 1K HDRs for the web, prefetch the next preset when the menu opens, and present the choice as a labelled radio group.

## Snapshot of the 3D view

```js
export function snapshot() {
  (composer || renderer).render(scene, camera);           // render and read in the same task; no preserveDrawingBuffer needed
  renderer.domElement.toBlob(async blob => {
    const file = new File([blob], `view-${Date.now()}.png`, { type: 'image/png' });
    if (navigator.canShare?.({ files: [file] })) await navigator.share({ files: [file], title: document.title }).catch(() => {});
    else download(file);
    toast('Snapshot saved');
  }, 'image/png');
}
```

For larger output, temporarily call `renderer.setPixelRatio(2)` before rendering, then restore it. If you add a watermark or caption, draw the snapshot onto a 2D canvas first.

## Measurement tool

```js
const ray = new THREE.Raycaster(), ndc = new THREE.Vector2();
let pts = [];
tourCanvas.addEventListener('click', e => {
  if (!measuring) return;
  const r = tourCanvas.getBoundingClientRect();
  ndc.set((e.clientX - r.left) / r.width * 2 - 1, -((e.clientY - r.top) / r.height) * 2 + 1);
  ray.setFromCamera(ndc, camera);
  const hit = ray.intersectObject(model, true)[0]; if (!hit) return;
  pts.push(hit.point.clone());
  if (pts.length === 2) {
    const mm = pts[0].distanceTo(pts[1]) * UNITS_TO_MM;
    drawMeasureLine(pts);
    showLabel(midpoint(pts), `${mm.toFixed(1)} mm`);
    announce(`Distance ${mm.toFixed(1)} millimetres`);
    pts = [];
  }
});
```

Keyboard alternative: a "From / To" pair of selects listing named anchor points on the model. Label lines in the DOM, projected like hotspots, so the text stays sharp and readable.

## AR view (`<model-viewer>`)

Load `@google/model-viewer` (pin an exact version, e.g. `@google/model-viewer@4.4.0`) only when the user presses "View in your room". It bundles its own three.js.

```html
<model-viewer id="ar" src="/models/engine.glb" ios-src="/models/engine.usdz" ar ar-modes="webxr scene-viewer quick-look"
  camera-controls alt="3D model of the engine" hidden></model-viewer>
```

```js
arButton.addEventListener('click', async () => {
  await import('@google/model-viewer');
  const mv = document.getElementById('ar');
  await mv.updateComplete;
  if (mv.canActivateAR) mv.activateAR();
  else toast('AR is not available on this device');
});
```

Show the AR button only on mobile devices that are likely to support it, and still handle `canActivateAR === false`. Provide a USDZ file for iOS Quick Look.

## Gyroscope tilt on mobile

Tilting the phone drives the same `tx`/`ty` targets as the pointer parallax, with the same limits.

```js
async function enableTilt() {
  if (typeof DeviceOrientationEvent?.requestPermission === 'function') {    // iOS: must be called from a tap
    if (await DeviceOrientationEvent.requestPermission().catch(() => 'denied') !== 'granted') return toast('Motion access was not allowed');
  }
  let base = null;
  addEventListener('deviceorientation', e => {
    if (motion.reduced || e.beta == null) return;
    base ??= { b: e.beta, g: e.gamma };                   // calibrate to how the phone is held
    setTiltTarget(Math.max(-1, Math.min(1, (e.gamma - base.g) / 25)), Math.max(-1, Math.min(1, (e.beta - base.b) / 25)));
  });
}
```

Make tilt an opt-in toggle ("Tilt to look around"), off by default and remembered. Recalibrate on `orientationchange`.
