# ScoutCam Native Stitch Worker — Implementation Planning

Read `docs/stitching-processing-architecture.md` completely before doing anything else. Its architectural decisions are authoritative.

This task is **PLANNING AND REPOSITORY INVESTIGATION ONLY**. Do not implement the worker, create services/packages/migrations/Dockerfiles/APIs/infrastructure/configuration, or modify Reco, ScoutCam, MinIO, Cloudflare, Windows/WSL, FFmpeg, NVIDIA drivers, or system configuration.

The goal is to inspect the actual ScoutCam repositories and produce a concrete implementation plan grounded in existing code.

## 1. Fixed Architecture

```text
ScoutCam → Mobile → ScoutCam Backend
                         ├─ durable jobs/state
                         └─ AWS S3 SDK → MinIO
                                           ↓
                              Native TypeScript Stitch Worker
                              claim → download → verify → scratch
                              → native Reco → validate → fast-start
                              → upload immutable result → publish
```

Current host:

```text
Windows 11 → Node.js/TypeScript → native reco.exe
→ wgpu/WGSL → DX12 → NVIDIA RTX 5060 Ti 16 GB
```

This native Windows/DX12 path has already rendered short test clips. Do NOT redesign this into Docker, WSL, Linux, Vulkan, AWS Batch, or another architecture.

Long term, preserve the same application contracts for Jetson Linux ARM64 and optional AWS GPU scale-out. Portability comes from contracts and shared TypeScript source where practical, not identical containers.

## 2. Discover the Actual ScoutCam Repositories

Inspect the current checked-out ScoutCam repositories. Identify which own:

- backend/API
- mobile
- UI
- infrastructure
- recording/session domain models
- authentication
- database/models/migrations
- object/file storage
- uploads
- video/game entities
- background jobs
- status/events
- configuration

Do not assume repository names from historical docs. Produce a repository/responsibility map. Explicitly note anything unavailable locally.

## 3. Inspect the Existing Backend

Determine the actual runtime, package manager, framework, database, ORM/query layer, module organization, API style, authentication, config, logging, error handling, jobs, status/events, storage abstractions, AWS SDK/MinIO usage, media/video entities, game/recording/session entities, and upload workflows.

Reference actual files, functions, classes, models, routes, configuration, and dependencies. Reuse existing abstractions where appropriate instead of inventing duplicates.

## 4. Inspect the Mobile Recording/Upload Flow

Trace:

```text
ScoutCam → completed session → mobile
→ session.json + left.mp4 + right.mp4
→ verification → current next step
```

Determine discovery, download, temporary storage, manifest handling, SHA-256 verification, existing uploads, resumable/multipart behavior, authentication, recording/game IDs, progress, successful-transfer behavior, and deletion/retention.

Do not modify it.

## 5. Inspect Storage

The decision is **Speak S3, not MinIO**.

Look for existing `@aws-sdk/client-s3`, `@aws-sdk/lib-storage`, `S3Client`, Get/Put/Head commands, multipart/presigning, and storage/media services.

MinIO is today's S3-compatible endpoint; AWS S3 is future storage. Do not introduce a MinIO-specific SDK.

## 6. Reconcile Object Layout

Current proposal:

```text
recordings/{recordingId}/
  source/{sourceRevision}/
    left.mp4
    right.mp4
    session.json
  calibration/{calibrationHash}/match.json
  output/{jobKey}/{attemptId}/
    stitched.mp4
    result.json
```

Compare this with the actual backend/domain model. Determine whether existing recording/session/game/media identities and object-key conventions should be reused.

Requirements: immutable source identity, left/right association, original manifest, calibration identity, attempt-specific output, stable backend asset identity, and no durable MinIO URL dependency.

## 7. Decide Where the Worker Belongs

Determine whether the native TypeScript/Node worker should:

A. be another process/entrypoint in the backend repo,
B. be a package in an existing workspace/monorepo, or
C. be a separate ScoutCam repository.

Base this on the real repository/deployment structure, not generic microservice advice.

Logical modules:

```text
StitchWorker
├─ JobOrchestrator
├─ StorageAdapter
├─ RecoRunner
├─ PlatformMediaAdapter
├─ Validator
└─ ResultPublisher
```

These are modules, not separate services.

## 8. Durable Job Model

Reconcile approximately these states with the existing domain:

```text
UPLOADING
UPLOADED
STITCH_QUEUED
STITCH_RUNNING
STITCH_VALIDATING
STITCH_COMPLETE
STITCH_FAILED
```

Determine how to represent logical job identity, attempts, lease/fencing token, expiry/heartbeat, timestamps, failures, progress, calibration/input/output identities, and accepted result.

Do not assume each requires a separate column.

## 9. Simplest Claim Model

There is one ScoutCam and one worker; concurrency is 1.

Prefer the simplest durable existing mechanism. Do not introduce Redis, RabbitMQ, Kafka, SQS, Kubernetes, Docker orchestration, Step Functions, EventBridge, or container-per-job execution unless already present and clearly simpler.

Plan atomic claim → attempt/fencing token → heartbeat → process → conditional publication, including crash/reboot/stale lease/duplicate claim/late attempt/backend restart behavior.

## 10. StitchJob Contract

Propose a versioned TypeScript schema using existing naming conventions. Likely concepts:

```text
schemaVersion
jobKey / jobId
recordingId
attemptId
leftObjectKey / rightObjectKey
sessionManifestObjectKey
leftHash / rightHash
calibrationObjectKey / calibrationHash
sync settings
output settings / outputObjectKey
engine version
```

Clearly distinguish logical job, attempt, and output asset.

## 11. StorageAdapter

Plan only the operations the worker actually needs, likely:

```text
headObject()
downloadObject()
uploadObject()
getRange()
objectExists()
```

Use AWS S3 SDK. MinIO/AWS differences belong in configuration. Avoid internal presigned URLs unless justified.

## 12. Scratch Workspace

Use cross-platform Node APIs (`node:path`, `node:os`, `node:fs/promises`), never hard-coded Windows/Linux paths.

Plan generated scratch directories, free-space checks, cleanup, failure retention, orphan reconciliation, and configurable scratch root for a high-capacity SSD.

## 13. RecoRunner

Use `child_process.spawn(executable, args)` or equivalent safe API. Never shell command strings.

Inspect the actual current Reco CLI and document the exact invocation, calibration/sync/output/codec/audio options, backend environment, stdout/stderr/progress handling, cancellation, timeout, and Windows process-tree termination.

Do not modify Reco. Identify CLI/API limitations explicitly rather than hiding them in worker hacks.

## 14. PlatformMediaAdapter

Centralize native configuration approximately as:

```text
PlatformMediaAdapter
├─ recoExecutable
├─ ffmpegExecutable
├─ ffprobeExecutable
├─ recoEnvironment
├─ codecPolicy
└─ process behavior
```

Initial target is native Windows/DX12. Future Jetson/Linux supplies different native settings without changing business/job logic.

## 15. Input Validation

Plan independent validation of downloaded inputs using the actual `session.json` plus ffprobe/full checks where needed:

- size and SHA-256
- readability
- H.264
- dimensions/FPS/duration/frame count
- expected audio policy
- left/right compatibility
- calibration compatibility
- synchronization policy

Identify what the current manifest cannot prove.

## 16. Output Validation

Reco exit code 0 is not success.

Validate `stitched.partial.mp4`, then fast-start remux if required, then validate the FINAL delivered bytes:

- ffprobe
- H.264/container/dimensions/FPS/audio
- complete strict decode
- expected frame count/duration
- valid beginning/end
- plausible size
- final SHA-256

`result.json` must describe the exact uploaded bytes.

## 17. result.json

Propose a versioned result contract containing job/attempt/input/calibration identity, engine/Reco/FFmpeg/GPU/backend/encoder provenance, output media properties and checksum, validation results, stage timings, reliable resource metrics, and producer identity (`local-windows` or equivalent).

Do not invent metrics that cannot be measured reliably.

## 18. Metrics

Plan lightweight Windows measurement of:

- total wall time
- download/stitch/validation/remux/upload times
- CPU
- process RAM
- GPU utilization
- VRAM
- scratch high-water mark
- output size/bitrate
- effective render FPS

Prefer existing OS/NVIDIA/application-observable metrics. No monitoring stack.

## 19. Keep Infrastructure and Panorama Quality Separate

The existing 720p calibration is provisional.

Record separate verdicts for:

**INFRASTRUCTURE SUCCESS** — storage/job/worker/validation pipeline works.

**PANORAMA QUALITY SUCCESS** — calibration, field coverage, lens correction, seam, temporal alignment, seam-crossing players, beginning/middle/end sync, and later 1080p30 calibration are accepted.

Do not conflate them.

## 20. No YOLO

YOLO is downstream and out of scope. Its future failure must never invalidate a successful stitched video.

## 21. Security Review

Identify minimum S3/MinIO permissions, backend worker authentication, scratch permissions, subprocess safety, object-key validation, cleanup and logging/secrets risks.

Worker must not have MinIO admin, source deletion, arbitrary bucket, arbitrary command, or job-supplied executable permissions.

## 22. Produce the Planning Document

Create:

```text
docs/stitch-worker-poc-implementation-plan.md
```

Include:

1. Executive summary
2. Repositories inspected
3. Existing backend architecture
4. Existing mobile recording/upload flow
5. Existing storage implementation
6. Existing database/domain model
7. Recommended worker repository/location
8. Proposed module structure
9. StitchJob contract
10. Result contract
11. StorageAdapter
12. Durable job/claim model
13. Scratch workspace
14. RecoRunner
15. PlatformMediaAdapter
16. Input validation
17. Output validation
18. MinIO integration
19. Backend integration
20. Metrics
21. Failure/recovery behavior
22. Security
23. Windows service/deployment considerations
24. Jetson portability
25. Explicitly deferred work
26. Implementation sequence
27. Acceptance criteria
28. Open questions

Clearly label **EXISTS TODAY** vs **PROPOSED** and cite actual repository files throughout.

## 23. File-Level Change Plan

At the end, include a proposed map using the ACTUAL repository structure:

```text
Repository: <actual repo>

CREATE:
...

MODIFY:
...

DATABASE:
...

CONFIG:
...

TESTS:
...
```

For each proposed file/change, explain responsibility, why it belongs there, and existing code it integrates with. Do not create these files yet.

## 24. Implementation Phases

Break implementation into small reviewable phases, approximately:

### Phase 1 — Contracts and storage qualification
- Reconcile existing backend/domain model.
- Define StitchJob/result contracts.
- Qualify AWS S3 SDK operations against current MinIO.
- Establish object-key conventions.
- No stitching yet.

### Phase 2 — Native process qualification
- Define RecoRunner and PlatformMediaAdapter.
- Verify native Windows Reco/DX12 identity.
- Define FFmpeg/ffprobe invocation.
- Define safe cancellation/failure semantics.

### Phase 3 — Validation
- Implement input hash/probe checks.
- Implement output probe/full-decode/count/duration checks.
- Implement fast-start remux and final-byte validation.

### Phase 4 — Durable worker lifecycle
- Add claim/lease/heartbeat/fencing/retry semantics.
- Add scratch lifecycle and crash reconciliation.
- Keep concurrency one.

### Phase 5 — End-to-end MinIO POC
- Seed real source pair/calibration through S3 SDK.
- Claim job → download → Reco → validate → upload → publish.
- Capture timing/resource metrics.

### Phase 6 — Failure testing
- Worker kill/restart.
- Duplicate claims/finalize.
- Corrupt/truncated media.
- Output upload failure.
- Stale attempt publication.
- Disk-space failure where safely testable.

### Phase 7 — Panorama-quality qualification
- Separate from infrastructure acceptance.
- Evaluate seam/coverage/sync and later 1080p30 calibration.

Adjust phases to fit actual code discovered.

## 25. Acceptance Criteria

Define explicit acceptance criteria for the implementation POC. At minimum:

- existing architecture is reused rather than duplicated
- MinIO access uses AWS S3 SDK
- no MinIO-specific business logic
- worker runs natively on Windows
- Reco uses RTX 5060 Ti/DX12
- jobs use object identities, not host paths
- scratch paths are generated cross-platform
- Reco/FFmpeg invoked without shell strings
- durable claim/recovery works
- source objects remain immutable
- partial/stale output cannot publish
- final output passes strict validation
- final uploaded bytes have verified SHA-256
- result provenance/timings are recorded
- infrastructure and panorama-quality verdicts remain separate
- no Docker/WSL/Vulkan/AWS/YOLO requirement is introduced

## 26. Explicitly Deferred Work

Keep out of this POC unless existing code requires otherwise:

- Jetson implementation
- AWS Batch/EC2
- Dockerizing the worker
- WSL/Vulkan qualification
- YOLO
- HLS/MediaConvert
- CloudFront
- multi-worker scaling
- live-camera stitching
- Reco renderer redesign
- timestamp/drift engine redesign beyond documenting blockers
- production parent streaming changes

## 27. Final Response

When finished, provide a concise summary of:

- repositories inspected
- where the worker should live
- existing components that can be reused
- proposed database/job changes
- proposed worker module structure
- exact Reco/FFmpeg integration approach
- MinIO/S3 integration approach
- biggest implementation risks
- unresolved questions
- recommended first implementation phase

Do not implement anything beyond creating the planning document.
