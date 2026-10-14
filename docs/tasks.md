# 🎬 VidroTube — Tasks (start here, learn + code)

**Order is fixed.** Do not skip phases.  
**Scale is last.** QA + launch (Phase 6) comes **before** expansion (7) and global scale (8).

```
You are here → Phase 0
Then: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8
```

---

## Where you start

1. Open this file.
2. Do **Phase 0** only.
3. When every checkbox in Phase 0 is done → move to Phase 1.
4. Never create Video models or S3 upload code before Phase 3 / 4.

---

# PHASE 0 — Requirements (documents only)
> Learn: what the product is, limits, reliability.  
> Code: **none**.

### Tasks

- [ ] **0.1** Create folder  
  ```bash
  mkdir -p docs/phase-0
  ```

- [ ] **0.2** Create and fill `docs/phase-0/01-mvp-scope.md`  
  Write:
  - Creator actions (register, channel, upload, publish, analytics, delete)
  - Viewer actions (browse, play, quality, captions)
  - Admin actions (moderate, retry processing, audit)

- [ ] **0.3** Create and fill `docs/phase-0/02-out-of-scope.md`  
  List what V1 will **not** have: live, monetization, DRM, full copyright, comments, likes, playlists, advanced recommendations.

- [ ] **0.4** Create and fill `docs/phase-0/03-capacity-limits.md`  
  Write numbers (change later if needed):
  - Max file size (e.g. 5 GB)
  - Max duration (e.g. 2 hours)
  - Uploads per day (e.g. 1 000)
  - Concurrent viewers (e.g. 10 000)
  - Regions (e.g. 1 × EU)
  - Retention (originals / renditions)

- [ ] **0.5** Create and fill `docs/phase-0/04-slos.md`  
  Write targets:
  - API p95 latency
  - Playback start time
  - Upload success rate
  - Processing time for 1080p

- [ ] **0.6** Create and fill `docs/phase-0/05-recovery-compliance.md`  
  Write: RPO, RTO, data region, soft-delete then hard-delete rules, privacy/export.

- [ ] **0.7** Create and fill `docs/phase-0/06-cloud-choice.md`  
  Choose cloud (e.g. AWS) and map: PostgreSQL, Redis, S3, CDN, Kafka, RabbitMQ, workers.

- [ ] **0.8** Create and fill `docs/phase-0/07-identity-api-rules.md`  
  Write: public ID format (ULID), API version `/api/v1`, auth method, roles (user / creator / admin), service auth idea.

- [ ] **0.9** Read all 7 files once. Approve them (you or team).

**Exit gate:** All 7 files exist and you understand the product boundaries.  
→ Next: Phase 1

---

# PHASE 1 — Architecture & contracts (still mostly docs)
> Learn: domain model, lifecycle, who owns what, how services talk.  
> Code: almost none (no Video upload code).

### Tasks

- [ ] **1.1** Create folder  
  ```bash
  mkdir -p docs/phase-1
  ```

- [ ] **1.2** Write `docs/phase-1/01-domain-model.md`  
  List entities and links:
  ```
  User → owns → Channel → contains → Video
  Video → UploadSession, MediaAsset, Thumbnail, Caption, Rendition,
          ProcessingAttempt, ModerationCase, WatchEvent
  ```

- [ ] **1.3** Write `docs/phase-1/02-data-ownership.md`  
  For each entity: owner system (Laravel/PostgreSQL vs S3 vs worker).

- [ ] **1.4** Write `docs/phase-1/03-video-lifecycle.md`  
  States and allowed transitions:
  ```
  created → uploading → uploaded → quarantined → processing → ready
                                                      ↘ blocked / deleted
  ```

- [ ] **1.5** Write `docs/phase-1/04-upload-flow.md`  
  Steps: create session → presigned URL → client → S3 quarantine → complete → verify.

- [ ] **1.6** Write `docs/phase-1/05-processing-flow.md`  
  Steps: probe → validate → transcode → thumbnails/captions → ready.

- [ ] **1.7** Write `docs/phase-1/06-api-contracts.md`  
  Example JSON for: create channel, create video, create upload session, complete upload (all under `/api/v1`).

- [ ] **1.8** Write `docs/phase-1/07-event-envelope.md`  
  Common fields: id, type, version, occurred_at, correlation_id, payload.

- [ ] **1.9** Write `docs/phase-1/08-kafka-topics.md`  
  At least: video.lifecycle, media.probe.result, watch.events (purpose + producer + consumers).

- [ ] **1.10** Write `docs/phase-1/09-rabbitmq-queues.md`  
  At least: media.probe, media.transcode (schema + retry + DLQ idea).

- [ ] **1.11** Write `docs/phase-1/10-failure-behavior.md`  
  What happens on: duplicate message, worker crash, bad checksum, expired URL.

- [ ] **1.12** Write `docs/phase-1/11-go-rust-responsibilities.md`  
  First Go job = probe consumer. Rust = only if you have a clear reason (else skip).

**Exit gate:** You can explain lifecycle + ownership + upload path without looking at code.  
→ Next: Phase 2

---

# PHASE 2 — Foundation (infra, still no video features)
> Learn: env, DB, Redis, S3, secrets, CI.  
> Code: config + Docker/CI only.

### Tasks

- [ ] **2.1** Set local PostgreSQL (Docker or install). Update `.env` and `.env.example`.

- [ ] **2.2** Set local Redis. Confirm cache/queue config points to it.

- [ ] **2.3** Create S3-compatible bucket (or MinIO local). Prefixes: `quarantine/`, `originals/`, `renditions/`, `thumbnails/`, `captions/`. Private only.

- [ ] **2.4** Configure Laravel filesystem disk for that storage in `config/filesystems.php` + env vars.

- [ ] **2.5** Document all required env vars in `.env.example` (no real secrets).

- [ ] **2.6** Add health route (e.g. `/up` or `/health`) that checks DB (and later Redis).

- [ ] **2.7** Add request/correlation ID to logs (middleware).

- [ ] **2.8** Set up CI (GitHub Actions or similar): `composer test` / `php artisan test`, fail on red tests.

- [ ] **2.9** Write `docs/phase-2/backup-restore.md` (how to backup/restore PostgreSQL once).

- [ ] **2.10** Practice one restore on a throwaway DB.

**Exit gate:** App boots with PostgreSQL + Redis; S3 disk configured; CI runs tests; you can restore a DB.  
→ Next: Phase 3

---

# PHASE 3 — Channels + Video catalog (first real product code)
> Learn: models, migrations, policies, thin controllers, services, API resources, tests.  
> Code: Laravel only. **No file upload.**

### Tasks

- [ ] **3.1** Finish / fix `Channel` model + migration  
  Fields: `user_id`, public id or handle, name, status, timestamps.  
  Relation: `User` hasMany `Channel`, `Channel` belongsTo `User`.

- [ ] **3.2** Create `Video` model + migration  
  ```bash
  php artisan make:model Video -m
  ```  
  Fields: `channel_id`, public id, title, description, visibility, status, timestamps.

- [ ] **3.3** Create `MediaAsset` model + migration (metadata only, no bytes)  
  ```bash
  php artisan make:model MediaAsset -m
  ```  
  Fields: `video_id`, type, storage_key, size, checksum, status.

- [ ] **3.4** Add relationships and factories  
  ```bash
  php artisan make:factory ChannelFactory
  php artisan make:factory VideoFactory
  ```

- [ ] **3.5** Create application services (or Actions)  
  e.g. `app/Services/ChannelService.php`, `app/Services/VideoService.php`  
  Methods: create, update, change visibility, soft delete.

- [ ] **3.6** Create Form Requests  
  `StoreChannelRequest`, `UpdateChannelRequest`, `StoreVideoRequest`, `UpdateVideoRequest`.

- [ ] **3.7** Create API Resources  
  `ChannelResource`, `VideoResource` (hide internal fields).

- [ ] **3.8** Create Policies  
  `ChannelPolicy`, `VideoPolicy` (owner or admin only for write).

- [ ] **3.9** Versioned routes  
  Create `routes/api_v1.php` (or prefix group):  
  - channels CRUD  
  - videos CRUD (metadata only)

- [ ] **3.10** Thin controllers  
  Controllers only: validate → call service → return resource.

- [ ] **3.11** Feature tests  
  ```bash
  php artisan make:test ChannelApiTest
  php artisan make:test VideoApiTest
  ```  
  Cover: create, ownership, forbidden access, visibility, soft delete.

- [ ] **3.12** Run  
  ```bash
  php artisan migrate
  php artisan test
  ```

**Exit gate:** You can create a channel and a video record via `/api/v1` with tests green. No binary files.  
→ Next: Phase 4

---

# PHASE 4 — Upload (direct to S3)
> Learn: presigned URLs, upload sessions, quarantine, verification.  
> Code: Laravel + S3. Still no transcoding.

### Tasks

- [ ] **4.1** Create `UploadSession` model + migration  
  ```bash
  php artisan make:model UploadSession -m
  ```  
  Fields: video/channel/user link, size, mime, checksum, status, expiry, storage_key, idempotency_key.

- [ ] **4.2** Implement `UploadSessionService`  
  - `createSession()` → returns presigned upload URL(s)  
  - `completeSession()` → check object on S3 (exists, size, checksum) → mark uploaded/quarantined

- [ ] **4.3** API endpoints  
  - `POST /api/v1/upload-sessions`  
  - `POST /api/v1/upload-sessions/{id}/complete`

- [ ] **4.4** Put new objects under **quarantine/** prefix only.

- [ ] **4.5** Scheduled cleanup for abandoned sessions + document S3 lifecycle rules.

- [ ] **4.6** Tests: happy path, expired session, bad checksum, unauthorized, duplicate complete.

**Exit gate:** Client uploads a real file to S3 and completes the session; Laravel never receives the video body.  
→ Next: Phase 5

---

# PHASE 5 — Processing + events + playback
> Learn: outbox, queues, first worker, transcoding pipeline, signed playback.  
> Code: Laravel + first Go worker + S3 + CDN path.

### Tasks (do in order)

- [ ] **5.1** Outbox table + write event in same transaction as upload complete.

- [ ] **5.2** Relay that publishes outbox events to the broker.

- [ ] **5.3** First Go worker: consume `media.probe`, validate object, call Laravel status API, publish result. Retries + DLQ + health.

- [ ] **5.4** Transcoding path (MediaConvert or containers): produce HLS/DASH ladder, store renditions/thumbnails/captions, record `ProcessingAttempt`.

- [ ] **5.5** Kafka (or chosen bus) lifecycle events + simple read projection if needed.

- [ ] **5.6** Playback: authorize (visibility/moderation/owner) → signed CDN URL → HLS/DASH.

- [ ] **5.7** E2E test of full path including retry and duplicate message.

**Exit gate:**  
`channel → video → upload → quarantine → probe → process → ready → play` works and survives crashes/retries.  
→ Next: Phase 6

---

# PHASE 6 — QA + production launch
> Learn: testing strategy, runbooks, alerts, safe deploy.  
> **This comes before scale.**

### Tasks

- [ ] **6.1** Business-rule tests (lifecycle, policies, idempotency).

- [ ] **6.2** Integration tests (DB, Redis, S3, queue, worker).

- [ ] **6.3** Security tests (private media, signed URL expiry, unauthorized upload).

- [ ] **6.4** Failure tests (worker crash, poison message, bad file).

- [ ] **6.5** Write runbooks: failed job, DLQ replay, revoke access, restore DB.

- [ ] **6.6** Dashboards + alerts (errors, queue depth, processing failures, playback errors).

- [ ] **6.7** Deploy strategy: feature flags + canary or blue-green.

- [ ] **6.8** Readiness review checklist signed off.

**Exit gate:** Core VOD can be operated by someone else.  
→ Next: Phase 7 (only after real metrics)

---

# PHASE 7 — Product expansion (after launch)
> Only after Phase 6. Add value features one by one.

### Tasks (pick order as needed)

- [ ] **7.1** Read projections for feeds / channel pages.

- [ ] **7.2** Subscriptions + notifications.

- [ ] **7.3** Engagement (likes, comments, playlists) + abuse controls.

- [ ] **7.4** Watch analytics.

- [ ] **7.5** Search.

- [ ] **7.6** Recommendations (last; needs real watch data).

Each feature: own contracts, tests, dashboard, rollback plan.

---

# PHASE 8 — Global scale (last)
> Only when metrics show you need it. **After QA.**

### Tasks

- [ ] **8.1** Multi-AZ DB and brokers.

- [ ] **8.2** Edge / multi-region delivery.

- [ ] **8.3** Controlled regional failover (one write region first).

- [ ] **8.4** Extract a service only with measured reason + owner.

- [ ] **8.5** Active/active writes only after conflict rules are designed.

---

## Answer to “scale > QA?”

**No.**  
Correct order:

```
... → Phase 5 (full path works)
    → Phase 6 (QA + launch)     ← quality and ops first
    → Phase 7 (more features)
    → Phase 8 (global scale)    ← scale last
```

You do not scale a system that is not proven and operable.

---

## Your next action (today)

```bash
mkdir -p docs/phase-0
```

Then create and fill the 7 files in **Phase 0** (tasks 0.2–0.8).  
When they are done, come back and start Phase 1 the same way.

You learn by doing each phase’s tasks in order.  
You code starting at Phase 3 (catalog), then 4 (upload), then 5 (processing).
