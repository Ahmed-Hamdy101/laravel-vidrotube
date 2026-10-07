# 🎬 VidroTube — Implementation Plan
### Laravel 13 · PHP 8.3 · Livewire 4 · Fortify · (Go + Rust + Kafka + RabbitMQ — later, when earned)

---

> [!IMPORTANT]
> **You write every line of code.** This file is the design, the task list, and the "how" — never the implementation. `[ ]` = something you go do. I keep this file in sync with your actual repo as you go.

---

## 📦 Current Project Audit *(scanned from your repo, not guessed)*

| Layer | Status |
|---|---|
| Laravel 13 / PHP 8.3, Fortify, Livewire, Flux, Vite | ✅ Scaffolded, running |
| `.env` `APP_KEY` | ❌ Empty — nothing encrypts/sessions won't work until `key:generate` runs |
| `channels` migration | 🟡 Exists — **no columns**, just `id` + `timestamps` |
| `Channel` model | 🟡 Exists — empty, no `$fillable`, no relationships |
| `ChannelController` | 🟡 Exists — every method is an empty stub |
| `Video` / upload / processing / anything media | ❌ Not started |
| `routes/api_v1.php` | ❌ Doesn't exist — `bootstrap/app.php`'s `withRouting()` has no `api:` key at all |
| `bootstrap/app.php` | ✅ Laravel 13's streamlined structure — no `app/Exceptions/Handler.php` in this version, exceptions configured via `withExceptions()`, which already has `shouldRenderJsonWhen()` wired up |
| `Services/`, `Repositories/`, `Http/Requests/`, `Http/Resources/` | ❌ None of these folders exist yet |
| `DB_CONNECTION` | `sqlite` — fine to keep for now; `database/database.sqlite` doesn't exist yet |
| `REDIS_*`, `AWS_*` in `.env` | Configured but unused — `CACHE_STORE`/`QUEUE_CONNECTION` are still `database`, `FILESYSTEM_DISK` is still `local` |
| Tests (`tests/Feature`, `tests/Unit`) | 🟡 Default Laravel scaffolding only — nothing written |
| Test tooling | ✅ PHPUnit, Larastan (phpstan), Pint all installed — `composer test` runs all three |
| `docs/mvp-scope.md` | 🟡 Written, clean up copy-pasted instruction text mixed into the bullets |
| `docs/out-of-scope.md` | 🟡 Written, missing its own heading, has a stray instruction line |
| Object storage, CDN, RabbitMQ, Kafka, Go/Rust workers | ❌ Not started — correct, nothing here should exist yet |

---

## 🔴 Gap Inventory — what's actually missing right now

| # | Gap | Why it matters | Fixed in |
|---|---|---|---|
| 1 | No system design docs beyond SD-A (partial) | You've been about to write code with no domain model, no contracts, no lifecycle rules | **SD-C → SD-H** |
| 2 | `channels` table has no real columns | Can't store a channel's name/owner yet | **Build 2** |
| 3 | No `Video` model/migration at all | Nothing to upload against | **Build 2** |
| 4 | No API route file | No versioned contract seam | **Build 1** |
| 5 | No Service/Repository layers | Business logic has nowhere correct to live | **Build 2** |
| 6 | No Form Requests / Resources | No boundary validation, no response shaping | **Build 2** |
| 7 | Zero tests written | No safety net for anything you build next | **Every phase ships its own tests, none deferred** |
| 8 | No RBAC / identity rules written down | "Who can do what" isn't decided | **SD-B, Task 7** |
| 9 | `docs/` files have copy-paste artifacts | Fix now before it becomes a habit | **Step 0** |
| 10 | `APP_KEY` empty | Sessions/encryption broken until generated | **Build 1** |

---

## 🗺️ Target Architecture

| Component | Owns | Does NOT own |
|---|---|---|
| Laravel | Identity, channels, video metadata, auth, upload sessions | Video bytes, transcoding |
| PostgreSQL | Transactional state | Media files |
| Redis | Cache, locks, rate limits | Source-of-truth records |
| S3-compatible storage | Original files, renditions, thumbnails, captions | Authorization |
| RabbitMQ | Short-lived processing jobs, retries, DLQs | Long-term event history |
| Kafka | *(deferred to Build 6)* | — |
| Go worker | Orchestration, media probing, consumers | Business/catalog data |
| Rust | *(only if proven necessary)* | General APIs |

```
controller → service → repository/client      (backend)
```
Infrastructure never leaks upward.

---

## ✅ Step 0 — Clean up what you already wrote (10 min)
- `[ ]` `docs/mvp-scope.md` — remove duplicated/stray instruction text
- `[ ]` `docs/out-of-scope.md` — add `# Out of Scope` heading, remove stray line

---

## PART I — SYSTEM DESIGN 🎯 no code yet

### SD-A / SD-B — Product & Quality Requirements
- `[x]` `docs/mvp-scope.md` — done (clean up above)
- `[x]` `docs/out-of-scope.md` — done (clean up above)
- `[ ]` Capacity table, SLO table, recovery & privacy, hosting choices, identity & API rules — add as `##` sections in `docs/mvp-scope.md`

### SD-C — Domain & Data Design 🎯
- `[ ]` Entity tree: `User → Channel → Video → {UploadSession, MediaAsset, Thumbnail, Caption, Rendition, ProcessingAttempt, ModerationCase}`, `Video → WatchEvent, AuditRecord`
- `[ ]` One row per entity: fields, owner, required/optional
- `[ ]` Video lifecycle: `created → uploading → uploaded → quarantined → processing → ready → published/blocked/deleted`

### SD-D — User-Flow Design 🎯
- `[ ]` Sketch upload, processing, moderation, playback, deletion flows — each with a failure branch

### SD-E — Architecture Design 🎯
Captured above in **Target Architecture** — confirm or edit.

### SD-F — Contract Design 🎯
- `[ ]` Example request/response for each core endpoint
- `[ ]` Event envelope: `event_id`, `type`, `occurred_at`, `payload`, `trace_id`
- `[ ]` Delivery rule: consumer checks `event_id` against `processed_events` before acting

### SD-G — QA & Operations Design 🎯
- `[ ]` QA strategy (unit/integration/contract/E2E), rule: no `[x]` without a passing test
- `[ ]` Observability: every log line carries a `request_id`
- `[ ]` Runbook template

### SD-H — Approval Gate
- `[ ]` Read SD-A → SD-G once, out loud, fix anything you couldn't defend in an interview

---

## PART II — IMPLEMENTATION

## ✅ Build 1 — Production Foundation 🎯
**Goal:** boring plumbing. No features.

- `[ ]` `php artisan key:generate` — your `.env` has a blank `APP_KEY`, nothing encrypts/sessions won't work until this runs
- `[ ]` `database/database.sqlite` created + `php artisan migrate` run (SQLite is a fine, correct choice for now)
- `[ ]` `routes/api_v1.php` — created, registered in `bootstrap/app.php`'s `withRouting()` via an `api:` key, every future route lives under `/api/v1/`
- `[ ]` `bootstrap/app.php` → `withExceptions()` — add a `render()` case for one consistent JSON error shape (it already has `shouldRenderJsonWhen`, this adds the actual shape)
- `[ ]` Redis/S3 — no action yet, staying on `database` cache/queue and `local` disk until a later phase actually needs them

🧪 One smoke test: app boots, migrations run, `/up` (already registered) returns 200, and your new `/api/v1/...` test route responds.

---

## ✅ Build 2 — Identity, Channels, and Video Catalog 🎯
**Goal:** full CRUD core — no upload pipeline yet, just the records.

- `[ ]` Add real columns to `channels` migration: `name`, `slug`, `user_id`, `description`
- `[ ]` `Video` model + migration, matching your SD-C field table
- `[ ]` `app/Http/Requests/StoreChannelRequest.php`, `StoreVideoRequest.php`, `UpdateVideoRequest.php`
- `[ ]` `app/Services/ChannelService.php`, `VideoService.php`
- `[ ]` `app/Repositories/Contracts/VideoRepositoryInterface.php` + `app/Repositories/Eloquent/EloquentVideoRepository.php`
- `[ ]` Rewrite `ChannelController` to be thin; add `VideoController` the same way
- `[ ]` `app/Http/Resources/VideoResource.php`, `ChannelResource.php`
- `[ ]` Visibility toggle enforced in the service, not the controller

🧪 Test cases: `VideoService` rejects invalid visibility · repository test against real DB · contract test on `GET /api/v1/videos/{id}` · lifecycle test blocking `created → published`

**Verify with:** `composer test` (lint + phpstan + `php artisan test`)

---

## ✅ Build 3 — Upload and Asset Lifecycle 🎯
- `[ ]` `UploadSession` model + migration
- `[ ]` `app/Services/UploadService.php` — direct-to-storage upload URL, browser uploads straight to S3
- `[ ]` Resume logic, checksum validation, quarantine path
- `[ ]` `MediaAsset` record on success

🧪 Interrupted upload resumes correctly · duplicate completion doesn't double-create `MediaAsset` · oversized file rejected · checksum gates `uploaded` status

---

## ✅ Build 4 — Processing and Playback 🎯
- `[ ]` RabbitMQ queue `video.processing`
- `[ ]` First Go worker: consumes queue, probes file, reports back
- `[ ]` Transcoding → `Rendition`s via ffmpeg
- `[ ]` `Thumbnail`, `ProcessingAttempt` (retry limit, DLQ), `ModerationCase`
- `[ ]` Signed/authorized playback URLs

🧪 Crashed worker leaves video recoverable · duplicate message doesn't double-transcode · unauthorized viewer rejected · 20 concurrent uploads all processed

---

## ✅ Build 5 — Reliability, QA, and Production Launch 🎯
- `[ ]` Business-rule + infra tests against real services · cross-language contract test · full journey CI test
- `[ ]` Security pass, kill-the-worker recovery test
- `[ ]` Real runbooks, tested rollback plan

---

## ✅ Build 6 — Product Expansion and Read Scale 🎯
Kafka enters here — not before.
- `[ ]` Kafka domain events, read projections, subscriptions, engagement (rate-limited), search index
- `[ ]` Recommendations/monetization/live streaming → their own SD-A→H pass each

🧪 Abuse-control, privacy/retention, projection-replay tests

---

## ✅ Build 7 — Global Scale and Service Extraction 🎯 Staff / Hero
- `[ ]` Multi-AZ, replicated storage/CDN, proven regional failover
- `[ ]` Extract services only with a metric justifying it
- `[ ]` Active/active → conflict resolution designed first

---

## 📋 Naming Standard
| Area | Rule |
|---|---|
| Services | own verbs — `VideoService::publish()` |
| Booleans | `isProcessing`, `canPublish` |
| I/O honesty | `fetchVideo()` vs `getTitle()` |
| Interfaces | only with a real second implementation |
| Every `[x]` | has a passing test behind it |

## 🔁 Verification (run after every task)
```
composer lint         # Pint
composer types:check  # Larastan
php artisan test      # PHPUnit
composer test         # all three
```

## 📋 Task Ownership
| Task | Owner |
|---|---|
| All code, migrations, workers, infra | **You** |
| Design docs, decisions | **You write; I explain and review** |
| "Is this right?" sanity checks | **Ask anytime** |

## 🗓️ Priority Order
```
Step 0 → SD-A/B → SD-C..H → Build 1 → Build 2 → Build 3 → Build 4 → Build 5 → Build 6 → Build 7
```

---

*Generated from a live scan of your repo.*
