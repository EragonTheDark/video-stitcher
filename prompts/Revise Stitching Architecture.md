# Revise ScoutCam Stitching Investigation: Local PC + Docker + MinIO

We have made an architectural decision based on the findings in:

```text
docs/aws-stitching-investigation.md
```

Update that document to reflect the new development and initial deployment architecture.

Do NOT discard the existing investigation or rewrite it from scratch. Preserve the valuable technical findings about:

* Current ScoutCam Cedar V3 recording architecture
* 1080p30 capture
* Reco architecture
* Calibration
* synchronization limitations
* GPU requirements
* FFmpeg
* wgpu/Vulkan/DX12
* validation requirements
* Jetson migration
* streaming
* object-storage contracts
* failure handling

The major change is **where stitching runs and where object storage lives initially**.

## New Decision

We currently have only ONE ScoutCam.

Rather than paying for AWS GPU compute while developing and operating a single device, we will initially run the ScoutCam backend and video-processing services on our existing local PC/server.

The development/initial deployment environment is:

* Windows 11 host
* NVIDIA GeForce RTX 5060 Ti 16 GB
* Docker / Docker Desktop
* Local MinIO object storage
* AWS S3 SDK for all object-storage access
* Cloudflare Tunnel for externally accessible services
* Reco as the stitching engine

Reco has already successfully detected and used the RTX 5060 Ti through DX12 on this machine.

The design must NOT tightly couple the application to MinIO, Windows, or this specific PC.

The objective is to develop the same portable processing service that can later run on:

1. NVIDIA Jetson
2. AWS GPU compute / AWS Batch if scale requires it

## Revised Architecture

The primary architecture should now be approximately:

```text
ScoutCam
   │
   │ completed left/right recording pair
   ▼
Mobile transfer/upload flow
   │
   ▼
ScoutCam Backend
   │
   │ AWS S3 SDK
   ▼
MinIO
   │
   ├── left.mp4
   ├── right.mp4
   ├── session.json
   └── calibration
          │
          ▼
Docker Stitch Worker
   │
   │ NVIDIA RTX 5060 Ti
   │ Reco
   ▼
stitched.mp4
   │
   ▼
MinIO
   │
   ▼
ScoutCam Backend
   │
   ▼
Cloudflare Tunnel
   │
   ▼
Client / Parent Video Playback
```

Cloudflare Tunnel is the network exposure mechanism.

MinIO should remain private behind the application/storage boundary unless there is a deliberate reason to expose a signed object URL.

## Critical Design Principle: Speak S3, Not MinIO

Application code must use the AWS S3 SDK and S3-compatible APIs.

Do NOT use a MinIO-specific SDK.

MinIO is simply the currently configured S3-compatible object-storage endpoint.

Conceptually:

```text
Storage Adapter
      │
      │ AWS S3 SDK
      ▼
S3-compatible API
      │
      ├── Development/V1 → MinIO
      │
      └── Future         → AWS S3
```

The storage configuration should be environment-driven, including concepts such as:

```text
S3_ENDPOINT
S3_REGION
S3_BUCKET
S3_ACCESS_KEY_ID
S3_SECRET_ACCESS_KEY
S3_FORCE_PATH_STYLE
```

Determine the exact configuration based on the language/runtime and existing codebase rather than blindly adopting these names.

The application should not contain business logic such as:

```text
if minio ...
else if aws ...
```

Storage differences belong in configuration or a storage adapter.

## Docker Requirement

The stitch worker should be designed and developed as a Dockerized service.

The Docker container should encapsulate as much of the processing environment as practical:

* Worker/orchestration application
* Reco
* FFmpeg
* ffprobe
* validation tools
* required runtime libraries
* processing configuration

GPU drivers remain host/platform responsibilities.

The worker should have a clean processing contract rather than depending on Windows paths or manually mounted game folders.

For example:

```text
StitchJob
    │
    ├── recordingId
    ├── leftObjectKey
    ├── rightObjectKey
    ├── calibrationObjectKey
    ├── outputObjectKey
    └── stitch settings
```

The worker performs:

```text
Receive job
   ↓
Download objects through S3 API
   ↓
Create isolated local scratch workspace
   ↓
Probe inputs
   ↓
Validate inputs
   ↓
Run Reco
   ↓
Validate output
   ↓
Fast-start MP4 if necessary
   ↓
Upload result through S3 API
   ↓
Publish result metadata/status
   ↓
Clean scratch workspace
```

Reco itself should NOT know about MinIO, S3, Cloudflare, or the backend.

## Local GPU Runtime

Investigate the implications of running the worker through Docker on the current Windows 11 + NVIDIA RTX 5060 Ti environment.

Reco currently works natively on Windows using DX12.

Determine what is required for the Dockerized version.

Specifically investigate whether the preferred Docker worker should use:

* Linux containers through Docker Desktop / WSL2
* NVIDIA Container Toolkit GPU passthrough
* Vulkan inside the Linux container
* CUDA/NVDEC/NVENC availability
* or another appropriate configuration

Do NOT assume that because Reco works through DX12 on Windows that the Linux Docker container will automatically work.

Document the exact GPU/runtime validation we should perform.

The first acceptance test should include verifying the GPU adapter selected by Reco inside the container.

## Portable Worker Boundary

Design the worker so that the orchestration layer is portable.

Separate responsibilities approximately as:

```text
Stitch Worker
│
├── Job orchestration
│
├── StorageAdapter
│     └── S3-compatible API
│
├── StitchEngine
│     └── Reco
│
├── Media/Platform Adapter
│     ├── desktop NVIDIA
│     └── future Jetson NVIDIA
│
├── Validator
│     └── ffprobe / decode validation
│
└── Result Publisher
```

Do not over-engineer interfaces if the existing codebase already has appropriate boundaries.

The important requirement is that MinIO, Windows, AWS and Jetson-specific assumptions do not leak throughout the application.

## Object Storage Layout

Revisit the S3 layout proposed in the AWS investigation.

Preserve the useful immutable/versioned concepts, but simplify where appropriate for one ScoutCam.

A likely shape is:

```text
recordings/{recordingId}/
    source/
        left.mp4
        right.mp4
        session.json

    calibration/
        match.json

    processing/
        stitch-request.json

    output/
        stitched.mp4
        result.json
```

However, inspect existing ScoutCam backend/domain models before adopting this exact layout if those repositories are available.

Do not introduce unnecessary multi-tenant complexity for the current single-device system merely because the previous AWS design proposed it.

Preserve enough structure that moving the bucket from MinIO to AWS S3 does not require changing object identities throughout the application.

## Local Job Processing

Replace AWS Batch as the primary job execution mechanism.

Investigate the simplest reliable local mechanism.

Possibilities include:

* backend launches/queues Docker worker jobs
* persistent worker container consumes jobs from a queue/database
* existing application job infrastructure
* lightweight database-backed job queue

Prefer the simplest reliable mechanism for ONE ScoutCam.

Do NOT introduce Kafka, Kubernetes, RabbitMQ, Redis, SQS, Step Functions, or similar infrastructure unless an existing component makes it clearly justified.

We need:

```text
Recording uploaded
      ↓
Stitch job queued
      ↓
Worker claims job
      ↓
STITCHING
      ↓
Result validated
      ↓
STITCH_COMPLETE
```

Failures must be retryable without destroying or overwriting the source recordings.

## Preserve the State Model

Retain the useful processing states from the previous investigation:

```text
UPLOADING
UPLOADED
STITCH_QUEUED
STITCHING
STITCH_COMPLETE
STITCH_FAILED
```

Keep recording/save state separate from cloud/local processing state where appropriate.

The worker exiting successfully is NOT sufficient for `STITCH_COMPLETE`.

The output must pass validation before publication.

## Output Validation

Preserve the previous investigation's strong validation requirements.

At minimum evaluate:

* ffprobe succeeds
* complete decode succeeds
* expected codec/container
* expected dimensions
* expected FPS
* expected duration tolerance
* expected frame-count tolerance
* non-zero file size
* output checksum
* beginning/end frame validity
* temporal alignment assumptions
* Reco process exit status

A truncated but playable MP4 must not be considered successful.

## Synchronization and Calibration

Do NOT remove the previous report's synchronization findings simply because processing is local.

The same limitations remain:

* integer frame offset
* ordinal frame pairing
* possible independent camera/drop history
* need to validate beginning/middle/end
* calibration must match the physical rig/lenses/resolution

Clearly distinguish these from infrastructure concerns.

Running on an RTX 5060 Ti does not fix synchronization or calibration.

## Streaming

Revise streaming recommendations around the local architecture.

Initial delivery should favor simplicity:

```text
stitched.mp4
      ↓
MinIO
      ↓
ScoutCam Backend / authorized media endpoint
      ↓
Cloudflare Tunnel
      ↓
Parent/client
```

Investigate HTTP range-request support and seeking.

Determine whether serving through an application endpoint, controlled signed object URL, or another approach is most appropriate with MinIO + Cloudflare.

Keep the bucket private.

Start with progressive H.264 MP4.

Retain the recommendation to perform a fast-start remux if Reco's output does not place MP4 metadata appropriately for progressive playback.

HLS can remain a future optimization if adaptive streaming becomes necessary.

## Jetson Migration

This is a major reason for Dockerizing the worker.

Design toward:

```text
TODAY

MinIO
  ↓
Docker Stitch Worker
  ↓
RTX 5060 Ti
  ↓
MinIO
```

Eventually:

```text
JETSON

Cameras / local recordings
        ↓
Docker Stitch Worker
        ↓
Jetson NVIDIA GPU
        ↓
S3 SDK
        ↓
AWS S3
```

The application-level worker logic should remain as similar as practical.

Investigate a multi-architecture image strategy:

```text
linux/amd64
linux/arm64
```

But do NOT assume the same media/GPU libraries will work unchanged.

Explicitly document the likely differences between:

* desktop NVIDIA GPU
* Jetson integrated NVIDIA GPU
* NVENC/NVDEC
* JetPack
* NVMM/NvBufSurface
* CUDA/Vulkan
* ARM64 vs x86_64

Prefer shared source and contracts with platform-specific image stages/adapters where necessary.

## AWS Becomes a Scale-Out Option

Do not delete the AWS research.

Move it into a section such as:

```text
Future Scale-Out: AWS GPU Processing
```

The previous AWS investigation contains useful information about:

* GPU EC2
* AWS Batch
* On-Demand
* Spot
* S3
* retries
* immutable attempts
* validation
* worker portability

Preserve that research as the path we can take if ScoutCam grows beyond the capacity or reliability requirements of one local processing machine.

Conceptually:

```text
                    Stitch Job
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          Local PC    Jetson    AWS Batch
```

The backend should not require knowledge of which compute target produced the final validated asset.

## Cost Model

Replace the AWS-first cost discussion with the local architecture.

The local development environment already exists, so distinguish:

* incremental infrastructure cost
* electricity
* storage
* Internet upload/download bandwidth
* Cloudflare considerations
* backup/retention
* operational availability of the local PC

Do not claim local processing is literally free.

AWS GPU cost should remain documented as future scale-out cost.

## Revised Proof of Concept

The immediate POC should now be:

1. Use one existing ScoutCam recording pair.
2. Establish/verify calibration and synchronization.
3. Build a Docker image containing Reco + FFmpeg + the minimal worker.
4. Run it locally with NVIDIA GPU access.
5. Verify Reco selects the intended GPU inside the container.
6. Upload/source the input through MinIO using the AWS S3 SDK.
7. Download to container scratch storage.
8. Stitch.
9. Validate the entire output.
10. Fast-start remux if necessary.
11. Upload `stitched.mp4` and `result.json` back to MinIO.
12. Measure:

* download time
* stitch/render time
* validation time
* upload time
* total processing time
* average/peak CPU
* average/peak GPU
* VRAM
* RAM
* scratch disk
* output bitrate
* output size

13. Verify browser playback and seeking through the intended Cloudflare/backend path.

Do NOT implement the POC as part of this task.

## Document Changes

Rename:

```text
docs/aws-stitching-investigation.md
```

to something architecture-neutral such as:

```text
docs/stitching-processing-architecture.md
```

Update the title accordingly.

The beginning of the report should clearly state the new decision:

**For the current single-ScoutCam deployment, use a Dockerized local stitching worker on the existing RTX 5060 Ti PC with MinIO accessed through the AWS S3 SDK. Preserve the worker and storage contracts so processing can later move to Jetson or AWS without redesigning the application.**

Then reorganize the document so the major sections are approximately:

1. Decision
2. Current ScoutCam Recording Architecture
3. Current Reco Stitching Implementation
4. Local Processing Architecture
5. Docker/GPU Runtime
6. S3-Compatible Storage Contract
7. Local Job Processing
8. Processing State Model
9. Synchronization and Calibration
10. Output Validation
11. Streaming / Cloudflare
12. Security
13. Jetson Migration
14. Future Scale-Out to AWS
15. Cost Considerations
16. Open Questions
17. Smallest Useful Proof of Concept

Preserve useful evidence and source references from the existing investigation.

Remove or rewrite AWS-first assumptions that are no longer valid, but do not throw away AWS research that remains useful for future scale.

## Important Constraints

* Investigation/documentation only.
* Do not modify ScoutCam production code.
* Do not modify Reco.
* Do not modify the Orange Pi.
* Do not create Dockerfiles yet.
* Do not install Docker/NVIDIA components.
* Do not start or stop containers.
* Do not change MinIO.
* Do not provision AWS resources.
* Do not change Cloudflare configuration.
* Do not create backend APIs.
* Do not implement the worker.
* Preserve existing working-tree changes unrelated to this document.

Before finishing, review the updated document for contradictions left over from the previous AWS-first architecture.

Specifically search for stale statements implying:

* AWS is the primary processing environment
* S3 means AWS S3 specifically
* AWS Batch is required
* EC2 is the first POC
* CloudFront is required for initial playback
* MediaMTX is still active
* the recorder is still 720p
* MinIO-specific APIs should be used
* the local PC architecture is temporary throwaway code

The revised architecture should make clear that **the local implementation is the real first implementation of the portable worker architecture**, not a disposable prototype.

When complete, summarize:

* what changed in the document
* the proposed local architecture
* Docker/GPU approach
* MinIO/S3 abstraction
* local job execution recommendation
* Jetson migration path
* what AWS functionality remains relevant for future scale
* the exact next POC we should build
