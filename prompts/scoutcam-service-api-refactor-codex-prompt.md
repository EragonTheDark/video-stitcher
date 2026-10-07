# Refactor ScoutCam Stitch Worker POC Plan — Introduce Dedicated Service API

Update the existing planning document:

`docs/stitch-worker-poc-implementation-plan.md`

This is a **PLANNING/DOCUMENTATION-ONLY** task. Do NOT implement anything or modify ScoutCam services, Reco, MinIO, Cognito, Cloudflare, infrastructure, or the Orange Pi.

Preserve the useful repository investigation and native worker design in the existing plan. The primary change is **domain/API ownership and service authentication**.

## Local Runtime Decision

GraphQL and mobile source are developed on the Mac. GitHub Actions deploys
GraphQL from `dev` to self-hosted DigitalPortal, where DEV binds
`127.0.0.1:4001 -> 4000` in Docker and is served through Cloudflare. V1 API and
services deploy on DigitalPortal. This Windows machine is development host for
service code and native GPU stitch-worker host because Orange Pi is under-resourced.

Plan Service API and GraphQL communication inside DigitalPortal deployment
networking. Plan authenticated Windows stitch-worker -> Service API control-plane
reachability. Both paths carry small control requests only. Do not proxy multi-GB
media through GraphQL, Cloudflare, or Service API.

DigitalPortal Docker hosts MinIO (S3-compatible storage), PostgreSQL, and Redis.
Plan direct Windows stitch-worker -> MinIO transfer through configured S3 endpoint
and least-privilege credentials; never proxy media through application APIs.
Qualify private Windows -> DigitalPortal storage reachability before implementation.

Moving GraphQL to AWS is future work. AWS is used for Cognito authentication and
authorization; no AWS S3, ECS, Fargate, ECR, Batch, Lambda, queues, or new cloud
deployment belongs in this plan. Current GraphQL deployment obtains its environment
from AWS SSM Parameter Store; preserve that existing secret-delivery mechanism
unless user separately changes it.

V2 may move native processing to Jetson after separate ARM64/JetPack/GPU/media
qualification. Preserve service contracts so that move changes platform adapters
and deployment, not domain/job/auth semantics.

## 1. New Architectural Decision

ScoutCam will have two distinct API planes.

### Client API / BFF — `PyrotechSolutions/ScoutCam-api`

The existing GraphQL API remains client-facing.

Responsibilities:
- Mobile/Web client API
- Cognito USER authentication
- user-facing authorization
- client-friendly GraphQL queries/mutations/subscriptions
- translate authorized client operations into Service API calls
- present service-domain state to clients

It should NOT own recording orchestration, object storage, calibration management, stitch jobs/attempts, worker leases/fencing, media processing, or service authentication.

### Service API — proposed `PyrotechSolutions/ScoutCam-service-api`

Use Node.js/TypeScript + Fastify unless repository evidence requires an implementation adjustment.

This becomes the central ScoutCam service/control plane:

```text
Mobile/Web
   │ Cognito user JWT
   ▼
ScoutCam GraphQL API
   │ Cognito M2M token
   ▼
ScoutCam Service API (Fastify)
   ├─ recordings/source revisions
   ├─ calibrations
   ├─ media assets
   ├─ object storage
   ├─ processing jobs/attempts
   ├─ leases/fencing
   ├─ processing state
   └─ orchestration
        ├─ PostgreSQL
        └─ MinIO/S3
```

The stitch worker is another Service API consumer:

```text
ScoutCam Stitch Worker
   │ Cognito M2M token
   ▼
Service API
   ├─ claim
   ├─ heartbeat
   ├─ progress
   ├─ complete
   └─ fail
```

## 2. Cognito: Separate Human and M2M Roles

Do NOT build a custom OAuth server, token issuer, client-secret database, or home-grown JWT system in Fastify.

Human flow remains:

```text
Mobile/Web → Cognito user auth → GraphQL
```

Machine flow uses Cognito M2M/client credentials:

```text
service client_id + client_secret
        ↓
      Cognito
        ↓
short-lived access token
        ↓
Fastify Service API
```

Potential service clients:
- `scoutcam-graphql`
- `scoutcam-stitch-worker`
- future Jetson
- future YOLO worker
- future internal services

Clients cache/reuse access tokens until near expiration. Do not send client secrets on every Service API request.

## 3. Authentication Is Not Authorization

Explicitly separate:

- Authentication: Which service are you?
- Authorization: What may that service do?

Plan Cognito resource-server/custom scopes with least privilege. Example concepts only:

```text
recordings.read
recordings.write
processing.request
processing.read
stitch.claim
stitch.heartbeat
stitch.complete
stitch.fail
media.read
media.write
```

Do not finalize names until endpoint/domain design is reconciled.

`scoutcam-graphql` should have client-orchestration permissions but not worker claim/heartbeat privileges.

`scoutcam-stitch-worker` should have processing-worker permissions but not arbitrary user/profile/admin capabilities.

## 4. GraphQL → Service API Trust Boundary

The Service API must not trust arbitrary user IDs merely because the caller has a valid M2M token.

GraphQL authenticates the human and performs user-facing authorization. GraphQL then calls the Service API using its own M2M identity plus trusted domain context required for the operation.

Plan a contract that authorizes:
1. the calling service capability, and
2. the requested domain resource/context.

Document how user/account context is propagated safely. Do not permit arbitrary external callers to choose owner IDs.

## 5. Service API Owns the Core Domain

Move authoritative ownership previously assigned to GraphQL into the Service API:

- Recording
- SourceRevision
- CalibrationAsset
- MediaAsset
- StitchJob
- StitchAttempt
- ProcessingState
- AcceptedAsset

Exact normalization/names remain planning decisions.

Rule:

```text
GraphQL = client-facing representation/BFF
Service API = authoritative domain/orchestration
Worker = native media execution
```

Do not duplicate business logic.

## 6. Service API Owns Storage Abstraction

Move AWS S3 SDK storage ownership to the Service API.

Rule: **Speak S3, not MinIO.**

```text
Service API → AWS S3 SDK → DigitalPortal-hosted MinIO
```

Service API owns object identities, upload coordination, source verification, calibration registration, media metadata, object-key policy, and accepted-result metadata.

No MinIO-specific SDK.

## 7. Re-evaluate Worker Storage Access

Compare:
A. worker directly accesses MinIO/S3 with scoped credentials
B. worker obtains temporary/presigned access from Service API

Evaluate local simplicity, least privilege, future IAM/workload identity, credential rotation, multi-GB transfers, Jetson, AWS Batch, and avoiding Fastify as a media proxy.

Service API remains control plane even if worker transfers bytes directly with object storage.

Do NOT route full game video through Fastify merely for architectural purity.

## 8. Database Ownership

Move recording/calibration/media/job/attempt authority out of GraphQL.

Evaluate whether Service API should:
- share existing PostgreSQL with its own tables/schema,
- reuse Sequelize conventions,
- use a separate DB/schema,
- or another arrangement justified by infrastructure.

There must be one authoritative owner. GraphQL accesses Service API-owned domain through the Service API, not direct table manipulation.

## 9. GraphQL Becomes an Internal Service Client

Plan a small internal Service API client abstraction in `ScoutCam-api`:

```text
GraphQL resolver
  → ServiceApiClient
  → obtain/cache Cognito M2M token
  → Fastify Service API
  → domain operation
```

Do not scatter raw HTTP/token logic across resolvers.

## 10. Stitch Worker Becomes Another Service Client

Keep `PyrotechSolutions/ScoutCam-stitch-worker` as a separate repo.

Native processing remains:

```text
Windows 11 → Node/TypeScript → reco.exe → wgpu/WGSL → DX12 → RTX 5060 Ti
```

Control plane becomes:

```text
Worker → Cognito M2M → Fastify Service API → claim job
```

Worker heartbeat/progress/finalize go to Service API. Media transfer goes to MinIO/S3. Reco stays local.

Preserve concurrency one, durable claims, attempts, leases, fencing, retries, immutable outputs, strict validation, and conditional publication.

## 11. Keep Reco Boundary Unchanged

Reco knows nothing about Cognito, Fastify, GraphQL, S3, MinIO, job tables, or service auth.

Worker still performs:

```text
claim → download → scratch → Reco → validate → fast-start → hash → upload → finalize
```

## 12. Fastify Service API Surface

Propose the smallest useful REST API. Conceptual examples:

```text
POST /v1/recordings
POST /v1/recordings/:id/uploads/...
POST /v1/recordings/:id/finalize
GET  /v1/recordings/:id
POST /v1/processing/stitch
GET  /v1/processing/:id

POST /v1/workers/stitch/claim
POST /v1/workers/stitch/:attemptId/heartbeat
POST /v1/workers/stitch/:attemptId/progress
POST /v1/workers/stitch/:attemptId/complete
POST /v1/workers/stitch/:attemptId/fail
```

These are examples, not mandated routes. Design actual REST resources during planning. Protect worker routes with appropriate scopes.

## 13. GraphQL Responsibilities

Document what remains in GraphQL: user-facing recording queries, processing status, video assets, upload workflow, stitch/retry/archive mutations, and subscriptions as appropriate.

GraphQL translates authorized user operations to Service API calls. It does not duplicate storage repositories or job orchestration.

## 14. Mobile Responsibilities

Preserve current Pi flow:

```text
Pi → Mobile → resumable download → SHA-256 verify → private completed bundle
```

Future cloud upload/status remains through client-facing GraphQL.

Never give mobile worker M2M credentials or Service API service credentials. Mobile authenticates as a human user.

## 15. Service Credentials / Secret Handling

Plan credential delivery for GraphQL, stitch worker, and future Jetson.

Requirements:
- never commit client secrets
- do not recreate Cognito secrets in app DB
- environment/secrets delivery appropriate to deployment
- short-lived tokens
- cache until near expiration
- reacquire as needed
- never log tokens/secrets
- support secret rotation

## 16. Cognito Infrastructure Ownership

Inspect `ScoutCam-infrastructure` and plan only Cognito changes needed for:
- Service API Cognito resource server
- custom scopes
- GraphQL M2M app client
- stitch-worker M2M app client
- future service clients
- client-credentials configuration
- secret delivery/reference
- Service API token validation settings

Reuse existing Cognito patterns. Do not introduce a second IdP.

## 17. Fastify Auth Middleware

Plan maintained JWT/JWKS/OIDC validation; do not hand-roll crypto.

Validate appropriate signature, issuer, expiration, token use, client/audience relationship as applicable, and scopes.

Model:

```text
request
 → validate Cognito access token
 → ServicePrincipal { client identity, scopes }
 → route authorization
```

Keep principal construction separate from route/domain logic.

## 18. Domain Authorization

Scopes are necessary but not always sufficient.

Plan:

```text
service capability + domain resource/context
```

GraphQL can act on behalf of authorized users within its service capabilities. Worker can only claim/update its processing attempts. Worker cannot arbitrarily modify recording ownership or publish stale attempts.

Fencing-token enforcement remains independent of authentication.

## 19. Repository Ownership Map

Refactor toward:

```text
ScoutCam-api
  GraphQL/BFF
  Cognito USER auth
  Service API client
  client subscriptions/presentation

ScoutCam-service-api
  Fastify
  Cognito M2M validation
  core domain
  recordings/calibration/media
  S3 abstraction
  processing jobs/attempts
  leases/fencing/orchestration

ScoutCam-stitch-worker
  native Windows TypeScript worker
  Cognito M2M client
  S3 transport
  RecoRunner
  validation/result publication

ScoutCam-mobile
  user client
  Pi recording download/verification
  GraphQL upload/status UX

ScoutCam-ui
  user client
  GraphQL

ScoutCam-infrastructure
  Cognito users + M2M resources
  deployment/storage infrastructure

video-stitcher
  Reco engine only
```

Reconcile with actual repository evidence.

## 20. Refactor File-Level Change Plan

### ScoutCam-service-api
Propose Fastify structure, config, Cognito M2M auth, scopes/authorization, domain modules, S3 adapter, DB repositories, worker routes, service-level recording/media routes, and tests. Do not create repo.

### ScoutCam-api
Reduce changes to Service API client, M2M token acquisition/cache, client-facing GraphQL schema/resolvers, user→service translation, subscriptions/status, and tests. No Service API DB repositories.

### ScoutCam-stitch-worker
Keep native worker plan; use Cognito M2M for Service API auth.

### ScoutCam-infrastructure
Plan Cognito resource server/scopes/app clients/secrets and local Service API
authentication wiring. Do not plan AWS service deployment, storage, database, or
compute resources.

### ScoutCam-mobile
Preserve Pi behavior; future cloud changes remain GraphQL-facing.

## 21. Refactor Implementation Phases

### Phase 1 — Service API contracts and identity design
- Define Service API ownership boundary and REST contract.
- Define Cognito resource server/scopes.
- Define GraphQL/worker service identities.
- Define trusted user/account context propagation.
- Define ServicePrincipal/authorization model.
- Define recording/source/calibration/media/job/attempt contracts.
- Documentation/contracts only.

### Phase 2 — Service API foundation
- Create Fastify service repo/app.
- Add configuration, health/readiness.
- Add Cognito M2M validation and scope middleware.
- Add initial DB/storage wiring following approved contracts.
- No Reco.

### Phase 3 — Domain/storage
- Implement recording/source/calibration/media/job/attempt persistence.
- Implement AWS S3 SDK adapter against MinIO.
- Implement immutable object-key and verification policy.
- Implement claim/lease/fencing transactions.

### Phase 4 — GraphQL integration
- Add ServiceApiClient and M2M token cache.
- Move client-facing operations to Service API.
- Preserve Cognito USER boundary.
- Add client status/subscription translation as appropriate.

### Phase 5 — Native worker
- Create stitch-worker repo.
- Add Cognito M2M client/token cache.
- Implement claim/heartbeat/progress/finalize.
- Implement native Reco/FFmpeg adapters and scratch handling.

### Phase 6 — Validation/end-to-end POC
- Seed verified pair/calibration.
- Claim → S3 download → Reco → strict validation → fast-start → upload → fenced finalize.
- Record metrics/provenance.
- Keep infrastructure and panorama-quality verdicts separate.

### Phase 7 — Failure/recovery
- worker kill/restart
- stale lease
- duplicate claim/finalize
- corrupt/truncated media
- upload failure
- stale attempt completion
- token expiry/refresh failure
- Service API restart
- safe disk exhaustion test

### Phase 8 — Panorama quality
- separately qualify 1080p30 calibration, seam, field coverage, player crossings, beginning/middle/end synchronization/drift.

## 22. Acceptance Criteria for the Refactored Plan

The revised plan must make these explicit:

- GraphQL is client-facing/BFF, not authoritative processing domain.
- Fastify Service API owns recording/media/calibration/job orchestration.
- Human auth remains Cognito USER auth at GraphQL.
- Service auth uses Cognito M2M/client credentials.
- No custom token issuer/client-secret database.
- Service API endpoints are authenticated and scope-authorized.
- Authentication and authorization are distinct.
- GraphQL and worker use different least-privilege service identities.
- Mobile never receives service credentials.
- Service API is control plane; multi-GB media need not proxy through it.
- S3 abstraction remains AWS SDK + DigitalPortal-hosted MinIO; no AWS S3 plan.
- Worker remains native Windows/DX12/RTX initially.
- Reco remains independent.
- Durable jobs/leases/fencing/immutable outputs/strict validation remain.
- Jetson/AWS remain future execution targets.
- No Docker/WSL/Vulkan/YOLO requirement is reintroduced.

## 23. Update the Existing Plan, Do Not Start Over

Preserve repository findings, Reco findings, mobile download/hash findings, validation design, object-identity concepts, worker module design, failure semantics, Windows runtime decisions, Jetson portability, and panorama-quality separation.

Rewrite only what is affected by the new Service API and Cognito M2M ownership.

Before finishing, search the entire plan for stale assumptions that:
- ScoutCam-api owns recording/job/storage repositories
- worker endpoints are GraphQL operations
- worker auth is generic/undefined
- Fastify issues its own client IDs/secrets/tokens
- worker impersonates a Cognito user
- mobile calls Service API directly with M2M credentials
- GraphQL and worker share the same permissions
- Service API trusts arbitrary owner IDs from service callers
- Redis is required as a queue
- full media must proxy through Fastify
- Service API and GraphQL both own authoritative domain tables

Correct them.

## 24. Completion Response

When finished, summarize:
1. what ownership moved from GraphQL to Service API
2. proposed Fastify Service API responsibilities
3. Cognito USER vs M2M model
4. proposed service clients/scopes
5. GraphQL→Service API trust/context model
6. worker→Service API auth model
7. storage/data-transfer boundary
8. database ownership
9. revised repository/file-level plan
10. revised implementation phases
11. remaining open questions

Do not implement anything.
