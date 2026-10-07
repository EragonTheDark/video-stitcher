# ScoutCam 720p30 Calibration Handoff

Calibrate ScoutCam’s existing two-camera recording for offline panoramic stitching.

Repository: `video-stitcher`

Inputs:

- `videos/scoutcam-20260920-173128/left.mp4`
- `videos/scoutcam-20260920-173128/right.mp4`
- `videos/scoutcam-20260920-173128/session.json`

Read `AGENTS.md` and these reports first:

- `docs/video-stitcher-evaluation.md`
- `docs/offline-stitcher-extraction-assessment.md`
- `docs/stitching-processing-architecture.md`

## Goal

Produce an evidence-backed calibration for THIS recorded camera pair, plus short stitched validation clips. Determine whether the footage and available lens information are sufficient; do not treat generating `match.json` as proof of successful calibration.

## Context

- The session manifest identifies `gstreamer-cedar-h264-720p30`.
- ScoutCam now records at 1080p30, but this calibration task concerns the older 720p30 files.
- Cameras are generic USB “4K U3 Camera” devices; their actual lens profiles are not established.
- Reco uses saved lens intrinsics/distortion plus optimized relative camera placement.
- The recordings are expected to be video-only; verify that.
- Reco’s calibration command itself initializes wgpu. CPU feature matching does not make the entire calibration workflow CPU-only.
- The current shell previously lacked Cargo and FFmpeg/ffprobe on PATH. Inspect available host/Windows tools and GPU support before choosing an execution environment.

## Constraints

- Preserve both MP4s and `session.json` byte-for-byte.
- No production source changes, Pi service/configuration changes, new recordings, or AWS provisioning.
- Use the local copies. Do not run calibration or heavy media processing on the production Pi.
- Keep all generated artifacts outside the source session directory: `calibration-results/scoutcam-20260920-173128/`.
- Do not commit videos, generated media, or large debug artifacts.
- Use existing project tools where possible. Explain any missing prerequisite precisely.
- Do not silently install system-wide software or change GPU drivers.

## Workflow

### 1. Verify the inputs

- Compare file sizes and SHA-256 hashes with `session.json`.
- Probe actual dimensions, codec, pixel format, frame rate, time base, duration, frame count and audio streams.
- Distinguish manifest wall-clock duration from encoded media duration.
- Identify decode errors or timing discontinuities that could invalidate calibration.

### 2. Inspect representative frame pairs

- Extract manageable samples near the beginning, middle and end.
- Identify stable camera mounting, overlapping scene content, lens distortion, exposure differences and possible camera movement.
- Prefer stationary background features for geometric fitting.
- Reserve separate timestamps for validation rather than evaluating only the fitted frames.

### 3. Establish temporal alignment separately from geometry

- Use common visible motion/events to estimate the frame offset.
- Check the offset near the beginning, middle and end.
- Verify the offset’s sign against the actual frame-selection implementation.
- Do not assume equal PTS, equal duration or a shared capture pipeline means simultaneous exposure.
- If drift or dropped frames prevent a single offset from working, document that limitation and identify a stable segment for a provisional calibration.
- Do not use audio/IMU synchronization unless those streams genuinely exist.

### 4. Resolve lens intrinsics before trusting placement optimization

- Inspect existing lens profiles and the profile-selection implementation.
- Do not accept a database profile merely because it matches 1280×720.
- Do not assume both lenses are identical without evidence.
- Explain whether available footage supports credible intrinsic calibration, or whether actual lens specifications/checkerboard footage are required.
- If testing an approximate profile, label every resulting calibration provisional and record its assumptions.
- Do not invent distortion coefficients or present a guessed profile as calibrated.
- Continue useful footage/synchronization analysis even if accurate lens calibration is blocked.

### 5. Run the existing calibration tooling when prerequisites are met

Relevant source:

- `crates/reco-cli/src/main.rs`: Calibrate options
- `crates/reco-cli/src/calibrate.rs`
- `crates/reco-calibrate/src/pipeline.rs`
- `crates/reco-calibrate/src/lens_database.rs`
- `crates/reco-core/src/calibration.rs`

Verify executable `--help` against this checkout before constructing commands. Build without default AI features if a build is required. Use explicit left/right profiles, measured offset, suitable sampling windows and a dedicated debug directory.

Important CLI detail: this checkout defines `--auto-sync` as a boolean-valued argument; disabling it is expected to be `--auto-sync false`, despite a comment mentioning `--no-auto-sync`. Verify `--help` rather than copying that comment. `--no-auto-imu` exists.

Start with a modest set of representative frame pairs. Inspect inliers, residuals and match distribution before increasing samples or tuning thresholds. Record exact commands, versions, settings and diagnostics.

### 6. Validate the result by rendering

- Produce short clips at held-out beginning/middle/end timestamps.
- Use a fixed panorama without YOLO or automatic camera tracking.
- Choose explicit output dimensions and view/FOV; width alone does not guarantee full-field coverage.
- Inspect field-line continuity, distortion, seam ghosting, players crossing the seam, exposure mismatch and coverage.
- Verify actual output frame counts/durations and first/last frames.
- Separate timing errors from lens, geometry, parallax and blending errors.
- Check which saved calibration settings the stitch path actually honors.
- Beware: the stitch CLI currently does not forward a zero sync override, so `--sync-offset 0` may leave a nonzero saved offset in effect.

### 7. Address future 1080p30 use

- Keep this result explicitly scoped to the 720p30 session and mounting.
- Explain which parameters might transfer if mounting, optics and sensor field of view are unchanged.
- Do not blindly scale or reuse intrinsics: 1080p capture may crop or change the sensor readout/FOV.
- Treat temporal offset as session-specific.
- List the evidence needed to validate a separate 1080p30 calibration.

## Deliverables

Under `calibration-results/scoutcam-20260920-173128/`:

- `report.md`: verified inputs, synchronization findings, lens-profile provenance, exact commands, fitting diagnostics, visual findings and remaining limitations.
- `match.json` only if a supported calibration can be generated; label provisional candidates distinctly.
- Lens profiles used or references/hashes identifying them.
- Representative comparison images and short validation MP4s.
- A clear verdict: validated for this session, provisional, or blocked—and why.

Proceed through the authorized local analysis and calibration work. Ask only for missing information that materially blocks progress. Do not claim success unless the rendered output has been inspected.
