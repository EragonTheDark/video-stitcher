# ScoutCam Native Stitch Worker POC — Refactored Plan

Planning only. No service, Reco, Pi, MinIO, Cognito, DB, Cloudflare, host, or
infrastructure change. Updated 2026-09-25.

## Decision

V1 splits BFF, authoritative control plane, native executor.

~~~
Mobile/Web -> Cognito USER -> GraphQL BFF
GraphQL -> Cognito M2M -> Service API -> PostgreSQL + MinIO
Windows Stitch Worker -> Cognito M2M -> Service API
Windows Stitch Worker -> S3 protocol -> MinIO
~~~

Mac develops GraphQL/mobile. GitHub Actions deploys GraphQL dev to self-hosted
DigitalPortal Docker; current bind is 127.0.0.1:4001 to 4000, Cloudflare-facing.
Service API also deploys DigitalPortal. Windows develops service code and runs
native RTX/DX12 stitch worker because Orange Pi is under-resourced. V2 may move
worker to Jetson after separate ARM64/JetPack/GPU/media qualification.

DigitalPortal Docker hosts GraphQL, Service API when built, MinIO, PostgreSQL,
Redis. GraphQL -> Service API stays internal DigitalPortal networking. Windows
worker -> Service API requires authenticated control reachability. Windows worker
-> MinIO requires qualified TLS/network/least-privilege direct transfer. No
multi-GB media proxy through GraphQL, Cloudflare, or Service API.

AWS scope: Cognito USER/M2M. No AWS S3/ECS/ECR/Fargate/Batch/Lambda/queues. Existing
GraphQL workflow reads environment from SSM Parameter Store; preserve unless user
separately changes secret delivery.

## Repositories investigated

| Repository | Commit | Finding |
|---|---:|---|
| ScoutCam-api | 8539402 | Bun Apollo GraphQL BFF; profile-only domain today. |
| ScoutCam-mobile | 779fb82 | Capacitor client; Pi library/download/verification. |
| ScoutCam-ui | 5b47172 | Browser GraphQL client. |
| ScoutCam-infrastructure | c166f94 | Cognito/legacy CloudFormation/deploy scripts. |
| ScoutCam-recorder-service | dc3c827 | Cedar V3 capture/archive/session library. |
| ScoutCam-wifi-control-service | b56f21f | Pinned-TLS Pi control bridge. |
| ScoutCam-workspace | d1fd647 | Shared docs/qualification. |
| video-stitcher | local | Rust GPU engine/CLI. |

EXISTS: recorder SessionManifest v1; mobile Range resume/full SHA-256/private
promotion; Reco local stitch CLI; GraphQL USER auth/SDL/Sequelize profile pattern;
DigitalPortal MinIO/PostgreSQL/Redis.

MISSING: Service API, core media/job domain, AWS SDK MinIO adapter, cloud upload,
Cognito M2M clients/scopes, worker service client, durable claims/fences.

## Ownership

GraphQL BFF owns client SDL, Cognito USER auth, client authorization, client-facing
state, and typed ServiceApiClient. It owns no recording/calibration/storage/media/
job/lease/domain tables.

New ScoutCam-service-api owns DeviceRig, Recording, SourceRevision, CalibrationAsset,
MediaAsset, StitchJob, StitchAttempt, accepted asset, object keys, source verify,
AWS SDK storage adapter, jobs, attempts, lease/fence, domain authorization.
One authoritative owner: GraphQL never manipulates its tables directly.

New ScoutCam-stitch-worker owns native Windows process, scratch, RecoRunner,
FFmpeg/ffprobe validation, direct object transfer, result publication. Concurrency
one. No DB access/client API ownership.

Mobile preserves Pi -> resumable download -> SHA-256 verify -> private pair.
Future cloud upload/status uses GraphQL only. Reco knows no cloud/auth/job detail.

## Cognito auth and authorization

Human path: Mobile/Web -> Cognito access token -> GraphQL.

Machine path: GraphQL M2M client or worker M2M client -> Cognito client credentials
-> short-lived access token -> Service API. No custom OAuth/token issuer/secret DB/
JWT crypto. Cache until near expiry; never log secrets/tokens; rotate deployment
secrets.

Plan resource server/custom least-privilege scopes. Concepts: recordings.read/write,
processing.request/read, stitch.claim/heartbeat/complete/fail, media.read/write.
GraphQL gets orchestration scopes; worker gets only attempt scopes.

M2M authentication does not authorize arbitrary owner IDs. GraphQL validates human
identity, then sends audited trusted actor context under GraphQL service identity.
Service API validates principal, scope, context shape, domain ownership. Worker can
update only claimed current-fenced attempt. Fencing remains independent from JWT.

## Cognito/infrastructure planning

Current Cognito template has Web/Mobile USER clients only. Plan Cognito-only changes:

- resource server and final scopes;
- confidential GraphQL and worker client-credentials clients;
- required User Pool OAuth token endpoint/domain settings;
- secret reference/delivery/rotation and Service API issuer/client/scope config;
- exports/parameters/tests.

Do not modify CloudFormation now. No AWS runtime/storage/compute plan. API GitHub
Actions verifies Bun/tests/codegen/typecheck/build, gets env from SSM, runs its
authorized deployment migration, builds Compose, waits health. Agents never run
migrations, dispatch workflows, or deploy.

## Domain, storage, lifecycle

Domain: DeviceRig, Recording, immutable SourceRevision, hash-addressed
CalibrationAsset, MediaAsset, idempotent StitchJob, StitchAttempt, accepted
pointer. A calibration belongs to DeviceRig: physical paired-camera mounting,
lens/camera identity and capture profile, not one recording. Recording selects a
specific immutable approved calibration revision; job/attempt/result retain that
exact calibration ID/hash for reproducibility.

~~~
recordings/recordingId/source/sourceRevision/left.mp4
recordings/recordingId/source/sourceRevision/right.mp4
recordings/recordingId/source/sourceRevision/session.json
devices/deviceId/rigs/rigId/calibrations/calibrationSha256/match.json
recordings/recordingId/output/jobKey/attemptId/stitched.mp4
recordings/recordingId/output/jobKey/attemptId/result.json
~~~

Generated validated segments. Identity key + SHA-256, never URL/path/ETag. Source
immutable. Service API uses AWS SDK against DigitalPortal MinIO; no MinIO SDK.
Worker reads allowed source/calibration prefixes, writes own attempt prefix; no
source delete/list-all/admin. Full SHA-256, not multipart ETag, is integrity truth.

~~~
UPLOADING -> UPLOADED -> STITCH_QUEUED -> STITCH_RUNNING
          -> STITCH_VALIDATING -> STITCH_COMPLETE | STITCH_FAILED
~~~

Claim transaction makes attempt ID, unpredictable fence, lease. Heartbeat/progress/
complete/fail require current attempt/fence/lease. Complete verifies final object
size/hash then conditionally updates accepted pointer. Stale/crashed/duplicate
attempt cannot publish. Redis not required queue.

## Service surface and worker

Service API REST plans recording upload/register/finalize/status, calibration
registration/selection, processing request/status/retry, worker claim/heartbeat/
progress/complete/fail. Actual routes after domain design.

GraphQL has one typed ServiceApiClient; no resolver-local HTTP/token logic.
Worker separate client for claim/heartbeat/progress/finalize. Decide direct scoped
MinIO credentials versus short-lived Service API grants after Windows/DigitalPortal
network and rotation qualification. Never Fastify media proxy.

Worker flow:

~~~
claim -> download -> scratch -> reco.exe -> validate -> faststart -> hash
-> upload -> fenced finalize
~~~

Use spawn executable+args, shell false. Adapter owns Reco/FFmpeg/ffprobe paths,
env, codec, timeout, process-tree kill, scratch, GPU evidence. Scratch generated,
contained, ACL-restricted, free-space checked, reconciled. Reco CLI current sync
override cannot explicitly replace nonzero calibration offset with zero: FRICTION,
not worker hack.

Input: strict SessionManifest v1, hashes/sizes/sides, ffprobe, compatibility,
selected DeviceRig calibration/sync. Reject calibration when rig, lens/camera,
mount, resolution, crop, or capture profile compatibility differs. Manifest does
not prove lens geometry/common clock/drift.
Output: Reco exit zero insufficient; probe/full decode/count/duration/end checks;
remux final faststart, revalidate, SHA-256 final bytes, upload/Head check.
result.json records final identity, provenance, validation, timings, reliable metrics.

## File-level plan

ScoutCam-service-api new repository: Bun/Fastify, config, health/readiness, Cognito
M2M middleware, ServicePrincipal, actor authorization, DeviceRig/domain modules, Sequelize
repositories/migrations, AWS SDK adapter, worker routes/tests, DigitalPortal Compose
deployment plan. Do not create yet.

ScoutCam-api: add M2M token cache/ServiceApiClient, GraphQL translation/status/tests.
No core-domain repository/storage code.

ScoutCam-stitch-worker new repository: Windows M2M client, Service API client,
storage, scratch, RecoRunner, validation, result/metrics/tests. No DB.

ScoutCam-infrastructure: Cognito resources/scopes/M2M clients/secrets/tests only.
ScoutCam-mobile/UI: GraphQL-only upload/status later; no M2M/direct Service API.
video-stitcher: no change; document FRICTION, no workaround.

## Phases

1. Contracts: ownership, REST, scope matrix, actor context, DigitalPortal/Windows
   reachability. Documentation only.
2. Service foundation: Fastify/Bun, config, health, M2M validation, tests.
3. Domain/storage: persistence, MinIO AWS SDK, object verification, claim/fence.
4. GraphQL: token cache, ServiceApiClient, user translation/status.
5. Worker: Windows M2M, native adapters, scratch/direct transfer.
6. E2E: verified pair -> claim -> stitch -> strict final -> publish -> metrics.
7. Failure: restart, token refresh, stale fence, duplicate finalize, corrupt media,
   upload failure, disk test.
8. Quality: separate 1080p30 calibration/seam/coverage/sync/drift.

## Acceptance, deferred, questions

Acceptance: BFF not domain owner; Service API sole domain owner; USER vs M2M
separation; scopes plus resource/fence authorization; mobile no service secrets;
DigitalPortal control plane; Windows native RTX/DX12 worker; direct MinIO transfer;
immutable source/attempt; strict final hash; no AWS runtime, Docker worker,
WSL/Vulkan, YOLO requirement.

Deferred: Jetson, GraphQL AWS move, AWS storage/compute, multi-worker scale, Redis
queue, HLS/MediaConvert/CloudFront, live stitch, Reco renderer/drift redesign, Pi
changes, automatic deletion.

Open questions: Windows -> DigitalPortal Service API/MinIO TLS/network; credential
versus presign; tenant/game/actor audit model; DB schema/backups/migrations; Service
API Compose port/health; Cognito token claims/rotation; mobile upload UX; output/
calibration quality policy; full-game throughput/VRAM/drift evidence.

Next: Phase 1 design review. No repository/app/migration/worker code before API
routes/domain, scopes, actor context, DigitalPortal topology, Windows reachability
are agreed.
