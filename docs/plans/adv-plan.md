# VidroTube — Advanced Plan (action + system design)

Same style as the dumb plan: **one step → one action**.  
Now each step also has: **Explain · Tip · You should not · System design**.

Do the steps in order. Do not skip.

---

# START

```bash
cd your-laravel-project
```

You already have Laravel 13 + Fortify. That is the catalog and identity system.  
Workers, S3, and brokers come later. Laravel never becomes the place that streams video bytes.

---

# STEP 1 — Make Channel real (domain foundation)

### Action

1. Open `app/Models/Channel.php`
2. Set fillable:
```php
protected $fillable = [
    'user_id',
    'name',
    'handle',
    'description',
    'status',
];
```
3. Relations:
```php
// Channel.php
public function user()
{
    return $this->belongsTo(User::class);
}

// User.php
public function channels()
{
    return $this->hasMany(Channel::class);
}
```
4. Migration columns: `id`, `user_id` (FK), `name`, `handle` (unique), `description` (nullable), `status` (default `active`), timestamps.
5. Run:
```bash
php artisan migrate
```
6. Prove it:
```bash
php artisan tinker
```
```php
$user = User::first();
$channel = $user->channels()->create([
    'name' => 'My Channel',
    'handle' => 'mychannel',
    'status' => 'active',
]);
```

### Explain
A **Channel** is the creator’s public space. It is not the video file. It belongs to one User and will contain many Videos. Ownership starts here so every later permission check has a clear root.

### Tip
Use a stable unique `handle` (like a slug) for public URLs. Prefer a public id (ULID) later if you need to rename handles without breaking links.

### You should not
- Put video binary fields on Channel.
- Allow a Channel without `user_id`.
- Skip the foreign key. Orphan channels break authorization forever.

### System design
**Entity:** User owns Channel.  
**Owner system:** Laravel + PostgreSQL.  
**Boundary:** Channel is catalog metadata, not media storage.

---

# STEP 2 — Make Video model (catalog, not the file)

### Action

```bash
php artisan make:model Video -m
```

Migration:
```php
$table->id();
$table->foreignId('channel_id')->constrained()->cascadeOnDelete();
$table->string('public_id')->unique();
$table->string('title');
$table->text('description')->nullable();
$table->string('visibility')->default('private'); // public | unlisted | private
$table->string('status')->default('created');
// created → uploading → uploaded → quarantined → processing → ready
// also: blocked, deleted
$table->timestamps();
$table->softDeletes();
```

Model:
```php
protected $fillable = [
    'channel_id', 'public_id', 'title', 'description', 'visibility', 'status',
];

public function channel()
{
    return $this->belongsTo(Channel::class);
}
```

In Channel:
```php
public function videos()
{
    return $this->hasMany(Video::class);
}
```

```bash
php artisan migrate
```

Tinker test: create a video under a channel with `public_id` = ULID, `status` = `created`.

### Explain
**Video** is the business record (title, owner, visibility, lifecycle). The binary file is a **MediaAsset** later. Mixing them causes timeouts, huge DB rows, and broken lifecycle.

### Tip
Generate `public_id` with ULID/UUID at create time. Never expose auto-increment ids in public URLs.

### You should not
- Store the file path as the only source of truth without a status field.
- Let status be free text with no agreed transitions.
- Make Video belong to User directly and skip Channel (you lose the creator space model).

### System design
**Lifecycle (must enforce later):**
```
created → uploading → uploaded → quarantined → processing → ready
                                                    ↘ blocked / deleted
```
**Owner:** Laravel owns status transitions. Workers may request transitions via API/events, never by writing Laravel tables directly.

---

# STEP 3 — Versioned API for Channel + Video

### Action

```bash
php artisan make:controller Api/ChannelController
php artisan make:controller Api/VideoController
```

`routes/api.php`:
```php
use App\Http\Controllers\Api\ChannelController;
use App\Http\Controllers\Api\VideoController;

Route::middleware('auth:sanctum')->prefix('v1')->group(function () {
    Route::apiResource('channels', ChannelController::class);
    Route::apiResource('videos', VideoController::class);
});
```

Controller rule (example `store`):
```php
public function store(Request $request)
{
    $data = $request->validate([
        'name' => 'required|string|max:255',
        'handle' => 'required|string|max:50|unique:channels,handle',
    ]);

    $channel = $request->user()->channels()->create($data);

    return response()->json($channel, 201);
}
```

Test with Postman/curl: create channel, list channels, create video, list videos.

### Explain
`/api/v1` is the **contract seam**. Clients depend on this shape. Versioning lets you change v2 later without breaking v1.

### Tip
Keep controllers thin from day one. Even if logic is small, call a service method as soon as a second caller appears (job, command, worker).

### You should not
- Put business rules only in the controller (next week a job will need the same rule).
- Return full Eloquent models with hidden/internal fields forever — introduce API Resources when the wire shape differs.
- Skip auth middleware on write endpoints.

### System design
**Inbound delivery only:** Controller = parse → authorize → call one service → shape response.  
**Identity:** Fortify/Sanctum for users. Service-to-service auth comes later for workers.

---

# STEP 4 — UploadSession (permission to upload, not the bytes)

### Action

```bash
php artisan make:model UploadSession -m
```

Migration:
```php
$table->id();
$table->foreignId('video_id')->constrained()->cascadeOnDelete();
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
$table->string('status')->default('pending'); // pending, uploading, completed, expired, failed
$table->string('storage_key')->nullable();
$table->unsignedBigInteger('size')->nullable();
$table->string('mime')->nullable();
$table->string('checksum')->nullable();
$table->timestamp('expires_at')->nullable();
$table->string('idempotency_key')->nullable()->unique();
$table->timestamps();
```

```bash
php artisan make:controller Api/UploadSessionController
```

Routes (inside `v1` group):
```php
Route::post('upload-sessions', [UploadSessionController::class, 'store']);
Route::post('upload-sessions/{uploadSession}/complete', [UploadSessionController::class, 'complete']);
```

**store:** create session, set short `expires_at`, return session id + (for now) fake upload URL.  
**complete:** mark session `completed`, set video status `uploaded`.

### Explain
An **UploadSession** is a short-lived permission record: who may upload what, how big, until when. The browser uploads **directly to object storage**. Laravel never receives the video body. That is how you avoid PHP timeouts and memory blowups.

### Tip
Always set `idempotency_key` on create. If the client retries the same key, return the same session instead of creating a second one.

### You should not
- Accept multipart file uploads into a Laravel controller for full-length videos.
- Trust the client’s “I finished uploading” without verifying the object exists.
- Leave sessions forever — expire them and clean storage.

### System design
**Flow:** create session → presigned URL → client → S3 quarantine → complete → server verifies size/checksum/ownership → status uploaded/quarantined.  
**Owner of permission:** Laravel. **Owner of bytes:** object storage.

---

# STEP 5 — Real S3 (quarantine first)

### Action

```bash
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

`.env`: AWS keys, region, bucket.  
`config/filesystems.php`: `s3` disk.

In `store()`:
- Build key: `quarantine/{user_id}/{video_id}/{ulid}.mp4`
- Create real presigned/temporary upload URL
- Save `storage_key` on the session
- Set video status `uploading`

In `complete()`:
- Head/get object on S3
- Verify exists + size (+ checksum if you collect it)
- Session → `completed`
- Video → `uploaded` (then quickly `quarantined` if you add that state)

Upload a small real file. Confirm object appears under `quarantine/`.

### Explain
**Quarantine** means untrusted files are isolated until validation. Nothing in quarantine is playable. Playback paths only point at approved renditions later.

### Tip
Use separate prefixes: `quarantine/`, `originals/`, `renditions/`, `thumbnails/`, `captions/`. Lifecycle rules can expire quarantine junk automatically.

### You should not
- Make the bucket public.
- Serve quarantine objects to viewers.
- Stream the whole file through PHP “just for now”.

### System design
**Object storage owns bytes.** Laravel owns metadata and authorization. CDN will own delivery later. Never put authorization decisions inside the bucket policy alone — check visibility/status in the app first, then issue a short-lived signed URL.

---

# STEP 6 — Processing path (start dumb, design for workers)

### Action

After successful complete, set video status to `processing`.

```bash
php artisan make:command ProcessVideoCommand
```

Command (learning version):
- Select videos in `uploaded` or `processing`
- Simulate work (sleep / log)
- Set status `ready`

```bash
php artisan videos:process
```

Later replace with: queue job → RabbitMQ consumer → Go worker.

### Explain
Processing is **asynchronous**. Upload HTTP request must not wait for transcoding. A command or worker picks up work, records attempts, and moves the lifecycle forward.

### Tip
Add a `processing_attempts` table early when you leave the fake command: attempt number, step, error, started_at, finished_at. Retries resume from a known step instead of restarting everything.

### You should not
- Transcode inside the web request.
- Let workers UPDATE Laravel tables with raw SQL across services.
- Skip failure states — failed processing must be visible and retryable.

### System design
**Preferred path later:**
1. Upload complete writes DB state + **outbox** event in one transaction  
2. Relay publishes to Kafka/RabbitMQ  
3. Go worker consumes `media.probe` / `media.transcode`  
4. Worker calls versioned Laravel API (or command contract) to update status  
5. Publish result events for projections  

**Rule:** workers never own catalog truth. Laravel does.

---

# STEP 7 — Playback authorization + URL

### Action

Route:
```php
Route::get('videos/{video}/play', [VideoController::class, 'play']);
```

`play()`:
1. Load video
2. Allow only if `status === ready` and (visibility public OR owner OR authorized private)
3. Deny blocked / deleted / processing
4. Return temporary signed URL to the playable object (or manifest path later)

Test: owner can play; stranger cannot play private; blocked returns 403.

### Explain
Playback is not “here is the S3 link”. It is **authorize, then issue short-lived access**. Signed URLs/cookies expire so links cannot be shared forever.

### Tip
Put CDN in front as soon as you leave local dev. App issues signed access; CDN delivers bytes. Laravel stays out of the hot path.

### You should not
- Return permanent public S3 URLs for private videos.
- Skip status checks because “the file exists”.
- Embed long-lived secrets in the player.

### System design
**Authorize in Laravel → signed CDN access → HLS/DASH segments.**  
Originals stay private. Viewers receive renditions only.

---

# STEP 8 — Harden structure (services, boundaries, tests)

### Action (one at a time)

1. **Services**
```text
app/Services/ChannelService.php
app/Services/VideoService.php
app/Services/UploadSessionService.php
```
Controllers call services. Commands/jobs call the same services.

2. **Form Requests**
```bash
php artisan make:request StoreChannelRequest
php artisan make:request StoreVideoRequest
php artisan make:request StoreUploadSessionRequest
```

3. **API Resources**
```bash
php artisan make:resource ChannelResource
php artisan make:resource VideoResource
```
Hide internal fields (cost, internal notes, raw storage keys if not needed).

4. **Policies**
```bash
php artisan make:policy ChannelPolicy --model=Channel
php artisan make:policy VideoPolicy --model=Video
```

5. **Tests**
```bash
php artisan make:test ChannelApiTest
php artisan make:test VideoApiTest
php artisan make:test UploadSessionTest
php artisan test
```

### Explain
This is the layer map:
```
Controller → Service → Model/Storage client
```
Same business rules for HTTP, jobs, and future workers that call your API.

### Tip
Add a service the moment a second entry point needs the same behavior. Do not wait for a “big refactor week”.

### You should not
- Grow 200-line controller methods.
- Duplicate ownership checks in controller, service, and policy differently.
- Ship upload/complete without tests for expiry, bad checksum, and unauthorized access.

### System design
**Dependency direction:** UI/API → application service → infrastructure (DB, S3).  
Infrastructure never leaks upward (no raw AWS types in controllers).

---

# STEP 9 — Real async processing (system design becomes code)

Do these **only after** Steps 1–8 work end-to-end with a real file.

### Action order

1. **Outbox table**  
   Save event in the same DB transaction as upload complete.

2. **Relay**  
   Publish outbox rows to the broker with retries.

3. **First worker (Go or Laravel job first)**  
   Consume `media.probe` → validate object → report status via Laravel API.

4. **Transcoding**  
   MediaConvert or FFmpeg workers → HLS/DASH ladder → store under `renditions/`.

5. **Thumbnails / captions**  
   Derived assets as MediaAsset rows.

6. **Kafka (or your event bus) for lifecycle**  
   Projections and analytics consume events; they do not read Laravel tables.

7. **Signed CDN playback**  
   Manifest + segments; revoke by status change + short TTL.

### Explain
Distributed systems deliver messages **at least once**. Design for duplicates with idempotency keys and inbox records. The failure path (retry, DLQ, partial processing) is part of the product.

### Tip
Start with **Laravel queue + one job** if Go is new to you. Extract the same contract to Go later. Contracts first, language second.

### You should not
- Start with Kafka + Go + Rust + CDN on day one.
- Let two systems both “own” video status.
- Ignore poison messages (they need a DLQ and an operator path).

### System design (target)
```
Client → Laravel API → PostgreSQL + Outbox
                ↓
         RabbitMQ commands / Kafka events
                ↓
         Go worker (probe/orchestrate)
                ↓
         S3 (quarantine → originals → renditions)
                ↓
         CDN signed playback
```
**Laravel:** identity, catalog, auth, upload sessions, moderation.  
**Workers:** media work only.  
**S3:** bytes. **CDN:** delivery.

---

# STEP 10 — Quality before scale

### Action
- Tests for lifecycle transitions and permissions
- Failure tests: expired URL, bad checksum, worker crash, duplicate complete
- Runbook notes: how to retry a failed video, how to revoke playback
- Basic metrics: upload success, processing failures, play errors
- Only then: more features (comments, feeds) or multi-region

### Explain
Scale multiplies whatever you already have — including bugs and unclear ownership. Prove one reliable path first.

### Tip
Write the “what do I do when processing is stuck?” note while you still remember how the system works.

### You should not
- Add multi-region or microservices before the single-path flow is boringly reliable.
- Treat load tests as optional if you claim a concurrency target.

### System design
**Order of the whole product:**
```
Catalog (Channel/Video)
  → Direct upload + quarantine
  → Async process → ready
  → Authorized playback
  → QA / runbooks / alerts
  → Extra product features
  → Global scale
```

---

# What you do RIGHT NOW

```text
1. Open app/Models/Channel.php
2. Fix fillable + user/channels relations
3. Fix migration + foreign key
4. php artisan migrate
5. php artisan tinker → create a channel under a user
```

When that works → **Step 2 (Video)**.

---

# Tiny map (advanced)

| Step | You build | System design idea |
|------|-----------|--------------------|
| 1 | Channel works | User owns Channel (catalog) |
| 2 | Video works | Video = metadata + lifecycle, not file |
| 3 | `/api/v1` CRUD | Versioned contract seam |
| 4 | UploadSession | Permission record, not bytes |
| 5 | S3 quarantine | Storage owns bytes; private by default |
| 6 | Process → ready | Async work; Laravel owns status |
| 7 | Play URL | Authorize then short-lived access |
| 8 | Services / policies / tests | Thin controllers, one rule in one place |
| 9 | Outbox + worker + renditions | Events/commands, no shared DB with workers |
| 10 | QA then scale | Reliability before geography |

Start at Step 1. One action at a time.
```
