# Minimal Offline Stitcher: Extraction Assessment

Initially assessed 2026-09-13; updated 2026-09-22 against the current Orange Pi deployment. Scope: two completed camera recordings in, one stitched panoramic MP4 out. This is an investigation and proposed design; no application code, packages, drivers, or running services were changed. No stitching benchmark or ARM64 build was performed. See the [processing architecture](stitching-processing-architecture.md) for the current recording-to-cloud boundary.

## Recommendation

**A small-purpose offline application is straightforward to build around the existing engine. Extracting only a handful of source files is substantially harder and is unnecessary for the first version.**

Use a small command-line wrapper with just two project dependencies: `reco-core` and `reco-io`, enabling only the latter's FFmpeg backend. Supply an existing calibration file. That removes the GUI, OBS integration, camera control, AI tracking, live capture, and calibration solver from the application's project dependency graph. The wrapper reads two local files, renders a fixed panoramic view, and encodes one MP4.

For one developer familiar with Rust and video processing, budget roughly **1–3 working days for a prototype on a provisioned, supported GPU host**, and **1–2 weeks total for a dependable, deliberately restricted batch workflow**. These are planning estimates, not measured delivery times. They exclude Orange Pi driver enablement and acquiring a valid calibration for the actual cameras. Literal source extraction and simplification would more plausibly take **2–4+ weeks**, with greater maintenance and regression risk.

On the inspected Orange Pi, **the graphics stack remains the first execution blocker**. Removing unrelated Rust code does not make its GPU available. Offline processing does, however, remove the requirement to keep up with capture: a 90-minute recording can take several hours to process if that is acceptable.

## What the Orange Pi already provides

The deployed ScoutCam recorder already produces the right basic inputs for an offline job:

| Observed item | Consequence for the proposed tool |
|---|---|
| Two completed `left.mp4` and `right.mp4` files per session | No RTSP client or live acquisition integration is needed. |
| Selected capture profile: H.264, **1920×1080, 30 fps**, video only, target **2.048 Mb/s per camera** | Start with this input contract, but probe every file: older sessions use other profiles. No completed pair from this selected release was validated during the investigation. |
| Native C++ GStreamer engine: USB MJPEG → software JPEG decode/conversion → NV12 → **Cedar/OpenMAX hardware H.264 encoding** | The two encoded tees feed paired Matroska archive branches; there are no active MediaMTX publishers or separate RTSP archive readers. |
| Each archive branch waits for its own first timestamped IDR; native finalization remuxes to fast-start MP4 | A shared pipeline/clock does not prove simultaneous camera exposure. Remuxing does not correct temporal alignment. |
| Latest completed sample inspected belongs to the earlier Cedar 720p30 release: 398 frames and 13.2667 s per side | This confirms that sample's media properties, not the selected 1080p30 release's output or exposure alignment. |
| Historical September 12 V2 samples include a long pair differing by one frame | Retain an explicit shorter-input/end policy, but do not use old V2 evidence to qualify Cedar V3. |
| Completed sessions have a validated `session.json` manifest with two source-file entries | Read the existing manifest without extending its schema. Keep derived-job metadata separately. |
| `SessionLibrary.deleteCompleted` removes the entire source session directory | Store stitched output outside that directory, or a normal source-session deletion will also delete it. |
| Approximately 416 GiB free on the 469 GiB NVMe filesystem at the September 22 recheck | Prefer that filesystem for output and temporary files; check free space again for every job. |

The actual USB devices identify as “4K U3 Camera”; a specific lens model and a valid calibration for this mounted pair have not been established. Existing footage alone does not establish that both cameras cover the desired field with sufficient overlap.

The active recorder/capture services now use `/home/orangepi/app/ScoutCam-recorder-service/dist/releases/20260921T154625565621Z-dc3c827f0b9c543d4fd1feb577ebfadb1a757fee`. Its sealed `app/v3/cedar/config/capture-profile.json` selects 1080p30. `/opt/scoutcam` and V2/MediaMTX are historical. The active architecture is native systemd, including fan control; Docker is not required for recording.

Authoritative remote code: `native/engine/main.cpp:89–117` builds capture/encoding; `native/engine/recording_pair.cpp:62–111,177–278` gates, queues and drains recording; `src/v3/nativeRecorder.ts:66–105` owns saving; `src/v3/archive.ts:35–78` and `native/archive/main.cpp:263–319` remux/validate; `src/sessionLibrary.ts:89–116` hashes and atomically publishes `session.json`. A session becomes available only after successful finalization, not immediately on Stop. Saving and failed saves retain recording ownership.

The phone downloads completed originals and the manifest through the authenticated Wi-Fi bridge, verifies size/SHA-256, and owns cloud upload. No Pi uploader or AWS worker exists in the inspected Pi repositories; mobile/backend cloud code was unavailable. A local offline tool should remain separate from recording, and an AWS variant should preserve this mobile-owned transfer boundary. The user-authorized runtime-document mode correction now says 1080p30, but its older release ID remains stale; the active service path above is authoritative.

## The smallest practical dependency graph

```text
new offline command-line application
├── reco-core
│   └── calibration data, lens/projection geometry, wgpu rendering,
│       frame/session machinery, GPU readback
└── reco-io [default-features = false, features = ["ffmpeg"]]
    └── paired file decoding, StitchJob, MP4 encoding

Runtime inputs: left.mp4 + right.mp4 + saved calibration + job settings
Runtime output: one panoramic MP4 + separate job metadata
```

This is three project crates including the new executable, plus their third-party dependencies and native libraries. It does **not** mean three standalone source files or a dependency-free binary.

| Existing component | Keep in the offline application? | Reason |
|---|---|---|
| `reco-core` | Yes | Contains the existing warp, projection, blend, session, and GPU output machinery. |
| `reco-io`, FFmpeg feature | Yes | Already implements file decoding and video output around the engine. |
| `reco-cli` | Replace with a small wrapper | Its dependency graph includes preview/windowing, camera control, and calibration functionality even without its default AI features. |
| `reco-calibrate` | Separate setup tool only | Compute calibration elsewhere, then load its result on the Pi. The runtime needs calibration data types in core, not the solver crate. |
| `reco-control` | No | The recorder already owns camera acquisition. |
| `reco-detect`, `reco-autocam` | No | A fixed panorama needs neither object detection nor an automatic virtual camera. |
| `reco-gui`, `reco-obs` | No | No interactive preview or OBS source is required. |
| GStreamer, V4L2, libcamera, replay/stacked-output features in `reco-io` | Do not enable | The new application's inputs are completed files. The existing recorder can continue using GStreamer independently. |

Source evidence: [workspace manifest](../Cargo.toml), [core manifest](../crates/reco-core/Cargo.toml), [I/O manifest](../crates/reco-io/Cargo.toml), and [CLI manifest](../crates/reco-cli/Cargo.toml).

As a baseline, the existing command can be built with `cargo build --locked --release -p reco-cli --no-default-features` on a host with the required toolchain and native dependencies. This avoids the CLI's default AI features, but its unconditional `reco-calibrate`, `reco-control`, `winit`, and image dependencies remain. It is a useful reference implementation for comparing outputs, not the smallest dependency set. This command was not executed during the investigation.

## Reuse versus literal extraction

| Approach | Work and tradeoff | Assessment |
|---|---|---|
| Use existing CLI with AI disabled | Configure and package an existing file-to-file path; retain extra commands and dependencies. | Fastest baseline, potentially hours to a day after the host is ready and calibration exists. |
| New wrapper using the two existing library crates | Define a narrow job contract, explicit rendering settings, validation, safe output handling, and useful status/errors. | Recommended. Small product surface and limited new code while preserving the tested engine structure. |
| Vendor the two crates into a small repository | Same functionality, plus workspace/dependency cleanup and ownership of updates. | Reasonable if a separate repository is required; keep the first copy close to upstream. |
| Copy only selected rendering and FFmpeg files | Reconstruct dependencies, replace session machinery or introduce feature boundaries, and prove equivalent geometry and frame handling. | Highest engineering cost. Do this only for a demonstrated size, portability, or maintenance requirement. |

The inspected source contains approximately 70,743 physical Rust/WGSL lines across the nine project crates. `reco-core` accounts for 25,412 and `reco-io` for 11,040: retaining both leaves about 36,452 lines, including comments and tests. These counts exclude other source formats and third-party dependencies. They explain why a focused executable can still sit on substantial library source; they do not predict shipped binary size or memory usage.

There is no existing “minimal offline core” feature. Some detection-related interfaces, replay structures, and Linux interop code are declared unconditionally within the retained crates. The interfaces do not themselves pull in the separate inference backends, but deleting them requires tracing their consumers. Linux builds also contain CUDA/interop probing code; selecting a CPU-resident input path does not remove that source dependency graph. Release linking may eliminate unused functions, but its result must be measured.

### The engine is more than the stitch shader

The relevant flow is already encapsulated by [`StitchJob`](../crates/reco-io/src/stitch_job.rs):

```text
load saved calibration and explicit settings
  → open and pair two decoded file sources
  → upload frames to wgpu
  → apply calibrated lens/projection mapping and feathered blending
  → convert rendered output to encoder-compatible pixels
  → read back and encode frames
  → drain delayed frames and finalize the MP4
```

For a literal extraction, these are the main responsibility groups to preserve. The table is an investigation map, not a proven closed list of compilable files.

| Responsibility | Starting points in the current source |
|---|---|
| Calibration loading and validation | [`calibration.rs`](../crates/reco-core/src/calibration.rs), lens and projection modules |
| GPU selection, device, buffers, textures | [`gpu/mod.rs`](../crates/reco-core/src/gpu/mod.rs) |
| Render pipeline, scene, planes, viewport | [`render/pipeline.rs`](../crates/reco-core/src/render/pipeline.rs), [`render/renderer.rs`](../crates/reco-core/src/render/renderer.rs), related render modules |
| Per-pixel camera mapping and blending | [`fisheye.wgsl`](../crates/reco-core/src/shaders/fisheye.wgsl) and the CPU code constructing its uniforms |
| Session/frame ordering and delayed GPU output | [`session/mod.rs`](../crates/reco-core/src/session/mod.rs), source/frame interfaces |
| Output pixel conversion | [`nv12_converter.rs`](../crates/reco-core/src/gpu/nv12_converter.rs), [`rgba_to_nv12.wgsl`](../crates/reco-core/src/shaders/rgba_to_nv12.wgsl) |
| Decoder pairing and encoder lifecycle | [`adapters.rs`](../crates/reco-io/src/adapters.rs), [`stitch_job.rs`](../crates/reco-io/src/stitch_job.rs), FFmpeg backend |
| Bounded asynchronous encoding | [`async_encode.rs`](../crates/reco-core/src/async_encode.rs), encoder interfaces |

Rendering shares viewport-position types with detection interfaces and uses `VirtualCamera` projection geometry and world-up calculations. The public pipeline also exposes interop types. Consequently, copying the WGSL shader and a renderer file will not reproduce a working stitcher.

Preserve the existing delayed-readback and encoder-drain lifecycle. In particular, the GPU conversion path uses staged readback with delayed frames; failing to flush it can lose final frames. A smaller loop must prove its first-frame, last-frame, and output-count behavior against the original.

## What the small application would actually do

Start with one explicit job at a time. Do not add a web server, queue service, or recorder integration to the first version.

1. Accept two completed local MP4s, a saved calibration, output path, output dimensions, fixed view/FOV settings, and an explicit frame offset.
2. Probe both files and validate the supported input contract: decodable video, expected dimensions, compatible cadence, plausible duration, and suitable calibration. Refuse to overwrite either input or an existing final output.
3. Check output-space availability and run the existing `StitchJob` with explicit MP4, no audio, and fixed rendering settings. Disable detection and lookahead behavior.
4. Write a uniquely named temporary `.partial.mp4` beside the intended final output, on the same filesystem. Treat interruption or encoding failure as an incomplete job.
5. Finish the session and encoder, then independently validate the output's video stream, dimensions, duration, and expected frame count. For qualification, also perform a full decode and visual checks.
6. Rename the validated file to its final name. Record input identity, calibration identity, settings, tool revision, frame counts, elapsed time, and outcome in separate job metadata. Preserve both original recordings.

`StitchJob` already exposes output format, resolution, audio, synchronization offset, and session customization. Its `on_session` hook can configure the pipeline's fixed FOV. Thus the wrapper need not introduce a new rendering API solely to choose a static view.

Set dimensions explicitly: although the builder's documentation says its default matches input resolution, the inspected implementation defaults to **1920×1080**. A proposed 2560×720 panorama is a useful test size, not proof of full-field coverage. Width, height, camera orientation, and projection/FOV must be selected together. Simply making the output wider does not guarantee that the field appears in it.

Warping changes image pixels, so the final panorama requires decoding and re-encoding. The recorder's current stream-copy/remux strategy cannot combine two viewpoints into a stitched image. Placing the files side by side is a different output and does not align their overlap.

### Calibration remains a requirement

The existing renderer uses its calibrated lens/projection model and a fixed feathered blend. It does not automatically discover alignment anew for every frame, solve moving seams, or guarantee matching exposure between cameras.

Generate and inspect the saved calibration on a desktop or other provisioned host using the existing calibration tooling. Keep the cameras rigidly mounted and retain the resolution, lens assumptions, and framing used to validate it. The Pi then only loads the saved result. A generic homography or an arbitrary JSON file is not interchangeable with the current `MatchCalibration` data.

Before committing to the port, validate actual overlapping soccer footage for straight-line distortion, seam ghosting, players crossing the seam, exposure differences, and field coverage. An engine that runs successfully can still produce an unsuitable panorama if the calibration/model does not fit these lenses.

### Synchronization and error handling need explicit limits

The current file adapter pairs frames sequentially after an integer-frame offset. It does not continuously reconcile original timestamps: decoded timestamps are discarded in the CPU pairing path. The encoder generates constant-frame-rate PTS from output frame count. Equal MP4 timestamps therefore do not establish simultaneous camera exposure, and a fixed offset cannot correct arbitrary drift or independent frame drops.

For the first version, support only the recorder's qualified constant-cadence file pairs. Measure alignment from common visible events at the beginning, middle, and end of a full recording. Define whether a small end-length difference is accepted by stopping at the shorter input and reporting the omitted frames; reject unexplained discontinuities. Treat temporal offset as session-specific, not necessarily a permanent camera calibration property.

The wrapper should always forward the chosen offset, including zero. The existing CLI only forwards a nonzero override, whereas the library's `.sync_offset(0)` can explicitly replace a stored nonzero calibration offset.

There are also existing reliability gaps to address before calling the workflow dependable:

- A decoder worker can log a decoding error and stop; channel closure can subsequently look like normal end-of-file to the pairing loop.
- `StitchJob`'s final probe can log that it is “trusting encoder” if reopening the output fails, rather than failing the job. It does not establish complete expected-frame delivery.

Independent output validation is necessary, but it does not replace proper propagation of decoder failures. Record these API gaps in the appropriate `FRICTION.md` and fix them in the library if implementing the reliable version; do not conceal them with consumer-side assumptions. General variable-frame-rate handling, timestamp resampling, and drift correction would extend the scope beyond the restricted estimate above.

## Orange Pi feasibility after simplifying the application

The inspected board runs Debian 11 with the vendor `5.15.147-sun60iw2` kernel and approximately 6 GB RAM. It has FFmpeg but no installed Cargo toolchain. Provisioning must cover a compatible ARM64 Rust build environment, the project's Rust 1.92 toolchain requirement, and native FFmpeg development/runtime dependencies. The installed FFmpeg executable alone does not establish that the Rust bindings can build and link.

The September 13 graphics inspection found an unbound GPU, no PowerVR ICD/loaded driver, and a minimal Vulkan probe returned `VK_ERROR_INCOMPATIBLE_DRIVER`. The September 22 recheck still lists only `/dev/dri/card0`, no render node or ICD in the standard directories checked; the Vulkan execution probe was not repeated. No working Reco graphics path has been established. BXM hardware capability and the now-working Cedar video encoder do not supply a Vulkan graphics stack.

The [existing device evaluation](video-stitcher-evaluation.md) documents the matching-driver investigation, including the **24.2.6603887** kernel/userspace DDK lead and matching **36.56.104.183** firmware identification. That is a candidate compatibility path, not a validated Orange Pi installation. Budget the driver work separately: a few days if a compatible vendor bundle works, potentially **1–3+ weeks** for integration problems, with no guaranteed upper bound for unsupported behavior.

Calling `.force_cpu_decode()` does **not** make Reco a CPU stitcher. It selects the CPU-resident source route; rendering still uses wgpu. Depending on decoder selection, hardware decoding can also still occur before frames are downloaded. Cedar hardware encoding already works in the recorder's GStreamer/OpenMAX path, but it is not integrated into Reco's FFmpeg output and does not establish hardware H.264 decoding for Reco. Reusing Cedar for stitching output remains separate adapter/qualification work.

### Offline processing changes the performance target

No live input queues, RTSP reconnection, streaming output, or concurrent capture ownership are required. Initially run the job after recording has stopped; separately qualify concurrent processing if that later becomes desirable.

For a 90-minute recording at the current 30 fps, there are approximately 162,000 frame pairs:

| Measured processing speed, if achieved | Approximate processing time |
|---|---|
| 5 frames/s | 540 minutes |
| 15 frames/s | 180 minutes |
| 30 frames/s | 90 minutes |
| 60 frames/s | 45 minutes |

These are arithmetic scenarios, not Orange Pi benchmark results. The relevant acceptance question is the allowable delay before a match's MP4 is ready.

At the selected 1920×1080 input size, two YUV420P frames contain about **6.221 MB per pair**. At 30 pairs/s, that is about **186.6 MB/s** of logical input upload—4.5 times the older 720p15 scenario. An illustrative 2560×720 NV12 output requires about **82.9 MB/s** of readback at 30 fps; that output size is not a requirement or a guarantee of full-field coverage. The existing output texture, readback, and conversion allocations total approximately 71 MB at that output size (79.8 MB at 1920×1080), before input textures, codec buffers, queues, and other services. Keeping the existing core retains allocations such as its eagerly created RGBA readback buffers even when the narrower application does not need every path.

At the configured combined target bitrate of 4.096 Mb/s, 90 minutes of originals would be approximately 2.765 GB before container overhead. This compressed storage estimate is separate from decoded-frame bandwidth and is not a measured bitrate for a current-profile game. Scratch must also hold the stitched output and any remux copy. Cedar reduces capture encoding cost but does not establish spare capacity for concurrent stitching.

These figures suggest that nominal RAM capacity alone is not the obvious blocker, but they do not establish actual peak RSS, memory traffic, thermal stability, or codec throughput. Use bounded buffers and process sequentially. Measure those properties on the real board once its graphics stack works.

### Optional route if GPU enablement stalls

A separate CPU implementation could precompute mapping images for the fixed cameras, use FFmpeg to remap each image into panorama coordinates, blend the overlap, and encode the result. The installed FFmpeg advertises the `remap` filter. Its documented interface takes a source image and two single-channel 16-bit coordinate maps; this confirms a possible building block, not an already working stitch pipeline. [FFmpeg remap documentation](https://ffmpeg.org/ffmpeg-filters.html#remap).

This would require implementing/exporting the inverse mapping from the lens and projection geometry, selecting overlap weights, reproducing boundary behavior, and validating quality. Reco does not currently expose a ready-made “export FFmpeg maps” path. Integer map coordinates and sampling behavior also need comparison with the shader output. FFmpeg cannot directly consume the existing calibration JSON as those maps.

Treat this as a separate prototype if avoiding GPU-stack work becomes the priority. It may be acceptable for slow offline jobs, but neither its speed nor image quality has been measured. It is not a benefit obtained simply by deleting unrelated Reco code.

## Packaging and ownership

Prefer a pinned upstream revision for the two library dependencies initially, or vendor them with minimal changes. Both manifests specify `publish = false`; do not assume these packages can be fetched as ordinary published registry dependencies.

If copying the crates into a new workspace, preserve or explicitly replace inherited workspace metadata and dependencies, retain a reproducible lockfile, and check relative paths. A directory copy without its workspace context is insufficient. Build against a compatible ARM64 userspace; a binary produced on a newer distribution must not be assumed to run on this Debian 11 installation.

The source manifests declare `AGPL-3.0-only`. Preserve the existing license files and source notices, and track the upstream revision when copying code. Extraction does not turn the copied engine into newly authored unlicensed code.

Keep derived files in a dedicated directory on the NVMe filesystem, associated with the source session ID through separate metadata. Do not change the recorder's strict two-file manifest or place the only copy of the derived MP4 inside a directory its normal deletion operation removes.

## Suggested delivery stages and acceptance evidence

These stages separate application scope from platform risk; their estimates overlap and should not be summed as independent commitments.

| Stage | Completion evidence | Indicative effort |
|---|---|---|
| Establish a reference result | Existing CLI produces an acceptable fixed panorama from actual files and calibration on a supported host. | Hours to a day once prerequisites and calibration are ready; acquiring calibration may take longer. |
| Build the small wrapper | Only the two library crates are direct project dependencies; explicit settings produce a readable MP4 equivalent to the reference. | 1–3 working days for a prototype. |
| Make the restricted batch workflow dependable | Decoder failures propagate; partial outputs stay partial; originals survive cancellation; geometry, alignment, and full-duration output pass checks. | Roughly 1–2 weeks total including prototype work, plus full-duration test time. |
| Enable and qualify the Pi | Physical BXM adapter is used; ARM64 build runs; full recording completes with acceptable time, memory, thermals, and image quality. | Driver work separately as above, then several build/integration/test days. |
| Prune source only if justified | Measured benefit in binary size, build cost, or maintenance; reference output and lifecycle tests still pass. | Roughly 2–4+ weeks for a deeper extraction, depending on retained functionality. |

Qualification should include a short known-good pair, real players crossing the overlap, a full-length session checked at multiple time points, a one-frame input-length difference, corrupt/truncated input, explicit zero offset over a stored nonzero offset, cancellation, insufficient output space, and an existing output filename. Confirm expected first and last frames and independently decode the finished MP4. No live-stream tests are required for this scope.

The recommended next implementation is therefore **a standalone offline command with the existing two-crate engine**, initially validated on a working GPU host and then qualified on the Pi. A new renderer, live-service integration, hardware codec backend, and extensive source pruning are not prerequisites for that deliverable.
