# ScoutCam Stitching and Processing Architecture

Revised 2026-09-22. Documentation only. No code, Pi, Reco, storage, host, driver, Docker, or cloud configuration changed.

## Decision

**Current runtime: one native TypeScript/Node.js stitch worker on Windows 11.** Worker claims durable jobs one at time, accesses private MinIO through AWS S3 SDK, uses scratch, invokes native `reco.exe` and FFmpeg/ffprobe, validates output independently, then conditionally publishes immutable result.

```text
Reco → wgpu/WGSL → DX12 → NVIDIA RTX 5060 Ti 16 GB
```

Native Windows short renders already work; see [local setup](../AGENTS.md) and [720p calibration report](../calibration-results/scoutcam-20260920-173128/report.md). This proves DX12 execution, not full-game performance, production calibration, or 1080p30 acceptance.

Docker is optional infrastructure, never stitch-worker requirement. Portability comes from stable application contracts and shared TypeScript worker source where practical, not identical containers. Docker can support MinIO/backend/development, future Jetson if useful, and likely future AWS GPU jobs.

## Current ScoutCam Recording Architecture

**Current capture is native Cedar V3; MediaMTX is not active path.** Active release `20260921T154625565621Z-dc3c827f0b9c543d4fd1feb577ebfadb1a757fee` uses `gstreamer-cedar-h264-1080p30`: 1920×1080, 30 fps, H.264, 2,048,000 bit/s per camera, video only. Existing inspected fixture is 720p30 and does not prove current calibration.

```text
USB cameras → C++ GStreamer → MJPEG decode/conversion/NV12
→ two Cedar/OpenMAX H.264 encoders → paired MKV
→ drain/EOS → native fast-start MP4 remux + validation
→ hashes + atomic session.json → mobile verified download
```

Cedar only encodes ScoutCam recordings. Do not move Cedar/GStreamer, Pi capture, MediaMTX, or live streaming into worker. Mobile owns upload after completed pair verification. Source MP4 fast-start does not make Reco output fast-start.

## Current Reco Stitching Implementation

Reco performs **decode → integer frame-offset alignment → calibrated GPU warp/blend → readback → encode**. It does not concatenate files. Calibration contains lens and camera-layout data. Reco consumes local files and produces local output; it stays independent from MinIO, AWS, Cloudflare, job tables, and authorization.

| Concern | Local evidence |
|---|---|
| CLI/job | [`reco-cli`](../crates/reco-cli/src), [`StitchJob`](../crates/reco-io/src/stitch_job.rs) |
| Decode/encode | [`ffmpeg`](../crates/reco-io/src/ffmpeg), [`OutputFrame`](../crates/reco-core/src/encoder.rs) |
| Render | [`GpuContext`](../crates/reco-core/src/gpu/mod.rs), [`renderer`](../crates/reco-core/src/render/renderer.rs), WGSL |
| Calibration | [`MatchCalibration`](../crates/reco-core/src/calibration.rs) |

File adapters pair ordinal frames and discard source PTS. No continuous drift correction. Decoder/final-probe behavior can hide failures. Exit code zero never proves accepted result; worker independently validates output.

## Native Cross-Platform Processing Architecture

```text
ScoutCam paired originals → Mobile → ScoutCam Backend
                                      ├─ durable jobs/state
                                      └─ AWS S3 SDK → MinIO
                                                           ↓
             Native TypeScript Stitch Worker
claim → download → verify → scratch → Reco → validate → fast-start
     → upload immutable result → fenced publication
                                                           ↓
Windows now: reco.exe / FFmpeg / DX12 / RTX
Jetson later: reco ARM64 / FFmpeg or Jetson media / qualified GPU
```

One application, not microservices:

```text
Stitch Worker
├── job orchestration
├── StorageAdapter: AWS S3 SDK
├── StitchEngine: RecoRunner
├── PlatformMediaAdapter
├── Validator: ffprobe + full decode
└── ResultPublisher
```

Use `child_process.spawn(executable, args)` or equivalent safe Node API. Never build shell strings. Keep path and arguments separate. Capture stdout, stderr, exit code, signal, start/end time, progress, timeout, cancellation.

## Windows Processing Runtime

Windows is production runtime, not reference/debug route.

```text
MinIO objects → generated scratch → reco.exe → DX12 → RTX
             → ffprobe/full decode/fast-start → MinIO stitched.mp4 + result.json
```

Native executable locations, graphics/backend, FFmpeg build, codec policy, scratch root and platform environment are configuration. Native service-manager choice remains open: Windows Service, WinSW, NSSM, Scheduled Task, or existing mechanism. Host crash/reboot/sleep follows same recovery model as former container crash.

## Platform and Filesystem Adapter

Centralize discovery in configuration-driven `PlatformMediaAdapter`: Reco, FFmpeg, ffprobe, graphics/backend, codecs, environment. No scattered OS checks or elaborate plugin framework.

Use `node:path`, `node:os`, `node:fs/promises`; `path.join`, `path.resolve`, `path.basename`, `os.tmpdir`, `fs.mkdtemp`. Never embed `C:\\ScoutCam`, `C:\\temp`, `/tmp/scoutcam`, or `/opt/scoutcam` in business logic. Scratch may use configured SSD root.

Jobs hold storage identities, never host paths:

```text
recordingId, leftObjectKey, rightObjectKey,
calibrationObjectKey, outputObjectKey, jobKey, attemptId
```

## S3-Compatible Storage Contract

**Speak S3, not MinIO.** Worker uses AWS S3 SDK. MinIO is current endpoint; AWS S3 is future endpoint. Endpoint, region, credentials, path style, bucket are deployment configuration. Do not use MinIO SDK. Durable identities are keys/hashes, never MinIO URLs, Docker names, presigned URLs, or filesystem paths.

```text
recordings/{recordingId}/source/{sourceRevision}/{left.mp4,right.mp4,session.json}
recordings/{recordingId}/calibration/{calibrationHash}/match.json
recordings/{recordingId}/output/{jobKey}/{attemptId}/{stitched.mp4,result.json}
```

Inputs immutable; calibration hash-addressed; outputs attempt-specific. `result.json` records validation, hash/size/media properties, actual tools/backend, timings, provenance. MinIO → AWS S3 changes storage deployment, not contracts. Full-file SHA-256 remains separate from multipart ETag.

## Local Job Processing and State

One persistent native worker, concurrency one, claims durable backend jobs. Polling sufficient initially. No Kubernetes, Redis, RabbitMQ, Kafka, SQS, Docker socket, container-per-job, or cloud executor required.

Backend verifies both originals, hashes/sizes/order, manifest, and calibration before `UPLOADED`; derives idempotent job key; transactionally claims with lease/fencing token. Worker heartbeat and conditional publication reject stale attempt.

```text
UPLOADING → UPLOADED → STITCH_QUEUED → STITCH_RUNNING
          → STITCH_VALIDATING → STITCH_COMPLETE | STITCH_FAILED
```

Keep durable jobs, attempts, leases, fencing, retries, scratch cleanup, source preservation, immutable attempt outputs, idempotency and conditional publication. Duplicate claims, host crash, corrupt input, full disk, failed upload, and lost completion response never publish partial/stale video.

## Synchronization, Calibration, and Output Validation

Docker removal does not solve offset, ordinal pairing, dropped frames, drift, fractional exposure skew, lens calibration, seam, or framing. Current 720p candidate is provisional: black borders, stretched edge views, limited lens-fit coverage, focus/history uncertainty, and full-field/seam checks remain. Do not reuse blindly for 1080p30.

`STITCH_COMPLETE` requires:

- `ffprobe`: expected container/codec/dimensions/FPS/audio policy.
- strict full decode; expected count/duration; first/end validity.
- plausible size; SHA-256 final bytes; post-upload size/hash match.
- synchronization plus visual seam/geometry qualification.
- current fencing token and immutable accepted object.

After Reco, remux `+faststart` if needed. Revalidate and hash delivered remux bytes. Rules identical on Windows, Jetson, AWS.

## Streaming and Security

Stitching, storage, control, delivery separate. Cloudflare Tunnel may expose authorized app services; it is not Reco, object store, authorization, or video-delivery guarantee. Private concept: MinIO → authorized backend range endpoint → Tunnel → player. Test actual range/seek/auth expiry, plan video/large-file limits, and origin upstream. Cloudflare Stream/Enterprise delivery remains future adapter.

Keep MinIO API/console, job DB, worker claims, recorder control, credentials private. Scope worker input/calibration read and attempt-output write only. Validate keys/sizes/media. Back up objects and job/asset metadata with restore test.

## Jetson Migration

```text
Windows → Node.js → same TypeScript source → reco.exe → DX12 → RTX
Jetson ARM64 → Node.js → same TypeScript source → reco → qualified Jetson GPU/media
```

Shared: StitchJob/result schemas, AWS SDK adapter, object keys, lifecycle, retries, validation, publication, metrics where practical. Allowed differences: OS, architecture, Reco, FFmpeg, GPU backend, JetPack/NVMM/NvBufSurface/CUDA-Vulkan interop, codecs, scratch, service manager, packaging.

Jetson needs ARM64 build and exact model/JetPack/GPU/codec qualification. Container may later help, but does not remove this work or require Windows media stack duplication. Live capture and YOLO remain separate work.

## Optional Containers and Future AWS Scale-Out

Optional Linux containers on Windows retain historical WSL/Vulkan gate: inspected Vulkan was software `llvmpipe`; prove hardware Reco adapter and render before use. This does not block native Windows DX12 worker.

```text
StitchJob → Windows native | Jetson native | AWS GPU Linux container
```

AWS becomes future executor when measured load, uptime, or delivery needs justify it. AWS Batch/EC2 GPU may use Linux container, same TypeScript contracts, validation, fencing, and result semantics. AWS needs separate Linux Reco/Vulkan/NVIDIA/media qualification. Fargate lacks GPU route; Lambda unsuitable for full-game stitch. Batch, CloudFront, MediaConvert, SQS, EventBridge optional future dependencies.

## Smallest Useful Proof of Concept — Proposed, Not Built

1. Use verified `videos/scoutcam-20260920-173128` 720p30 pair; establish calibration/sync and gate provisional quality.
2. Create TypeScript worker skeleton and durable job contract.
3. Configure native Windows Reco, FFmpeg, ffprobe; capture RTX/DX12 identity.
4. Seed pair, unchanged manifest, calibration into MinIO through AWS S3 SDK.
5. Claim one job; download into generated scratch; verify hashes/probe inputs.
6. Invoke Reco via argument arrays; record process/progress/tool identity.
7. Strictly validate; fast-start remux if needed; revalidate final bytes.
8. Upload immutable output/result; conditionally publish; clean scratch.
9. Measure stage times, CPU/GPU/VRAM/RAM, scratch high-water, output size/bitrate.
10. Exercise restart, duplicate claim/finalize, corrupt input, failed output upload, truncated output; separately assess panorama quality.

No POC implementation belongs in this task.

## Open Questions

1. Existing backend/mobile runtime, DB, recording/game identity, upload contracts?
2. Accepted viewport/FOV, output policy, skew/seam quality, retention, queue delay, uptime?
3. MinIO version/volume/backup and Cloudflare delivery plan?
4. Target Jetson model and JetPack?
5. Full-game Windows throughput, scratch, reliability, cost measurements?

## Revision Record

Removed Docker-first, Linux-first, WSL/Vulkan initial blocker, multi-architecture-container, and Windows-reference-only assumptions. Retained Cedar V3, Reco, synchronization/calibration, validation, security, streaming, and future AWS strategy. WSL/Vulkan findings now optional historical deployment evidence.
