# Final Revision: Native Cross-Platform ScoutCam Stitch Worker

Update the existing:

```text
docs/stitching-processing-architecture.md
```

to reflect one final architectural decision.

This is a DOCUMENTATION-ONLY task.

Do not modify production code, Reco, ScoutCam, MinIO, Cloudflare, Windows/WSL configuration, drivers, or infrastructure.

## Final Architectural Decision

Docker is NO LONGER a requirement for the ScoutCam stitch worker.

The worker will be implemented as a **cross-platform TypeScript/Node.js service running natively on the host operating system**.

The primary portability boundary is the application/service contract, NOT the operating system container.

The current processing host is:

* Windows 11
* NVIDIA GeForce RTX 5060 Ti 16 GB
* Native Windows Reco executable
* Native Windows FFmpeg/ffprobe
* Reco using its already-qualified DX12 backend
* Local MinIO for S3-compatible object storage
* AWS S3 SDK from the TypeScript worker
* Cloudflare Tunnel for externally exposed application services where appropriate

The future processing host is expected to be:

* NVIDIA Jetson
* Linux ARM64
* Native Linux/ARM64 Reco executable
* Jetson-qualified GPU/media stack
* Native Linux FFmpeg/ffprobe or appropriate Jetson media implementation
* The SAME TypeScript worker source wherever practical
* AWS S3 as the eventual object store

AWS GPU processing remains an optional future scale-out executor.

## Why This Decision Changed

The previous architecture made Linux Docker a portability requirement.

Further investigation established that this creates an unnecessary graphics qualification problem.

Reco's stitching renderer uses wgpu/WGSL.

The existing native Windows path is already proven:

```text
Reco
  ↓
wgpu
  ↓
DX12
  ↓
RTX 5060 Ti
```

The proposed Linux Docker path would require:

```text
Reco
  ↓
wgpu
  ↓
Vulkan
  ↓
WSL2 / Docker Desktop GPU virtualization
  ↓
Windows NVIDIA driver
  ↓
RTX 5060 Ti
```

Existing investigation showed NVIDIA compute visibility in WSL but did NOT establish working hardware Vulkan for Reco. Existing Vulkan evidence exposed software llvmpipe.

CUDA cannot simply replace Vulkan/DX12 for Reco's renderer because CUDA is not a wgpu graphics backend. Reco uses CUDA for NVIDIA interop/accelerated input paths, while its actual warp/blend renderer is implemented using wgpu/WGSL.

Therefore:

**Do not introduce a new Linux/Vulkan problem on a machine where the native Windows/DX12 path already works.**

Also make clear that Docker would not eliminate Jetson-specific platform work anyway. Jetson will still require ARM64 compilation and qualification of its own GPU, codec, JetPack and media stack.

## Core Portability Principle

The portable component is:

```text
scoutcam-stitch-worker
```

The worker should be written in TypeScript and designed to run under Node.js on both Windows and Linux.

Conceptually:

```text
                     StitchJob
                         │
                         ▼
              TypeScript Stitch Worker
              ┌───────────────────────┐
              │ Job orchestration     │
              │ S3 storage adapter    │
              │ scratch management    │
              │ Reco invocation       │
              │ validation            │
              │ result publication    │
              └───────────┬───────────┘
                          │
                    Native tools
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
         WINDOWS                     JETSON
        reco.exe                    reco
        ffmpeg.exe                  ffmpeg
        ffprobe.exe                 ffprobe
           │                          │
          DX12                     qualified
           │                       Jetson GPU
      RTX 5060 Ti                    stack
```

The application-level contract should remain the same.

The native media/GPU implementation underneath it is allowed to vary by platform.

## Native Process Model

The worker should invoke native executables using Node's process APIs.

Prefer:

```text
child_process.spawn(executable, args)
```

or an equivalent safe Node API.

Do NOT build shell command strings.

Arguments must remain separate from executable paths.

Capture:

* stdout
* stderr
* exit code
* termination signal
* process start/end time
* relevant Reco progress
* timeout/cancellation behavior

The worker must never treat exit code 0 alone as successful stitching.

Independent output validation remains required.

## Platform-Agnostic Filesystem Rules

The worker must avoid hard-coded Windows or Linux paths.

Do NOT embed paths such as:

```text
C:\ScoutCam\...
C:\temp\...
/tmp/scoutcam/...
/opt/scoutcam/...
```

in business logic.

Use Node APIs such as:

```text
node:path
node:os
node:fs/promises
```

Use:

```text
path.join(...)
path.resolve(...)
path.basename(...)
os.tmpdir()
fs.mkdtemp(...)
```

where appropriate.

Job requests must contain S3 object identities, NOT host filesystem paths.

Example:

```text
recordingId
leftObjectKey
rightObjectKey
calibrationObjectKey
outputObjectKey
```

The worker creates host-appropriate scratch paths internally.

Scratch locations may also be configurable for hosts where a dedicated high-capacity SSD is preferable to the OS temporary directory.

## Executable Discovery / Platform Adapter

Do not scatter platform checks throughout the worker.

Centralize native-tool resolution.

For example, conceptually:

```text
PlatformMediaAdapter
  ├── reco executable
  ├── ffmpeg executable
  ├── ffprobe executable
  ├── graphics/backend settings
  ├── codec settings
  └── platform-specific environment
```

This does NOT need to become an elaborate plugin framework.

A configuration-driven implementation is sufficient.

Example conceptual configuration:

```text
Windows:

RECO_EXECUTABLE=<path to reco.exe>
FFMPEG_EXECUTABLE=<path to ffmpeg.exe>
FFPROBE_EXECUTABLE=<path to ffprobe.exe>
RECO_GPU_BACKEND=dx12


Jetson/Linux:

RECO_EXECUTABLE=<path to reco>
FFMPEG_EXECUTABLE=<path to ffmpeg>
FFPROBE_EXECUTABLE=<path to ffprobe>
RECO_GPU_BACKEND=<qualified Jetson/Linux backend>
```

Do not assume the exact environment-variable names until implementation.

The important point is that platform-specific executable locations and GPU/media settings are configuration, not business logic.

## Windows Is Now the First-Class Initial Runtime

Revise the document so Windows is no longer described merely as a reference/debug route.

For the current deployment, Windows is the actual processing runtime.

The intended flow is:

```text
ScoutCam
   ↓
Mobile
   ↓
ScoutCam Backend
   ↓
AWS S3 SDK
   ↓
MinIO
   │
   ├── left.mp4
   ├── right.mp4
   ├── session.json
   └── calibration
          │
          ▼
Native TypeScript Stitch Worker
          │
          ├── S3 download
          ├── scratch workspace
          ├── reco.exe
          ├── DX12
          ├── RTX 5060 Ti
          ├── ffprobe validation
          └── fast-start preparation
          │
          ▼
MinIO
   │
   └── stitched.mp4 + result.json
```

Preserve concurrency one for the current single-ScoutCam deployment.

## Docker's New Role

Docker should become OPTIONAL infrastructure.

It may still be useful for:

* MinIO
* backend services
* PostgreSQL or another database
* development dependencies
* future cloud workers
* future Jetson deployment if it proves beneficial

But:

**The ScoutCam stitching architecture must not require the stitch worker itself to run in Docker.**

Do not spend engineering effort forcing native Windows Reco through Linux Docker merely for deployment uniformity.

Docker can be reconsidered per target later.

If AWS GPU processing is adopted, a Linux container will likely still be appropriate there.

If Jetson deployment benefits from NVIDIA's container ecosystem, the worker may later be containerized there.

Those are deployment decisions, not requirements of the worker architecture.

## S3-Compatible Storage Contract Remains Unchanged

Keep the existing decision:

**Speak S3, not MinIO.**

The TypeScript worker should use the AWS S3 SDK.

MinIO remains the current S3-compatible endpoint.

Future AWS S3 should require primarily configuration/deployment changes rather than rewriting application storage logic.

Conceptually:

```text
TypeScript Worker
      │
      ▼
AWS S3 SDK
      │
      ├── Today  → MinIO
      └── Future → AWS S3
```

Do NOT use a MinIO-specific SDK.

Preserve the existing immutable/versioned object-key strategy where useful.

## Worker Boundary

Retain approximately:

```text
Stitch Worker
│
├── Job orchestration
│
├── StorageAdapter
│     └── AWS S3 SDK
│
├── StitchEngine
│     └── RecoRunner
│
├── PlatformMediaAdapter
│     ├── Windows / DX12
│     ├── Jetson / Linux
│     └── future AWS Linux
│
├── Validator
│     └── ffprobe / full decode
│
└── ResultPublisher
```

These should normally be modules within one TypeScript application.

Do NOT create unnecessary microservices.

## Reco Remains Platform-Native

Reco itself should remain independent from:

* MinIO
* AWS
* S3
* Cloudflare
* backend job tables
* application authorization

Reco consumes local files and produces a local file.

The TypeScript worker handles:

```text
Object storage
      ↓
local scratch
      ↓
Reco
      ↓
local result
      ↓
validation
      ↓
object storage
```

Preserve this separation.

## FFmpeg Remains Platform-Native

Do not attempt to embed FFmpeg behavior into TypeScript.

Use the native FFmpeg/ffprobe executables appropriate to the host.

The worker should invoke them through the same safe process-runner abstraction used for Reco.

Windows and Linux binaries may differ.

The validation contract must not.

## Job Execution

Retain the recommendation:

**one persistent TypeScript worker, concurrency one, claiming durable jobs from the backend.**

No Docker lifecycle orchestration is required.

The worker can run natively as an ordinary host service/process.

For Windows, investigate later whether deployment should use:

* Windows Service
* WinSW
* NSSM
* Scheduled Task
* another existing service-management mechanism

Do NOT select or implement one during this documentation task.

For Jetson/Linux, systemd is a likely deployment option, but again that is deployment configuration rather than application architecture.

Do not introduce:

* Kubernetes
* Redis
* RabbitMQ
* Kafka
* SQS
* Docker socket control
* container-per-job execution

for the current one-ScoutCam environment.

## Preserve Existing Reliability Design

Do NOT weaken the existing architecture simply because Docker is removed.

Preserve:

* durable jobs
* job keys
* attempt IDs
* leases
* fencing tokens
* idempotent completion
* immutable source objects
* attempt-specific output objects
* retries
* scratch cleanup
* source preservation
* independent validation
* conditional result publication

A native process crash must be treated exactly as a container crash would have been.

## Validation Remains Critical

Keep the existing output-validation requirements.

Success requires more than:

```text
reco exited 0
```

Validate:

* ffprobe succeeds
* expected codec/container
* expected dimensions
* expected FPS
* full-file decode
* expected frame count
* expected duration
* beginning/end validity
* non-zero/plausible file size
* SHA-256
* synchronization expectations
* visual seam/geometry qualification where required

Only validated final bytes can become `STITCH_COMPLETE`.

## Calibration and Synchronization

Preserve all current findings.

Removing Docker does NOT solve:

* integer frame offset
* ordinal pairing
* drift
* dropped frames
* fractional exposure skew
* calibration quality
* lens profile accuracy
* seam quality
* full-field framing

Keep infrastructure qualification separate from panorama-quality qualification.

The native Windows/DX12 path is already proven to execute Reco, but that does NOT mean the current calibration is production-quality.

## Jetson Migration

Rewrite the Jetson section around **source portability rather than container portability**.

The intended migration is:

```text
TODAY

Windows 11
   ↓
Node.js
   ↓
same TypeScript worker source
   ↓
native reco.exe
   ↓
DX12
   ↓
RTX 5060 Ti


FUTURE

Jetson Linux ARM64
   ↓
Node.js
   ↓
same TypeScript worker source
   ↓
native ARM64 Reco
   ↓
qualified Jetson GPU/media stack
```

Application contracts shared between the platforms should include:

* StitchJob schema
* result schema
* StorageAdapter
* S3 object identities
* job lifecycle
* retries
* validation rules
* publication semantics
* logging/metrics schema where practical

Platform-specific differences may include:

* Reco executable
* graphics backend
* FFmpeg build
* NVENC/NVDEC availability
* JetPack
* NVMM/NvBufSurface
* CUDA/Vulkan interoperability
* codec selection
* service manager
* filesystem/scratch configuration

Do not pretend these differences disappear because both systems could theoretically run containers.

## Future AWS Scale-Out

Preserve the existing AWS research.

AWS remains a future executor:

```text
                    StitchJob
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
       Windows PC     Jetson      AWS Worker
```

The same application-level job/result/storage contracts should apply.

AWS may use a Linux Docker container because containerization makes sense for Batch/EC2 deployment.

That does not mean the Windows or Jetson deployments must use the same container.

If AWS is adopted later:

```text
AWS Batch / GPU EC2
        ↓
Linux worker deployment
        ↓
same logical TypeScript worker
        ↓
Linux Reco
        ↓
Vulkan / NVIDIA
```

Qualification remains target-specific.

## Streaming

Keep the current streaming investigation and Cloudflare caveats.

The worker architecture decision does not change the media-delivery architecture.

Continue to treat:

* stitching
* storage
* application control
* media delivery

as separate concerns.

Do not make Cloudflare Tunnel or MinIO part of Reco.

## Revised Proof of Concept

Replace the Docker-oriented POC with a native Windows POC.

The smallest useful implementation should now be:

1. Use one existing verified ScoutCam recording pair.
2. Establish/verify calibration and synchronization.
3. Create the cross-platform TypeScript worker skeleton.
4. Configure native Windows:

   * Reco
   * FFmpeg
   * ffprobe
5. Verify Reco identifies and uses:

   * NVIDIA RTX 5060 Ti
   * DX12
6. Seed/upload the source pair and calibration into MinIO using the AWS S3 SDK.
7. Create one durable stitch job.
8. Worker claims the job.
9. Download objects through the AWS S3 SDK into a generated scratch workspace.
10. Verify hashes and probe inputs.
11. Invoke native Reco safely using argument arrays.
12. Capture progress/stdout/stderr/exit status.
13. Produce the stitched MP4.
14. Perform strict independent validation.
15. Fast-start remux if required.
16. Validate the final delivered bytes.
17. Upload immutable `stitched.mp4` and `result.json` through the AWS S3 SDK.
18. Conditionally publish the accepted result.
19. Clean scratch.
20. Measure:

    * download time
    * stitch time
    * validation time
    * remux time
    * upload time
    * total wall time
    * CPU usage
    * GPU usage
    * VRAM
    * RAM
    * scratch high-water mark
    * output size
    * output bitrate
21. Exercise restart/failure behavior.
22. Separately evaluate panorama quality.

Do NOT implement this POC during this task.

## Remove the WSL/Vulkan Gate From the Initial Architecture

This is important.

The document currently treats:

```text
Windows
  ↓
Docker Desktop
  ↓
WSL2
  ↓
Linux
  ↓
Vulkan
  ↓
RTX
```

as the intended first runtime and therefore treats hardware Vulkan inside WSL as a blocker.

That is no longer true.

Move the WSL/Vulkan findings into a historical/optional deployment note.

They remain useful if we later experiment with Linux containers on Windows, but they are NOT a blocker for the current architecture.

The current runtime gate is simply:

```text
native Windows Reco
      ↓
DX12
      ↓
RTX 5060 Ti
```

which has already demonstrated successful short renders.

The remaining gates are now worker integration, full-job validation, calibration/synchronization quality and sustained full-game performance.

## Update the Document Structure

Keep:

```text
docs/stitching-processing-architecture.md
```

Suggested organization:

1. Decision
2. Current ScoutCam Recording Architecture
3. Current Reco Stitching Implementation
4. Native Cross-Platform Processing Architecture
5. Windows Processing Runtime
6. TypeScript Worker Design
7. Platform/Media Adapter
8. S3-Compatible Storage Contract
9. Local Job Processing
10. Processing State Model
11. Synchronization and Calibration
12. Output Validation
13. Streaming / Cloudflare
14. Security
15. Jetson Migration
16. Optional Container Deployment
17. Future Scale-Out to AWS
18. Cost Considerations
19. Open Questions
20. Smallest Useful Proof of Concept

Preserve useful evidence/source references already present in the document.

## Search for Stale Assumptions

Before finishing, explicitly review the entire document for stale statements implying:

* Docker is required
* Linux is the initial processing OS
* WSL2 is the initial processing runtime
* Vulkan inside WSL is an initial blocker
* the stitch worker must be a Linux container
* the PC worker uses Vulkan instead of DX12
* Docker is necessary for Jetson portability
* a multi-architecture container is required
* MinIO-specific APIs should be used
* AWS is the initial processing environment
* AWS S3 is required instead of S3-compatible storage
* Windows native processing is merely a debug/reference path
* CUDA can replace wgpu's graphics backend
* GStreamer/Cedar must be moved from ScoutCam into the processing worker
* the Orange Pi recording stack needs to be containerized
* the TypeScript worker requires identical native dependencies on every platform
* platform-specific Reco/FFmpeg binaries are an architectural problem rather than an expected platform-adapter concern
* host filesystem paths belong in StitchJob requests
* Windows and Linux path separators should be manually constructed
* shell command strings should be used to invoke Reco or FFmpeg
* Reco exit code 0 alone constitutes successful processing
* removing Docker changes the existing validation, retry, fencing, or immutable-artifact requirements
* Jetson must reproduce the Windows multimedia stack exactly
* AWS Batch, EC2, CloudFront, or MediaConvert are initial dependencies
* YOLO belongs inside the stitching worker
* Cloudflare Tunnel is part of the stitching engine
* MinIO URLs are durable application identities
* the local PC implementation is disposable development infrastructure

Correct any such statements so the document consistently reflects the final architecture.

## Final Architecture Summary

The finished document should make the following architecture unambiguous:

```text
                         SCOUTCAM
                            │
                     paired originals
                            │
                            ▼
                         Mobile
                            │
                            ▼
                    ScoutCam Backend
                      │           │
                      │           └── durable jobs/state
                      │
                      ▼
                   AWS S3 SDK
                      │
                      ▼
                    MinIO
                      │
                      ▼
              TypeScript Stitch Worker
              ┌───────────────────────┐
              │ claim job             │
              │ download objects      │
              │ verify inputs         │
              │ create scratch        │
              │ invoke Reco           │
              │ validate output       │
              │ fast-start MP4        │
              │ upload result         │
              │ publish completion    │
              └───────────┬───────────┘
                          │
                          ▼
                   Platform Adapter
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
        CURRENT PC                 FUTURE JETSON
        Windows 11                Linux ARM64
        reco.exe                  reco
        FFmpeg Windows            FFmpeg/Jetson media
        DX12                      qualified GPU backend
        RTX 5060 Ti               Jetson GPU
             │                         │
             └────────────┬────────────┘
                          │
                    same contracts
                          │
              ┌───────────┴───────────┐
              │                       │
         MinIO today             AWS S3 later
```

And future scale-out remains possible:

```text
                       StitchJob
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
       Windows PC        Jetson        AWS GPU
          native          native       container
```

The important portability guarantees are:

```text
Same TypeScript worker source where practical
Same StitchJob schema
Same result schema
Same AWS S3 SDK storage abstraction
Same object identities
Same validation rules
Same job lifecycle
Same publication semantics
```

The things that are explicitly allowed to differ are:

```text
Operating system
CPU architecture
Reco executable
FFmpeg build
GPU backend
GPU/media libraries
codec acceleration
scratch location
service manager
deployment packaging
```

## Final Principle

State this clearly in the decision/conclusion:

**ScoutCam achieves portability through stable application contracts and a cross-platform TypeScript worker, not by forcing every execution target into an identical container. Native platform capabilities should be used where they provide the simplest and most reliable implementation.**

For the current PC, that means using the already-working native Windows Reco → wgpu → DX12 → RTX 5060 Ti path rather than introducing Linux/WSL/Vulkan solely for deployment uniformity.

For Jetson, that means compiling and qualifying Reco and its media dependencies for the actual Jetson/JetPack environment while preserving the worker's application-level contracts.

For future AWS scale-out, Linux containers remain appropriate because they fit AWS GPU job execution, but that deployment choice must not leak into the application architecture.

## Scope

This task updates documentation only.

Do NOT:

* implement the TypeScript worker
* create package.json or source files
* modify Reco
* rebuild Reco
* modify FFmpeg
* install Node packages
* install Windows services
* modify WSL
* modify Vulkan
* modify NVIDIA drivers
* create Docker images
* start/stop Docker containers
* modify MinIO
* modify Cloudflare
* modify ScoutCam
* modify the Orange Pi
* provision AWS resources

Preserve unrelated working-tree changes.

## Completion Report

When finished, provide a concise summary containing:

1. What Docker-first assumptions were removed.
2. The final native Windows processing architecture.
3. The TypeScript worker portability strategy.
4. How native Reco/FFmpeg execution is abstracted.
5. How filesystem/path portability is maintained.
6. How MinIO → AWS S3 portability is maintained.
7. The Jetson migration strategy.
8. The future AWS scale-out strategy.
9. Which WSL/Vulkan findings remain relevant only as optional/historical information.
10. The exact smallest useful implementation POC that should be built next.

Do not implement that POC as part of this task.
