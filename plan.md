# Meme Machine — Build Plan

One self-contained HTML page (`index.html`), no backend, no build step. Three
features stitched together with a shared shell. Built and tested in the
phase order below so there's always something demoable.

Stack: vanilla HTML/CSS/JS, MediaPipe Tasks Vision (FaceLandmarker +
HandLandmarker, loaded from CDN, runs entirely client-side), CodeMirror
(CDN) for the editor, Acorn (CDN) for JS syntax checking, Web Audio API for
the error sound. Everything runs in the browser off `getUserMedia` — no
server, no API keys, no model training.

---

## Phase 0 — Skeleton & camera check (~30 min)

**Goal:** one working page with navigation between three empty sections, and
proof the camera pipeline works before anything else is built on top of it.

- [x] Create `index.html` with a header/nav and three `<section>`s: "Mood
      Meme", "Vibe Check Editor", "6-7 Counter". Nav buttons toggle which
      section is visible (simple `display: none` swap, no router needed).
      (Built as a title-card home screen with three named buttons — "what
      do you meme?", "Are you coding yet?", "its 6 7 time" — routing to
      each section, per direct user request, instead of a persistent top
      nav.)
- [x] Add a shared `<video>` element (hidden, `autoplay muted playsinline`)
      and a `<canvas>` overlay, used by both Phase 2 and Phase 4.
- [x] Write `initCamera()`: calls `navigator.mediaDevices.getUserMedia({video: true})`,
      pipes the stream into the `<video>` element, resolves once
      `loadeddata` fires. Wrap in try/catch — on failure, show an inline
      message ("camera access needed for this part") instead of failing
      silently.
- [x] Add a big "Enable camera" button on first load (browsers block
      autoplay-without-gesture on some setups) that calls `initCamera()`.
      (Superseded per user request: the home-screen buttons themselves
      trigger the permission prompt directly — no separate "Enable
      camera" button.)
- [x] Sanity check: draw the raw video feed onto the canvas for a few
      seconds to confirm the permission + stream pipeline works end to end.
      (Runs continuously now, not just a few seconds — confirmed live and
      no longer freezes; feed is mirrored via `scaleX(-1)` for a natural
      selfie view.)
- [x] **Test:** load the page, click "Enable camera", confirm you see
      yourself on screen. This de-risks the one dependency all three
      features share. (Verified by user via screenshot — live feed
      renders, continuously updates, mirrored correctly.)

**Exit criteria:** page loads, nav switches sections, camera preview shows
your face on screen.

---

## Phase 1 — Vibe Check Editor (code editor + error sound) (~1 hr)

Built first because it has zero camera/CV dependency — fastest guaranteed
win, good for morale and for demoing early.

- [ ] Load CodeMirror from CDN (JS mode) into the "Vibe Check Editor"
      section. Pre-fill with a short broken snippet as a starting example.
- [ ] Load Acorn from CDN for parsing.
- [ ] On every keystroke (debounced ~300ms) or on an explicit "Run" button,
      run `acorn.parse(code, {ecmaVersion: 2020})` in a try/catch.
- [ ] On success: green border / "looks clean" status text.
- [ ] On `SyntaxError`: red border, show the error message near the line,
      and trigger the sound (see below). Don't spam the sound on every
      keystroke while mid-typing a valid partial statement — only re-fire
      when the error *state* changes from valid→invalid, not on every
      invalid keystroke.
- [ ] Build the "FAHHH" sound with the Web Audio API: a short descending
      sawtooth/square oscillator sweep (e.g. 400Hz → 80Hz over ~400ms) plus
      a touch of distortion — a synthesized "wrong buzzer," not a ripped
      audio clip (avoids copyright issues and any asset loading).
- [ ] Add a small counter: "syntax errors triggered: N" for comic effect.
- [ ] **Test:** type valid JS (silence, green), then break it — comma
      splice, unmatched bracket, stray keyword — confirm the sound fires
      once per new error, and clears when fixed.

**Exit criteria:** typing broken JS reliably triggers the buzzer sound and
visual error state; fixing it clears both.

---

## Phase 2 — Mood Meme (face expression → meme image) (~1.5–2 hrs)

Ported from a reference implementation (github.com/kristelTech/make_me_a_meme,
Python/OpenCV) rather than built from scratch — its feature-extraction and
similarity-matching approach is better than a hand-tuned threshold table, and
it's a near-direct port since MediaPipe Tasks Vision (JS) exposes the same
478 face landmark indices the Python version uses.

**Assets: our own 6 images** (`assets/`) — the Gibraltar "reaction monkey"
meme series, not the source repo's human meme photos:

| File | Expression/gesture to detect |
|---|---|
| `praying_monkey.jpg` | eyes closed + calm mouth + hands together, not raised |
| `unbothered_lion_chimp.jpg` | neutral/calm face + hand near chin |
| `shocked_monkey.jpg` | mouth wide open + eyes wide + hands near chest |
| `flex_pointing_monkey.jpg` | big smile, teeth showing + one hand raised, finger up |
| `wink_smirk_monkey.jpg` | one eye closed (wink) + smirk + hand near chin/mouth |
| `pondering_monkey.jpg` | neutral/curious face + finger touching mouth |

**Important deviation from the source repo — tested and confirmed before
building on it:** the source repo auto-extracts each meme's "feature
fingerprint" by running the *same* human face landmarker on the meme image
itself. That only works because its memes are human faces. Ours aren't —
I ran a face detector against all 6 monkey images to check, and results
were inconsistent (0 faces found on 2 of them, spurious multi-detections
on others from fur texture), and one image (`unbothered_lion_chimp.jpg`)
is a side-profile illustration, not a photo, which face landmarkers
generally can't fit at all. So: **we don't run the face/hand landmarker on
the meme images.** Only your live webcam feed gets landmarked (that's a
real human face — no issue there). Each meme's target feature vector is
hand-authored based on what it visually shows, then tuned live against
your own face during testing.

- [ ] Load `@mediapipe/tasks-vision` from CDN, initialize a `FaceLandmarker`
      (`outputFaceBlendshapes: false` — not needed, we use raw landmarks)
      and a `HandLandmarker`, both `runningMode: "VIDEO"`.
- [ ] Port `_compute_features()` to JS as `computeFeatures(faceLandmarks, handResult)`,
      operating on the same landmark indices as the source repo:
      - eye-aspect-ratio (EAR) per eye from `LEFT/RIGHT_EYE_UPPER/LOWER`
        indices, averaged → `eye_openness`, plus `eyes_symmetry` (abs
        difference between the two).
      - mouth-aspect-ratio from landmarks 13/14 (vertical) vs 61/291
        (horizontal) → `mouth_openness`; inner-mouth width (78/308) over
        outer width → `mouth_width_ratio`.
      - eyebrow height: mean y of `LEFT/RIGHT_EYEBROW` indices vs. mean y of
        that eye's landmarks → `eyebrow_height`, plus `brow_symmetry`.
      - `mouth_elevation`: nose tip (landmark 4) y minus mouth-center y.
      - hand features from `HandLandmarker` result: `num_hands`, and
        `hand_raised` (1 if any wrist/middle-fingertip landmark is above a
        y-threshold relative to face center/top), plus a `hand_near_face`
        flag (any fingertip landmark within a small radius of the chin/mouth
        region) — needed to tell `unbothered_lion_chimp` and
        `wink_smirk_monkey` apart, since both involve a hand near the face.
      - derived scores exactly as source: `surprise_score`, `smile_score`,
        `concern_score`, `cheers_score` (products of the above).
- [ ] **Author the 6 target feature vectors by hand** (JS object, one per
      meme file above), using the same `feature_keys` as
      `computeFeatures()` outputs. Starting-point logic per image:
      - `praying_monkey`: low `eye_openness` (near 0), low `mouth_openness`,
        `hand_raised = 0`, `hand_near_face = 0` (hands are together at
        chest, not at face or overhead).
      - `unbothered_lion_chimp`: mid `eye_openness`, low `mouth_openness`,
        `hand_near_face = 1`, `hand_raised = 0`.
      - `shocked_monkey`: high `eye_openness`, high `mouth_openness`,
        `hand_raised = 0`, `hand_near_face = 0` (hands clasped at chest).
      - `flex_pointing_monkey`: high `smile_score` (wide `mouth_width_ratio`,
        low `mouth_openness`), `hand_raised = 1`.
      - `wink_smirk_monkey`: `eyes_symmetry` high (one eye open, one closed),
        mid `smile_score`, `hand_near_face = 1`.
      - `pondering_monkey`: mid `eye_openness`, low-mid `mouth_openness`,
        `hand_near_face = 1`, slight `eyebrow_height` raise — closest to
        `unbothered_lion_chimp` and `wink_smirk_monkey`, so this is the
        trio to watch closely when tuning (see test step below).
      These are starting guesses, not measurements — expect to nudge the
      actual numbers once you're testing live against your own face.
- [ ] Port `computeSimilarity()`/`findBestMatch()`: same `feature_weights`
      and `feature_factors` arrays as the source repo to start, same
      `sum(weights * exp(-|diff| * factors))` scoring — trivial in JS, no
      numpy needed for 6 reference vectors. Expect to re-weight
      `hand_near_face` and `eyes_symmetry` higher than the source repo did,
      since those are what separate our closest-together trio above.
- [ ] Per live frame: compute the viewer's feature vector, run
      `findBestMatch()` against the 6 hand-authored vectors, display the
      winning meme image + name + match score.
- [ ] Add a hold/debounce (~400–600ms) so the match doesn't flicker between
      two close-scoring memes.
- [ ] **Test, in this order (easy separations first, hard ones last):**
      1. Praying (eyes closed) vs. shocked (wide eyes+mouth) vs. flex
         (big smile+hand up) — these three should separate immediately,
         they differ on multiple features at once.
      2. The hard trio — unbothered / wink-smirk / pondering, all "hand
         near face, calm-ish mouth" — this is where you'll spend actual
         tuning time. Exaggerate the wink deliberately; exaggerate the
         eyebrow-raise for pondering; keep the chin-hand still and neutral
         for unbothered. Adjust feature weights/targets until each wins
         clearly for its intended pose.
      3. Full run-through, all 6, checking nothing flickers on a resting
         neutral face between images.

**Exit criteria:** all 6 expression/gesture combos reliably match their
intended image with a clear score gap over the others, and holding a
neutral face doesn't flicker between matches.

---

## Phase 3 — "6-7" Hand Counter (~2–3 hrs, most fiddly phase)

- [ ] Initialize a `HandLandmarker` (same MediaPipe package) with
      `runningMode: "VIDEO"`, `numHands: 2`.
- [ ] Per frame, read the wrist landmark's `y` coordinate (landmark index
      0) for each detected hand. Normalize (0=top,1=bottom of frame).
- [ ] Maintain a rolling buffer (e.g. last ~1.5s of y-values per hand) and
      run simple peak/valley detection: a "rep" = a local max followed by
      a local min (or vice versa) whose amplitude exceeds a minimum
      threshold (filters out small jitter from natural stillness).
- [ ] Count a rep only once per full up-down cycle (state machine:
      `idle → going_up → at_top → going_down → at_bottom → idle`, using
      direction reversal). This avoids double-counting on noisy frames.
- [ ] Maintain a rolling 60-second window of timestamped rep events;
      display "reps in the last 60s" live (recompute each frame from the
      timestamp list, dropping events older than 60s).
- [ ] Add a manual "reset counter" button and a big animated number
      display for the demo moment.
- [ ] Tune amplitude/timing thresholds against your own arm-pump speed —
      expect this to be the one part you iterate on the most.
- [ ] **Test:** do slow deliberate reps, count by hand, compare to the
      on-screen counter; do it fast; hold still and confirm it doesn't
      free-count from small movements.

**Exit criteria:** deliberate up-down hand motion counts accurately
(±1) against a manual count, and stillness doesn't accumulate false counts.

---

## Phase 4 — Integration & polish (~1–1.5 hrs)

- [ ] Make sure only one MediaPipe task runs at a time (Phase 2 and Phase
      3 shouldn't both be running full inference when their section isn't
      visible — pause the `requestAnimationFrame` loop for the hidden
      section to save CPU/battery).
- [ ] Consistent visual theme across all three sections (pick a palette,
      apply it once — don't hand-tune colors per section).
- [ ] Landing/home view: a title screen with the three feature names and
      a "which one do you want" entry point, instead of dumping straight
      into section 1.
- [ ] Sound/UX pass: transition animations between meme swaps, a little
      flourish on hitting counter milestones, favicon, page title.
- [ ] Error states: no camera permission, no hands/face detected for a
      while, browser lacking `getUserMedia` support — friendly fallback
      messages instead of a blank screen.
- [ ] Final run-through of all three sections back to back, on the actual
      device/browser you'll demo on.

**Exit criteria:** a stranger can open the page, click through all three
features, and understand what to do without you narrating.

---

## Cut list (if time runs out)

If the night gets short, cut in this order and it still demos fine:

1. Phase 4 polish → ship it rougher, narrate over the rough edges live.
2. Phase 3 accuracy tuning → looser thresholds, call it "vibes-based
   counting."
3. Phase 2's hard trio (unbothered / wink-smirk / pondering) → drop to
   just the 3 easy-to-separate memes (praying, shocked, flex-pointing) if
   the hand-near-face tuning is eating too much time. Still 3 working
   matches beats 6 flaky ones.

Never cut Phase 0 or Phase 1 — they're the cheapest and most reliable wins.

---

## Open decisions to make before/while building

- Whether "6 7" counting requires both hands or just one (spec above
  assumes tracking both, counting either).
- Where this ends up living for the demo — a shareable hosted link vs.
  running locally off the HTML file (affects nothing in the build, only
  how you show it).