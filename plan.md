# Meme Machine — Build Plan

One self-contained HTML page (`index.html`), no backend, no build step. Two
features stitched together with a shared shell (originally scoped as three —
see Phase 3's cut note below). Built and tested in the phase order below so
there's always something demoable.

Stack: vanilla HTML/CSS/JS, MediaPipe Tasks Vision (FaceLandmarker +
HandLandmarker, loaded from CDN, runs entirely client-side), CodeMirror
(CDN) for the editor, Pyodide for Python syntax/execution, a user-provided
MP3 for the error sound. Everything runs in the browser off `getUserMedia`
— no server, no API keys, no model training.

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

## Phase 1 — Vibe Check Editor (Python editor + error sound) (~2–2.5 hrs)
 
Built second (after the camera check) because it has zero camera/CV
dependency of its own — still a fast, self-contained feature to get fully
working before moving into the two CV-heavy phases.
 
**Python-only, syntax errors only** (deliberately descoped from full
linting/undefined-name checks — see discussion above). Uses **Pyodide**
(real CPython compiled to WebAssembly, running client-side) instead of a
JS-only parser like Acorn, since the checking has to understand actual
Python syntax. Sound is a **user-provided MP3 file** (`assets/fahh-sound-effect.mp3`),
not synthesized — swap out the earlier Web-Audio-oscillator idea.
 
- [x] Load CodeMirror from CDN with **Python mode** (not JS mode) into the
      "Vibe Check Editor" section, for correct syntax highlighting.
      Pre-fill with a short broken Python snippet as a starting example.
      (Also added the closebrackets addon for VS-Code-style auto-closing
      of `()`/`[]`/`{}`/quotes, per direct user request — not in the
      original spec but a natural editor-feel addition.)
- [x] Load CodeMirror's **lint addon/extension** — this is what actually
      draws the squiggly underline given a list of
      `{line, ch, message, severity}` diagnostics; we just need to supply
      that list from Pyodide's errors.
- [x] Load Pyodide from its CDN script tag and call `loadPyodide()`
      **immediately on page load** (not gated behind opening the Vibe
      Check section or clicking anything) — this is the pre-load-before-
      the-demo requirement. Show a small status indicator ("Python
      loading…" → "Python ready ✓") somewhere visible on the page so it's
      obvious when it's safe to start the demo, since the download +
      init takes a few seconds. (Also added a distinct "Python failed to
      load — reload the page" error state, so a stuck/failed CDN load is
      visibly different from a slow one.)
- [x] Disable/grey out the editor's syntax-checking and Run button until
      `loadPyodide()` resolves, so nothing silently no-ops while it's
      still loading.
- [x] **Live syntax check** (debounced ~300ms after the last keystroke):
      run `pyodide.runPython("compile(<code>, '<input>', 'exec')")` (or
      equivalent via `ast.parse`) wrapped so a `SyntaxError` is caught in
      JS with its `lineno`/`offset`/message intact. Feed that into the
      lint addon to draw the squiggly line at the actual failing
      location — not just a generic "there's an error somewhere" banner.
      (Switched from raw `ast.parse` to `codeop.compile_command` — the
      same check the real Python REPL uses — because `ast.parse` can't
      tell "still typing a multi-line block/bracket/string" apart from an
      actual syntax error, which caused false-positive buzzer triggers on
      completely ordinary in-progress typing.)
- [x] **Error-state transition tracking**, same principle as originally
      planned: only trigger the sound when the code goes from
      valid→invalid, not on every keystroke while it's already invalid
      (otherwise it fires constantly while mid-typing a multi-line broken
      block). Clear the squiggle and reset state the moment it re-parses
      clean. (Refined further per user testing: the sound decision is now
      on its own ~1.5s debounce separate from the live squiggle, so it
      only fires once you've paused typing, not mid-keystroke; and it now
      fires on every *new distinct* error signature — not just the first
      valid→invalid flip — while still not repeating for the same
      unchanged error. The very first lint pass against the pre-filled
      broken starting snippet is also suppressed from playing the sound,
      since that's not something the user typed.)
- [x] **Sound playback**: `<audio id="fahh" src="assets/fahh.mp3">` in the
      page; on each new syntax error, `fahh.currentTime = 0; fahh.play()`
      so rapid re-triggers restart cleanly instead of overlapping oddly
      or getting ignored.
- [x] **"Run" button** (separate from the live syntax check): executes the
      current code via `pyodide.runPythonAsync(code)`, with
      `pyodide.setStdout`/`setStderr` wired to append into an on-page
      output panel. A successful run shows printed output; a runtime
      exception (as opposed to a syntax error caught live while typing)
      shows the real Python traceback in that same output panel, styled
      as an error. (Known accepted limitation, documented in the UI: no
      Web Worker, so a genuine infinite loop freezes the tab — a caption
      under the Run button says so. Also confirmed `input()` doesn't work
      since Pyodide has no wired-up stdin; user declined a browser-prompt
      workaround, so this stays unimplemented.)
- [x] Add a small counter: "syntax errors triggered: N" for comic effect
      (kept from the original plan).
- [x] **Test, in this order:**
      1. Load the page fresh, confirm the "Python loading…" → "ready"
         indicator actually flips, and that typing/Run are inert before
         it does.
      2. Type valid Python (no squiggle, silence), then break it a few
         different ways — bad indentation, unclosed bracket/paren,
         dangling `:` , stray keyword — confirm the squiggle appears at
         the right line and the sound fires exactly once per new error.
      3. Fix the error back to valid — confirm the squiggle clears and no
         extra sound fires.
      4. Hit Run on valid code that prints something — confirm output
         shows up in the panel.
      5. Hit Run on code that's valid syntax but throws at runtime (e.g.
         `1/0`) — confirm the real traceback shows in the output panel
         (this is a separate path from the live squiggle/sound, since
         it's not a syntax error).
      6. Reload the page, immediately mash the keyboard before the
         "ready" indicator flips — confirm nothing breaks or throws in
         the console while Pyodide is still loading.
      (All verified by user via live testing, including two rounds of bug
      reports that were fixed and reconfirmed: a broken gutter/line-number
      layout from a hidden-container CodeMirror init, a dark-theme pass,
      and the false-positive sound triggering on normal multi-line typing.)
**Exit criteria:** Pyodide finishes loading automatically on page open
with a visible ready-state; typing broken Python reliably draws an
accurate squiggly line and plays the FAHHH sound exactly once per new
error; fixing the code clears both; Run executes valid code and shows
real output, and shows a real traceback for runtime (non-syntax) errors.
 

 
---
## Phase 2 — Mood Meme (face expression → meme image) (~1.5–2 hrs)

Originally scoped as a port of a reference implementation
(github.com/kristelTech/make_me_a_meme, Python/OpenCV), but that source repo
was never actually added to this project's workspace — its exact
`_compute_features()`/`computeSimilarity()` formulas and `feature_weights`/
`feature_factors` numbers were never available to copy. What's actually
implemented is an **original reauthoring** built from this file's own feature
*descriptions* below (EAR-based eye openness, MAR-based mouth openness, the
per-image bullet points, etc.), using the same MediaPipe Tasks Vision (JS)
478-point face landmark indices the Python version would have used. This is
consistent with this section's own "starting guesses, not measurements"
framing further down — reauthoring against the spec was judged preferable to
pulling in the actual unreviewed source repo mid-project.

**Assets: 4 images** (`assets/`) from the Gibraltar "reaction monkey" meme
series, not the source repo's human meme photos. Originally scoped as 6 —
see the deviation note below.

| File | Expression/gesture to detect |
|---|---|
| `praying_monkey.jpg` | eyes closed + calm mouth + hands together, not raised |
| `unbothered_lion_chimp.jpg` | neutral/calm face + hand near chin |
| `shocked_monkey.jpg` | mouth wide open + eyes wide + hands near chest |
| `flex_pointing_monkey.jpg` | big smile, teeth showing + one hand raised, finger up |

`wink_smirk_monkey.jpg` and `pondering_monkey.jpg` are still in `assets/`
but no longer wired into `MOOD_TARGETS` — see deviation note below.

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

- [x] Load `@mediapipe/tasks-vision` from CDN, initialize a `FaceLandmarker`
      (`outputFaceBlendshapes: false` — not needed, we use raw landmarks)
      and a `HandLandmarker`, both `runningMode: "VIDEO"`. (Loaded via a
      dynamic `import()` inside `initVision()`, called the first time the
      Mood Meme section opens — genuinely lazy, and wrapped in try/catch so
      a load failure can't take down Phase 0's camera code or Phase 1's
      Pyodide/editor code sharing the same script tag. Package version
      `@mediapipe/tasks-vision@1.0.1` and both model asset URLs verified
      live via `curl` before committing to them. GPU delegate attempted
      first with a CPU fallback on failure.)
- [x] Reauthored `computeFeatures(face, hands)` (see the reauthoring note
      above — not a literal port), operating on canonical MediaPipe face
      mesh indices:
      - eye-aspect-ratio (EAR) per eye from six-point index sets, averaged
        → `eye_openness`, plus `eyes_symmetry` (abs difference between the
        two). Already scale-invariant (ratio of two face-relative
        distances) — no extra normalization.
      - mouth-aspect-ratio from landmarks 13/14 (vertical) vs 61/291
        (horizontal) → `mouth_openness`; inner-mouth width (78/308) over
        outer width → `mouth_width_ratio`. Also already scale-invariant,
        left un-normalized.
      - eyebrow height: mean y of `LEFT/RIGHT_EYEBROW` indices vs. mean y of
        that eye's landmarks → `eyebrow_height`, plus `brow_symmetry`. These
        are raw single distances (not ratios), so both are divided by
        interocular distance to stay stable across camera distance.
      - `mouth_elevation`: nose tip (landmark 4) y minus mouth-center y,
        also interocular-normalized for the same reason.
      - hand features from `HandLandmarker` result (`.landmarks`, not
        `.handLandmarks` — verified against the actual API to avoid a known
        mix-up): `num_hands`, `hand_raised` (1 if any wrist/middle-fingertip
        landmark is above a y-threshold relative to face top), and
        `hand_near_face` (any fingertip within an interocular-scaled radius
        of the chin/mouth region) — needed to tell `unbothered_lion_chimp`
        and `wink_smirk_monkey` apart.
      - derived scores: `surprise_score`, `smile_score` — original authoring
        (see reauthoring note; `concern_score`/`cheers_score` from the
        unseen source repo were dropped since nothing in this file's own
        6-target descriptions actually references them).
- [x] **Author the 6 target feature vectors by hand** (JS object, one per
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
- [x] **Deviation — in-app calibration instead of hand-edited constants:**
      live testing confirmed the guessed vectors above don't separate
      reliably (same meme dominating, flickering, hard trio confused with
      each other and with the easy poses) — expected, since the last test
      step below was never actually run before this was flagged. Rather
      than re-guessing numbers offline again, added a **"Calibrate my
      poses" flow** in the Mood Meme section: walks through all 6 memes,
      captures ~1.5s of your real `computeFeatures()` output per pose on
      "Hold pose & Capture", averages it, and uses that as the target
      vector — replacing only the guessed values, not the scoring scheme
      (`FEATURE_WEIGHTS`/`FEATURE_FACTORS`/confidence threshold unchanged).
      Persists to `localStorage` so it survives reloads; a "Recalibrate"
      button re-runs it any time.
- [x] **Deviation — cut down to 4 memes:** after calibration, praying,
      unbothered, shocked, and flex/pointing all matched reliably, but
      wink-smirk and pondering stayed unreliable even after recalibration
      (both share "hand near face, calm-ish mouth" with unbothered, and
      wink-smirk's `eyes_symmetry` signal proved hard to separate live).
      Given the scope of this project, descoped `wink_smirk_monkey.jpg` and
      `pondering_monkey.jpg` — removed both entries from `MOOD_TARGETS` and
      their now-unused `eyes_symmetry`/`eyebrow_height` weighting from
      `FEATURE_WEIGHTS`/`FEATURE_FACTORS`. The 4 remaining memes are the
      ones that were actually verified working. Image files are left in
      `assets/` unused, not deleted.
- [x] `computeSimilarity()`/`findBestMatch()`: `sum(weights * exp(-|diff| *
      factors))` scoring exactly as specified (weights/factors are original
      starting values, not ported from the unseen source repo — see
      reauthoring note above) — trivial in JS, no numpy needed for 6
      reference vectors. `hand_near_face` and `eyes_symmetry` already given
      weight 1 (full weight) from the start, ready to be tuned further live.
- [x] Per live frame: compute the viewer's feature vector, run
      `findBestMatch()` against the 4 calibrated target vectors, display the
      winning meme image + name + match score.
- [x] Add a hold/debounce (~400–600ms) so the match doesn't flicker between
      two close-scoring memes. (Implemented at 500ms.)
- [x] **Test:** praying (eyes closed) vs. unbothered (calm face + hand near
      chin) vs. shocked (wide eyes + open mouth) vs. flex/pointing (big
      smile + hand up) — all four separate reliably after calibration and
      the `findBestMatch`/`hand_raised` fixes above; holding a neutral face
      doesn't flicker between matches.

**Exit criteria:** all 4 remaining expression/gesture combos reliably match
their intended image with a clear score gap over the others, and holding a
neutral face doesn't flicker between matches.

---

## Phase 3 — "6-7" Hand Counter — CUT

Dropped from scope entirely. The project ships as two features: Mood Meme
(Phase 2) and Vibe Check Editor (Phase 1). The home screen, camera-target
branching, and `section-counter` markup/JS for this phase have been removed
from `index.html`. Original spec (hand rep counting via wrist-y peak/valley
detection, rolling 60s window, reset button) is left out of this document
since it was never built.

---

## Phase 4 — Integration & polish (~1–1.5 hrs)

- [ ] Consistent visual theme across both sections (pick a palette,
      apply it once — don't hand-tune colors per section).
- [ ] Landing/home view: a title screen with the two feature names and
      a "which one do you want" entry point, instead of dumping straight
      into section 1.
- [ ] Sound/UX pass: transition animations between meme swaps, favicon,
      page title.
- [ ] Error states: no camera permission, no hands/face detected for a
      while, browser lacking `getUserMedia` support — friendly fallback
      messages instead of a blank screen.
- [ ] Final run-through of both sections back to back, on the actual
      device/browser you'll demo on.

**Exit criteria:** a stranger can open the page, click through both
features, and understand what to do without you narrating.

---

## Cut list (if time runs out)

If the night gets short, cut in this order and it still demos fine:

1. Phase 4 polish → ship it rougher, narrate over the rough edges live.
2. Phase 2's hard trio (unbothered / wink-smirk / pondering) → drop to
   just the easy-to-separate memes if the hand-near-face tuning is eating
   too much time. (This actually happened — see Phase 2's deviation note:
   wink-smirk and pondering were cut, landing on 4 working memes —
   praying, unbothered, shocked, flex-pointing — instead of 6 flaky ones.)

Never cut Phase 0 or Phase 1 — they're the cheapest and most reliable wins.

---

## Open decisions to make before/while building

- Whether "6 7" counting requires both hands or just one (spec above
  assumes tracking both, counting either).
- Where this ends up living for the demo — a shareable hosted link vs.
  running locally off the HTML file (affects nothing in the build, only
  how you show it).