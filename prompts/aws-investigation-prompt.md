# ScoutCam AWS Video Stitching Investigation

I want to investigate temporarily moving ScoutCam's video stitching workload to AWS until the Jetson-based version of the hardware is ready.

Do NOT make any code changes yet. This is an architecture and codebase investigation only.

## Goal

ScoutCam currently records video from two cameras. I want to determine the best way to:

1. Record the two camera videos on ScoutCam.
2. Upload the original left/right recordings to Amazon S3.
3. Trigger an AWS compute job.
4. Run our existing stitching pipeline in AWS.
5. Produce a single stitched MP4.
6. Store the stitched MP4 back in S3.
7. Eventually make the stitched game available for streaming through CloudFront or another delivery layer.

Later, the AWS stitching worker will be replaced by on-device stitching on an NVIDIA Jetson. Because of that, I want the AWS implementation to be modular and replaceable.

The desired long-term contract is approximately:

```text
Current:

ScoutCam
   ├── left.mp4
   └── right.mp4
          ↓
         S3
          ↓
    AWS Stitch Worker
          ↓
    stitched.mp4
          ↓
         S3
          ↓
   Streaming / Download


Future:

Jetson
   ├── left camera
   └── right camera
          ↓
   On-device Stitcher
          ↓
    stitched.mp4
          ↓
         S3
          ↓
   Streaming / Download
```

The rest of ScoutCam should ideally not care where the stitching occurred.

## Phase 1 — Understand the Existing System

Inspect the repository and document:

* How ScoutCam currently records each camera.
* Camera interfaces and drivers being used.
* Current GStreamer pipelines.
* Whether MediaMTX is still involved anywhere.
* Video codec, container, resolution, framerate, bitrate, and other relevant recording settings.
* How left/right recordings are named and organized.
* How recordings are currently uploaded or transferred.
* Existing S3/AWS integration.
* Existing job/workflow infrastructure.
* Dockerfiles and container architecture.
* Node/TypeScript services involved in recording and processing.
* Any Python, FFmpeg, GStreamer, OpenCV, or native components involved.
* Configuration/environment variables relevant to video processing.
* How the mobile/client application learns that a recording is complete or available.

Trace the complete lifecycle of a recording from:

```text
Camera
  ↓
Capture
  ↓
Encode
  ↓
Local storage
  ↓
Upload
  ↓
Processing
  ↓
Final video
```

Reference actual files, modules, functions, classes, commands, and configuration from the repository.

Do not assume the architecture from documentation if the implementation disagrees with it. Treat the current code as authoritative and point out discrepancies.

## Phase 2 — Investigate the Existing Stitcher

Find all code related to video stitching.

Determine exactly how it works.

Document:

* Stitching algorithm.
* Whether it uses FFmpeg.
* Whether it uses GStreamer.
* Whether it uses OpenCV.
* Whether it uses CUDA/GPU acceleration.
* Whether it uses hardware video decoding/encoding.
* CPU requirements.
* GPU requirements.
* Memory requirements.
* Temporary disk/storage requirements.
* Input assumptions.
* Synchronization requirements between the two recordings.
* Calibration data required.
* Output codec/container.
* Whether processing is streaming or requires complete input files.
* Whether the current implementation can run inside a Linux Docker container.

Identify any dependencies that are specific to the Orange Pi/Allwinner hardware and would need to be replaced in AWS.

Also identify which parts could later run unchanged on NVIDIA Jetson.

## Phase 3 — Determine AWS Compute Options

Evaluate these options for running the existing stitcher:

* ECS/Fargate
* ECS on EC2
* AWS Batch
* AWS Batch using EC2 Spot
* Standard EC2
* GPU EC2
* Lambda only if technically appropriate

Do not assume GPU is necessary.

Base the recommendation on what the existing stitching implementation actually requires.

For each viable option, evaluate:

* Compatibility with the current stitcher.
* Container support.
* CPU/GPU availability.
* Ephemeral disk requirements.
* S3 integration.
* Startup latency.
* Operational complexity.
* Ability to scale to multiple games.
* Approximate cost characteristics.
* Whether Spot instances make sense.
* Changes required to the existing code.

Pay particular attention to whether AWS Batch + EC2 Spot would be appropriate for this workload since stitching occurs after a game and does not necessarily need to be real-time.

If GPU acceleration would materially improve processing, identify suitable NVIDIA-backed EC2 instance families and explain why.

## Phase 4 — Design a Proposed AWS Workflow

Design an architecture similar to:

```text
ScoutCam
   │
   ├── left.mp4
   └── right.mp4
          │
          ▼
         S3
          │
          ▼
     Job Trigger
          │
          ▼
   Stitch Worker
          │
          ▼
    stitched.mp4
          │
          ▼
         S3
```

Determine what should trigger the stitch job.

Consider:

* Explicit API call after both uploads complete.
* S3 events.
* SQS.
* EventBridge.
* Step Functions.

Prefer the simplest reliable architecture.

The system must not start stitching until BOTH camera recordings have successfully uploaded.

Consider failure/retry behavior including:

* One upload fails.
* Stitching crashes.
* Worker is interrupted.
* Spot instance is reclaimed.
* Output upload fails.
* Duplicate job events occur.

The workflow should be idempotent where practical.

## Phase 5 — Define the Storage Contract

Propose a clean S3 object structure.

For example:

```text
games/
  {gameId}/
      source/
          left.mp4
          right.mp4

      processed/
          stitched.mp4

      metadata/
          game.json
```

Determine whether this structure fits the existing ScoutCam domain model.

If another structure better matches the current codebase, recommend it instead.

Define what metadata ScoutCam should store to represent processing states such as:

```text
RECORDING
UPLOADING
UPLOADED
STITCH_QUEUED
STITCHING
STITCH_COMPLETE
STITCH_FAILED
```

Do not implement these states yet.

## Phase 6 — Preserve the Jetson Migration Path

This is important.

AWS stitching is temporary.

Eventually ScoutCam hardware will move to NVIDIA Jetson and perform stitching locally.

Design the boundary so we can eventually replace:

```text
S3 sources
    ↓
AWS Stitch Worker
    ↓
stitched.mp4
```

with:

```text
Jetson cameras
    ↓
Local Stitch Worker
    ↓
stitched.mp4
```

without redesigning the backend, mobile application, game model, or streaming system.

Identify the interface/contract that should separate:

* Recording
* Stitching
* Storage
* Streaming
* Future YOLO processing

YOLO should remain a separate downstream processing stage and should NOT be coupled to stitching.

## Phase 7 — Streaming

Briefly evaluate how the finished video could later be delivered to parents.

Consider:

```text
stitched.mp4
      ↓
     S3
      ↓
 CloudFront
      ↓
 Web/Mobile Player
```

Also explain whether we should initially stream the MP4 directly through CloudFront or add an HLS/MediaConvert stage.

Prefer simplicity for the first version.

Do NOT implement streaming yet.

## Phase 8 — Produce an Investigation Report

Create a Markdown report at:

```text
docs/aws-stitching-investigation.md
```

The report should contain:

1. Current ScoutCam recording architecture.
2. Current stitching architecture.
3. Relevant source files/modules.
4. Current video formats/settings.
5. Hardware-specific dependencies.
6. AWS compatibility findings.
7. Comparison of AWS compute options.
8. Recommended AWS architecture.
9. Proposed S3 layout.
10. Proposed processing lifecycle.
11. Failure/retry strategy.
12. Security considerations for S3/IAM.
13. Jetson migration strategy.
14. Streaming options.
15. Open questions.
16. Recommended proof-of-concept.

For the proof-of-concept, identify the minimum work necessary to take ONE existing ScoutCam left/right recording pair, upload it to S3, run the stitcher in AWS, and return a stitched MP4.

Do not build the proof-of-concept yet.

## Important Constraints

* Do not modify production code.
* Do not provision AWS resources.
* Do not run Terraform/CDK/CloudFormation against AWS.
* Do not change recording behavior.
* Do not change the existing mobile application.
* Do not introduce YOLO into the stitching pipeline.
* Do not assume GPU is required.
* Do not redesign working ScoutCam components unnecessarily.
* Prefer existing code and abstractions where practical.
* Clearly distinguish what exists today from what you recommend adding.

When finished, give me a concise summary containing:

* How stitching currently works.
* Whether it can run in AWS unchanged.
* What must change.
* Which AWS compute option appears most appropriate.
* Whether CPU or GPU compute is appropriate.
* The proposed end-to-end architecture.
* The smallest useful AWS proof-of-concept we should build next.
