# Audio: voiceover, sound effects, spatial audio

Covers audio unlock on the first gesture, the Kokoro AI voiceover (English and Hindi), keeping voice in sync with the timeline, UI sound design, spatial audio and mute settings.

## Contents
- Rules
- Unlock on first gesture
- Kokoro voiceover: build-time pipeline (recommended)
- Kokoro in the browser (optional, English)
- Voice playback in sync with the timeline
- Sound design (UI sound effects)
- Spatial audio
- Mute, volume and fallbacks

## Rules
- Voiceover is **on by default**, but browsers block sound until the user interacts with the page. Start narration on the first tap, click or keypress, and show a visible "Tap anywhere to turn on sound" hint until then.
- Voice and sound effects have separate mute toggles, both remembered. Sound effects default to a low volume.
- Narration is never the only way to get information: captions and the transcript always exist.
- Narration plays through an `<audio>` element. That keeps the Media Session controls working and lets `playbackRate` keep pitch (`preservesPitch`).

## Unlock on first gesture

```js
let ctx = null, unlocked = false;
export function unlockAudio() {
  if (unlocked) return;
  unlocked = true;
  ctx = new AudioContext();
  ctx.resume();
  voiceEl.play().then(() => voiceEl.pause()).catch(() => {});   // prime the element inside the gesture
  hint.remove();
  bus.emit('audio:unlocked');
  if (store.get().playing) playVoiceAt(store.get().time);
}
['pointerdown', 'keydown'].forEach(t => addEventListener(t, unlockAudio, { once: true, capture: true }));
```

The first `player.play()` also calls `unlockAudio()`. In step mode, the very first tap anywhere usually happens before the user presses Play, so the narration for the current scene starts as soon as the timeline plays.

## Kokoro voiceover: build-time pipeline (recommended)

Generate all narration ahead of time and ship compressed audio files plus a manifest of their durations. This is faster to start, works offline, costs nothing on the user's device, and supports Hindi.

Kokoro-82M ships Hindi voices (`hf_alpha`, `hf_beta`, `hm_omega`, `hm_psi`) through the Python `kokoro` package with `lang_code='h'`. English uses `lang_code='a'` (American) or `'b'` (British), with voices such as `af_heart` and `bf_emma`. Check current voice names in the Kokoro documentation before generating.

```python
# tools/gen_voice.py — run at build time; pip install kokoro soundfile
import json
import numpy as np
import soundfile as sf
from kokoro import KPipeline

SCRIPT = json.load(open('narration.json'))       # { "en": { "intro": "Welcome…" }, "hi": { "intro": "स्वागत है…" } }
VOICES = { 'en': ('a', 'af_heart'), 'hi': ('h', 'hf_alpha') }
manifest = {}
for lang, lines in SCRIPT.items():
    code, voice = VOICES[lang]
    pipe = KPipeline(lang_code=code)
    manifest[lang] = {}
    for scene_id, text in lines.items():
        wav = np.concatenate([a for _, _, a in pipe(text, voice=voice)])
        path = f'public/voice/{lang}/{scene_id}.wav'
        sf.write(path, wav, 24000)
        manifest[lang][scene_id] = { 'src': f'/voice/{lang}/{scene_id}.opus', 'duration': round(len(wav) / 24000, 3) }
json.dump(manifest, open('public/voice/manifest.json', 'w'), ensure_ascii=False, indent=2)
```

Then convert each file to Opus (`ffmpeg -i x.wav -c:a libopus -b:a 48k x.opus`) with an M4A/AAC copy for older Safari. Generate the captions file from the same narration script, timed per sentence, so captions and voice always match. The timeline is built from `manifest.json` (`tour-player.md`).

## Kokoro in the browser (optional, English)

`kokoro-js` (pin an exact version, e.g. `kokoro-js@1.2.1`) runs the ONNX model in the browser, through WebGPU when available, otherwise WASM. Its phonemiser supports English only at the time of writing, so Hindi must still be pre-generated. Use it only for text that can't be prepared in advance, for example reading out a user's quiz result.

```js
// voice-worker.js — keep the model off the main thread
import { KokoroTTS } from 'kokoro-js';
let tts;
self.onmessage = async ({ data: { id, text, voice } }) => {
  tts ??= await KokoroTTS.from_pretrained('onnx-community/Kokoro-82M-v1.0-ONNX', {
    dtype: 'q8', device: 'gpu' in navigator ? 'webgpu' : 'wasm' });
  const audio = await tts.generate(text, { voice: voice ?? 'af_heart' });
  const blob = audio.toBlob();
  self.postMessage({ id, blob }, []);
};
```

The model download is large (tens of MB even quantised). Only start it after an explicit action or when the device is idle on a fast, unmetered connection (`navigator.connection?.saveData`), cache it, and show progress.

## Voice playback in sync with the timeline

```js
const voiceEl = new Audio();
voiceEl.preload = 'auto';
voiceEl.preservesPitch = true;
let currentClip = null;

export function playVoiceAt(time) {
  const s = store.get();
  if (!unlocked || s.mute.voice || !s.playing) { voiceEl.pause(); return; }
  const seg = timeline[s.scene], clip = seg.voice;
  const offset = time - seg.start;
  if (!clip || offset >= clip.duration) { voiceEl.pause(); return; }   // silent tail of the scene
  if (currentClip !== clip.src) { voiceEl.src = clip.src; currentClip = clip.src; }
  voiceEl.currentTime = offset;
  voiceEl.playbackRate = s.speed;
  voiceEl.play().catch(() => showSoundHint());
}

bus.on('scene:mid', () => playVoiceAt(store.get().time));
bus.on('seek:end', () => playVoiceAt(store.get().time));
store.subscribe((s, p) => {
  if (s.playing !== p.playing) s.playing ? playVoiceAt(s.time) : voiceEl.pause();
  if (s.speed !== p.speed) voiceEl.playbackRate = s.speed;
  if (s.lang !== p.lang || s.mute.voice !== p.mute.voice) playVoiceAt(s.time);
  if (s.playing && !voiceEl.paused && Math.abs(voiceEl.currentTime - (s.time - timeline[s.scene].start)) > .3)
    voiceEl.currentTime = s.time - timeline[s.scene].start;   // correct drift
});
```

- Prefetch the next scene's clip during the current one (`new Audio(next.src).preload = 'auto'`, or `<link rel="prefetch">`).
- Scene durations already include each clip's length, so no clip is ever cut off.
- Switching language keeps the same relative position. The timeline is rebuilt first (`tour-player.md`), then the voice restarts.

## Sound design

Subtle UI clicks, hover ticks and a curtain "whoosh", using one decoded sound sprite through Web Audio.

```js
const SPRITE = { click: [0, .08], tick: [.1, .04], whoosh: [.2, .9], correct: [1.2, .5], wrong: [1.8, .4] };
let buffer, sfxGain;
bus.on('audio:unlocked', async () => {
  sfxGain = ctx.createGain(); sfxGain.gain.value = .25; sfxGain.connect(ctx.destination);
  buffer = await ctx.decodeAudioData(await (await fetch('/audio/ui-sprite.mp3')).arrayBuffer());
});

const lastPlayed = {};
export function sfx(name, { pan = 0 } = {}) {
  if (!buffer || store.get().mute.sfx || document.hidden) return;
  const now = ctx.currentTime;
  if (now - (lastPlayed[name] ?? 0) < .05) return;             // throttle hover ticks
  lastPlayed[name] = now;
  const [start, dur] = SPRITE[name];
  const src = ctx.createBufferSource(); src.buffer = buffer;
  const p = ctx.createStereoPanner(); p.pan.value = pan;
  src.connect(p).connect(sfxGain);
  src.start(0, start, dur);
}

bus.on('scene:before', () => { if (!motion.reduced) sfx('whoosh'); });
document.addEventListener('click', e => { if (e.target.closest('button, [role="tab"]')) sfx('click'); });
```

While narration plays, duck the sound effects: `sfxGain.gain.setTargetAtTime(voiceEl.paused ? .25 : .08, ctx.currentTime, .1)`. Sounds always accompany an action the user took or a visible change; they never carry information on their own.

## Spatial audio

Ambience and hotspot sounds placed in the 3D scene. Narration stays centred and non-spatial, for clarity.

```js
import * as THREE from 'three';
const listener = new THREE.AudioListener();      // reuses its own AudioContext; call listener.context.resume() on unlock
camera.add(listener);

const hum = new THREE.PositionalAudio(listener);
hum.setRefDistance(2); hum.setRolloffFactor(1.5); hum.setDistanceModel('inverse');
hum.panner.panningModel = 'HRTF';
new THREE.AudioLoader().load('/audio/engine-hum.opus', b => { hum.setBuffer(b); hum.setLoop(true); hum.setVolume(.4); });
engineMesh.add(hum);
bus.on('audio:unlocked', () => { listener.context.resume(); if (!store.get().mute.sfx) hum.play(); });
```

Offer a "Mono audio" option that sets `panningModel = 'equalpower'` and centres everything, for listeners with hearing in one ear. Stop positional sounds when the tab is hidden, in mini mode, or when sound effects are muted.

## Mute, volume and fallbacks

- The controls include voice on/off (`M`), sound effects on/off and a volume slider. All are remembered (`quiz.md`, "Progress persistence").
- If a clip fails to load (`error` event) or Kokoro fails: continue with captions, show a toast ("Narration couldn't load. Captions are on."), switch captions on, and offer Retry.
- If autoplay is blocked mid-session (`play()` rejects), show the sound hint again.
- Honour the Save-Data preference: don't prefetch upcoming clips, and load each one only when its scene starts.
