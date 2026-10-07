# VidroTube — Dumb Plan (do this, then this)

No theory. No long docs.  
One step = one action.  
Finish the step. Check it. Go to the next.

---

# START HERE

```bash
cd your-laravel-project
```

You already have Laravel. Good.

---

# STEP 1 — Make the Channel real

### 1.1 Open the Channel model
File: `app/Models/Channel.php`

### 1.2 Make sure it has:
```php
protected $fillable = [
    'user_id',
    'name',
    'handle',
    'description',
    'status',
];
```

### 1.3 Add relation to User
In `Channel.php`:
```php
public function user()
{
    return $this->belongsTo(User::class);
}
```

In `User.php`:
```php
public function channels()
{
    return $this->hasMany(Channel::class);
}
```

### 1.4 Fix the migration if needed
File: `database/migrations/*_create_channels_table.php`

Columns:
- id
- user_id (foreign)
- name
- handle (unique)
- description (nullable)
- status (default: active)
- timestamps

### 1.5 Run it
```bash
php artisan migrate
```

### 1.6 Test in tinker
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
$channel;
```
If you see the channel → Step 1 done.

---

# STEP 2 — Make the Video model

### 2.1 Create model + migration
```bash
php artisan make:model Video -m
```

### 2.2 Edit the migration
File: `database/migrations/*_create_videos_table.php`

```php
$table->id();
$table->foreignId('channel_id')->constrained()->cascadeOnDelete();
$table->string('public_id')->unique();
$table->string('title');
$table->text('description')->nullable();
$table->string('visibility')->default('private'); // public, unlisted, private
$table->string('status')->default('created');     // created, uploading, ready, blocked, deleted
$table->timestamps();
$table->softDeletes();
```

### 2.3 Edit the model
File: `app/Models/Video.php`

```php
protected $fillable = [
    'channel_id',
    'public_id',
    'title',
    'description',
    'visibility',
    'status',
];

public function channel()
{
    return $this->belongsTo(Channel::class);
}
```

In `Channel.php` add:
```php
public function videos()
{
    return $this->hasMany(Video::class);
}
```

### 2.4 Run migration
```bash
php artisan migrate
```

### 2.5 Test
```bash
php artisan tinker
```
```php
$channel = Channel::first();
$video = $channel->videos()->create([
    'public_id' => (string) \Illuminate\Support\Str::ulid(),
    'title' => 'First video',
    'visibility' => 'private',
    'status' => 'created',
]);
$video;
```
If it works → Step 2 done.

---

# STEP 3 — Simple API for Channel + Video

### 3.1 Create controllers
```bash
php artisan make:controller Api/ChannelController
php artisan make:controller Api/VideoController
```

### 3.2 Add routes
File: `routes/api.php`

```php
use App\Http\Controllers\Api\ChannelController;
use App\Http\Controllers\Api\VideoController;

Route::middleware('auth:sanctum')->prefix('v1')->group(function () {
    Route::apiResource('channels', ChannelController::class);
    Route::apiResource('videos', VideoController::class);
});
```

### 3.3 Make controllers simple
**ChannelController** — only these methods first:
- `index` → list my channels
- `store` → create channel
- `show` → one channel
- `update` → update channel
- `destroy` → delete channel

**VideoController** — same idea for videos (create under a channel).

Rule: controller does little.
```php
// example store
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

### 3.4 Test with Postman or curl
Login first (Fortify / Sanctum), then:
```bash
POST /api/v1/channels
GET  /api/v1/channels
POST /api/v1/videos
GET  /api/v1/videos
```

If you can create channel + video via API → Step 3 done.

---

# STEP 4 — UploadSession (prepare for real upload)

### 4.1 Create model
```bash
php artisan make:model UploadSession -m
```

### 4.2 Migration columns
```php
$table->id();
$table->foreignId('video_id')->constrained()->cascadeOnDelete();
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
$table->string('status')->default('pending'); // pending, uploading, completed, expired
$table->string('storage_key')->nullable();
$table->unsignedBigInteger('size')->nullable();
$table->string('mime')->nullable();
$table->string('checksum')->nullable();
$table->timestamp('expires_at')->nullable();
$table->string('idempotency_key')->nullable()->unique();
$table->timestamps();
```

### 4.3 Model fillable + relations
```php
protected $fillable = [
    'video_id', 'user_id', 'status', 'storage_key',
    'size', 'mime', 'checksum', 'expires_at', 'idempotency_key',
];

public function video()
{
    return $this->belongsTo(Video::class);
}
```

### 4.4 Migrate
```bash
php artisan migrate
```

### 4.5 Create controller
```bash
php artisan make:controller Api/UploadSessionController
```

### 4.6 Two endpoints only (for now)
```php
// routes/api.php inside the v1 group
Route::post('upload-sessions', [UploadSessionController::class, 'store']);
Route::post('upload-sessions/{uploadSession}/complete', [UploadSessionController::class, 'complete']);
```

### 4.7 store() — create session + fake presigned URL first
For learning, return a fake URL first:
```php
return response()->json([
    'upload_session_id' => $session->id,
    'upload_url' => 'https://example.com/fake-upload', // replace later with real S3
    'expires_at' => $session->expires_at,
]);
```

### 4.8 complete() — mark session completed + set video status to uploaded
```php
$session->update(['status' => 'completed']);
$session->video->update(['status' => 'uploaded']);
return response()->json(['ok' => true]);
```

Test with curl/Postman.  
If session creates and completes → Step 4 done.

---

# STEP 5 — Real S3 upload (replace the fake URL)

### 5.1 Install AWS SDK if needed
```bash
composer require league/flysystem-aws-s3-v3 "^3.0" --with-all-dependencies
```

### 5.2 Add to `.env`
```
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=...
AWS_BUCKET=...
```

### 5.3 Config disk
In `config/filesystems.php` make sure `s3` disk exists.

### 5.4 In UploadSessionController@store
Generate a real presigned URL (or temporary upload URL) for key like:
```
quarantine/{user_id}/{video_id}/{ulid}.mp4
```
Save that key on the session as `storage_key`.

### 5.5 In complete()
Use Storage facade to check the object exists on S3.  
If yes → status completed + video status uploaded.  
If no → return 422.

### 5.6 Test with a small real file from browser or Postman.

If a real file lands in S3 and complete works → Step 5 done.

---

# STEP 6 — Simple “processing” (no Go yet)

### 6.1 After complete, set video status to `processing`

### 6.2 Create a simple Artisan command
```bash
php artisan make:command ProcessVideoCommand
```

### 6.3 Command does fake work for now
- Find videos with status `uploaded` or `processing`
- Sleep 2 seconds (pretend transcoding)
- Set status to `ready`
- Maybe set a fake thumbnail url field later

### 6.4 Run it
```bash
php artisan videos:process
```

### 6.5 Later replace this command with a real queue job, then with a Go worker.

If video goes uploaded → processing → ready → Step 6 done.

---

# STEP 7 — Playback (simple)

### 7.1 Add a route
```php
Route::get('videos/{video}/play', [VideoController::class, 'play']);
```

### 7.2 play() method
- Check video status is `ready`
- Check visibility (public or owner)
- Return a URL (for now the S3 object URL or a temporary signed URL)

### 7.3 Test
Open the play URL. If you can reach the file when allowed, and get 403 when not → Step 7 done.

---

# STEP 8 — Make it less dumb (one improvement at a time)

Do these only after Steps 1–7 work:

### 8.1 Move logic from controllers into Services
```bash
# create by hand
app/Services/ChannelService.php
app/Services/VideoService.php
app/Services/UploadSessionService.php
```
Controller calls service. Service does the work.

### 8.2 Add Form Requests
```bash
php artisan make:request StoreChannelRequest
php artisan make:request StoreVideoRequest
```

### 8.3 Add API Resources
```bash
php artisan make:resource ChannelResource
php artisan make:resource VideoResource
```

### 8.4 Add Policies
```bash
php artisan make:policy ChannelPolicy --model=Channel
php artisan make:policy VideoPolicy --model=Video
```

### 8.5 Write a few tests
```bash
php artisan make:test ChannelApiTest
php artisan make:test VideoApiTest
php artisan test
```

---

# STEP 9 — Real processing later (when you are ready)

Only after the whole path works:

1. Add `outbox` table (save event when upload completes)
2. Install queue / RabbitMQ or use Laravel queue first
3. Make a real job that probes the file
4. Then add Go worker if you still want it
5. Then real transcoding (FFmpeg or MediaConvert)
6. Then signed CDN playback

Do **one** of these at a time. Never all together.

---

# What you do RIGHT NOW

```text
1. Open app/Models/Channel.php
2. Fix fillable + relations (Step 1.2 and 1.3)
3. Fix migration if needed
4. php artisan migrate
5. php artisan tinker → create a channel
```

When that works → do Step 2 (Video).

When Step 2 works → do Step 3 (API).

Keep going one step at a time.  
Do not jump to S3 or Go until Step 4 and 5.

---

# Tiny map

```
Step 1  Channel works in DB
Step 2  Video works in DB
Step 3  API create channel + video
Step 4  UploadSession (fake URL)
Step 5  Real S3 upload
Step 6  Fake process → ready
Step 7  Play URL
Step 8  Clean code (services, requests, tests)
Step 9  Real workers / transcoding / CDN
```

That is the whole plan.  
Start at Step 1.1.
```
