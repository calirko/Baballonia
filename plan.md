# Headset-slip robust gaze

## Problem

When the headset moves on the face (smiling/frowning lifts or drops the facial interface, head motion
during calibration, re-seating the headset between sessions), the eye shifts in the camera image while
the real gaze stays put. The gaze net has learned "pupil at pixel X -> angle Y", so slip reads as gaze
movement and the output wobbles.

The labels are already correct under slip: the reticle is world-locked and the label is computed from
the HMD eye pose every frame (`BabbleCalibration/Scripts/Routines/ReticleRoutine.cs:113`), so it is
headset-relative and follows the slip. The problem is purely appearance: the training data almost never
contains *the same gaze under different headset positions*, so the model never learns to read gaze from
the eye's shape (pupil vs. eye corners/lids, iris ellipse) instead of its pixel position.

Constraint: no extra smoothing on the gaze output - precision must stay the same.

Repos involved:
- Baballonia (this repo) - calibration steps, capture, inference
- [BabbleCalibration](https://github.com/Project-Babble/BabbleCalibration) - VR overlay, reticle routines
- [BabbleTrainer](https://github.com/Project-Babble/BabbleTrainer) - trainer (`trainer_distsampler.py` is what ships, per `BabbleTrainer.spec`)

### Repo / branch setup

Work happens on forks under `calirko`, one feature branch per repo, PR'd back upstream when ready.

| Repo | Fork | Branch | Based on |
|---|---|---|---|
| Baballonia | [calirko/Baballonia](https://github.com/calirko/Baballonia) | `feat/slip-robust-gaze` | `expr-dev` |
| BabbleCalibration | [calirko/BabbleCalibration](https://github.com/calirko/BabbleCalibration) | `feat/slip-robust-gaze` | `experimental-expressions` |
| Baballs (native `Overlay/flags.h`) | [calirko/Baballs](https://github.com/calirko/Baballs) | `feat/slip-robust-gaze` | `main` |
| BabbleTrainer | not forked yet (only needed for step 2) | `feat/slip-robust-gaze` | tbd |

Remotes: `origin` = fork, `upstream` = Project-Babble. Keep in sync with
`git fetch upstream && git rebase upstream/expr-dev`.

---

## Step 1 - Slip calibration pass (now)

A new calibration step where the reticle stops at fixed points and the user keeps looking at it while
moving the headset on their face.

### User flow

1. Tutorial: "Keep looking at the dot. Gently move the headset up, down, sideways and make faces."
2. Runs after the normal gaze pass (which is unchanged - moving dot, follow with the head).
3. Reticle appears at ~5 points in turn: center, up, down, left, right (relative to where the head faces
   when each point starts), and stays fixed in the world while held.
4. At each point (~5-6 s): eyes locked on the dot, user slowly pushes/tilts the headset with their hands,
   then smiles / frowns / squints while still fixating. Then the dot jumps to the next point.
5. Small head motion is fine - labels follow the HMD pose. The only rule is: eyes stay on the dot.

### BabbleCalibration

- [x] Add `SlipRoutine : ReticleRoutine`
  - `protected override bool SpeedCheck => false;` (same as `ConvergenceRoutine`)
  - Holds the reticle at fixed points, ~6 s each, instead of moving it
  - Keeps sending `HmdPositionalDataPacket` every frame (same `TransformToReticule` path), so labels are per-frame
  - Prompt text near the dot: "Keep looking - move the headset"
  - Optional: suppress sending packets for the first ~0.5 s at each point (eye still saccading to the dot)
- [x] Point placement: **spawn head-relative, hold world-locked**
  - At the start of each point, read `Backend.HeadTransform()` and place the dot 2 m forward of it,
    rotated by that point's offset: center, then ~12-15 deg up / down / left / right
  - Freeze that world transform for the whole hold; next point is computed from the head pose at that moment
  - Why: the dot always spawns inside the FOV regardless of where the user is facing (15 deg is well within
    a ~90+ deg FOV), and because it's world-locked the eye stays still while the headset slides - no
    re-fixation saccades in the frames we care about
  - Rejected alternatives: one world dot for the whole pass (user must hold eccentric gaze while
    fidgeting, no coverage guarantee); head-locked dot (labels still correct, but the dot moves with every
    slip and the eye has to chase it)
- [x] Head-rotation guard (inverse of the speed check): if the head rotates more than ~20 deg from the
      pose at point start, stop sending packets and show "Keep your head still - move only the headset"
      until it's back in range
- [x] Optional guided prompts per point so slip directions are covered deliberately, e.g. cycle
      "push up" -> "push down" -> "tilt" -> "smile" -> "frown" during the hold
- [x] Register the routine name (`"slip"`) and add a tutorial (`"sliptutorial"`) (text, all languages in `translations.csv`)
- [x] ~~Bump OverlaySDK~~ not needed: routine names are free-form strings

### Baballonia

- [x] `src/Baballonia.CaptureBin.IO/CaptureFlags.cs`: add `FLAG_SLIP` (alias a free routine bit, e.g.
      `FLAG_ROUTINE_3`) so the trainer can identify these frames later. Keep in sync with Baballs `Overlay/flags.h` (done there too).
- [x] `src/Baballonia.Desktop/Calibration/ICalibrationRoutine.cs`, in `BasicAllCalibration` (and a shorter
      version in `BasicAllCalibrationQuick`), after the gaze/gazeexpr passes:
      ```csharp
      new BaseTutorialStep("sliptutorial", TimeSpan.FromSeconds(10)),
      eyeCaptureStepFactory.CreateGazeExpression("slip",
          CaptureFlags.FLAG_GOOD_DATA | CaptureFlags.FLAG_VERSION_BIT1 | CaptureFlags.FLAG_SLIP,
          TimeSpan.FromSeconds(30), lid: 1),
      ```
      `GazeExpressionCaptureStep` already forces the gaze-valid bit and gates on fresh positional data.
      No `FLAG_IN_MOVEMENT` - the reticle is static.
- [x] Add `"slip.bin"` to every `MergeBinsStep` that includes this step.
- [ ] Optional, low cost: stamp real timestamps instead of `DateTimeOffset.Now` for both label and frames
      in `FrameCollector.cs:25` so label/frame skew becomes measurable.

### BabbleTrainer

- No change required: slip frames carry the gaze bit, so they go straight into gaze training.
- Nice to have: oversample `FLAG_SLIP` frames in the gaze batcher (they're a small share of the data).

### Guidelines / pitfalls

- Move the headset **slowly** - fast jerks cause motion blur and label/frame time skew.
- Don't push so far that the eye leaves the camera FOV; stay within normal-wear slip range.
- Reticle is 2 m away, so a few mm of headset translation is negligible label error (<0.2 deg).

### How to validate

- Before/after: run inference while fixating a static dot and deliberately smiling / pushing the headset;
  log gaze output std-dev. Should drop clearly with no change to normal saccade responsiveness.
- Compare loss on a held-out slip segment between a model trained with and without `slip.bin`.

---

### Status (2026-10-15)

- Implemented and committed on `feat/slip-robust-gaze` in Baballonia, BabbleCalibration and Baballs.
- Overlay tested on the desktop debug backend only (scripted head): 5 points spawn head-relative 13 deg / 2 m,
  0.5 s settle, head guard at 20 deg, prompts cycle, `RoutineFinished("slip")` fires.
- Not yet tested: Baballonia + overlay together, and on a real headset (Quest via SteamVR on Windows).
- **Ship Baballonia and overlay together**: an overlay without the `slip` routine never sends
  `RoutineFinished`, so calibration hangs at that step.
- Overlay: Godot 4.6.1 .NET (matches `Godot.NET.Sdk/4.6.1`). Flags go without `--`
  (e.g. `--use-debug --test-routines`); `--test-routines` auto-starts the routine hardcoded in `MainScene.cs`.
- Windows test build from Linux: `IsOSPlatform(...)` and `'$(OS)' == 'Windows_NT'` in the csproj files pick
  native deps / the `WINDOWS` define by build host, so force them in a throwaway copy before
  `dotnet publish -r win-x64`, then put the exported overlay + trainer in `Calibration/Windows/{Overlay,Trainer}`.

---

## Step 2 - More realistic slip augmentation (later, BabbleTrainer)

Current augmentation (`trainer_distsampler.py:278`, called at `:982`): 30% chance, one affine
(+/-22 px shift on 128 px, +/-12 deg rotation, +/-10% scale) applied identically to all 4 temporal frames.

- [ ] **Per-temporal-frame jitter**: add a small extra shift (1-3 px) that differs between the 4 stacked
      frames. Real slip happens *inside* the temporal window; today the model never sees that, so it
      reads inter-frame slip as eye motion.
- [ ] Re-tune distribution: smaller shifts (realistic slip is a few px) applied more often (~60-70%),
      keep the large ones rare.
- [ ] Optional: geometry-aware labels - when the augmentation applies a rotation that represents
      camera rotation, adjust the gaze label by the equivalent angle instead of keeping it fixed.
- [ ] Oversample `FLAG_SLIP` frames (from step 1).

---

## Step 3 - Per-session re-anchor (later, Baballonia)

Handles re-seating the headset between sessions, which no amount of training covers exactly.

- [ ] At the end of calibration, save a reference eye image (center-dot fixation, neutral face) next to
      the tuned model.
- [ ] Re-anchor action: 2 s "look at the center" -> estimate image offset vs. the reference using only
      gaze-independent features (eye corners, lid edges, skin; mask out the iris), e.g.
      `Cv2.PhaseCorrelate` / `FindTransformECC`.
- [ ] Apply the offset as a constant crop correction in
      `src/Baballonia/Services/Inference/Transformers/ImageTransformer.cs`. Constant per session -> zero
      cost to responsiveness. Calibration frames go through the same transform, so train/infer stay
      consistent.
- [ ] Trigger: app start and on demand. Wire up the existing but unused `AppSettings_RecenterAddress`
      OSC setting (`AppSettingsViewModel.cs:23`).

Possible follow-up (only if 1-3 aren't enough): continuous slow alignment - same offset estimate run
live, heavily low-passed (slip is 100s of ms, saccades are 30-50 ms), applied to the crop. The gaze
output itself stays unfiltered.
