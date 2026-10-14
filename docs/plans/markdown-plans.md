# 🎬 VidroTube — Concrete Implementation Plan
### What you actually create, write, and run (step by step)
### Stack: Laravel 13 · PHP 8.3 · PostgreSQL · S3 · Kafka · RabbitMQ · Go/Rust workers

---

> [!IMPORTANT]
> **Do the phases in order.**  
> Do not create models, migrations, or workers before the phase that owns them.  
> Each phase ends with an **Exit Gate**. Do not start the next phase until the gate is checked.

---

## 📦 Starting Point (what you have today)

| Item | Status |
|------|--------|
| Laravel 13 + Fortify + Livewire + Flux + Vite | ✅ Exists |
| Incomplete `Channel` model + migration | ⚠️ Partial |
| Video catalog, upload, storage, workers, brokers | ❌ None |

---

## PHASE 0 — Requirements & Decisions
> **Goal:** Decide what V1 is, how big it is, and how reliable it must be.  
> **You write documents only. No code. No models. No migrations.**

### What you do

```bash
mkdir -p docs/phase-0
```

Create these 7 files and fill them:

| File | What you write inside |
|------|------------------------|
| `docs/phase-0/01-mvp-scope.md` | List of what creators and viewers can do at first public launch |
| `docs/phase-0/02-out-of-scope.md` | Explicit list of features that are **not** in V1 (live, monetization, DRM, comments, etc.) |
| `docs/phase-0/03-capacity-limits.md` | Numbers: max file size, max duration, uploads/day, concurrent viewers, regions, retention |
| `docs/phase-0/04-slos.md` | Measurable targets: API latency, playback start time, upload success rate, processing time |
| `docs/phase-0/05-recovery-compliance.md` | RPO, RTO, data region, deletion rules, GDPR/export rules, budget |
| `docs/phase-0/06-cloud-choice.md` | Chosen cloud + mapping (PostgreSQL, Redis, S3, CDN, Kafka, RabbitMQ, workers) |
| `docs/phase-0/07-identity-api-rules.md` | Public ID format, API version (`/api/v1`), login method, roles, service-to-service auth |

### Example content for `01-mvp-scope.md`

```markdown
# MVP Scope

## Creator can
- Register / sign in
- Create and manage one Channel
- Create a Video (title, description, visibility)
- Start upload session and upload a recorded video
- Resume interrupted upload
- See processing status
- Publish after processing + moderation
- Change visibility (public / unlisted / private)
- See basic analytics (views, watch time)
- Delete a video

## Viewer can
- Browse public channels and videos
- Open video page (title, thumbnail, description, captions)
- Play approved public or authorized private video
- Select playback quality
- Cannot play private / blocked / deleted / processing videos without permission

## Admin can
- Review moderation status
- Block / approve / restrict / restore video
- Inspect processing failures and retry
- Audit important changes
```

### What you do **not** do in Phase 0
- ❌ Create any model
- ❌ Create any migration
- ❌ Touch `app/`, `routes/`, `database/`
- ❌ Install Kafka, RabbitMQ, Go, or Rust

### Exit Gate
All 7 files exist, numbers are filled, and you (or the team) approve them.

---

## PHASE 1 — Architecture & Contracts
> **Goal:** Decide boundaries, data ownership, lifecycle, and message shapes.  
> **Still mostly documents + diagrams. Almost no product code.**

### What you do

```bash
mkdir -p docs/phase-1
```

Create these files:

| File | What you write |
|------|----------------|
| `docs/phase-1/01-domain-model.md` | Entities + relationships (User → Channel → Video → UploadSession, MediaAsset, etc.) |
| `docs/phase-1/02-data-ownership.md` | Who owns each field (Laravel/PostgreSQL vs object storage vs workers) |
| `docs/phase-1/03-video-lifecycle.md` | States and allowed transitions: `created → uploading → uploaded → quarantined → processing → ready` |
| `docs/phase-1/04-upload-flow.md` | Step-by-step: create session → presigned URL → client uploads to S3 → complete → quarantine |
| `docs/phase-1/05-processing-flow.md` | Probe → validate → transcode → thumbnails → captions → ready |
| `docs/phase-1/06-api-contracts.md` | Request/response examples for channel, video, upload session endpoints (versioned `/api/v1`) |
| `docs/phase-1/07-event-envelope.md` | Common wrapper for every event (id, type, version, occurred_at, correlation_id, payload) |
| `docs/phase-1/08-kafka-topics.md` | Topic list: purpose, producer, consumers, key, retention |
| `docs/phase-1/09-rabbitmq-queues.md` | Queue list: command schema, consumer, retry, DLQ |
| `docs/phase-1/10-failure-behavior.md` | What happens on duplicate, timeout, crash, out-of-order, poison message |
| `docs/phase-1/11-go-rust-responsibilities.md` | First Go worker purpose + optional Rust task (only if justified) |

Also draw (or describe in text):
- System context diagram
- Container / service boundary diagram

### What you do **not** do in Phase 1
- ❌ Create Laravel models for Video / UploadSession
- ❌ Install brokers or write worker code
- ❌ Start real S3 uploads

### Exit Gate
All Phase 1 docs exist and are reviewed. Boundaries and contracts are approved.

---

## PHASE 2 — Production Foundation
> **Goal:** Make staging real: database, Redis, S3, secrets, CI, observability.  
> **Infrastructure work. Still no video catalog features.**

### What you do

1. **PostgreSQL**
   - Local: Docker or local install
   - Staging: provision (RDS / equivalent)
   - Document connection vars in `.env.example`

2. **Redis**
   - Local + staging
   - Use for cache / locks / rate limits only

3. **Object storage (S3-compatible)**
   - Create buckets (or prefixes): `quarantine/`, `originals/`, `renditions/`, `thumbnails/`, `captions/`
   - Enable encryption
   - Private by default

4. **CDN prep**
   - Origin protection
   - Plan for signed URLs / cookies (no public direct S3 links)

5. **Environment config**
   - Document all env vars for local / staging / production
   - Keep secrets out of git

6. **Secrets**
   - Use a secret store or at least secure env injection
   - Never commit real keys

7. **Observability**
   - Structured logs
   - Request ID / correlation ID
   - Health + readiness endpoints

8. **Infrastructure as code** (Terraform / Pulumi / CloudFormation — pick one)
   - Networks, DB, Redis, buckets, IAM

9. **CI**
   - Run tests, static analysis, migration checks
   - Fail on broken tests or vulnerable dependencies

10. **Recovery**
    - Document backup + restore steps
    - Practice restore once on a test environment

### Files you will touch
- `.env.example`
- `config/database.php`, `config/filesystems.php`, `config/cache.php`
- Docker / compose files (if used)
- CI config (`.github/workflows/` or equivalent)
- `docs/phase-2/runbooks.md` (backup, restore, rollback)

### Exit Gate
Staging deploys from code, DB/Redis/S3 work, secrets are not in git, logs are visible, restore was tested, CI blocks broken changes.

---

## PHASE 3 — Identity, Channels, Video Catalog
> **Goal:** Real business models and APIs for channels and video **metadata** only.  
> **No binary upload yet.**

### What you do (concrete)

#### 3.1 Models & migrations
```bash
php artisan make:model Channel -m
php artisan make:model Video -m
php artisan make:model MediaAsset -m
# (and related models as needed)
```

Fill migrations with:
- Channel: `user_id`, public id / handle, display name, status, timestamps
- Video: `channel_id`, public id, title, description, visibility, status (lifecycle), timestamps
- MediaAsset: link to video, type (original/rendition/thumbnail/caption), storage key, size, checksum, status

#### 3.2 Relationships
- `User` hasMany `Channel`
- `Channel` belongsTo `User`, hasMany `Video`
- `Video` belongsTo `Channel`, hasMany `MediaAsset`

#### 3.3 Application services (or Actions)
Create under `app/Services/` or `app/Actions/`:
- `CreateChannel`
- `UpdateChannel`
- `CreateVideo`
- `UpdateVideoVisibility`
- `SoftDeleteVideo`

Controllers stay thin: validate → call service → return resource.

#### 3.4 Policies
- `ChannelPolicy`, `VideoPolicy`
- Only owner (or admin) can mutate

#### 3.5 Versioned API
```php
// routes/api_v1.php (or routes/api.php with prefix)
Route::prefix('v1')->group(function () {
    // channel + video routes
});
```

#### 3.6 Form Requests + API Resources
- `StoreChannelRequest`, `UpdateChannelRequest`
- `StoreVideoRequest`, `UpdateVideoRequest`
- `ChannelResource`, `VideoResource`

#### 3.7 Tests + factories
```bash
php artisan make:factory ChannelFactory
php artisan make:factory VideoFactory
php artisan make:test ChannelTest
php artisan make:test VideoTest
```

Cover: ownership, visibility, soft delete, forbidden access.

### What you do **not** do in Phase 3
- ❌ Presigned URLs
- ❌ S3 upload
- ❌ Transcoding
- ❌ Kafka / RabbitMQ / Go workers

### Exit Gate
A user can create/manage channel and video metadata through tested `/api/v1` endpoints. No file bytes involved.

---

## PHASE 4 — Upload & Asset Lifecycle
> **Goal:** Client uploads large files **directly to S3**. Laravel only manages sessions and verification.

### What you do (concrete)

#### 4.1 UploadSession model
```bash
php artisan make:model UploadSession -m
```

Fields: `video_id` (or `user_id` + `channel_id`), size, mime, checksum, status, expiry, region, idempotency_key, storage_key.

#### 4.2 Service methods
In `UploadSessionService` (or Actions):
- `createSession(...)` → returns presigned multipart URLs
- `completeSession(...)` → verifies object exists + size + checksum on S3, moves state to `uploaded` / `quarantined`
- `expireAbandonedSessions()` (scheduled)

#### 4.3 API endpoints
- `POST /api/v1/upload-sessions`
- `POST /api/v1/upload-sessions/{id}/complete`
- (optional resume / list parts)

#### 4.4 S3 integration
- Use Laravel filesystem disk (or AWS SDK) for presigned URLs
- Upload goes to **quarantine** prefix first
- Never stream the full video through PHP

#### 4.5 Cleanup
- Lifecycle rules on S3 for abandoned multipart uploads
- Artisan command or scheduler to expire old sessions

#### 4.6 Tests
- Happy path upload
- Resume / retry complete
- Expired URL
- Bad checksum
- Unauthorized user

### Exit Gate
A real client can upload a file to S3, resume, complete, and get a durable quarantined asset **without Laravel receiving the video bytes**.

---

## PHASE 5 — Processing, Events, Playback
> **Goal:** One full path: upload → quarantine → probe → process → ready → signed playback.

### What you do (in order)

#### 5.1 Outbox
- Table `outbox_messages`
- Write event in same DB transaction as state change
- Relay process publishes to Kafka / RabbitMQ

#### 5.2 First Go worker
- Small service that consumes RabbitMQ `media.probe` command
- Validates object, extracts metadata
- Calls Laravel internal API (or approved contract) to update status
- Publishes `MediaProbeCompleted` / `MediaProbeFailed`
- Retries + DLQ + metrics + health check

#### 5.3 Optional Rust worker
- Only if Phase 1 justified it (e.g. secure validation or thumbnails)
- Versioned contract
- Benchmark before keeping

#### 5.4 Transcoding
- Submit job to MediaConvert or container workers
- Produce HLS/DASH ladder
- Store manifests + renditions + thumbnails + captions in S3
- Record `ProcessingAttempt` for resume

#### 5.5 Kafka
- Lifecycle events after commit
- Read-model consumers for feeds / catalog projections
- Separate watch-event stream for analytics

#### 5.6 Playback
- Authorization check (visibility, moderation, ownership, not deleted)
- Issue short-lived signed CDN URL / cookie
- Serve HLS/DASH via CDN
- Test: private, blocked, revoked, missing rendition

### Exit Gate
This flow works end-to-end and survives duplicates, retries, and worker crashes:

```text
create channel → create video → create upload session → upload → complete
→ quarantine → probe → process → publish manifest → authorize playback
```

---

## PHASE 6 — Reliability, QA, Launch
> **Goal:** Prove it is safe and operable for real users.

### What you do
- Full QA matrix (business rules, infra, contracts, E2E, load, security, failure, data safety)
- Incident severity + escalation doc
- Service ownership list
- Runbooks: failed processing, DLQ replay, Kafka replay, takedown, restore
- Dashboards + alerts
- Canary / blue-green + feature flags
- Formal readiness review

### Exit Gate
Someone other than the original implementer can deploy and operate the VOD core.

---

## PHASE 7 — Product Expansion
> **After** production metrics exist.

Add only when the core is stable:
- Read projections (feeds, dashboards)
- Subscriptions / notifications
- Likes, comments, playlists, watch history (with abuse controls)
- Watch analytics
- Search
- Recommendations (last)

Each feature gets its own contracts, tests, dashboards, rollback plan.

---

## PHASE 8 — Global Scale
> Only when measured demand requires it.

- Multi-AZ
- Edge delivery
- Regional failover (start with one write region)
- Extract services only with evidence + owner
- Active/active writes only after conflict rules are designed

---

## Quick “What do I do right now?” answer

**Right now you are in Phase 0.**

1. Run:
   ```bash
   mkdir -p docs/phase-0
   ```
2. Create the 7 markdown files listed in Phase 0.
3. Fill the numbers and lists (scope, limits, SLOs, cloud, identity rules).
4. Approve them.
5. Only then start Phase 1 documents.

Do **not** create `Channel` / `Video` models yet. That starts in **Phase 3**.

---

## Phase order (never skip)

```
0  Requirements docs
1  Architecture + contracts docs
2  Foundation (DB, Redis, S3, CI, secrets)
3  Channel + Video metadata models & APIs
4  Upload sessions + direct-to-S3
5  Processing + events + playback
6  QA + launch
7  Extra product features
8  Global scale
```

---

*This file is the concrete “what do I create / run” version of the VidroTube plan.*
