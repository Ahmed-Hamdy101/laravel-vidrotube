# VidroTube — Staff Engineer Plan
### Same style as advanced-plan: one step = one action
Each step: **Action · Explain · Tip · You should not · System design**

```text
TOOLS → DESIGN A→H → BUILD 1→9 → PERF P1→P7 → STAFF HABITS
```

Finish left before right. This path trains staff habits: design, ownership, measure, operate.

---

# HOW YOU IMPLEMENT SYSTEM DESIGN
### Connect something → something → something

System design is not only boxes on a whiteboard.  
You implement it by **wiring real connections**. Every arrow below is something you build.

---

## 1) Big picture (what talks to what)

```text
Browser / Mobile
       │
       │  HTTPS  /api/v1/...
       ▼
┌─────────────────┐
│  Laravel API    │──────► PostgreSQL     (users, channels, videos, sessions, status)
│  (identity +    │
│   catalog +     │──────► Redis          (cache, locks, rate limits)
│   authz)        │
└────────┬────────┘
         │
         │  presigned URL (upload)     signed URL (play)
         ▼                             ▲
┌─────────────────┐                    │
│  S3 / Object     │────────────────────┘
│  Storage        │
│  quarantine/    │
│  originals/     │
│  renditions/    │──────► CDN ──────► Player
└────────┬────────┘
         │
         │  events / jobs
         ▼
┌─────────────────┐
│  Queue / Worker │──────► Laravel status API (never raw DB from worker)
│  (probe,        │
│   transcode)    │
└─────────────────┘
```

**Read it as connections you implement:**

```text
Client  →  Laravel
Laravel  →  PostgreSQL
Laravel  →  Redis
Laravel  →  S3          (presign + verify)
Client   →  S3          (direct upload bytes)
Laravel  →  Queue       (after complete)
Worker   →  S3          (read object, write renditions)
Worker   →  Laravel     (update status via API)
Laravel  →  CDN sign    (playback)
Player   →  CDN         (HLS/DASH bytes)
```

---

## 2) Upload path (implement this chain)

```text
User
  → POST /api/v1/videos                         (create Video status=created)
  → POST /api/v1/upload-sessions                (Laravel creates UploadSession)
  → Laravel returns presigned URL
  → Browser PUT/POST file bytes  →  S3 quarantine/
  → POST /api/v1/upload-sessions/{id}/complete
  → Laravel checks object on S3
  → Video status = uploaded / quarantined
  → Outbox row written (same DB transaction)
  → Relay publishes job/event
  → Worker probes file on S3
  → Worker calls Laravel “status=processing/ready”
  → Renditions written to S3 renditions/
  → Video status = ready
```

**What you connect in code:**

| From | To | How you connect it |
|------|----|--------------------|
| Controller | Service | method call |
| Service | Eloquent / DB | repository or model |
| Service | S3 | filesystem disk / AWS SDK presign |
| Browser | S3 | presigned URL (no Laravel in the middle of bytes) |
| Service | Outbox table | same DB transaction as status update |
| Outbox | Queue | relay command / scheduler |
| Worker | S3 | read/write objects |
| Worker | Laravel | HTTP internal API or signed command |

---

## 3) Play path (implement this chain)

```text
Viewer
  → GET /api/v1/videos/{id}/play
  → Laravel loads Video from PostgreSQL
  → Laravel checks status + visibility + owner
  → if denied → 403
  → if ok → create short-lived signed CDN/S3 URL
  → Player requests manifest/segments from CDN
  → CDN fetches from S3 origin if needed
  → Viewer watches
```

**Connections:**

```text
Player → Laravel (authz only)
Laravel → PostgreSQL (status/visibility)
Laravel → Signer (CDN or S3 temporary URL)
Player → CDN → S3 (bytes only)
```

Laravel never streams the whole video file to the viewer.

---

## 4) Domain connections (data ownership)

```text
User
  owns → Channel
           owns → Video
                    has → UploadSession
                    has → MediaAsset (original / rendition / thumb / caption)
                    has → ProcessingAttempt
                    has → ModerationCase
```

**Who is allowed to write what:**

```text
Laravel/PostgreSQL  →  User, Channel, Video.status, UploadSession, ModerationCase
S3                  →  file bytes only
Worker              →  may create rendition files on S3
Worker              →  may request status change via Laravel API only
Redis               →  cache copies, not source of truth
CDN                 →  delivery, not authorization truth
```

---

## 5) Lifecycle connection (status machine you enforce)

```text
created
  → uploading          (session created, presign issued)
  → uploaded           (complete verified object exists)
  → quarantined        (isolated until probe)
  → processing         (worker accepted job)
  → ready              (renditions available)
  → blocked / deleted  (admin or owner)
```

**Connection rule:**  
Only Laravel applies these transitions (service method).  
Worker sends “probe ok” / “transcode ok”; Laravel sets `ready`.

---

## 6) Event / job connection (async)

```text
DB transaction:
  Video.status = quarantined
  + Outbox(event = upload.completed)

Outbox relay
  → Queue message media.probe

Worker
  → read S3 object
  → POST Laravel /internal/videos/{id}/probe-result

Laravel
  → status = processing
  → enqueue media.transcode

Worker
  → write renditions to S3
  → POST Laravel /internal/videos/{id}/process-result

Laravel
  → status = ready
  → optional Kafka/event for analytics/projections
```

**Implement order of connections:**

```text
1. Laravel ↔ PostgreSQL
2. Laravel ↔ S3 (presign + head object)
3. Client ↔ S3 (bytes)
4. Laravel ↔ Outbox table
5. Outbox ↔ Queue
6. Worker ↔ Queue
7. Worker ↔ S3
8. Worker ↔ Laravel API
9. Laravel ↔ CDN signer
10. Player ↔ CDN
```

Do not connect 8 before 1–3 work.

---

## 7) Cache / performance connections

```text
Read-heavy path:

Client → Laravel → (Redis hit?) 
                      yes → return cached channel/video metadata
                      no  → PostgreSQL → fill Redis → return

Write path:

Client → Laravel → PostgreSQL (source of truth)
                 → invalidate Redis key
```

```text
List endpoints:

Client → Laravel → PostgreSQL with eager load (with('channel'))
                 → optional cache page fragment
```

```text
Scale path:

DNS/CDN → Load Balancer → Laravel instance × N
                              ↓
                         PostgreSQL (shared)
                         Redis (shared)
                         S3 (shared)
Workers × M (separate from web)
```

---

## 8) How design steps become these connections

```text
A Product requirements
  → tells you which arrows must exist (upload, play, moderate)

B Quality requirements
  → tells you which arrows must be fast/reliable (play < 2s → CDN)

C Domain
  → tells you which DB tables and who owns each field

D Flows
  → is exactly the upload path and play path sequences above

E Architecture
  → is the big picture boxes and allowed arrows

F Contracts
  → is the JSON on each HTTP arrow and event on each queue arrow

G QA/Ops
  → tests each arrow failure (expired presign, worker down, 403 play)

H Approval
  → you agree these connections are the ones you will build

BUILD 1–9
  → you implement the arrows one by one in order

PERF
  → you strengthen Laravel→DB, Laravel→Redis, Client→CDN arrows
```

---

## 9) One rule for every connection

```text
Ask: Who owns the truth on this arrow?

Client → Laravel     truth starts after validation
Laravel → PostgreSQL truth for catalog/status
Laravel → S3         permission + verify, not long-term truth of status
Client → S3          bytes only
Worker → Laravel     request, not silent DB write
Laravel → CDN        short-lived access, not permanent public
Redis ← Laravel      copy of truth, can be deleted
```

If two boxes both think they own the same truth → redesign before coding.

---

# STEP 0 — Tools (what you use to design)


### Action
Install / bookmark only these:

1. **Excalidraw** (https://excalidraw.com) — boxes and arrows
2. **Markdown** folder `docs/` in the repo — requirements and ADRs
3. **Mermaid live** (https://mermaid.live) — optional sequence diagrams
4. **Swagger Editor** (https://editor.swagger.io) — when you write OpenAPI
5. Later: `EXPLAIN ANALYZE` in PostgreSQL, Chrome Lighthouse, `k6` or `hey` for load

Create:
```bash
mkdir -p docs/adr docs/D-flows docs/runbooks
```

### Explain
Staff engineers leave artifacts. A diagram and a short decision file beat a long chat history. You design with tools so the team (or future you) can read the system without reading your mind.

### Tip
First diagram: maximum 5 boxes. If you need 20, your boundaries are wrong.

### You should not
- Spend days choosing tools.
- Design 12 microservices in Figma before Channel works in the DB.
- Keep the only design in your head.

### System design
```text
Tools  →  produce A–H artifacts  →  unlock BUILD
```

---

# DESIGN STEP A — Product requirements

### Action
Create `docs/A-product-requirements.md`.

Write four lists only:

1. **Creator can** — register, channel, upload, publish, analytics, delete, …
2. **Viewer can** — browse, open page, play, quality, captions, …
3. **Admin can** — moderate, retry processing, audit, …
4. **Out of scope V1** — live, monetization, DRM, full copyright, comments, likes, …

### Explain
Product requirements answer: **what must the system do?**
Every later diagram and table exists to support this list. If the list is fuzzy, the architecture is decoration.

### Tip
One plain sentence per capability. No Kafka, no Redis, no “scalable”. Example: “Creator uploads a recorded video directly to storage.”

### You should not
- Hide live streaming inside “upload”.
- Write a novel. One clear page is enough for V1.

### System design
```text
A Product requirements  →  drives B C D E F G H
```

---

# DESIGN STEP B — Quality requirements

### Action
Create `docs/B-quality-requirements.md`.

Write numbers (you can revise later):

- Max file size / max duration
- Uploads per day (target)
- Concurrent viewers (target)
- API p95 latency
- Playback start time (e.g. 95% under 2 seconds)
- Upload success rate
- RPO / RTO
- Region(s)
- Rough monthly budget ceiling

### Explain
Quality requirements answer: **how well must it work?**
“Fast” is not testable. “p95 play start < 2s” is. These numbers constrain cache, CDN, DB, and worker design.

### Tip
Pick numbers you can measure with logs, CDN metrics, or `k6`. Wrong numbers can change; missing numbers cannot be tested.

### You should not
- Copy Google-scale targets for a first launch.
- Skip RPO/RTO (they force backup and recovery design).

### System design
```text
A → B Quality  →  constrains architecture and PERF steps
```

---

# DESIGN STEP C — Domain and data design

### Action
Create `docs/C-domain-model.md`.

Write this tree and fill key fields + owner for each:

```text
User
  └── owns → Channel
               └── contains → Video
                               ├── UploadSession
                               ├── MediaAsset
                               ├── Thumbnail / Caption / Rendition
                               ├── ProcessingAttempt
                               └── ModerationCase
```

For each entity: main fields, **owner system** (Laravel/PostgreSQL vs S3 vs worker), delete rule.

### Explain
Domain design answers: **what information exists and who is allowed to change it?**
Two systems must not both own the same field (classic staff-level bug source).

### Tip
Bytes → object storage. Status `ready` → Laravel catalog. Worker may *request* a transition via API/event, not by writing Laravel tables.

### You should not
- Put video files in PostgreSQL.
- Invent 15 services before entities are clear.

### System design
```text
A → B → C Domain  →  tables, models, ownership matrix
```

---

# DESIGN STEP D — User-flow design

### Action
Open Excalidraw. Draw one happy path and one failure path for:

1. Create channel
2. Create video → upload session → upload → complete
3. Process → ready
4. Viewer plays
5. Admin blocks

Export or screenshot into `docs/D-flows/`.

Label every arrow with an API name or event name.

### Explain
Flows answer: **in what order do things happen, and what fails?**
Sequence thinking catches “who waits for whom” before you write code.

### Tip
If you cannot name an arrow, the contract (Step F) is missing.

### You should not
- Draw only the happy path.
- Use a magic box (“system does AI”) with no owner.

### System design
```text
A → B → C → D Flows  →  become API + event lists in F
```

---

# DESIGN STEP E — Architecture design

### Action
In Excalidraw, one context diagram:

```text
Client → Laravel API → PostgreSQL
              ↓
            Redis
              ↓
         S3 (quarantine / renditions)
              ↓
            CDN
              ↓
         Worker / queue   [later]
```

Under the diagram write 5 lines: what Laravel owns, what S3 owns, what workers own, what CDN owns, what Redis is for (cache/locks only).

### Explain
Architecture answers: **which part does which job?**
Staff default is clear boundaries, not maximum services.

### Tip
Modular monolith (Laravel) + S3 + one worker path until a **measured** bottleneck forces extraction.

### You should not
- Start with Kafka + Go + Rust + mesh on day one.
- Let workers UPDATE Laravel tables with raw SQL.

### System design
```text
A → B → C → D → E Architecture  →  deployment and ownership
```

---

# DESIGN STEP F — Contract design

### Action
Create:

- `docs/F-api-v1.md` — example JSON for create channel, create video, create upload session, complete upload, play
- `docs/F-events.md` — event envelope: id, type, version, occurred_at, correlation_id, payload + 3 examples

Optional: small OpenAPI stub in Swagger Editor.

### Explain
Contracts answer: **exactly what is exchanged between client, API, and workers?**
Field names and error shapes are product surface.

### Tip
Version everything: `/api/v1`, `event_version: 1`. Breaking change = new version.

### You should not
- Different JSON shapes per controller mood.
- Laravel serialized jobs as the contract to a Go service.

### System design
```text
… → F Contracts  →  clients and workers depend on this, not on DB schema
```

---

# DESIGN STEP G — QA and operations design

### Action
Create `docs/G-qa-ops.md` with:

1. Test list: ownership, visibility, expired upload, bad checksum, forbidden play, lifecycle transition
2. Alerts list: upload fail rate, processing stuck, 5xx, queue depth
3. One runbook file: `docs/runbooks/stuck-processing.md` (3–7 steps a human can follow)

### Explain
Ops design answers: **how do we know it is broken and who fixes it?**
If only you can operate it, delivery is not staff-level.

### Tip
Every critical path needs at least one automated test and one runbook line.

### You should not
- “Monitoring later.”
- Retries without thinking about duplicates.

### System design
```text
… → G QA/Ops  →  required before calling the system “done”
```

---

# DESIGN STEP H — Approval gate

### Action
Check every box before heavy build past catalog:

- [ ] A–G exist (short is fine)
- [ ] Out of scope is explicit
- [ ] Lifecycle states written
- [ ] Upload will not go through PHP body
- [ ] Video status owner = Laravel
- [ ] You know how you will test upload failure

### Explain
The gate stops you from building the wrong system quickly.

### Tip
Solo project: still check the boxes. Excitement is not approval.

### You should not
- Jump to workers/Kafka before H.

### System design
```text
A → B → C → D → E → F → G → H  →  BUILD starts
```

---

# BUILD STEP 1 — Channel

### Action
1. Open `app/Models/Channel.php`
2. Fillable: `user_id`, `name`, `handle`, `description`, `status`
3. Relations: Channel belongsTo User; User hasMany Channel
4. Migration: FK `user_id`, unique `handle`, timestamps
5. `php artisan migrate`
6. Tinker: create a channel under a user

### Explain
Channel is the creator space. Ownership of the whole catalog starts here.

### Tip
Stable unique `handle` for URLs; add public ULID later if handles must rename.

### You should not
- Channel without `user_id`.
- Video binary fields on Channel.

### System design
Entity from **C**. Root of ownership matrix.
```text
H → Build 1 Channel
```

---

# BUILD STEP 2 — Video catalog

### Action
```bash
php artisan make:model Video -m
```
Migration: `channel_id`, `public_id` unique, `title`, `description`, `visibility`, `status` default `created`, soft deletes.
Relations: Video belongsTo Channel; Channel hasMany Video.
Migrate. Tinker create video with ULID `public_id`.

### Explain
Video is metadata + lifecycle. The file is MediaAsset later. Mixing them causes timeouts and broken states.

### Tip
Only allow known status values when you harden validation.

### You should not
- Store mp4 bytes in `videos`.
- Skip `channel_id` and hang Video only on User.

### System design
Lifecycle from **C/D**:
`created → uploading → uploaded → quarantined → processing → ready`
Status owner = Laravel.

---

# BUILD STEP 3 — Versioned API

### Action
```bash
php artisan make:controller Api/ChannelController
php artisan make:controller Api/VideoController
```
Routes: `auth` + prefix `v1` + `apiResource` for channels and videos.
`store` validates → creates under current user → returns JSON 201.
Test with Postman/curl.

### Explain
`/api/v1` is the contract seam from **F**. Clients depend on this shape.

### Tip
Thin controller from day one. Call a service method as soon as a second caller appears.

### You should not
- Unversioned public API forever.
- Fat controllers with all business rules.

### System design
Inbound delivery only (**E** Laravel box).
```text
1 → 2 → 3 API
```

---

# BUILD STEP 4 — UploadSession

### Action
```bash
php artisan make:model UploadSession -m
```
Fields: `video_id`, `user_id`, `status`, `storage_key`, `size`, `mime`, `checksum`, `expires_at`, `idempotency_key`.
Controller: `store` + `complete`.
`store` returns session id + fake upload URL first.
`complete` marks session completed and video `uploaded`.

### Explain
UploadSession is a short-lived **permission** to upload, not the bytes. Supports expiry, resume, and verification from flow **D**.

### Tip
Always set `idempotency_key`. Same key → same session on retry.

### You should not
- Accept full-length video body in a Laravel controller.
- Trust “client says complete” without checking storage later.

### System design
```text
create session → presigned URL → client → storage → complete → verify
```
Laravel owns permission; storage owns bytes.

---

# BUILD STEP 5 — S3 quarantine

### Action
Configure S3 disk + env.
`store` builds key `quarantine/{user}/{video}/{ulid}.mp4` and real presigned URL.
`complete` heads object, checks size, then status completed / quarantined.
Upload one small real file; confirm object in quarantine prefix.

### Explain
Quarantine isolates untrusted files until validation. Nothing quarantined is playable (**B** security + **E** storage).

### Tip
Prefixes: `quarantine/`, `originals/`, `renditions/`, `thumbnails/`, `captions/`.

### You should not
- Public bucket.
- Stream full video through PHP “temporarily”.

### System design
S3 owns bytes. Laravel owns authz and metadata. CDN later owns delivery.

---

# BUILD STEP 6 — Async process (simple first)

### Action
After complete → video `processing`.
```bash
php artisan make:command ProcessVideoCommand
```
Command finds `uploaded`/`processing`, simulates work, sets `ready`.
Run: `php artisan videos:process`.
Later: queue job → then external worker.

### Explain
Processing is async so upload HTTP stays inside latency targets from **B**.

### Tip
When you leave the fake command, add `processing_attempts` (step, error, times).

### You should not
- Transcode inside the web request.
- Workers writing Laravel tables directly.

### System design
Worker box in **E**. Status transitions still owned by Laravel via API/command.

---

# BUILD STEP 7 — Playback

### Action
`GET /api/v1/videos/{video}/play`
Checks: status `ready` + (public OR owner OR allowed private).
Deny blocked/deleted/processing.
Return short-lived signed URL.

### Explain
Play = authorize, then temporary access. Matches flow **D** and security **B**.

### Tip
Put CDN in front as soon as you leave local dev.

### You should not
- Permanent public URLs for private videos.
- Skip status check because “file exists on S3”.

### System design
Authorize in Laravel → signed CDN → segments. Originals stay private.

---

# BUILD STEP 8 — Harden structure

### Action (one at a time)
1. `app/Services/ChannelService.php`, `VideoService.php`, `UploadSessionService.php` — controllers call services
2. Form Requests for store/update
3. API Resources (hide internal fields)
4. Policies for Channel and Video
5. Feature tests: ownership, expiry, bad complete, forbidden play
```bash
php artisan test
```

### Explain
One place for rules so HTTP, jobs, and future workers stay consistent (**G**).

### Tip
Extract a service the moment a second entry point needs the same logic.

### You should not
- 200-line controllers.
- Ship complete/upload without failure tests.

### System design
```text
Controller → Service → DB / S3 client
```

---

# BUILD STEP 9 — Real pipeline

### Action (only after 1–8 work with a real file)
1. Outbox table + write event in same transaction as complete
2. Relay publishes events
3. Probe job/worker validates object, reports status via Laravel API
4. Transcode → renditions under `renditions/`
5. MediaAsset rows for derivatives
6. Signed CDN manifest for play

### Explain
Distributed delivery is at-least-once. Idempotency and DLQ are part of the product (**F** + **G**).

### Tip
Laravel queue first; extract Go when the message contract is stable.

### You should not
- Kafka + Go + Rust before one boring working path.
- Two systems both owning `status`.

### System design
```text
Client → Laravel → PostgreSQL + Outbox
            ↓
     Queue / Kafka
            ↓
     Worker → S3 → CDN
```

---

# PERF STEP P1 — Big O on your endpoints

### Action
For `GET` list endpoints:

1. Write “what happens with n videos?”
2. Count queries (Debugbar or `DB::listen`)
3. If queries grow with n → fix in P2

### Explain
Big O is not only interviews. N+1 queries → N+1 latency and cost under load. Staff engineers ask “what at 10× data?”

### Tip
Fix query count before micro-optimizing PHP syntax.

### You should not
- Optimize CPU while still doing N+1.

### System design
Supports latency numbers from **B**.

---

# PERF STEP P2 — Eager loading

### Action
Replace:
```php
Video::all(); // then $v->channel in a loop
```
With:
```php
Video::with('channel')->paginate(20);
```
Apply on every nested API response.

### Explain
Eager loading turns 1+N queries into a small fixed number.

### Tip
Only `with()` relations you actually return. Over-fetch wastes memory.

### You should not
- `Model::all()` on production catalogs.

### System design
Read-path cost is part of architecture quality.

---

# PERF STEP P3 — Indexes and EXPLAIN

### Action
```sql
EXPLAIN ANALYZE SELECT * FROM videos WHERE channel_id = 1 AND status = 'ready';
```
Add indexes that match real WHERE/ORDER:
```php
$table->index(['channel_id', 'status']);
```

### Explain
Without indexes, scans grow with table size — practical bad Big O.

### Tip
Index what you filter on. Drop unused indexes; they slow writes.

### You should not
- Index every column “just in case”.

### System design
Access patterns from **C** drive indexes.

---

# PERF STEP P4 — Cache (Redis)

### Action
Redis up. Cache hot public reads:
```php
Cache::remember("channel:{$id}", 60, fn () => Channel::with(...)->findOrFail($id));
```
Invalidate on update/delete.

### Explain
Cache is for read-heavy, rarely changing data. Truth stays in PostgreSQL.

### Tip
Cache public channel/video metadata first. Be careful caching authz decisions.

### You should not
- Shared cache key for private user-specific data.
- Forget invalidation.

### System design
Redis in **E** = cache/locks only, not source of truth.

---

# PERF STEP P5 — Front-end performance

### Action
On watch page:

1. Lighthouse (LCP, CLS, TTFB)
2. Lazy-load player until play
3. Correct thumbnail sizes
4. Cache headers on static assets
5. Avoid huge JS for a title + play button

### Explain
User-visible latency is part of **B**. Staff own the path to first play, not only JSON.

### Tip
Test once with Network throttling (Fast 3G).

### You should not
- Multi-MB JS for the watch page by default.

### System design
Client + CDN are part of play flow **D/E**.

---

# PERF STEP P6 — Practical algorithms / aggregates

### Action
For “top videos” or counts later:

1. Precompute counters with a job; do not `COUNT(*)` huge tables per request
2. Serve feeds from a projection/table built offline
3. Do not load whole tables into PHP to sort

### Explain
Algorithm choice = where you pay CPU and how often. Prefer indexed reads and precomputed aggregates online.

### Tip
Heavy work offline → simple read online (projection pattern).

### You should not
- Sort 1M rows in PHP on each request.

### System design
Read models match event consumers in full **E**.

---

# PERF STEP P7 — Load balance and scale out

### Action (when app is ready)
1. Auth stateless enough (token or Redis sessions)
2. Multiple Laravel instances behind a load balancer
3. Shared PostgreSQL + Redis + S3
4. Scale workers separately from web
5. CDN for media so app LB does not serve video bytes

### Explain
Load balancers spread **stateless** HTTP. Local disk uploads and sticky-only design fight scale.

### Tip
Scale workers (transcode) before mindlessly scaling web JSON nodes.

### You should not
- Store uploads on one app server’s disk.
- 10 app nodes before fixing N+1 and missing indexes.

### System design
```text
CDN → LB → Laravel × N → PostgreSQL
               ↘ Redis
Workers × M → S3
```

---

# STAFF STEP S1 — ADRs

### Action
When you decide something non-obvious, add:
```text
docs/adr/001-presigned-s3-uploads.md
```
Sections: Context / Decision / Consequences.

### Explain
Architecture Decision Records make tradeoffs visible. Staff leave reversible, documented choices.

### Tip
One ADR per important choice (presigned upload, status owner, queue first vs Go first).

### You should not
- Only tribal knowledge in chat.

### System design
Links **E/F** decisions to the repo forever.

---

# STAFF STEP S2 — Runbooks and metrics

### Action
1. Keep `docs/runbooks/stuck-processing.md` updated when behavior changes
2. Track at least: upload success rate, processing failures, API 5xx, p95 latency
3. After each build step, add one line to `docs/risks.md` if needed

### Explain
Operability is part of delivery. A system only you can fix is not staff-owned.

### Tip
Write the runbook the same day you implement the path—while you still remember it.

### You should not
- Ship production paths with zero “what if it sticks?” note.

### System design
Closes **G** with real practice.

---

# STAFF STEP S3 — Teach the model

### Action
Once a week, explain in 2 minutes without notes:

- Channel vs Video vs MediaAsset
- Why upload is not through PHP
- Who owns `status`
- What happens when processing fails

### Explain
If you can teach the design, you own it. Staff engineers align others with clear models.

### Tip
Use the Excalidraw from Step E as the only slide.

### You should not
- Hide complexity behind jargon when a simple diagram works.

### System design
Reinforces **C/D/E** until they are muscle memory.

---

# What you do RIGHT NOW

### Action
1. `mkdir -p docs/adr docs/D-flows docs/runbooks`
2. Create `docs/A-product-requirements.md`
3. Fill creator / viewer / admin / out of scope
4. Open Excalidraw → draw Client → Laravel → PostgreSQL → S3

### Explain
You start in design so build steps have a target. That is the staff order.

### Tip
A and a 5-box diagram today beat coding the wrong upload path for a week.

### You should not
- Open `make:model` for workers or Kafka today.

### System design
```text
Step 0 Tools → A → B → C → D → E → F → G → H
    → Build 1…9 → Perf P1…P7 → Staff S1…S3
```

---

# Full arrow map

```text
0 Tools
  → A Product requirements
    → B Quality requirements
      → C Domain & data
        → D User flows
          → E Architecture
            → F Contracts
              → G QA & ops
                → H Approval gate
                  → 1 Channel
                    → 2 Video
                      → 3 API v1
                        → 4 UploadSession
                          → 5 S3 quarantine
                            → 6 Async process
                              → 7 Playback
                                → 8 Services / tests
                                  → 9 Real pipeline
                                    → P1 Big O
                                      → P2 Eager load
                                        → P3 Indexes
                                          → P4 Cache
                                            → P5 Front-end
                                              → P6 Aggregates
                                                → P7 Load balance
                                                  → S1 ADRs
                                                    → S2 Runbooks & metrics
                                                      → S3 Teach the model
```

One step. Check it. Next step.
That is how this project makes you practice staff engineer work.
```
