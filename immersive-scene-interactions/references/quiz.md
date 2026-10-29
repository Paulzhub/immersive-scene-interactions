# Quiz and learning features

Covers quiz structure, answer feedback, explanations, an optional accessible timer, spaced-repetition review, streaks and badges, confetti, haptics, the shareable result card, progress persistence and resume, and leaderboard security.

The Basics, Advanced and Leaderboard tabs use `switchPanel` (`motion.md`). Flashcard-style questions can use the 3D card flip in `motion.md`.

## Contents
- Question data and markup
- Answer feedback
- Explanations
- Optional timer (accessible)
- Spaced-repetition review
- Streaks and badges
- Confetti with a reduced-motion alternative
- Haptic feedback
- Shareable result card
- Progress persistence and resume
- Leaderboard: security and privacy

## Question data and markup

```js
// questions.json — ids are stable; never reuse an id for a different question
[{ "id": "b-01", "level": "basics", "q": { "en": "What does the injector do?", "hi": "…" },
   "options": { "en": ["…", "…", "…"], "hi": ["…", "…", "…"] }, "explain": { "en": "…", "hi": "…" }, "scene": 2 }]
```

The correct answers are **not** shipped to the browser for the leaderboard quiz; the server checks them (see "Leaderboard" below). For a practice-only quiz with no leaderboard, shipping answers is fine.

```html
<form class="question" aria-labelledby="q-b-01">
  <fieldset>
    <legend id="q-b-01">What does the injector do?</legend>
    <label class="opt"><input type="radio" name="b-01" value="0"> …</label>
    <label class="opt"><input type="radio" name="b-01" value="1"> …</label>
    <label class="opt"><input type="radio" name="b-01" value="2"> …</label>
  </fieldset>
  <button type="submit">Check answer</button>
  <p class="feedback" role="status"></p>
</form>
```

Native radios in a fieldset give arrow-key navigation, grouping and labels without extra code. Each option label is a 44px-tall target.

## Answer feedback

Correct: a brief pulse plus a check icon. Wrong: a small shake plus a cross icon, and the correct answer is revealed. Feedback never relies on colour alone.

```css
.opt.is-correct { border-color: var(--good); }
.opt.is-wrong   { border-color: var(--bad); }
.opt.is-correct::after { content: '✓ Correct'; color: var(--good); }
.opt.is-wrong::after   { content: '✗ Your answer'; color: var(--bad); }
.question.pulse { animation: pulse var(--dur-3) var(--ease-out); }
.question.shake { animation: shake 300ms linear; }
@keyframes pulse { 50% { transform: scale(1.02); } }
@keyframes shake { 20%, 60% { transform: translateX(-6px); } 40%, 80% { transform: translateX(6px); } }
@media (prefers-reduced-motion: reduce) { .question.pulse, .question.shake { animation: none; } }
```

```js
function showResult(form, { correct, correctIndex, chosen, explanation }) {
  const opts = form.querySelectorAll('.opt');
  opts[correctIndex].classList.add('is-correct');
  if (!correct) opts[chosen].classList.add('is-wrong');
  form.querySelectorAll('input').forEach(i => { i.disabled = true; });
  form.classList.remove('pulse', 'shake'); void form.offsetWidth;         // restart the animation
  form.classList.add(correct ? 'pulse' : 'shake');
  form.querySelector('.feedback').textContent = correct ? 'Correct.' : `Not quite. The answer is: ${opts[correctIndex].textContent.trim()}.`;
  sfx(correct ? 'correct' : 'wrong');
  haptic(correct ? 'correct' : 'wrong');
  showExplanation(form, explanation);
}
```

Write messages that encourage rather than punish ("Not quite"), and never just "Wrong!".

## Explanations

After each answer, show a short explanation (two or three sentences), with a "Watch this part" button that seeks the tour to the related scene (`player.seekTo(timeline[q.scene].start)`). The explanation is part of the `role="status"` message, or follows it in reading order, so screen-reader users hear it.

## Optional timer (accessible)

Off by default. When it's on, it must meet WCAG 2.2.1: the user can turn it off, or extend it, before time runs out.

```js
function startTimer(seconds, onExpire) {
  let left = seconds, warned = false, paused = false;
  const id = setInterval(() => {
    if (paused || document.hidden) return;                 // pauses when the tab is hidden
    left--;
    timerEl.textContent = `${left}s`;
    if (left === 20 && !warned) { warned = true; toast('20 seconds left', { action: { label: 'Add 60 seconds', run: () => { left += 60; } } }); }
    if (left <= 0) { clearInterval(id); onExpire(); }
  }, 1000);
  return { pause: () => { paused = true; }, resume: () => { paused = false; }, stop: () => clearInterval(id) };
}
```

Show the remaining time visually, but don't announce every second; announce only the 20-second warning. Provide "Pause timer" and "Turn off timer" buttons.

## Spaced-repetition review

Missed questions come back later in a "Review" set, using simple Leitner boxes.

```js
const INTERVALS = [0, 1, 3, 7, 14].map(d => d * 864e5);   // box 0–4, in milliseconds
export function recordAnswer(id, correct) {
  const srs = load('srs', {});
  const box = correct ? Math.min((srs[id]?.box ?? 0) + 1, 4) : 0;
  srs[id] = { box, due: Date.now() + INTERVALS[box] };
  save('srs', srs);
}
export const dueForReview = () => Object.entries(load('srs', {})).filter(([, v]) => v.due <= Date.now() && v.box < 4).map(([id]) => id);
```

Show "3 questions to review" on the quiz tab when items are due. Review answers update the boxes, but don't affect the leaderboard score.

## Streaks and badges

- Streaks: consecutive correct answers in a session, and days visited in a row (by local calendar day).
- Badges: First quiz, Perfect score, Finished the tour, Reviewed 10 questions, 5-day streak. Each badge has a text name and description, and an icon with alt text.
- Announce unlocks with a toast and a short sound effect. Show badges on the results screen and in a "Your badges" list.
- When a streak breaks, keep the tone neutral ("Streak reset. Your best is 7."). Never shame the user or nag them to come back.
- Store badges locally. If the leaderboard is connected, send only badge ids along with the display name.

## Confetti with a reduced-motion alternative

```js
export function confetti({ count = 120, duration = 2500 } = {}) {
  if (motion.reduced) { showStaticBadge(); return; }        // static badge and check mark instead
  const c = Object.assign(document.createElement('canvas'), { className: 'confetti' });
  c.setAttribute('aria-hidden', 'true');
  Object.assign(c.style, { position: 'fixed', inset: 0, pointerEvents: 'none', zIndex: 'var(--z-toast)' });
  document.body.append(c);
  const dpr = Math.min(devicePixelRatio, 2); c.width = innerWidth * dpr; c.height = innerHeight * dpr;
  const ctx = c.getContext('2d'); ctx.scale(dpr, dpr);
  const colors = ['--accent', '--good', '--blush', '--text'].map(color);
  const ps = Array.from({ length: count }, () => ({ x: innerWidth / 2, y: innerHeight * .35, vx: (Math.random() - .5) * 12,
    vy: Math.random() * -12 - 4, r: Math.random() * 6 + 3, a: Math.random() * 6.28, va: (Math.random() - .5) * .3, c: colors[(Math.random() * colors.length) | 0] }));
  const t0 = performance.now();
  (function frame(now) {
    const p = (now - t0) / duration;
    ctx.clearRect(0, 0, innerWidth, innerHeight);
    ctx.globalAlpha = Math.max(0, 1 - p * p);
    for (const q of ps) {
      q.vy += .35; q.vx *= .99; q.x += q.vx; q.y += q.vy; q.a += q.va;
      ctx.save(); ctx.translate(q.x, q.y); ctx.rotate(q.a); ctx.fillStyle = q.c; ctx.fillRect(-q.r / 2, -q.r / 4, q.r, q.r / 2); ctx.restore();
    }
    p < 1 ? requestAnimationFrame(frame) : c.remove();
  })(t0);
}
```

Use it only for real achievements (a perfect score, finishing the tour), not every correct answer. The result is always also given as text.

## Haptic feedback

```js
export function haptic(kind) {
  if (!store.get().haptics || !('vibrate' in navigator)) return;   // Android browsers; iOS Safari ignores it
  navigator.vibrate({ correct: 12, wrong: [30, 40, 30], unlock: [10, 30, 10, 30, 60] }[kind] ?? 10);
}
```

Make haptics a setting, on by default where supported and hidden where `vibrate` doesn't exist. Haptics only ever reinforce feedback that is also shown on screen.

## Shareable result card

```js
async function resultCard({ name, score, total, badge }) {
  await document.fonts.ready;
  const W = 1200, H = 630, c = new OffscreenCanvas(W, H), x = c.getContext('2d');
  x.fillStyle = color('--bg'); x.fillRect(0, 0, W, H);
  x.fillStyle = color('--accent'); x.font = '700 120px YourDisplayFont, system-ui'; x.fillText(`${score}/${total}`, 80, 300);
  x.fillStyle = color('--text'); x.font = '500 44px YourTextFont, system-ui';
  x.fillText(name ? `${name} completed the engine quiz` : 'Engine quiz completed', 80, 380);
  if (badge) x.drawImage(await createImageBitmap(await (await fetch(badge.icon)).blob()), W - 280, 80, 200, 200);
  x.drawImage(logoBitmap, 80, H - 120, 160, 48);
  return c.convertToBlob({ type: 'image/png' });
}

async function share(result) {
  const blob = await resultCard(result);
  const file = new File([blob], 'quiz-result.png', { type: 'image/png' });
  const data = { files: [file], title: 'My quiz result', text: `I scored ${result.score}/${result.total}`, url: location.origin + location.pathname };
  if (navigator.canShare?.(data)) return navigator.share(data).catch(() => {});
  download(file);                                            // fallback 1: save the image
  await navigator.clipboard?.writeText(data.url).then(() => toast('Image saved and link copied'), () => showLinkField(data.url));  // fallback 2: visible link
}
```

The card shows only the display name the user chose, and only if they typed one. Show a preview of the card with alt text ("Quiz result card: 9 out of 10, Perfect score badge") before sharing.

## Progress persistence and resume

```js
const KEY = 'tour:v1';
export function load(k, fallback) { try { return JSON.parse(localStorage.getItem(`${KEY}:${k}`)) ?? fallback; } catch { return fallback; } }
export function save(k, v) { try { localStorage.setItem(`${KEY}:${k}`, JSON.stringify(v)); return true; } catch { return false; } }

const PERSIST = ['scene', 'time', 'lang', 'speed', 'tab', 'captions', 'mute', 'haptics', 'shortcuts', 'quiz'];
let t;
store.subscribe(s => {
  clearTimeout(t);
  t = setTimeout(() => { if (!save('state', Object.fromEntries(PERSIST.map(k => [k, s[k]])))) storageWarningOnce(); }, 1000);
});
document.addEventListener('visibilitychange', () => { if (document.hidden) save('state', Object.fromEntries(PERSIST.map(k => [k, store.get()[k]]))); });

const saved = load('state', null);
if (saved && (saved.time > 5 || Object.keys(saved.quiz?.answers ?? {}).length)) {
  showResumePrompt({                                         // "Continue where you left off?" [Resume] [Start over]
    resume: () => store.set({ ...saved, playing: false }),   // never auto-play on resume
    restart: () => { localStorage.removeItem(`${KEY}:state`); },
  });
}
```

`storageWarningOnce()` shows one toast if saving fails ("Your progress can't be saved in this browser"), and the state then lives only in memory. Bump the version in `KEY` when the state shape changes, and ignore old versions. Deep-link parameters (`navigation.md`) take priority over saved state.

## Leaderboard: security and privacy

Assume every request can be forged. The server decides the score.

- **Start:** `POST /quiz/start` returns `{ sessionId, questions }`, without answers, and records the start time and the issued question ids.
- **Submit:** `POST /quiz/submit { sessionId, answers: { id: optionIndex } }`. The server marks the answers, rejects unknown or duplicate sessions, rejects implausible timing (faster than about 1 second per question), and returns the score and the correct answers for review.
- **Names:** 2–20 characters, a Unicode letters/numbers/space allow-list, normalised (NFKC), passed through a profanity and impersonation filter, and checked again on the server. Always render names with `textContent`, never as HTML.
- **Rate limits:** per IP and per session, on both start and submit, plus an optional proof-of-work or a privacy-friendly challenge if abuse appears.
- **Reads:** `GET /leaderboard` is cacheable for 30–60 seconds, paginated, and returns only name, score, badge ids and date.
- **Privacy:** no emails, accounts or tracking ids are required. Joining the leaderboard is opt-in at submit time, with a "Remove my entry" option (a delete token stored locally).
- **Offline or unreachable:** the quiz still works locally. The Leaderboard tab shows "Couldn't reach the leaderboard. Check your connection and try again", with a Retry button. Pending submissions can be retried later, with the user's consent.
