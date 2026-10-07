I just forked the `reco-project/video-stitcher` repository. Before modifying any code, I want you to perform a thorough technical audit of this repository and determine whether it is useful for a project I am building.

## My project

I have an Orange Pi 4 Pro that records a soccer game using two cameras.

The cameras are positioned as:

* Left camera
* Right camera

Each camera is recorded independently using GStreamer into an MP4 file.

Conceptually, each recording session produces something like:

```text
left.mp4
right.mp4
```

The long-term goal is to combine the two camera feeds into one wide/panoramic soccer-field video.

The Orange Pi is primarily responsible for recording. It has limited CPU/GPU resources compared with a desktop/server, so I do not want to add heavy processing to it unless doing so is realistically practical.

I am considering two architectures:

### Architecture A — Edge/Inline Processing

```text
Camera Left ──┐
              ├─> Orange Pi ─> Stitch / Combine ─> final.mp4
Camera Right ─┘
```

Potentially this could be:

```text
GStreamer Left ──┐
                 ├─> stitching pipeline
GStreamer Right ─┘
```

or stitching the two recorded MP4 files locally after recording.

### Architecture B — Cloud Processing

```text
Camera Left ──> left.mp4 ──┐
                           ├─> upload to AWS ─> stitch/process ─> final.mp4
Camera Right ─> right.mp4 ─┘
```

My existing architectural direction has been:

```text
Orange Pi
   │
   ├── Record Left MP4
   ├── Record Right MP4
   │
   └── Upload recordings
           │
           ▼
          AWS
           │
           ├── Stitch
           ├── Additional video processing
           └── eventually computer vision / YOLO
```

I need to determine whether this repository changes that architecture.

---

# Phase 1 — Repository Inventory

Do NOT modify the repository yet.

First, inspect the entire repository and build a technical inventory.

Determine:

1. What the project actually does.
2. What problem it was designed to solve.
3. Whether it performs actual image/video stitching or something else that happens to be called "stitching."
4. What languages, frameworks, libraries, and native dependencies it uses.
5. Its major modules/components.
6. Its entrypoints.
7. How data flows through the system.
8. What inputs it expects.
9. What outputs it produces.
10. Whether it operates on:

    * individual images
    * prerecorded video
    * live video streams
    * RTSP streams
    * GStreamer pipelines
    * OpenCV frames
    * FFmpeg
    * some other mechanism
11. Whether processing is frame-by-frame or uses a video-native pipeline.
12. Whether it requires camera calibration.
13. Whether it calculates homography/perspective transforms.
14. Whether it finds feature matches automatically.
15. Whether camera geometry is recalculated continuously or calculated once and reused.
16. How it handles overlap between cameras.
17. Whether it performs:

    * warping
    * alignment
    * cropping
    * blending
    * seam finding
    * exposure compensation
    * distortion correction
    * synchronization
18. Whether it encodes the final output itself or only produces stitched frames.
19. Whether hardware acceleration is supported.
20. Whether there are assumptions about NVIDIA CUDA, x86, GPU availability, or specific hardware.

Create a concise architecture diagram of the repository based on the actual code.

Do not rely heavily on the README. Verify claims against the implementation.

---

# Phase 2 — Trace the Real Stitching Pipeline

Identify the exact code responsible for stitching.

Trace the flow from input to final output.

For every important stage, identify:

* source file
* function/class
* purpose
* inputs
* outputs
* external library being used

I want to understand whether the core pipeline is roughly:

```text
Decode
  ↓
Synchronize
  ↓
Feature Detection
  ↓
Feature Matching
  ↓
Homography
  ↓
Warp
  ↓
Blend
  ↓
Encode
```

or whether it works differently.

Call out which portions of the project are reusable independently.

For example, if the repository contains a useful OpenCV stitching engine buried inside application-specific code, identify that clearly.

---

# Phase 3 — Determine Resource Requirements

Evaluate the computational requirements of this code.

I specifically care about running it on an Orange Pi 4 Pro.

Determine:

* CPU requirements
* RAM requirements
* GPU requirements
* native library requirements
* ARM64 compatibility
* whether dependencies compile/run on Linux ARM64
* whether CUDA is required or optional
* whether OpenCL is supported
* whether hardware video decoding/encoding can be used
* whether GStreamer integration already exists
* whether FFmpeg integration exists

Look for obvious computational hotspots.

Pay particular attention to operations such as:

* feature detection every frame
* feature matching every frame
* homography calculation every frame
* full-resolution OpenCV frame copies
* CPU-based video decode
* CPU-based encoding
* large intermediate frame buffers
* expensive blending algorithms

Identify places where the current implementation would be unrealistic on the Orange Pi.

---

# Phase 4 — Evaluate My Specific Camera Setup

My cameras are fixed relative to each other during a recording session.

That is important.

For a fixed two-camera rig, investigate whether the repository can take advantage of this by doing something like:

```text
Calibration / setup
       │
       ▼
Calculate transform once
       │
       ▼
Store homography / calibration
       │
       ▼
For every frame:
    warp left
    warp right
    blend
    encode
```

instead of detecting and matching features on every frame.

Determine whether the repository already supports this architecture.

If it does not, determine how difficult it would be to extract/adapt its useful components to support it.

Also determine whether the repository assumes handheld/moving cameras, because that would make its design significantly different from my fixed-camera use case.

---

# Phase 5 — Video Synchronization

Two separate GStreamer pipelines generate the recordings.

Investigate whether the repository handles synchronization between the two videos.

Consider:

* differing start timestamps
* frame count differences
* dropped frames
* variable frame rate
* presentation timestamps
* clock drift
* one camera starting slightly before the other

Determine whether synchronization is already handled by the repository.

If not, explain what would need to be added.

For soccer footage, synchronization errors would be obvious when a player crosses the seam between cameras.

---

# Phase 6 — GStreamer Compatibility

Determine how difficult it would be to integrate the stitching engine directly with my GStreamer pipeline.

An ideal architecture might eventually look something like:

```text
Camera Left
    │
GStreamer
    │
    ├─────────────┐
                  ▼
             Stitcher
                  ▲
    ├─────────────┘
GStreamer
    │
Camera Right
```

or:

```text
left.mp4  ─┐
           ├─> GStreamer decode
right.mp4 ─┘
                 │
                 ▼
             stitch frames
                 │
                 ▼
          hardware encoder
                 │
                 ▼
             final.mp4
```

Evaluate whether the repository's code can realistically be inserted into either pipeline.

Do not assume that because something uses OpenCV it will integrate efficiently with GStreamer. Look for unnecessary copies between GStreamer buffers, OpenCV Mats, and encoder buffers.

---

# Phase 7 — Compare Edge vs AWS

Based on the implementation you find, compare these options:

## Option 1 — Stitch while recording

Orange Pi receives both camera streams and produces the stitched video in real time.

## Option 2 — Record first, stitch locally afterward

Orange Pi records:

```text
left.mp4
right.mp4
```

Then performs stitching after the game has finished.

## Option 3 — Upload originals and stitch in AWS

Orange Pi records:

```text
left.mp4
right.mp4
```

Uploads them to object storage.

AWS performs:

```text
decode
→ synchronization
→ stitching
→ encoding
→ computer vision
```

## Option 4 — Hybrid approach

Determine whether there is a useful middle ground, such as performing calibration or lightweight preprocessing on the Orange Pi while leaving expensive frame processing for AWS.

For each architecture evaluate:

* Orange Pi CPU usage
* RAM usage
* thermal impact
* likelihood of dropped recording frames
* implementation complexity
* recoverability if stitching fails
* upload bandwidth/storage requirements
* AWS compute requirements
* processing speed
* operational complexity
* scalability
* ability to run YOLO later
* ability to reprocess footage using improved algorithms

Preserving the original left/right recordings has value because they could potentially be reprocessed later.

---

# Phase 8 — AWS Suitability

If AWS remains the better architecture, evaluate how portable this repository would be to cloud compute.

Consider:

* Docker/container compatibility
* ARM vs x86
* ECS/Fargate suitability
* EC2 suitability
* AWS Batch suitability
* GPU instance suitability
* CPU-only processing
* ephemeral disk requirements
* memory requirements
* ability to pull source videos from S3 and upload the final artifact back to S3

Do not redesign the entire AWS system yet.

I mainly want to know whether this repository's stitching engine could reasonably become a containerized AWS processing worker.

---

# Phase 9 — Code Quality and Project Health

Evaluate the repository itself.

Look for:

* abandoned dependencies
* deprecated APIs
* incomplete features
* hard-coded paths
* hard-coded camera assumptions
* poor error handling
* race conditions
* synchronization problems
* memory leaks
* unnecessary frame copies
* dead code
* platform-specific code
* undocumented assumptions
* test coverage
* build reproducibility

Also examine commit history if available locally and determine whether the repository appears mature, experimental, or proof-of-concept quality.

---

# Phase 10 — Final Recommendation

At the end, give me a clear recommendation.

Use one of these conclusions if appropriate:

### A — Use this repository substantially as-is

### B — Use this repository, but extract/refactor its stitching engine

### C — Use parts of this repository only as a reference

### D — This repository is not appropriate for this project

Then answer the primary architectural question:

> Should the Orange Pi stitch the two videos itself, or should it preserve the two source recordings and upload them to AWS for processing?

Do not base this answer on generic assumptions about edge vs cloud processing.

Base it specifically on:

* this repository's actual implementation
* the Orange Pi 4 Pro's realistic capabilities
* the fact that recording reliability is the primary responsibility of the Orange Pi
* two simultaneous camera streams
* MP4/GStreamer input
* eventual YOLO/computer-vision processing
* the ability to reprocess original footage later

---

# Deliverable

Create a report in:

```text
docs/video-stitcher-evaluation.md
```

Use this structure:

```text
# Video Stitcher Repository Evaluation

## Executive Summary

## What This Repository Actually Does

## Repository Architecture

## Dependency Inventory

## Stitching Pipeline

## Input and Output Formats

## Camera Calibration and Homography

## Video Synchronization

## Performance Characteristics

## ARM64 / Orange Pi Compatibility

## GStreamer Integration Potential

## Fixed Two-Camera Optimization Opportunities

## Edge Processing Evaluation

## AWS Processing Evaluation

## Architecture Comparison

## Reusable Components

## Missing Components

## Technical Risks

## Recommended Architecture

## Proposed Next Steps
```

Include file paths and function/class names throughout the report so that conclusions can be verified against the code.

When discussing an important behavior, reference the implementation responsible for it.

Do not modify application code.

Do not begin refactoring.

Do not install unnecessary dependencies.

The purpose of this task is **repository reconnaissance and architectural evaluation**, not implementation.
