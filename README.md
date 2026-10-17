# The Feed Scrolls You

A role-reversal webcam game. **You are the content.** An AI "algorithm" (a bored cartoon face slumped on a beanbag, thumb hovering over the *next* button) is watching you. Do what it asks, stay interesting, and keep its attention for 60 seconds. The moment you get boring, it swipes you away.

Everything runs in your browser. There is no backend, and your video never leaves your device: pose and face tracking run locally with [MediaPipe Tasks Vision](https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker).

---

## Running it

Browsers only allow camera access on `https://` or `localhost`, so opening `index.html` directly from disk (`file://`) won't work. Serve the folder instead:

```bash
python3 -m http.server 8000
```

Then open **http://localhost:8000**, click **Go live**, and allow camera access.

**Requirements**

- A recent desktop browser (Chrome, Edge, Firefox or Safari). Chrome/Edge give the best frame rate.
- A webcam.
- An internet connection on first load. The MediaPipe library comes from `cdn.jsdelivr.net`, the models (~10 MB in total) from `storage.googleapis.com`, and the font from Google Fonts. An aggressive ad-blocker can block these.
- Enough room to step back. Jumping and "energy" work best with your upper body (ideally your hips too) in frame.

---

## How to play

1. **Start screen:** "You are the content now." Press **Go live**.
2. **Countdown:** 3, 2, 1. Use it to get your upper body in frame.
3. **Requests:** the algorithm asks for something in its speech bubble. Each request lasts up to **10 seconds** (the thin bar at the bottom of the bubble).
4. **Attention meter:** drains about **12% per second**, all the time. Doing what was asked refills it, and the more convincingly you do it, the faster it refills.
5. **Swiped away:** if attention hits 0, the thumb presses NEXT, a snarky line ("Nah.", "Seen it.", "Next.", "Boring.", "Is that it?") gets stamped on your post, the post slides up, and you get a new request.
6. **Liked:** do the request *strongly* (the ring around the heart icon fills up) and the algorithm likes you: a heart burst, a delighted face, +2 s bonus watch time, and a new request.
7. **Don't repeat yourself:** if your pose and expression barely change over ~3 seconds, attention drains **2.5× faster** and the algorithm says *"You did that already."* (Freeze is exempt.)
8. **After 60 seconds** you get your analytics.

### The requests

| Request | What it wants | How it's detected |
|---|---|---|
| **Show me energy!** | Move a lot | Average speed of your body landmarks is high |
| **Jump for me.** | Actually jump | Your hips (or shoulders, if hips are out of frame) rise quickly above their resting height, then come back down |
| **Hands up!** | Both hands above your head | Both wrists are above your nose |
| **Look surprised.** | Jaw drop + eyebrows up | `jawOpen` and `browInnerUp` face blendshapes are both high |
| **Smile for me.** | A big smile | `mouthSmileLeft` + `mouthSmileRight` blendshapes are high |
| **Freeze. Don't move.** | Stay perfectly still for 2 s | Movement stays very low for 2 seconds (moving restarts the timer) |
| **Get closer to the screen.** | Lean into the camera | Your face (or shoulders) take up a large part of the frame |

A request never repeats either of the two requests before it.

### The algorithm's moods

- **Bored:** the default. Half-closed eyes, chewing gum, slumped back.
- **Curious:** you're doing what it asked. Eyes open, one eyebrow raised.
- **Delighted:** it just liked you. Happy eyes, blushing, floating hearts.
- **Disgusted:** attention is low, or it just swiped you. Green tint, tongue out.

When attention drops below ~35%, its thumb starts twitching impatiently over the button.

### Scoring and analytics

- **Watch time** is the number of seconds the algorithm actually spent watching you, plus a bonus for each like. Transitions between posts don't count.
- The **end screen** shows your total watch time, how many times it swiped you away, how many times it liked you, your **best-performing request**, and what *"the algorithm now thinks you are"*: a label built from your best and second-best requests (e.g. Jump + Surprised → *"Jumpy. Mildly surprising."*).
- "Best-performing" is scored as `likes × 4 + seconds spent satisfying it − swipes × 1.5`.

---

## Controls

| Key / button | Action |
|---|---|
| **Go live** | Start a session (asks for camera access the first time) |
| **D** | Toggle the debug view |
| **Go live again** | Replay from the end screen (the camera stays on) |

---

## Tuning detection (CONFIG)

Every threshold lives in one `CONFIG` object at the top of the `<script>` in `index.html`. Detection is rule-based, so lighting, camera angle and distance all affect it. Expect to adjust a few values for your setup.

**Use the debug view (press D).** It shows a small mirrored webcam feed with pose landmarks (pink/cyan), face landmarks (yellow), the jump baseline (dashed green line), and each live measurement next to the threshold it's compared against. Do the move, read the number, and set the threshold just below it.

**Units used:**

- **sw** = shoulder widths. Movement speed and jump height are divided by your shoulder width, so they work the same whether you're near or far.
- **Face width** = fraction of the video frame's width (0–1).
- **Blendshapes** = MediaPipe face scores from 0 to 1.

**Key settings:**

| Setting | Default | Meaning |
|---|---|---|
| `sessionSec` | 60 | Session length |
| `requestMaxSec` | 10 | Max length of one request |
| `meter.drainPerSec` | 12 | Constant attention drain (% / s) |
| `meter.refillPerSec` | 34 | Refill while satisfied (scaled 50–100% by strength) |
| `likeHoldSec` | 1.2 | Seconds of *strong* performance needed for a like |
| `likeBonusSec` | 2 | Bonus watch time per like |
| `novelty.minDistance` | 0.12 | Below this, your pose counts as repetitive |
| `novelty.drainMultiplier` | 2.5 | Drain multiplier while repetitive |
| `energy.min` / `.strong` | 1.0 / 2.0 | Movement speed (sw/s) to satisfy / to earn a like |
| `freeze.maxMotion` | 0.3 | Max movement (sw/s) that still counts as frozen |
| `freeze.holdSec` | 2.0 | How long to stay frozen |
| `jump.rise` / `.strong` | 0.22 / 0.45 | Jump height (sw) to count / to earn an instant like |
| `jump.jumpsForLike` | 2 | Normal jumps needed for a like |
| `handsUp.margin` / `.strong` | 0.05 / 0.7 | How far (sw) the lower wrist must be above the nose |
| `surprised.jawOpen` / `.browInnerUp` | 0.3 / 0.3 | Both must pass (strong: 0.5 / 0.5) |
| `smile.sum` / `.strong` | 0.8 / 1.3 | Left + right smile scores (0–2) |
| `closer.faceWidth` / `.strongFaceWidth` | 0.22 / 0.32 | Face width as a fraction of the frame |

**Common fixes:**

- *Freeze fails even when I'm still:* raise `freeze.maxMotion` (check the `motion` value in debug while standing still).
- *Energy is too hard or too easy:* adjust `energy.min` to sit between your "standing" and "dancing" `motion` values.
- *Surprised never triggers:* many people's `browInnerUp` scores stay low. Lower `surprised.browInnerUp`.
- *Constant "You did that already.":* lower `novelty.minDistance` or raise `novelty.graceSec`.
- *Jumps aren't detected:* step back so your hips are visible, or lower `jump.rise`.
- *"Get closer" is too easy at a desk:* raise `closer.faceWidth`.

---

## Troubleshooting

| Message / symptom | Fix |
|---|---|
| **Open this page from localhost** | You opened the file directly. Run `python3 -m http.server 8000` and open `http://localhost:8000`. |
| **Camera access was blocked** | Click the camera icon in the address bar (or site settings), allow the camera, then press **Try again**. |
| **No camera found** | Connect a webcam or open the laptop lid, then try again. |
| **Your camera is busy** | Close other apps or tabs using it (Zoom, Meet, OBS…). |
| **The algorithm failed to load** | Models couldn't download. Check your connection, or disable blockers for jsDelivr / googleapis. |
| **"The algorithm can't see you."** | Step back and make sure your head, shoulders and hands are in frame with decent lighting. |
| Low frame rate / laggy | Close other heavy tabs. The page tries GPU first and falls back to CPU automatically, which is slower. Check fps in the debug view. |

---

## How it works

- **Single file:** `index.html` contains all the HTML, CSS and vanilla JavaScript. No build step, no dependencies to install.
- **Tracking:** MediaPipe `PoseLandmarker` (lite model, 33 body points) and `FaceLandmarker` (478 face points plus 52 blendshapes) run in `VIDEO` mode on each new webcam frame.
- **Signals:** landmarks are lightly smoothed, then turned into a few features (movement speed, hip rise, wrist height, blendshape scores, face width), plus a small "pose signature" sampled every 0.1 s for the repetition check.
- **Rendering:** the webcam video is hidden. You're drawn on a canvas as a neon stick figure whose face mirrors your blinks, smile and open mouth. The algorithm is an inline SVG whose moods are driven by CSS.
- **Game loop:** one `requestAnimationFrame` loop runs detection, game logic, drawing and the HUD.

**Files**

```
index.html   the whole app
README.md    this file
```

**Privacy:** the camera stream is processed in memory, in the browser tab. Nothing is recorded, stored or uploaded.
