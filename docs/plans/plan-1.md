# 🎬 VidroTube — Phases & Tasks Master Plan
### `vidrotube` · Laravel 13 + PHP 8.3 + PostgreSQL + S3 + Go/Rust Workers + Future CDN Scale
### ⚡ Governed by System Design + Delivery Discipline

---

> [!IMPORTANT]
> **Ordering is law here.**
> - *Design before code* — product scope, SLOs, domain model, contracts, and ownership must be approved before any production feature is implemented.
> - *Gates are non-negotiable* — a later phase must not start until its prerequisite exit gate is complete.
> - *Boundaries over language* — Laravel owns catalog and identity; workers never read Laravel tables; all cross-service communication uses versioned contracts.
> - *MVP first* — the first public launch is the smallest useful VOD product. Live streaming, monetization, DRM, and full copyright systems are later products.
> - SOLID / layer discipline is a diagnostic tool, not permission to add abstraction for its own sake.

---

## 📦 Current Stack Audit

| Layer | Technology | Status |
|-------|-----------|--------|
| **Backend** | Laravel 13, PHP 8.3, Fortify, Livewire, Flux, Vite | ✅ Starter running |
| **Auth** | Laravel Fortify | ✅ Present |
| **DB** | (local default) — target PostgreSQL | ⚠️ Not production-ready |
| **Models** | Incomplete `Channel` model + migration | ⚠️ Partial |
| **Video Catalog** | None | ❌ Missing |
| **Upload Pipeline** | None | ❌ Missing |
| **Object Storage** | None | ❌ Missing |
| **CDN / Playback** | None | ❌ Missing |
| **Message Brokers** | None (Kafka / RabbitMQ) | ❌ Missing |
| **Workers** | None (Go / Rust) | ❌ Missing |
| **Deployment / IaC** | None | ❌ Missing |
| **Observability** | Basic Laravel only | ⚠️ Insufficient |

---

## 🔴 Smell Inventory (before any implementation phase starts)

| # | Smell | Principle | Location | Fix Phase |
|---|-------|-----------|----------|-----------|
| 1 | Incomplete Channel model — ownership and public identity undefined | Domain integrity | `Channel` model / migration | **Phase 0–3** |
| 2 | No video catalog — binary files and business records not separated | Data ownership | — | **Phase 3** |
| 3 | No upload session / direct-to-storage path | Scale & timeouts | — | **Phase 4** |
| 4 | No video lifecycle state machine | Invalid states can be constructed | — | **Phase 1 & 3** |
| 5 | No service / action layer — future controllers will become god methods | Thin controllers | Controllers (future) | **Phase 3** |
| 6 | No versioned API contract seam | Explicit boundaries | Routes | **Phase 1 & 3** |
| 7 | No outbox / inbox pattern planned | At-least-once delivery safety | Events | **Phase 1 & 5** |
| 8 | No multi-language contracts (JSON Schema / Protobuf) | Cross-service safety | — | **Phase 1** |
| 9 | No object storage quarantine / signed access design | Security | Storage | **Phase 2 & 4** |
| 10 | No observability across API → queue → worker → playback | Debuggability | — | **Phase 2** |
| 11 | Premature technology selection risk (Kafka/Go/Rust before requirements) | Design-before-code | System Design | **Phase 0–1** |
| 12 | No RPO/RTO, capacity limits, or SLOs defined | Non-functional requirements | — | **Phase 0** |

---

## 🗺️ Target Architecture

### System Context (who talks to whom)

```
Browser / Mobile Client
        │
        ▼
┌───────────────────────┐
│  Laravel (API + Auth) │  ← Identity, Channels, Video catalog, Upload sessions,
│  + PostgreSQL         │    Authorization, Moderation, Admin APIs
└───────────┬───────────┘
            │ versioned events / commands
            ▼
┌───────────────────────┐     ┌──────────────────────┐
│  Kafka (durable       │     │  RabbitMQ (commands, │
│  domain events +      │     │  processing jobs,    │
│  analytics streams)   │     │  retries, DLQs)      │
└───────────┬───────────┘     └──────────┬───────────┘
            │                            │
            ▼                            ▼
┌───────────────────────┐     ┌──────────────────────┐
│  Go Workers           │     │  Optional Rust       │
│  (orchestration,      │     │  (specialized media  │
│   consumers,          │     │   validation / CPU)  │
│   projections)        │     └──────────────────────┘
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐     ┌──────────────────────┐
│  S3-compatible        │────▶│  CDN (signed access) │
│  Object Storage       │     │  HLS / DASH playback │
│  (originals,          │     └──────────────────────┘
│   quarantine,         │
│   renditions,         │
│   thumbnails,         │
│   captions)           │
└───────────────────────┘
```

### Technology Responsibilities

| Component | Owns | Does **not** own |
|-----------|------|------------------|
| **Laravel** | Identity, channels, video metadata, authorization, upload sessions, admin APIs, moderation decisions | Video bytes, transcoding, analytics processing |
| **PostgreSQL** | Transactional metadata & state transitions | Media files, event replay, cache |
| **Redis** | Cache, locks, rate limits, ephemeral state | Source-of-truth business records |
| **S3-compatible storage** | Originals, quarantine, renditions, manifests, captions, thumbnails | Authorization decisions |
| **Kafka** | Durable domain events, analytics streams, replay, projections | Short-lived work retries |
| **RabbitMQ** | Commands, processing jobs, priorities, retries, DLQs | Long-term event history |
| **Go** | Distributed workers, orchestration, consumers, projections, internal APIs | Laravel business data ownership |
| **Rust** | Specialized media validation or CPU-sensitive work (only if justified) | General product APIs |
| **CDN** | Global playback delivery | Original media processing |

> Cross-language communication uses **versioned JSON Schema or Protobuf contracts**.  
> No Go/Rust service reads Laravel tables. Laravel serialized jobs are never used as service contracts.

### Core Domain Model

```text
User
  └── owns → Channel
                └── contains → Video
                                  ├── has → UploadSession
                                  ├── has → MediaAsset
                                  ├── has → Thumbnail
                                  ├── has → Caption
                                  ├── has → Rendition
                                  ├── has → ProcessingAttempt
                                  ├── has → ModerationCase
                                  └── produces → WatchEvent / Analytics
```

### Video Lifecycle (must be enforced)

```text
created → uploading → uploaded → quarantined → processing → ready
                                                          ↘ blocked / deleted
```

Every transition has a trigger, an owner, a failure path, and a test.

### Dependency Direction (must hold)

```
Controller / Action  →  Application Service  →  Repository / Client
Worker               →  Contract (API or event)  →  Laravel catalog
UI                   →  API client               →  transport
```

Infrastructure never leaks upward. No SQL type in a service signature. No raw broker type above the worker boundary.

---

## ✅ DESIGN PHASES (must complete before heavy implementation)

### Design Phase A — Product Requirements
- [ ] Approve functional requirements (creator / viewer / admin actions)
- [ ] Explicitly list out-of-scope items (live, monetization, DRM, full copyright, advanced recommendations)
- [ ] Produce approved MVP product scope document

### Design Phase B — Quality Requirements
- [ ] Define capacity limits (max file size, duration, uploads/day, concurrent viewers, regions, retention)
- [ ] Define SLOs (API latency, upload reliability, processing time, playback startup)
- [ ] Define RPO / RTO, privacy, retention, deletion, and cost targets

### Design Phase C — Domain & Data Design
- [ ] Finalize entities, fields, relationships, ownership matrix
- [ ] Define indexes, soft-delete, retention, and deletion rules

### Design Phase D — User-Flow Design
- [ ] Sequence diagrams: channel creation, upload, processing, moderation, playback, deletion
- [ ] Failure paths for each flow

### Design Phase E — Architecture Design
- [ ] System context + container diagrams
- [ ] Assign responsibilities to Laravel, PostgreSQL, Redis, S3, CDN, Go, Rust, Kafka, RabbitMQ

### Design Phase F — Contract Design
- [ ] API request/response contracts + versioning
- [ ] Event envelope + Kafka topics catalog
- [ ] RabbitMQ queue catalog + command schemas
- [ ] Idempotency, retries, DLQ behavior

### Design Phase G — QA & Operations Design
- [ ] Unit / integration / E2E / load / security / failure test strategy
- [ ] Dashboards, alerts, runbooks, backup/restore procedures

### Design Phase H — Approval Gate
- [ ] Team review and sign-off of all design outputs
- [ ] Prioritized implementation backlog locked

> **Exit gate:** No architecture or production feature work proceeds until Phases A–H are approved.

---

## ✅ IMPLEMENTATION PHASE 0 — Requirements and Decisions
> **Goal**: Agree on what is being built, its scale, and operating constraints before designing services or writing production code.  
> **Duration estimate**: 1–2 weeks

### Tasks

| # | Task | Done when |
|---|------|-----------|
| 0.1 | Define first-release (MVP) product | A new developer can read one list and know exactly what creators and viewers can do at launch |
| 0.2 | Record what is out of scope | Each excluded feature has a later placeholder and is removed from V1 estimates |
| 0.3 | Define capacity limits | Numbers exist for file size, duration, uploads/day, viewers, users, regions, retention |
| 0.4 | Define service-level objectives | Every target has a number, measurement method, and owner |
| 0.5 | Define recovery & compliance | RPO, RTO, regions, retention, deletion rules, and budget are approved |
| 0.6 | Choose initial cloud platform | Provider selected; each required capability mapped to a service |
| 0.7 | Define identity & API rules | Public IDs, API versions, login method, roles, permissions, service auth documented |

### Outputs
- Approved product scope
- Capacity and SLO document
- Risk and assumption register
- Initial cost model
- Product glossary

### Exit Gate
No architecture work proceeds until product scope, scale targets, SLOs, RPO/RTO, regions, and budget are approved.

---

## ✅ IMPLEMENTATION PHASE 1 — Architecture and Contracts
> **Goal**: Define boundaries and contracts so Laravel, Go, Rust, Kafka, and RabbitMQ have non-overlapping responsibilities.  
> **Duration estimate**: 2–3 weeks

### Tasks

| # | Task | Done when |
|---|------|-----------|
| 1.1 | Define domain model | Team can draw objects, relationships, required fields, ownership rules |
| 1.2 | Define data ownership | Every important field has one owner; others access via API or event |
| 1.3 | Define video lifecycle | Each transition has trigger, owner, failure path, and test |
| 1.4 | Design upload flow | Flow describes every request, response, stored record, and failure case |
| 1.5 | Design processing flow | Every step has input, output, retry rule, failure state, and owner |
| 1.6 | Define API contracts | Examples, validation, errors, auth, and versioning documented |
| 1.7 | Define event envelope | One valid example exists for each required field; invalid versions handled |
| 1.8 | Define delivery guarantees | Duplicate, timeout, crash, and replay scenarios have safe outcomes |
| 1.9 | Define Kafka topics | Each topic has purpose, producer, consumers, key, retention, schema, privacy |
| 1.10 | Define RabbitMQ queues | Every queue has command schema, consumer, concurrency, retry, DLQ |
| 1.11 | Define failure behavior | Duplicate, delayed, out-of-order, invalid, partial-processing examples covered |
| 1.12 | Choose first Go / Rust responsibilities | Each selected service has purpose, owner, contract, deployment target, success metric |

### Outputs
- System context diagram
- Container / service boundary diagram
- Domain model
- Video lifecycle state machine
- API contracts
- Event and message contracts
- Kafka topic catalog
- RabbitMQ queue catalog
- Data ownership matrix
- Architecture Decision Records (ADRs)

### Exit Gate
No production implementation proceeds until boundaries, contracts, state transitions, ownership, retry behavior, and security assumptions are reviewed and approved.

---

## ✅ IMPLEMENTATION PHASE 2 — Production Foundation
> **Goal**: Create the environments and platform capabilities required by every later phase.  
> **Duration estimate**: 2–3 weeks  
> **Prerequisites**: Phase 0 and Phase 1 exit gates approved

### Tasks

| # | Task | Done when |
|---|------|-----------|
| 2.1 | Set up PostgreSQL | Local + staging can connect and run migrations |
| 2.2 | Set up Redis | Configured, monitored, safe to lose without losing business data |
| 2.3 | Set up object storage | Test file stored privately, retrieved by authorized service, expired by policy |
| 2.4 | Prepare CDN security | Storage private; only approved access path can reach playback files |
| 2.5 | Define environment configuration | New developer and CI can configure each environment from documented variables |
| 2.6 | Manage secrets and credentials | Secrets injected securely, rotated, absent from repos and logs |
| 2.7 | Add observability basics | One upload can be traced across API → storage → queue → worker → playback |
| 2.8 | Create infrastructure as code | Staging can be created and updated from source-controlled definitions |
| 2.9 | Strengthen CI | CI blocks deliberately broken test, migration, contract, or vulnerable dependency |
| 2.10 | Define recovery procedures | Test environment has restored data and completed a documented rollback |

### Parallel work allowed
- Infrastructure code and CI
- Observability with environment setup
- No product feature may depend on an unapproved production service

### Exit Gate
Staging deploys repeatably, secrets are managed, PostgreSQL/Redis/object storage work, logs and alerts are visible, backups restore successfully, and CI blocks broken migrations or contracts.

---

## ✅ IMPLEMENTATION PHASE 3 — Identity, Channels, and Video Catalog
> **Goal**: Create the transactional business model before adding asynchronous media processing.  
> **Duration estimate**: 2–3 weeks  
> **Prerequisites**: Phase 2 exit gate complete

### Tasks

| # | Task | Result |
|---|------|--------|
| 3.1 | Connect channels to users | Ownership and permissions are enforceable |
| 3.2 | Add public channel identity | Channels safely exposed through URLs and APIs |
| 3.3 | Create video metadata models | Catalog data is separate from media data |
| 3.4 | Add video visibility states | Every access decision has a defined status |
| 3.5 | Protect catalog integrity | Invalid ownership and duplicate operations prevented |
| 3.6 | Add application services | Controllers and future workers use the same business rules |
| 3.7 | Add versioned APIs | Clients have a stable contract |
| 3.8 | Add catalog tests and factories | Catalog is safe before uploads are added |

### Files initially affected
- `app/Models/Channel.php`
- `app/Models/User.php`
- `app/Models/Video.php` (new)
- `app/Models/MediaAsset.php` (new)
- `database/migrations/`
- `database/factories/`
- `app/Http/Controllers/`
- `app/Policies/`
- `app/Services/` or `app/Actions/`
- `routes/`
- `tests/Feature/`

### Exit Gate
A user can create and manage an authorized channel and video metadata through tested APIs. **No binary upload is part of this phase.**

---

## ✅ IMPLEMENTATION PHASE 4 — Upload and Asset Lifecycle
> **Goal**: Allow users to upload large files directly to object storage without sending video bytes through Laravel.  
> **Duration estimate**: 2–3 weeks  
> **Prerequisites**: Phase 3 exit gate + object storage from Phase 2

### Tasks

| # | Task | Result |
|---|------|--------|
| 4.1 | Create upload sessions | Each upload has a controlled server-side record (size, MIME, checksum, expiry, owner, idempotency key) |
| 4.2 | Issue presigned upload URLs | Laravel does not carry large video files |
| 4.3 | Upload to quarantine storage | Untrusted files cannot immediately become playable |
| 4.4 | Verify upload completion | Platform does not trust the browser alone (existence, size, checksum, ownership) |
| 4.5 | Record atomic upload states | Database reflects real upload lifecycle (uploading → uploaded → quarantined) |
| 4.6 | Clean up abandoned uploads | Unused storage does not grow indefinitely |
| 4.7 | Add validation integration points | Hooks ready for virus scanning, file inspection, content validation |
| 4.8 | Test upload failure cases | Interruption, resume, duplicate completion, expired URLs, bad checksums, unauthorized access covered |

### Exit Gate
A real test client can upload an allowed file, resume an interrupted upload, safely retry completion, and receive a durable quarantined asset **without Laravel handling the file bytes**.

---

## ✅ IMPLEMENTATION PHASE 5 — Processing, Events, and Playback
> **Goal**: Complete one reliable upload-to-playback path before adding social features or global scale.  
> **Duration estimate**: 4–6 weeks  
> **Prerequisites**: Phase 4 exit gate + Phase 1 event/message contracts approved

### 5.1 Transactional Outbox
- [ ] Write outbox record in the same DB transaction as upload completion
- [ ] Build event relay with retry behavior and metrics
- [ ] Protect every consumer with inbox / idempotency handling

### 5.2 First Go Worker
- [ ] Create small RabbitMQ consumer with health checks and metrics
- [ ] Consume versioned `media.probe` command
- [ ] Validate object + extract media metadata
- [ ] Update processing status via approved Laravel API / command contract
- [ ] Publish `MediaProbeCompleted` or `MediaProbeFailed`
- [ ] Handle timeouts, retries, idempotency, DLQ

### 5.3 First Rust Workload (optional, evidence-based)
- [ ] Choose one specialized task only if approved (secure container validation or thumbnail generation)
- [ ] Define versioned contract
- [ ] Benchmark against Go / managed alternative
- [ ] Keep only if measured benefit justifies cost

### 5.4 Transcoding and Derivatives
- [ ] Run long transcoding outside Lambda (MediaConvert or dedicated containers)
- [ ] Produce approved HLS/DASH quality ladder
- [ ] Store immutable manifests, renditions, thumbnails, captions
- [ ] Track ProcessingAttempt records so retries resume from a known step

### 5.5 Kafka Streams
- [ ] Publish lifecycle events after transactional commits
- [ ] Build read-model consumers for catalog / feed projections
- [ ] Create separate watch-event analytics stream
- [ ] Monitor consumer lag and support replay

### 5.6 Playback
- [ ] Authorize playback (visibility, moderation, ownership, expiry, deletion, takedown)
- [ ] Issue signed CDN access with limited lifetime
- [ ] Deliver HLS/DASH through CDN
- [ ] Test revocation, private/blocked videos, missing renditions, CDN/origin failures

### Exit Gate
The complete flow works repeatedly:

```text
create channel → create video → create upload session → upload → complete
→ quarantine → probe → process → publish manifest → authorize playback
```

Must pass with: duplicate requests, duplicate messages, worker retries, worker crashes, and failed-processing recovery.

---

## ✅ IMPLEMENTATION PHASE 6 — Reliability, QA, and Production Launch
> **Goal**: Prove the VOD core is secure, observable, operable, and ready for real users.  
> **Duration estimate**: 2–4 weeks  
> **Prerequisites**: Phase 5 end-to-end flow complete

### QA Tracks
- [ ] Business rules (domain, state transitions, policies, idempotency, retries, contracts)
- [ ] Infrastructure integration (PostgreSQL, Redis, S3, Kafka, RabbitMQ, Go, Rust, transcoding)
- [ ] Cross-language contracts (Laravel ↔ Go ↔ Rust)
- [ ] Complete user journey (upload → process → playback)
- [ ] Client behavior (browser + mobile/API)
- [ ] Capacity (uploads, catalog reads, workers, Kafka lag, RabbitMQ, CDN origin)
- [ ] Security (private media, signed URLs, upload abuse, service identity, secrets, injection, privilege)
- [ ] Failure recovery (outages, duplicates, out-of-order, poison messages, corrupt media, worker crashes)
- [ ] Data safety (migrations, backup restore, deletion, takedown, disaster recovery)

### Launch Tasks
- [ ] Define incident response (severity, escalation, expectations)
- [ ] Assign service ownership (Laravel, Go, Rust, infra, brokers, storage, media)
- [ ] Write operational runbooks (failed processing, DLQ replay, Kafka replay, revocation, takedown, restore, failover)
- [ ] Create dashboards and alerts (API errors, processing failures, queue depth, Kafka lag, CDN errors, playback startup)
- [ ] Safe deployment strategy (feature flags + canary / blue-green)
- [ ] Readiness review (security, QA, operations, recovery, ownership)

### Exit Gate
Security, QA, observability, runbooks, backup restore, rollback, and incident ownership are approved. The first VOD release can be deployed and operated by someone other than the original implementer.

---

## ✅ IMPLEMENTATION PHASE 7 — Product Expansion and Read Scale
> **Goal**: Add features that create user value without destabilizing the VOD core.  
> **Duration estimate**: 4–8 weeks  
> **Prerequisites**: Phase 6 production launch complete + production metrics show actual bottlenecks

### Tasks (ordered)
- [ ] Build read projections (channel pages, video feeds, creator dashboards, moderation queues)
- [ ] Add subscriptions and notifications
- [ ] Add engagement features (likes, comments, playlists, watch history) with abuse controls
- [ ] Add watch analytics (privacy-aware retention + late-event handling)
- [ ] Add search (dedicated projection / index from approved catalog events)
- [ ] Add recommendations (only after reliable watch + interaction data exists)
- [ ] Design advanced products separately (monetization, copyright claims, age restrictions, live streaming)

### Exit Gate
Each new feature has its own ownership, data model, API/event contracts, QA coverage, abuse controls, dashboards, and rollback plan.

---

## ✅ IMPLEMENTATION PHASE 8 — Global Scale and Service Extraction
> **Goal**: Scale based on measured demand and clear ownership, not premature microservice separation.  
> **Duration estimate**: Ongoing / as needed  
> **Prerequisites**: Phase 6 stability + Phase 7 bottlenecks measured + business case for each extraction

### Tasks (ordered)
- [ ] Make core infrastructure highly available (multi-AZ databases and brokers)
- [ ] Replicate media and deliver from the edge
- [ ] Controlled regional failover (one write region first)
- [ ] Regional processing where latency or residency requires it
- [ ] Multi-region read models
- [ ] Extract media orchestration to Go only when justified by measured needs
- [ ] Extract additional services (analytics, search, recommendations, engagement) only with evidence + owner
- [ ] Design active/active writes carefully (conflict resolution + ownership + consistency first)
- [ ] Global readiness tests (load, soak, chaos, failover, restore, replay, cost)

### Exit Gate
Regional failure, data restoration, broker replay, scaling, security, cost, and operational exercises meet the approved global SLOs.

---

## 🏛️ SOLID Applied — Real Definitions

| Principle | Diagnostic Test | Fix Applied |
|-----------|-----------------|-------------|
| **S** — One reason to change | Would a DB admin + UI designer + domain expert all edit the same controller? | Controllers/Actions handle only HTTP or entry; Services own domain; Repos own SQL |
| **O** — Open/Closed | Adding a new video status or permission requires editing conditionals? | Lifecycle + policy registry; new states/permissions are configuration or new handlers |
| **L** — Liskov Substitution | Does an Eloquent repository work wherever the interface is expected? | Test doubles substitute cleanly |
| **I** — Interface Segregation | Does a broad repository force unused methods? | Split into focused reader/writer or capability interfaces |
| **D** — Dependency Inversion | Does a service `new` a concrete repository or client? | Inject via container; bound to interfaces |

> **Guardrail**: An interface earns its place only with a real second implementation, a genuine test seam, or a true policy/detail boundary.

---

## 📋 Naming Standard

| Pattern | ✅ Correct | ❌ Wrong |
|---------|-----------|---------|
| Service / Action verbs | `VideoService.publish()`, `UploadSessionService.complete()` | `VideoManager.handle()` |
| Repository | `findByPublicId()`, `findReadyForPlayback()` | `getData()`, `getAll()` |
| Booleans | `isReady`, `hasPermission`, `canRetry` | `flag`, `status`, `isNotReady` |
| I/O honest naming | `fetchPlaybackManifest()`, `loadChannel()` | `getPlaybackManifest()` (implies cheap) |
| No noise words | `ProcessingAttemptService` | `ProcessingHelper`, `ProcessingUtil`, `ProcessingManager` |
| Units in name | `timeoutMs`, `expirySeconds`, `maxFileSizeBytes` | `timeout`, `expiry`, `maxSize` |

---

## 🔁 Per-Step Verification Protocol

After every structural or behavioral move, before marking the task done:

```bash
# Backend
php artisan test                    # existing + new tests must pass
php artisan route:list              # all versioned routes resolve
php artisan migrate --pretend       # migrations are safe

# Contracts
# Validate API examples + event schemas against the approved contracts

# Workers (when present)
# Health checks green, metrics emitted, DLQ empty under normal load

# End-to-end (from Phase 5 onward)
# Full flow: channel → video → upload → quarantine → probe → process → playback
# Must survive duplicates, retries, and worker crashes
```

> “It compiles / tests pass” is not full proof. Also check by eye: state transitions, null vs missing, async ordering, thrown-vs-returned errors, and whether infrastructure types leak upward.

---

## 📋 Your Tasks / My Tasks Split

| Task Type | Owner |
|-----------|-------|
| Product decisions, business rules, capacity numbers, SLOs, budget | **You** |
| Approve each phase exit gate before the next phase begins | **You** |
| Verify business flows make real-world sense | **You** |
| All structural design, contracts, and code writing | **Me** |
| Domain modeling, lifecycle, ownership matrix | **Me** |
| Per-step verification and reporting | **Me** |
| Smell → principle → fix mapping | **Me** |

---

## 🗓️ Phase Priority Order

```
DESIGN A–H  →  Requirements, quality, domain, flows, architecture, contracts, QA design, approval
     ↓
PHASE 0     →  Requirements and decisions          ← START HERE
PHASE 1     →  Architecture and contracts
PHASE 2     →  Production foundation
PHASE 3     →  Identity, channels, video catalog
PHASE 4     →  Upload and asset lifecycle
PHASE 5     →  Processing, events, and playback    ← First full VOD path
PHASE 6     →  Reliability, QA, and production launch
PHASE 7     →  Product expansion and read scale
PHASE 8     →  Global scale and service extraction
```

> No phase bleeds into the next. Phase N is fully verified and gated before Phase N+1 begins.  
> Kafka, RabbitMQ, Go, and Rust appear only after architecture and contracts are approved.

---

## Pre-Coding Approval Checklist

- [ ] Phase 0 requirements and scale targets approved
- [ ] Phase 1 system design and contracts approved
- [ ] Data ownership and lifecycle states approved
- [ ] Security, privacy, retention, and takedown rules approved
- [ ] QA and failure-testing strategy approved
- [ ] Deployment, observability, backup, and rollback strategy approved
- [ ] Team ownership and operating model approved

**Until every item is checked, the project is still in design and should not begin feature implementation.**

---

*Generated from system-design.md + implementation_plan.md style | Stack: Laravel 13 · PHP 8.3 · PostgreSQL · S3 · Kafka · RabbitMQ · Go/Rust workers*
