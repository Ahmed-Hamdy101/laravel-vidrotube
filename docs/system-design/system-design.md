# VidroTube System Design and Delivery Plan

## Purpose

This document is the implementation roadmap for VidroTube. It is intentionally ordered. A later phase must not start until its prerequisite gate is complete.

The first release is the first public launch of VidroTube, also called the MVP (Minimum Viable Product). It is the smallest useful version that lets creators upload recorded videos and lets viewers watch them. It includes users, channels, uploads, processing, playback, visibility, moderation hooks, and basic creator analytics. Live streaming, monetization, advanced recommendations, DRM, and full copyright fingerprinting are later products.

## Current starting point

The repository is a Laravel 13 / PHP 8.3 starter application with Fortify, Livewire, Flux, and Vite. It currently has an incomplete Channel model and migration, but no video catalog, upload pipeline, object storage, CDN, Kafka, RabbitMQ, Go service, Rust service, or deployment infrastructure.

## What system design means

System design answers these questions before coding:

1. **What must the product do?** These are the functional requirements.
2. **How well must it work?** These are the non-functional requirements.
3. **What information must the system store?** These are the core entities and their relationships.
4. **Which parts of the system perform each job?** These are the architecture and service boundaries.
5. **How do the parts communicate and recover from failure?** These are the API, event, retry, security, and operations designs.

The order is important. Do not choose Kafka, RabbitMQ, Go, or Rust before answering the first three questions.

## Functional requirements: what users can do

Functional requirements describe user actions and business behavior. They answer: “What should VidroTube do?”

### Creator functions

1. A user can register and sign in.
2. A user can create and manage a Channel.
3. A creator can create a Video record with a title, description, and visibility setting.
4. A creator can start an Upload Session and upload a recorded video directly to storage.
5. A creator can resume an interrupted upload and safely retry its completion.
6. The platform can validate the uploaded file and show processing status.
7. The platform can create playback versions, thumbnails, and captions.
8. A creator can publish a video after processing and required moderation checks finish.
9. A creator can change a video between public, unlisted, and private visibility.
10. A creator can view basic analytics such as views and watch time.
11. A creator or administrator can delete a video and revoke its playback access.

### Viewer functions

1. A viewer can browse public Channels and Videos.
2. A viewer can open a Video page and see its title, thumbnail, description, and captions when available.
3. A viewer can play an approved public or authorized private Video.
4. The player can select a suitable playback quality.
5. A viewer cannot play a private, blocked, deleted, or still-processing Video without permission.

### Administrator functions

1. An administrator can review a Video's moderation status.
2. An administrator can block, approve, restrict, or restore a Video according to policy.
3. An administrator can inspect processing failures and retry or stop work.
4. An administrator can audit important changes and access decisions.

## Non-functional requirements: how the system must work

Non-functional requirements describe quality, limits, security, and operations. They answer: “How fast, safe, reliable, and maintainable must VidroTube be?”

1. **Performance:** Define API latency, upload speed expectations, processing time, and playback startup time.
2. **Scalability:** Define maximum users, uploads per day, concurrent viewers, video size, storage growth, and regions.
3. **Availability:** Define how often the API, upload service, processing service, and playback service may be unavailable.
4. **Reliability:** Define retry behavior, duplicate-message safety, backup frequency, and recovery procedures.
5. **Security:** Require authentication, authorization, private media protection, encryption, secret rotation, malware checks, and audit logs.
6. **Privacy:** Define data retention, deletion, export, regional storage, PII handling, and legal takedown behavior.
7. **Observability:** Require logs, metrics, traces, alerts, request IDs, correlation IDs, queue depth, and processing status visibility.
8. **Maintainability:** Require versioned APIs and events, automated tests, documentation, code ownership, and safe migrations.
9. **Cost control:** Define storage, bandwidth, transcoding, broker, database, and worker budgets.
10. **Disaster recovery:** Define RPO, RTO, regional failover, restore testing, and operator runbooks.

## Core entities and relationships

These are the main records the business needs. Start with these before designing microservices.

```text
User
    |
    +-- owns --> Channel
                                 |
                                 +-- contains --> Video
                                                                     |
                                                                     +-- has --> UploadSession
                                                                     +-- has --> MediaAsset
                                                                     +-- has --> Thumbnail
                                                                     +-- has --> Caption
                                                                     +-- has --> Rendition
                                                                     +-- has --> ProcessingAttempt
                                                                     +-- has --> ModerationCase
                                                                     +-- produces --> WatchEvent / Analytics
```

| Entity | What it represents | First owner |
|---|---|---|
| User | Account and identity of a person | Laravel/PostgreSQL |
| Channel | Creator space containing Videos | Laravel/PostgreSQL |
| Video | Title, description, owner, visibility, and lifecycle state | Laravel/PostgreSQL |
| UploadSession | Permission and progress record for one upload | Laravel/PostgreSQL |
| MediaAsset | A stored original or generated media file | Laravel metadata plus object storage file |
| Thumbnail | Preview image for a Video | Laravel metadata plus object storage file |
| Caption | Subtitle or transcript file for a Video | Laravel metadata plus object storage file |
| Rendition | A playable quality such as 360p or 1080p | Laravel metadata plus object storage file |
| ProcessingAttempt | One attempt to validate, probe, transcode, or generate media | Processing worker |
| ModerationCase | Review decision and reason for a Video | Laravel/admin system |
| WatchEvent | A viewer activity record used for analytics | Event stream/analytics system |
| AuditRecord | History of important changes and administrative actions | Laravel/PostgreSQL |

## System design phases

These are the design phases. They explain the system before the later delivery phases explain implementation.

### Design Phase A: Product requirements

Write the user actions, business rules, first-release scope, exclusions, limits, and user roles. Output: approved functional requirements.

### Design Phase B: Quality requirements

Write performance, scale, availability, reliability, security, privacy, observability, cost, and recovery targets. Output: approved non-functional requirements.

### Design Phase C: Domain and data design

Define the core entities, fields, relationships, lifecycle states, ownership, indexes, retention, and deletion rules. Output: entity model and data ownership matrix.

### Design Phase D: User-flow design

Draw the channel creation, upload, processing, moderation, playback, analytics, and deletion flows. Output: sequence diagrams and failure paths.

### Design Phase E: Architecture design

Assign responsibilities to Laravel, PostgreSQL, Redis, object storage, CDN, Go, Rust, Kafka, RabbitMQ, and transcoding infrastructure. Output: system context and container diagrams.

### Design Phase F: Contract design

Define API requests, responses, events, commands, schemas, authentication, idempotency, retries, and dead-letter behavior. Output: API, event, and message contracts.

### Design Phase G: QA and operations design

Define unit, integration, end-to-end, load, security, failure, backup, restore, deployment, monitoring, and incident tests. Output: QA strategy, dashboards, alerts, and runbooks.

### Design Phase H: Approval gate

Review all outputs with the team. Only after approval should implementation begin. Output: signed-off architecture and a prioritized implementation backlog.

## Non-negotiable ordering

```text
DESIGN PHASE A: Product requirements
    |
DESIGN PHASE B: Quality requirements
    |
DESIGN PHASE C: Domain and data design
    |
DESIGN PHASE D: User-flow design
    |
DESIGN PHASE E: Architecture design
    |
DESIGN PHASE F: Contract design
    |
DESIGN PHASE G: QA and operations design
    |
DESIGN PHASE H: Approval gate
    |
IMPLEMENTATION PHASE 0: Requirements and decisions
    |
IMPLEMENTATION PHASE 1: Architecture and contracts
    |
IMPLEMENTATION PHASE 2: Production foundation
    |
IMPLEMENTATION PHASE 3: Identity, channels, and video catalog
    |
IMPLEMENTATION PHASE 4: Upload and asset lifecycle
    |
IMPLEMENTATION PHASE 5: Processing and playback
    |
IMPLEMENTATION PHASE 6: Reliability, QA, and production launch
    |
IMPLEMENTATION PHASE 7: Product expansion and read scale
    |
IMPLEMENTATION PHASE 8: Global scale and service extraction
```

The existing numbered phases below are implementation phases. The design phases above must be completed first. Kafka, RabbitMQ, Go, and Rust are introduced only after the architecture and contracts are approved.

## Technology responsibilities

| Component | Responsibility | Does not own |
|---|---|---|
| Laravel | Identity, channels, video metadata, authorization, upload sessions, admin APIs | Video bytes, transcoding, analytics processing |
| PostgreSQL | Transactional metadata and state transitions | Media files, event replay, cache |
| Redis | Cache, locks, rate limits, ephemeral state | Source-of-truth business records |
| S3-compatible storage | Original files, quarantine files, renditions, manifests, captions, thumbnails | Authorization decisions |
| Kafka | Durable domain events, analytics streams, replay, projections | Short-lived work retries |
| RabbitMQ | Commands, processing jobs, priorities, retries, dead-letter queues | Long-term event history |
| Go | Distributed workers, orchestration, consumers, projections, internal APIs | Laravel business data ownership |
| Rust | Specialized media validation or CPU-sensitive processing | General product APIs |
| Lambda | Short stateless event-driven tasks | Long-running full-video transcoding |
| MediaConvert/ECS | Long-running and resource-intensive transcoding | User identity and catalog ownership |
| CDN | Global playback delivery | Original media processing |

All cross-language communication uses versioned JSON Schema or Protobuf contracts. No Go or Rust service reads Laravel tables directly, and Laravel serialized jobs are never used as service contracts.

## How to understand every task

Each task below answers four questions:

- **What:** What are we creating or deciding?
- **Related to:** Which users, data, services, or later tasks depend on it?
- **Why:** What problem does it solve?
- **Done when:** What evidence proves the task is complete?

### Basic product vocabulary

- **User:** A person with an account who can create a channel or watch videos.
- **Channel:** A creator's public space. It belongs to a User and contains that creator's videos.
- **Video:** The business record describing a video: title, owner, channel, visibility, status, and playback information. It is not the binary file itself.
- **Media asset:** A stored file related to a Video, such as the original upload, a thumbnail, a caption file, or a streaming rendition.
- **Upload session:** A temporary record that gives one user permission to upload one file and tracks whether the upload completed.
- **Rendition:** A playback version of the same video, such as 360p, 720p, or 1080p.
- **Moderation:** The process that decides whether a video is allowed, blocked, restricted, or still under review.
- **Playback:** Delivering approved video renditions to a viewer through a player and CDN.
- **Analytics:** Aggregated information about viewing, such as views, watch time, and completion rate.
- **MVP (Minimum Viable Product):** The first public launch with only the features required for creators to publish recorded videos and viewers to watch them. It is not the final version of VidroTube.
- **Domain model:** The important business objects and their relationships.
- **Contract:** The agreed structure of an API request, event, or message between systems.
- **Exit gate:** A review condition. The next phase cannot begin until it is satisfied.

### Basic technical vocabulary

- **PostgreSQL:** The main relational database for users, channels, videos, upload sessions, and processing state.
- **Redis:** A fast temporary data store for cache, locks, rate limits, and short-lived state. It is not the permanent business database.
- **Object storage:** File storage such as S3. It holds large video files and generated media instead of putting them in PostgreSQL.
- **CDN:** A network of edge servers that delivers playback files close to viewers.
- **Outbox:** A database table that records an event inside the same transaction as a business change, preventing lost events.
- **Inbox/idempotency:** A record showing that a consumer has already handled a message, making duplicate delivery safe.
- **Kafka topic:** A durable named stream of events that can be retained, replayed, and consumed by multiple systems.
- **RabbitMQ queue:** A work list from which a worker receives a command, completes it, and acknowledges it.
- **Worker:** A program that performs background work outside the web request, such as probing or transcoding a video.
- **Go service:** A suitable language choice for high-concurrency workers, message consumers, and orchestration.
- **Rust service:** A suitable language choice for specialized secure or CPU-intensive media work; it is not required for every worker.
- **Lambda:** A short-lived serverless function. It is useful for small event-driven steps, not normally for full-length transcoding.
- **HLS/DASH:** Streaming formats that split a video into small segments and quality levels for adaptive playback.
- **DLQ:** Dead-letter queue. It stores messages that failed too many times so an operator can inspect or replay them.
- **Projection/read model:** A read-optimized view built from events for feeds, search, dashboards, or moderation screens.
- **SLO:** Service-level objective, such as “95% of playback sessions start within two seconds.”
- **RPO/RTO:** Recovery Point Objective is the maximum acceptable data loss; Recovery Time Objective is the maximum acceptable recovery time.
- **Infrastructure as code:** Version-controlled definitions for servers, networks, databases, permissions, storage, and deployments.
- **CI:** Continuous integration. Automated checks that run when code changes, such as tests, static analysis, and security scans.

# Implementation Phase 0: Requirements and decisions

## Goal


Agree on what is being built, its scale, and its operating constraints before designing services or writing code.

## Tasks

1. **Define the first-release product.**
    - **What:** Decide what users can do when VidroTube launches publicly for the first time. This first launch is called the MVP (Minimum Viable Product).
    - **Related to:** Users create Channels; Channels contain Videos; Videos use Upload Sessions, Media Assets, Moderation, Playback, and Analytics.
    - **Why:** We need a small, understandable starting product before building advanced features.
    - **Example creator actions:** Create a channel, upload a recorded video, add a title and thumbnail, choose public/private visibility, and see basic view statistics.
    - **Example viewer actions:** Find a public video, open its page, play it, change quality, and read captions.
    - **Done when:** A new developer can read one list and understand exactly what creators and viewers can do at the first public launch.
2. **Record what is out of scope.**
    - **What:** Explicitly list features that version one will not build.
    - **Related to:** Live video, billing, advertising, copyright, security, and recommendation teams.
    - **Why:** These features create separate systems and must not appear as hidden work inside the VOD release.
    - **Done when:** Each excluded feature has a later placeholder and is removed from version-one estimates.
3. **Define capacity limits.**
    - **What:** Set the maximum size and expected volume of the system.
    - **Related to:** S3 storage, upload APIs, transcoding workers, PostgreSQL, Kafka, RabbitMQ, CDN bandwidth, and cost.
    - **Why:** A 100 MB video and a 100 GB video require completely different designs.
    - **Done when:** The team has numbers for file size, duration, uploads per day, viewers, users, regions, and retention.
4. **Define service-level objectives.**
    - **What:** Set measurable performance and availability targets.
    - **Related to:** API servers, upload storage, media processing, CDN playback, monitoring, and incident response.
    - **Why:** “Fast” and “reliable” are not testable requirements.
    - **Done when:** Every target has a number, a measurement method, and an owner.
5. **Define recovery and compliance requirements.**
    - **What:** Decide how much data may be lost, how quickly service must recover, where data may live, and what privacy rules apply.
    - **Related to:** Backups, PostgreSQL replication, object storage, regional deployment, deletion, privacy, and legal operations.
    - **Why:** Recovery and privacy requirements affect architecture before production exists.
    - **Done when:** RPO, RTO, regions, retention, deletion rules, and budget are approved.
6. **Choose the initial cloud platform.**
    - **What:** Choose where the production systems will run.
    - **Related to:** PostgreSQL/Aurora, Redis, S3, CloudFront, Lambda, Kafka, RabbitMQ, IAM, and deployment tooling.
    - **Why:** Each provider offers different services, limits, pricing, and operational tools.
    - **Done when:** The team has selected a provider and mapped each required capability to a service.
7. **Define identity and API rules.**
    - **What:** Decide how people, services, and public resources are identified and authorized.
    - **Related to:** Users, Channels, Videos, browser/mobile clients, Laravel APIs, Go workers, Rust workers, and private playback.
    - **Why:** Changing identity and authorization after data exists is expensive and risky.
    - **Done when:** Public IDs, API versions, login method, roles, permissions, and service authentication are documented.

## Outputs

- Approved product scope
- Capacity and SLO document
- Risk and assumption register
- Initial cost model
- Product glossary

## Exit gate

No architecture work proceeds until product scope, scale targets, SLOs, RPO/RTO, regions, and budget are approved.

# Implementation Phase 1: Architecture and contracts

## Goal

Define boundaries and contracts so Laravel, Go, Rust, Kafka, and RabbitMQ have non-overlapping responsibilities.

## Tasks

1. **Define the domain model.**
    - **What:** Design the business objects and their relationships: User owns Channel; Channel contains Video; Video has UploadSession, MediaAsset, Rendition, Thumbnail, Caption, ProcessingAttempt, ModerationCase, and AuditRecord records.
    - **Related to:** PostgreSQL tables, Laravel models, API resources, authorization, workers, and tests.
    - **Why:** Every later feature needs to know which object it changes.
    - **Done when:** The team can draw the objects, relationships, required fields, and ownership rules.
2. **Define data ownership.**
    - **What:** Decide which system is the authority for each type of data.
    - **Related to:** Laravel/PostgreSQL catalog state, object storage media, worker processing attempts, and Kafka-fed read projections.
    - **Why:** Two systems must not both claim to be allowed to change the same record.
    - **Done when:** Every important field has one owner and other systems access it through an API or event.
3. **Define the video lifecycle.**
    - **What:** Define every status a video can have and the allowed transitions between statuses.
    - **Related to:** Uploads, moderation, workers, playback authorization, deletion, and creator UI messages.
    - **Why:** A video must not become playable while it is still uploading, unsafe, or incomplete.
    - **Done when:** Each transition has a trigger, an owner, a failure path, and a test.

    ```text
    created -> uploading -> uploaded -> quarantined -> processing -> ready
    ```

    - **Result:** Every team knows which state transitions are valid.

4. **Design the upload flow.**
    - **What:** Describe how a user starts, performs, resumes, and completes an upload.
    - **Related to:** Laravel upload sessions, the browser/mobile client, S3, checksums, quarantine, and Phase 4.
    - **Why:** Sending large video files through PHP would cause timeouts, memory pressure, and scaling problems.
    - **Done when:** The flow describes every request, response, stored record, and failure case.
5. **Design the processing flow.**
    - **What:** Describe how an uploaded file becomes a playable video.
    - **Related to:** RabbitMQ commands, Go workers, optional Rust workers, transcoding, object storage, moderation, and CDN playback.
    - **Why:** Processing is asynchronous and may take minutes or fail partway through.
    - **Done when:** Every processing step has an input, output, retry rule, failure state, and owner.
6. **Define API contracts.**
    - **What:** Specify the exact requests and responses exchanged by clients and services.
    - **Related to:** Laravel controllers, browser/mobile clients, Go workers, Rust workers, and playback authorization.
    - **Why:** A contract prevents each service from inventing a different field name or behavior.
    - **Done when:** Examples, validation rules, error responses, authentication, and versioning are documented.
7. **Define the event envelope.**
    - **What:** Define the common wrapper around every asynchronous event.
    - **Related to:** Kafka, RabbitMQ messages, outbox records, Go consumers, Rust workers, tracing, and replay.
    - **Why:** Consumers need to identify, validate, trace, and safely repeat events.
    - **Done when:** One valid example exists for each required field and invalid versions are handled.
8. **Define delivery guarantees.**
    - **What:** Decide how events are stored before publishing and how consumers remember completed work.
    - **Related to:** PostgreSQL outbox, consumer inbox, Kafka offsets, RabbitMQ acknowledgements, retries, and dead-letter queues.
    - **Why:** Distributed systems normally deliver messages at least once, so duplicates are expected.
    - **Done when:** A duplicate, timeout, crash, and replay scenario has a defined safe outcome.
9. **Define Kafka topics.**
    - **What:** Define the durable streams of facts and analytics events.
    - **Related to:** Domain events, watch analytics, projections, recommendations, retention, and replay.
    - **Why:** Kafka is useful only when event ownership and history are managed deliberately.
    - **Done when:** Each topic has a purpose, producer, consumers, key, retention, schema, and privacy policy.
10. **Define RabbitMQ queues.**
    - **What:** Define temporary commands and jobs that workers must execute.
    - **Related to:** Media probing, thumbnails, transcoding, captions, moderation, retries, and DLQs.
    - **Why:** Work queues need fast acknowledgement and retry behavior, not Kafka-style history.
    - **Done when:** Every queue has a command schema, consumer, concurrency limit, retry policy, and DLQ.
11. **Define failure behavior.**
    - **What:** Decide the response to every common distributed-system failure.
    - **Related to:** Video statuses, retries, idempotency, operator tools, user-facing errors, and alerts.
    - **Why:** The failure path is part of the product, not only an infrastructure detail.
    - **Done when:** Duplicate, delayed, out-of-order, invalid, and partial-processing examples have expected outcomes.
12. **Choose the first Go and Rust responsibilities.**
    - **What:** Select the first small Go service and, only if justified, one Rust worker.
    - **Related to:** RabbitMQ, Kafka, media validation, thumbnails, deployment, metrics, and team skills.
    - **Why:** Multiple languages add operational cost and should solve a specific problem.
    - **Done when:** Each selected service has a purpose, owner, contract, deployment target, and success metric.

## Outputs

- System context diagram
- Container/service boundary diagram
- Domain model
- Video lifecycle state machine
- API contracts
- Event and message contracts
- Kafka topic catalog
- RabbitMQ queue catalog
- Data ownership matrix
- Initial architecture decision records

## Exit gate

No production implementation proceeds until boundaries, contracts, state transitions, ownership, retry behavior, and security assumptions are reviewed and approved.

# Implementation Phase 2: Production foundation

## Goal

Create the environments and platform capabilities required by every later phase.

## Prerequisites

- Phase 0 and Phase 1 exit gates approved

## Tasks

1. **Set up PostgreSQL.**
    - **What:** Choose PostgreSQL for production and document local setup.
    - **Related to:** Users, Channels, Videos, migrations, backups, and Laravel models.
    - **Why:** Business records need transactions, constraints, and reliable recovery.
    - **Done when:** Local and staging applications can connect and run migrations.
2. **Set up Redis.**
    - **What:** Configure cache, locks, rate limits, and Laravel-local queues.
    - **Related to:** API performance, duplicate-request protection, and background work.
    - **Why:** Temporary and high-speed state should not overload PostgreSQL.
    - **Done when:** Redis is configured, monitored, and safe to lose without losing business data.
3. **Set up object storage.**
    - **What:** Configure S3-compatible storage with encryption, lifecycle rules, quarantine prefixes, and restricted access.
    - **Related to:** Uploads, originals, thumbnails, captions, renditions, and deletion.
    - **Why:** Video files are too large and numerous for the relational database.
    - **Done when:** A test file can be stored privately, retrieved by an authorized service, and expired by policy.
4. **Prepare CDN security.**
    - **What:** Configure origin protection and signed playback access prerequisites.
    - **Related to:** Video visibility, private playback, CDN delivery, and takedowns.
    - **Why:** Viewers must not bypass authorization by accessing storage directly.
    - **Done when:** Storage is private and only an approved access path can reach playback files.
5. **Define environment configuration.**
    - **What:** Define settings for local, development, staging, and production.
    - **Related to:** Databases, storage, brokers, secrets, workers, and deployment.
    - **Why:** Different environments need different resources without changing application behavior.
    - **Done when:** A new developer and CI can configure each environment from documented variables.
6. **Manage secrets and credentials.**
    - **What:** Add environment validation, secret storage, key rotation, and service credentials.
    - **Related to:** Database access, object storage, Kafka, RabbitMQ, Go, Rust, and deployment.
    - **Why:** Credentials in source code or logs can compromise every service.
    - **Done when:** Secrets are injected securely, rotated, and absent from repositories and logs.
7. **Add observability basics.**
    - **What:** Add logs, metrics, traces, request IDs, correlation IDs, health checks, and readiness checks.
    - **Related to:** Every Laravel, Go, Rust, worker, broker, and media-processing operation.
    - **Why:** A distributed failure cannot be debugged from one application log.
    - **Done when:** One upload can be traced across API, storage, queue, worker, and playback steps.
8. **Create infrastructure as code.**
    - **What:** Define networks, IAM, databases, Redis, buckets, CDN, and workers as versioned configuration.
    - **Related to:** Deployment, disaster recovery, security review, and staging parity.
    - **Why:** Manually configured infrastructure cannot be reliably reproduced.
    - **Done when:** Staging can be created and updated from source-controlled definitions.
9. **Strengthen CI.**
    - **What:** Run tests, static analysis, frontend checks, migration checks, dependency audits, and image scans.
    - **Related to:** Every later code change and production release.
    - **Why:** Errors should be rejected before reaching shared environments.
    - **Done when:** CI blocks a deliberately broken test, migration, contract, or vulnerable dependency.
10. **Define recovery procedures.**
     - **What:** Document backup, restore, migration, rollback, and disaster-recovery procedures.
     - **Related to:** PostgreSQL, object storage, brokers, deployments, RPO, and RTO.
     - **Why:** A backup is not useful until restoration has been practiced.
     - **Done when:** A test environment has restored data and completed a documented rollback.

## Parallel work allowed

- Infrastructure code and CI can proceed in parallel.
- Observability can proceed in parallel with environment setup.
- No product feature may depend on an unapproved production service.

## Exit gate

Staging can deploy repeatably, secrets are managed, PostgreSQL/Redis/object storage work, logs and alerts are visible, backups restore successfully, and CI blocks broken migrations or contracts.

# Implementation Phase 3: Identity, channels, and video catalog

## Goal

Create the transactional business model before adding asynchronous media processing.

## Prerequisites

- Phase 2 exit gate complete

## Tasks

1. **Connect channels to users.**
    - Make every channel owned by a user.
    - Result: ownership and permissions are enforceable.
2. **Add public channel identity.**
    - Add public IDs, unique handles, display metadata, lifecycle status, and authorization policies.
    - Result: channels can be safely exposed through URLs and APIs.
3. **Create video metadata models.**
    - Add Video and MediaAsset records without storing binary files in the database.
    - Result: catalog data is separate from media data.
4. **Add video visibility states.**
    - Support public, unlisted, private, blocked, deleted, and processing states.
    - Result: every video access decision has a defined status.
5. **Protect catalog integrity.**
    - Add constraints, foreign keys, indexes, soft deletion, audit records, and idempotency records.
    - Result: invalid ownership and duplicate operations are prevented.
6. **Add application services.**
    - Centralize channel and video state changes in actions or services.
    - Result: controllers and future workers use the same business rules.
7. **Add versioned APIs.**
    - Expose channel and video operations through versioned endpoints with authorization tests.
    - Result: clients have a stable contract.
8. **Add catalog tests and factories.**
    - Test ownership, visibility, moderation blocks, deletion, and lifecycle transitions.
    - Result: the catalog is safe before uploads are added.

## Files initially affected

- `app/Models/Channel.php`
- `app/Models/User.php`
- `database/migrations/`
- `database/factories/`
- `app/Http/Controllers/`
- `app/Policies/`
- `routes/`
- `tests/Feature/`

## Exit gate

A user can create and manage an authorized channel and video metadata through tested APIs. No binary upload is part of this phase.

# Implementation Phase 4: Upload and asset lifecycle

## Goal

Allow users to upload large files directly to object storage without sending video bytes through Laravel.

## Prerequisites

- Phase 3 exit gate complete
- Object storage and security controls from Phase 2 available

## Tasks

1. **Create upload sessions.**
    - Store size, MIME type, checksum, expiry, region, and owner with an idempotency key.
    - Result: each upload has a controlled server-side record.
2. **Issue presigned upload URLs.**
    - Return short-lived multipart URLs for direct client-to-storage uploads.
    - Result: Laravel does not carry large video files.
3. **Upload to quarantine storage.**
    - Keep new files isolated until validation completes.
    - Result: untrusted files cannot immediately become playable.
4. **Verify upload completion.**
    - Check object existence, size, checksum, and ownership on the server.
    - Result: the platform does not trust the browser alone.
5. **Record atomic upload states.**
    - Move assets through uploading, uploaded, and quarantined without partial database updates.
    - Result: the database reflects the real upload lifecycle.
6. **Clean up abandoned uploads.**
    - Expire unfinished sessions and apply object lifecycle policies.
    - Result: unused storage does not grow indefinitely.
7. **Add validation integration points.**
    - Prepare hooks for virus scanning, file inspection, and content validation.
    - Result: security checks can run before processing.
8. **Test upload failure cases.**
    - Test interruption, resume, duplicate completion, expired URLs, bad checksums, and unauthorized access.
    - Result: uploads are reliable beyond the happy path.

## Exit gate

A real test client can upload an allowed file, resume an interrupted upload, safely retry completion, and receive a durable quarantined asset without Laravel handling the file bytes.

# Implementation Phase 5: Processing, events, and playback

## Goal

Complete one reliable upload-to-playback path before adding social features or global scale.

## Prerequisites

- Phase 4 exit gate complete
- Phase 1 event and message contracts approved

## Tasks in order

### 5.1 Transactional outbox

1. **Write the outbox record.**
    - Save the event in the same database transaction as upload completion.
    - Result: state changes cannot succeed while their event is lost.
2. **Build the event relay.**
    - Publish versioned events with retry behavior and metrics.
    - Result: database events reach the messaging system reliably.
3. **Protect consumers from duplicates.**
    - Add inbox or idempotency handling to every consumer.
    - Result: repeated delivery is safe.

### 5.2 First Go worker

1. **Create the first Go worker.**
    - Build a small RabbitMQ consumer with health checks and metrics.
    - Result: the first external worker has a clear operational shape.
2. **Consume the probe command.**
    - Read a versioned `media.probe` message.
    - Result: processing starts from a stable command contract.
3. **Validate and inspect the object.**
    - Check the object reference and extract media metadata.
    - Result: the platform knows whether the file is processable.
4. **Update processing status.**
    - Call an approved Laravel API or command contract instead of reading Laravel tables.
    - Result: Laravel remains the catalog authority.
5. **Publish the probe result.**
    - Publish `MediaProbeCompleted` or `MediaProbeFailed`.
    - Result: later processing steps can react asynchronously.
6. **Handle worker failures.**
    - Add timeouts, retries, idempotency, and dead-letter behavior.
    - Result: temporary failures do not lose work.

### 5.3 First Rust workload

1. **Choose one specialized Rust task.**
    - Use Rust for secure container validation or thumbnail generation only if approved.
    - Result: Rust has a specific reason to exist.
2. **Define its contract.**
    - Give the worker versioned inputs and outputs.
    - Result: other services do not depend on Rust implementation details.
3. **Benchmark the workload.**
    - Compare memory, CPU, latency, and failures against Go or a managed alternative.
    - Result: the language choice is evidence-based.
4. **Keep or remove the service.**
    - Keep Rust only when the measured benefit justifies its additional cost.
    - Result: polyglot complexity remains controlled.

### 5.4 Transcoding and derivatives

1. **Run long transcoding outside Lambda.**
    - Submit jobs to MediaConvert or dedicated container workers.
    - Result: long-running work has suitable compute resources.
2. **Create streaming renditions.**
    - Produce the approved HLS/DASH quality ladder.
    - Result: viewers can stream at different network speeds.
3. **Store derived media.**
    - Store immutable manifests, renditions, thumbnails, and captions.
    - Result: playback assets can be cached and versioned safely.
4. **Track processing attempts.**
    - Record each attempt and resume retries from a known processing step.
    - Result: failed jobs do not restart unnecessarily or corrupt output.

### 5.5 Kafka streams

1. **Publish lifecycle events.**
    - Send events to Kafka after transactional state changes are committed.
    - Result: other systems can react without database sharing.
2. **Build read-model consumers.**
    - Consume events to update catalog and feed projections.
    - Result: hot reads do not require heavy transactional joins.
3. **Create the watch-event stream.**
    - Send viewing activity to a separate analytics stream.
    - Result: analytics load is isolated from catalog writes.
4. **Operate Kafka safely.**
    - Monitor consumer lag and support replay from retained events.
    - Result: projections can recover after failures.

### 5.6 Playback

1. **Authorize playback.**
    - Check visibility, moderation, ownership, expiry, deletion, and takedown state.
    - Result: viewers receive access only when allowed.
2. **Issue signed CDN access.**
    - Return signed URLs or cookies with a limited lifetime.
    - Result: media links cannot be reused indefinitely.
3. **Deliver streaming media.**
    - Serve HLS/DASH through the CDN.
    - Result: playback scales without routing every byte through Laravel.
4. **Test playback failures.**
    - Test revocation, private and blocked videos, missing renditions, CDN errors, and origin failure.
    - Result: playback behavior is known before launch.

## Exit gate

The complete flow works repeatedly:

```text
create channel -> create video -> create upload session -> upload -> complete
-> quarantine -> probe -> process -> publish manifest -> authorize playback
```

This flow must pass with duplicate requests, duplicate messages, worker retries, worker crashes, and failed processing recovery.

# Implementation Phase 6: Reliability, QA, and production launch

## Goal

Prove that the VOD core is secure, observable, operable, and ready for real users.

## Prerequisites

- Phase 5 end-to-end flow complete

## QA tracks

1. **Test business rules.**
    - Test domain rules, state transitions, policies, idempotency, retries, and contracts.
2. **Test infrastructure integration.**
    - Test PostgreSQL, Redis, object storage, Kafka, RabbitMQ, Go, Rust, and transcoding adapters.
3. **Test cross-language contracts.**
    - Verify API and event contracts across Laravel, Go, and Rust.
4. **Test the complete user journey.**
    - Test upload through processing and playback end to end.
5. **Test client behavior.**
    - Test browser and mobile/API clients.
6. **Test capacity.**
    - Load-test uploads, catalog reads, workers, Kafka lag, RabbitMQ queues, and CDN origin traffic.
7. **Test security.**
    - Test private media, signed URLs, upload abuse, service identity, secrets, injection, and privilege boundaries.
8. **Test failure recovery.**
    - Test outages, duplicate and out-of-order events, poison messages, corrupt media, delayed storage, and worker crashes.
9. **Test data safety.**
    - Test migrations, backup restore, deletion, takedown, and disaster recovery.

## Launch tasks

1. **Define incident response.**
    - Set incident severity, escalation paths, and response expectations.
    - Result: failures have an accountable response.
2. **Assign service ownership.**
    - Name owners for Laravel, Go, Rust, infrastructure, brokers, storage, and media processing.
    - Result: every production component has an owner.
3. **Write operational runbooks.**
    - Document failed processing, DLQ replay, Kafka replay, access revocation, takedown, restore, and region failover.
    - Result: operators have repeatable recovery steps.
4. **Create dashboards and alerts.**
    - Track API errors, processing failures, queue depth, Kafka lag, CDN errors, and playback startup.
    - Result: important problems are visible quickly.
5. **Use a safe deployment strategy.**
    - Deploy with feature flags and canary or blue-green releases.
    - Result: changes can be limited or rolled back.
6. **Run the readiness review.**
    - Review security, QA, operations, recovery, and ownership before launch.
    - Result: the release decision is evidence-based.

## Exit gate

Security, QA, observability, runbooks, backup restore, rollback, and incident ownership are approved. The first VOD release can be deployed and operated by someone other than the original implementer.

# Implementation Phase 7: Product expansion and read scale

## Goal

Add features that create user value without destabilizing the VOD core.

## Prerequisites

- Phase 6 production launch complete
- Production metrics show the actual bottlenecks

## Tasks in order

1. **Build read projections.**
    - Use Kafka events for channel pages, video feeds, creator dashboards, and moderation queues.
    - Result: high-volume reads avoid heavy catalog queries.
2. **Add subscriptions and notifications.**
    - Let viewers follow channels and receive relevant notifications.
    - Result: creators and viewers can build ongoing relationships.
3. **Add engagement features.**
    - Add likes, comments, playlists, and watch history with abuse controls.
    - Result: viewers can interact with and organize content safely.
4. **Add watch analytics.**
    - Process watch events with privacy-aware retention and late-event handling.
    - Result: creators receive useful viewing reports.
5. **Add search.**
    - Build a dedicated search projection or index from approved catalog events.
    - Result: search does not overload the transactional database.
6. **Add recommendations.**
    - Use recommendations only after reliable watch and interaction data exists.
    - Result: ranking is based on trustworthy signals.
7. **Design advanced products separately.**
    - Treat monetization, copyright claims, age restrictions, and live streaming as separate designs.
    - Result: complex products do not silently expand the VOD scope.

## Exit gate

Each new feature has its own ownership, data model, API/event contracts, QA coverage, abuse controls, dashboards, and rollback plan.

# Implementation Phase 8: Global scale and service extraction

## Goal

Scale based on measured demand and clear ownership, not on premature microservice separation.

## Prerequisites

- Phase 6 production stability
- Phase 7 bottlenecks measured
- Business case for each extracted service

## Tasks in order

1. **Make core infrastructure highly available.**
    - Add multi-AZ production databases and brokers.
    - Result: a single availability-zone failure does not stop the platform.
2. **Replicate media and deliver from the edge.**
    - Use replicated object storage and CDN edge delivery.
    - Result: users receive media from locations near them.
3. **Start with controlled regional failover.**
    - Keep one write region and test how traffic moves during regional failure.
    - Result: recovery is proven before active/active writes.
4. **Add regional processing where needed.**
    - Route uploads and processing by latency or data-residency requirements.
    - Result: the platform can meet regional needs without unnecessary complexity.
5. **Build multi-region read models.**
    - Replicate projections for feeds, search, analytics, and catalog reads.
    - Result: reads remain available during regional pressure or failure.
6. **Extract media orchestration only when justified.**
    - Move orchestration to Go when its scaling or deployment needs differ from Laravel.
    - Result: service extraction solves a measured problem.
7. **Extract additional services by evidence.**
    - Separate analytics, search, recommendations, or engagement only when load and team ownership justify it.
    - Result: each service has a real reason and owner.
8. **Design active/active writes carefully.**
    - Define conflict resolution, data ownership, and regional consistency before enabling multiple write regions.
    - Result: global writes do not create silent data corruption.
9. **Run global readiness tests.**
    - Perform load, soak, chaos, failover, restore, replay, and cost tests.
    - Result: global scale is demonstrated rather than assumed.

## Exit gate

Regional failure, data restoration, broker replay, scaling, security, cost, and operational exercises meet the approved global SLOs.

# Pre-coding approval checklist

- [ ] Phase 0 requirements and scale targets approved
- [ ] Phase 1 system design and contracts approved
- [ ] Data ownership and lifecycle states approved
- [ ] Security, privacy, retention, and takedown rules approved
- [ ] QA and failure-testing strategy approved
- [ ] Deployment, observability, backup, and rollback strategy approved
- [ ] Team ownership and operating model approved

Until every item is checked, the project is still in design and should not begin feature implementation.
