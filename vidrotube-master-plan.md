# 🎬 VidroTube — Master Plan: Level 0 → Staff Engineer
### Laravel 13 / PHP 8.3 core · Go + Rust workers · Kafka + RabbitMQ · You write every line

---

> [!IMPORTANT]
> **The rule for this whole plan:** you write all the code. I write the design, the concrete task list, and the "why" — never the implementation. Every `[ ]` below is something *you* go do.

---

## 🧭 How to read this

- `[ ]` — a concrete task. It names a real file, model, service, queue, or document — never "figure out X" left vague.
- `💡` — an explanation callout. Only appears where a term or decision isn't self-explanatory. No callout = the task is plain enough to just do.
- `🧪` — QA for that specific phase, sitting right next to the work it tests. Nothing is "we'll test it later" — that's exactly what the ERP plan you showed me does right, and what your original VidroTube doc skipped.
- `🎯 Level N` — the skill level this phase is training. Levels stack; you don't skip one.
- Each phase has a **Goal**, a **Duration** (realistic for solo, learning-as-you-go pace — treat these as floors, not deadlines), and an **Exit gate** — the bar that must be true before the next phase starts.

---

## 🎓 The Level Map

| Level | What it proves about you | Phases |
|---|---|---|
| **0 — Junior** | You can define a product in writing before touching code | SD-A, SD-B *(done in your [Beginner Build Roadmap](#) — Tasks 1–7)* |
| **1 — Mid** | You can design a domain model, contracts, and flows *before* coding, and ship a safe, working monolith | SD-C → SD-H, Build 1, Build 2 |
| **2 — Senior** | You can handle async processing, failure modes, and put a real system into production with QA, monitoring, and a rollback plan | Build 3, Build 4, Build 5 |
| **3 — Staff / Hero** | You can grow a live system deliberately — extend it without breaking the core, then scale it only when evidence demands it, not by guessing | Build 6, Build 7 |

This is also, almost line for line, what a staff-engineer interview loop tests: product scoping, domain modeling, failure-mode reasoning, and "when do you *not* add complexity." That's why this plan is shaped like this, not by accident.

---

## 📦 Current Starting Point

| Layer | Status |
|---|---|
| Laravel 13, PHP 8.3, Fortify, Livewire, Flux, Vite | ✅ Scaffolded |
| Channel model + migration | 🟡 Incomplete |
| Video catalog, upload pipeline | ❌ Not started |
| Object storage, CDN | ❌ Not started |
| Kafka, RabbitMQ | ❌ Not started |
| Go / Rust services | ❌ Not started |
| Deployment infra | ❌ Not started |
| System design docs (SD-A/B) | 🟡 In progress — the roadmap I gave you last turn |

---

## PART I — SYSTEM DESIGN 🎯 Level 0 → 1

> No code in this whole part. If you catch yourself opening a terminal to `php artisan make:model`, you've jumped ahead — come back.

### SD-A / SD-B — Product & Quality Requirements
Already broken down task-by-task in the roadmap from last message — nothing new here, just finish those 7 tasks if you haven't:
- `[ ]` `docs/mvp-scope.md` (creator/viewer action lists)
- `[ ]` `docs/out-of-scope.md`
- `[ ]` Capacity table
- `[ ]` SLO table
- `[ ]` `docs/recovery-and-privacy.md`
- `[ ]` Hosting table
- `[ ]` Identity & API rules doc

**Exit gate:** all seven files exist and you could defend every line in an interview.

---

### SD-C — Domain & Data Design 🎯 Level 0→1

💡 A domain model is just: *what are the "things" in this app, and who's allowed to change them?* You're about to design the shape of your database before writing a single migration.

- `[ ]` Draw the entity tree in `docs/domain-model.md` (plain text art is fine, Excalidraw if you prefer):
  ```
  User → owns → Channel → contains → Video
                                        ├─ UploadSession
                                        ├─ MediaAsset (original file)
                                        ├─ Thumbnail
                                        ├─ Caption
                                        ├─ Rendition (360p/720p/1080p)
                                        ├─ ProcessingAttempt
                                        ├─ ModerationCase
                                        └─ WatchEvent (analytics)
  Video → produces → AuditRecord (any important change)
  ```
- `[ ]` For each entity, write one row in a table: **fields**, **who owns it** (Laravel/Postgres vs. a worker vs. object storage), **required vs. optional**.
  💡 "Owner" matters because two systems must never both be allowed to change the same field — that's how data gets silently corrupted in real systems.
- `[ ]` Define the **Video lifecycle** — every status a video can be in and which transitions are legal:
  ```
  created → uploading → uploaded → quarantined → processing → ready → published
                                                              ↘ blocked / deleted
  ```
- `[ ]` For each arrow above, write: what triggers it, what can make it fail, what the user sees on failure.

🧪 **QA for this phase:** write a plain-language test list — "a video in `processing` must never be playable," "a `blocked` video can't move straight to `published`." You'll turn these into real automated tests in Build 2, but the *rule* gets written now, before code exists to contradict it.

**Done when:** you can point at any two connected entities and say, in one sentence, who owns the relationship and what happens if it's deleted.

---

### SD-D — User-Flow Design 🎯 Level 1

💡 A "flow" is just the step-by-step path one action takes through your system, including the ways it can go wrong — not just the happy path.

- `[ ]` `docs/flows/upload.md` — creator picks file → Laravel issues an upload session → browser uploads straight to storage → Laravel confirms completion → what happens if the connection drops mid-upload
- `[ ]` `docs/flows/processing.md` — uploaded file → validation → transcode → thumbnail/caption generation → ready
- `[ ]` `docs/flows/moderation.md` — ready → review → published/blocked, plus who can re-review a blocked video
- `[ ]` `docs/flows/playback.md` — viewer opens video → authorization check → CDN delivers the right rendition
- `[ ]` `docs/flows/deletion.md` — creator deletes → what happens to storage, playback access, and analytics already recorded

For each flow, sketch it as boxes and arrows (pen and paper, a phone photo pasted into the doc — doesn't need to be pretty).

**Done when:** every flow has at least one *failure* branch drawn, not just the success path.

---

### SD-E — Architecture Design 🎯 Level 1

💡 This is where you decide **who does what job** — before you decide *how*. Get this table right and Kafka/Go/Rust stop being buzzwords and become tools with a specific, narrow reason to exist.

- `[ ]` `docs/architecture.md` — one row per component:

| Component | Owns | Does NOT own |
|---|---|---|
| Laravel | Identity, channels, video metadata, auth, upload sessions | Video bytes, transcoding |
| PostgreSQL | Transactional state | Media files |
| Redis | Cache, locks, rate limits | Source-of-truth records |
| S3-compatible storage | Original files, renditions, thumbnails, captions | Authorization |
| RabbitMQ | Short-lived processing jobs, retries, DLQs | Long-term event history |
| Kafka | *(deferred — see note)* | — |
| Go worker | Orchestration, media probing, consumers | Business/catalog data |
| Rust worker | *(only if a CPU-heavy step actually needs it)* | General APIs |
| CDN | Playback delivery | Original processing |

  💡 **Kafka note:** at your Task-3 scale (tens to low-thousands of uploads/day), RabbitMQ alone covers processing jobs. Kafka earns its place in Build 6 when you build read projections/analytics from event history — introducing it now would be solving a problem you don't have yet. Write that reasoning down; it's a real architectural decision, not a shortcut.

- `[ ]` Draw the system context diagram: browser → Laravel API → {Postgres, Redis, S3, CDN}. No Go/Rust/RabbitMQ boxes yet — they appear in Build 3–4 when there's real async work to hand them.

**Done when:** every technology in your stack has exactly one row, and you could explain to someone why it's *not* doing the neighboring job.

---

### SD-F — Contract Design 🎯 Level 1

💡 A "contract" is just: the exact shape of a request/response or event, written down *before* two pieces of code have to agree on it by accident.

- `[ ]` `docs/contracts/api.md` — for each of these, write an example request and response body (as a description of the fields, not code): create channel, create video, start upload session, complete upload, get video, list public videos
- `[ ]` `docs/contracts/event-envelope.md` — define the common wrapper every async event uses: `event_id`, `type`, `occurred_at`, `payload`, `trace_id`
- `[ ]` `docs/contracts/delivery.md` — write the rule: *"a consumer checks `event_id` against a `processed_events` table before acting, so a message delivered twice only does its work once."*
  💡 This is called an **inbox pattern** — it's how real distributed systems survive "at least once" delivery, which is the normal (not exceptional) behavior of queues.

**Done when:** every API/event mentioned in SD-D's flows has a matching entry here.

---

### SD-G — QA & Operations Design 🎯 Level 1→2

**This is the section your ERP-style plan had and your VidroTube doc was missing until now — QA and ops decided *before* code, not bolted on at the end.**

- `[ ]` `docs/qa-strategy.md` — decide, in writing, what gets which kind of test:
  - **Unit tests** → Services (business logic), pure and fast
  - **Integration tests** → Repositories + real DB
  - **Contract tests** → API responses match SD-F examples exactly
  - **End-to-end test** → one full upload→process→publish→watch happy path
- `[ ]` Pick your tools: PestPHP or PHPUnit for backend; Go's built-in `testing` package for workers later
- `[ ]` Write the rule you'll hold yourself to for every Build phase from here on: **a `[ ]` isn't `[x]` until it has a passing test.** (Borrow this straight from your own ERP plan — it's the single biggest reason that plan feels trustworthy.)
- `[ ]` `docs/observability.md` — decide from day one: every log line includes a `request_id`; every error gets logged with enough context to reproduce it without guessing
- `[ ]` Write one runbook template you'll fill in per feature going forward: *"If X breaks → check Y → do Z."* Empty now, real entries start in Build 3.

**Done when:** you could open this doc mid-incident (even a fake one) and know what kind of test should have caught it.

---

### SD-H — Approval Gate 🎯 Level 1

- `[ ]` Read SD-A → SD-G start to finish, out loud, in one sitting
- `[ ]` For every decision, ask: *"could I defend this to a senior engineer interviewing me?"* If no, fix it now — it's still free to change on paper.

**Exit gate:** yes to the question above, for everything. Once you check this, you've done what most bootcamp grads never do — and what most senior interviews actually probe for.

---

## PART II — IMPLEMENTATION 🎯 Level 1 → Staff

> Numbering note: these are **Build 1–7**, not "Implementation Phase 1–8" like the original doc — SD-C through SD-H above already absorbed what used to be a separate "Implementation Phase 1," so nothing is duplicated.

---

## ✅ Build 1 — Production Foundation 🎯 Level 1
**Goal:** the boring plumbing every later phase needs. No features yet.
**Duration:** 1–2 weeks

- `[ ]` PostgreSQL running locally (Docker) + one staging instance; migrations run clean
- `[ ]` Redis running locally; confirm Laravel cache/queue driver connects
- `[ ]` S3-compatible bucket created (Cloudflare R2/Backblaze B2, per your SD-B hosting choice); confirm Laravel can read/write a test file
- `[ ]` `routes/api_v1.php` created from day one — every route lives under `/api/v1/`
  💡 versioning from the start means you can introduce `/v2` later without breaking anything that already depends on `/v1`
- `[ ]` `Handler.php` — one consistent JSON error shape for every API error, decided now per SD-G
- `[ ]` CI pipeline (GitHub Actions is fine): runs `php artisan test` + a lint check on every push

🧪 **QA:** one smoke test — app boots, migrations run, a health-check endpoint returns 200. Trivial, but it's the first `[x]` earned under your new "no green box without a test" rule.

**Exit gate:** a fresh clone of the repo, on a fresh machine, can run migrations and pass the smoke test with nothing but the README.

---

## ✅ Build 2 — Identity, Channels, and Video Catalog 🎯 Level 1→2
**Goal:** the full monolith CRUD core — no upload pipeline yet, just the business records.
**Duration:** 2–3 weeks

### Backend
- `[ ]` `User`, `Channel`, `Video` models + migrations, matching SD-C's field table exactly
- `[ ]` `Http/Requests/` — `StoreChannelRequest`, `StoreVideoRequest`, `UpdateVideoRequest` (boundary validation, per SD-G)
- `[ ]` `Services/ChannelService.php`, `Services/VideoService.php` — all business logic lives here, not in controllers
- `[ ]` `Repositories/Contracts/VideoRepositoryInterface.php` + `Repositories/Eloquent/EloquentVideoRepository.php`
  💡 an interface here earns its place because you'll swap in a test double for it in your unit tests — that's the real second implementation, not abstraction for its own sake
- `[ ]` `Http/Controllers/API/ChannelController.php`, `VideoController.php` — thin: parse request → call one service → shape response
- `[ ]` `Http/Resources/VideoResource.php`, `ChannelResource.php` — shape the wire response, hide internal fields
- `[ ]` Video visibility toggle (public/unlisted/private) enforced in the service layer, not the controller

🧪 **QA:**
- `[ ]` Unit tests: `VideoService` — creating a video with invalid visibility is rejected
- `[ ]` Integration tests: `EloquentVideoRepository` against a real test DB
- `[ ]` Contract tests: `GET /api/v1/videos/{id}` response matches your SD-F example field-for-field
- `[ ]` Lifecycle test: a video cannot jump from `created` straight to `published` (this is the SD-C rule, now enforced in code)

**Exit gate:** you can register, create a channel, create a video record, and list public videos — through the API, with tests proving it, and nothing yet involves an actual video file.

---

## ✅ Build 3 — Upload and Asset Lifecycle 🎯 Level 2
**Goal:** a creator can actually get a video file into storage, safely, resumably.
**Duration:** 3–4 weeks — this is the first genuinely hard phase.

- `[ ]` `UploadSession` model + migration — tracks who's allowed to upload what, and whether it completed
- `[ ]` `Services/UploadService.php` — issues a direct-to-storage upload URL (creator's browser uploads straight to S3, never through PHP — per SD-E)
  💡 routing large files through PHP causes timeouts and memory pressure; direct-to-storage upload is the standard fix
- `[ ]` Resume logic: an interrupted upload can be resumed without starting over
- `[ ]` Checksum validation on completion — reject corrupt/partial files
- `[ ]` Quarantine step: file lands in a `quarantine/` bucket path until validated, then moves to its real location
- `[ ]` `MediaAsset` record created on successful upload, linked to the `Video`

🧪 **QA:**
- `[ ]` Test: an upload interrupted at 50% can be resumed and completes correctly
- `[ ]` Test: completing an upload twice (duplicate request) doesn't create two `MediaAsset` records — your first real use of the SD-F inbox/idempotency rule
- `[ ]` Test: an oversized file (beyond your SD-B capacity limit) is rejected with a clear error, not a silent failure
- `[ ]` Test: a video record cannot move to `uploaded` status until its checksum passes

**Exit gate:** you can upload a real video file end to end, kill your wifi mid-upload, resume it, and watch it land correctly in storage — repeatably, not "it worked once."

---

## ✅ Build 4 — Processing and Playback 🎯 Level 2
**Goal:** an uploaded file becomes an actually-watchable video. This is where Go and RabbitMQ finally earn their place.
**Duration:** 4–6 weeks — the centerpiece of the whole project.

- `[ ]` RabbitMQ queue: `video.processing` — Laravel publishes a command when a `MediaAsset` finishes upload
- `[ ]` First Go worker: consumes `video.processing`, probes the file (duration, codec, resolution), reports back via a small internal API or a result queue
  💡 this is the "choose the first Go responsibility" decision from SD-E, now made concrete — Go because you need a fast, concurrent worker, not because it's fashionable
- `[ ]` Transcoding step: generate at least two `Rendition`s (e.g. 480p, 720p) — start with a straightforward tool (ffmpeg via the Go worker) before reaching for MediaConvert-style managed services
- `[ ]` Thumbnail generation → `Thumbnail` record
- `[ ]` `ProcessingAttempt` record per try, with a retry limit and a dead-letter path (RabbitMQ DLQ) for permanent failures
- `[ ]` `ModerationCase` created automatically when processing succeeds; a video can't reach `published` without one being resolved
- `[ ]` Playback: signed/authorized URLs so private/unlisted videos can't be watched by guessing a link
- `[ ]` Basic HLS or simple quality-selector playback on the frontend

🧪 **QA:**
- `[ ]` Test: a crashed worker mid-transcode leaves the video in a recoverable state, not stuck forever
- `[ ]` Test: the same processing message delivered twice doesn't double-transcode (inbox pattern again — you should notice this pattern repeating; that repetition *is* the lesson)
- `[ ]` Test: a private video's playback URL rejects an unauthorized viewer
- `[ ]` Load test (even a crude one): queue 20 uploads at once, confirm none silently get dropped

**Exit gate:** upload → process → moderate → publish → play, fully working end to end, with a Go worker doing real work and RabbitMQ actually earning the complexity it adds.

---

## ✅ Build 5 — Reliability, QA, and Production Launch 🎯 Level 2→3
**Goal:** prove this is a real, operable system — not a demo that only works on your laptop.
**Duration:** 3–4 weeks

### QA tracks
- `[ ]` Business-rule tests: every lifecycle transition from SD-C, every policy from SD-F
- `[ ]` Infrastructure tests: Postgres, Redis, S3, RabbitMQ, Go worker — each adapter tested against the real thing, not just mocks
- `[ ]` Cross-language contract test: Laravel's published message matches exactly what the Go worker expects
- `[ ]` Full journey test: upload → play, automated, run in CI
- `[ ]` Security pass: private media can't be accessed without auth; secrets aren't in the repo; basic injection/abuse checks on upload endpoints
- `[ ]` Failure-recovery test: kill the Go worker mid-job, confirm RabbitMQ redelivers and nothing corrupts

### Launch tasks
- `[ ]` Define your own incident response: what "down" means, what you do first, in what order
- `[ ]` Write real runbooks (using the SD-G template): failed processing, DLQ replay, access revocation, restore-from-backup
- `[ ]` Dashboards/alerts: API errors, queue depth, processing failures — even a simple log-based dashboard counts at this scale
- `[ ]` Deploy with a rollback plan you've actually tested once (not just written down)

**Exit gate:** someone other than you could deploy, break, and recover this system using only your runbooks. That sentence is the actual definition of "production-ready," at any company size.

---

## ✅ Build 6 — Product Expansion and Read Scale 🎯 Level 3
**Goal:** add real user-facing value without destabilizing the core you just hardened. **Kafka enters here — not before.**
**Duration:** 4–6 weeks

- `[ ]` Introduce Kafka: domain events (`video.published`, `video.watched`, etc.) published to durable topics
  💡 the reason it's here and not in Build 4: you now have real production data and a real reason to replay/project it — introducing Kafka in Build 4 would've been solving an imaginary problem
- `[ ]` Read projections built from Kafka events: channel pages, video feeds, creator dashboards — so these don't hammer the transactional Postgres tables
- `[ ]` Subscriptions + notifications
- `[ ]` Engagement: likes, comments, watch history — with basic abuse controls (rate limits, spam checks) from day one, not bolted on later
- `[ ]` Watch analytics with a privacy-aware retention policy (tie back to your SD-B recovery/privacy doc)
- `[ ]` Search — a dedicated index built from approved catalog events, not ad-hoc queries against Postgres
- `[ ]` Note (don't build): recommendations, monetization, live streaming, copyright fingerprinting each get their **own** SD-A→H design pass. Bolting them onto this plan would be exactly the scope creep SD-A's "out of scope" list exists to prevent.

🧪 **QA:** abuse-control tests for engagement features (can't spam-comment, can't fake-watch for analytics), privacy tests for retention rules, and a replay test — can you rebuild a projection from Kafka history if it gets corrupted?

**Exit gate:** every new feature here has its own owner (you), its own test coverage, and its own rollback plan — none of them silently expanded the core VOD system's scope.

---

## ✅ Build 7 — Global Scale and Service Extraction 🎯 Level 3 (Staff / Hero)
**Goal:** scale based on *evidence*, not on what sounds impressive on a résumé.
**Duration:** open-ended — this is the phase you grow into over months, not weeks.

- `[ ]` Multi-AZ database/broker setup
- `[ ]` Replicated object storage + CDN edge delivery
- `[ ]` Controlled regional failover — proven with an actual drill, not just documented
- `[ ]` Multi-region read models for feeds/search/analytics
- `[ ]` Extract media orchestration to a standalone Go service **only when you can point at a real metric** that justifies it (deploy cadence, scaling mismatch with Laravel — not "microservices are cooler")
- `[ ]` Extract further services (analytics, search) by the same evidence rule
- `[ ]` If you ever go active/active across regions: design conflict resolution and data ownership explicitly before enabling it — this is where silent data corruption happens in real companies

🧪 **QA:** load, soak, chaos, failover, and replay tests — this phase basically *is* a QA phase with infrastructure attached.

**Exit gate:** you can explain, with numbers, why each extracted service earned its complexity — and you'd say the same thing in a staff-engineer interview that you wrote here.

---

## 🏛️ Cross-Cutting Standards (apply in every Build phase)

| Area | Rule |
|---|---|
| **Naming** | Services own verbs (`VideoService::publish()`), not vague nouns (`VideoManager::handle()`) |
| **Booleans** | `isProcessing`, `canPublish` — never bare `status`/`flag` |
| **I/O honesty** | `fetchVideo()` (network call) vs `getTitle()` (cheap, in-memory) — name implies cost |
| **Interfaces** | Only where there's a real second implementation (a test double counts) — not by default |
| **No task is `[x]`** | ...without a passing test, per SD-G. This one rule does more for a portfolio project's credibility than any framework choice. |

---

## 📋 Task Ownership

| Task type | Owner |
|---|---|
| All code — models, services, workers, migrations, everything | **You** |
| All infrastructure setup — Docker, CI, deploys | **You** |
| Design docs, decisions, explanations | **You write them; I explain concepts and review your reasoning** |
| "Is this task actually done / does this decision hold up" sanity checks | **Ask me anytime — I won't write it, but I'll tell you straight if it's wrong** |

---

## 🗓️ Phase Priority Order

```
SD-A/B (done in roadmap) → SD-C..H (system design)
        ↓
Build 1 (foundation) → Build 2 (catalog) → Build 3 (upload)
        ↓
Build 4 (processing/playback) → Build 5 (QA + launch)
        ↓
Build 6 (expansion, Kafka) → Build 7 (global scale)
```

No phase bleeds into the next. Each exit gate is a real bar, not a formality — that discipline *is* the staff-engineer skill this whole plan is training.

---

*Companion doc: the Beginner Build Roadmap (SD-A/B, task-by-task). This plan starts where that one leaves off.*
