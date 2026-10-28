# Too Raw Media — Product Requirements Document (PRD)

**Status:** Proposed MVP specification  
**Updated:** 2026-10-28  
**Repository:** `kofiarhin/too-raw-media`  
**Implementation status:** Documentation scaffold only; no application, backend, schedule or publishing is deployed.

## 1. Product summary

Too Raw Media is a small, server-hosted Christian prayer video production system. It turns approved visual and music assets into cinematic, captioned vertical prayer videos, saves them for review/download, and optionally publishes a fresh video automatically to YouTube Shorts each morning. The owner controls the entire library and can generate or publish videos manually through a simple admin frontend.

**Core problem:** Writing prayers, selecting appropriate media, recording narration, editing, captioning and publishing daily content are repetitive and time-consuming; maintaining a consistent visual identity is difficult.

**Outcome:** Produce consistent, high-quality, original Christian prayer content with less manual work while retaining owner control over assets, publishing and data.

## 2. Goals and success criteria

1. A single admin can upload, inspect, edit and delete library files safely from a browser.
2. The admin can trigger a video on demand; it is saved as a draft and never auto-published as a result of manual generation.
3. The same generation pipeline can produce a scheduled video each day at **05:00 Europe/London**.
4. When unattended publishing is explicitly activated, a completed, quality-checked scheduled video is published to **YouTube Shorts**.
5. Each completed video has a downloadable MP4, SRT captions, transcript and copyable title/description/hashtags.
6. Files, metadata, account credentials and processing remain on the Beast server; the browser is an API client, not permanent storage.
7. The system avoids duplicate publications and overlapping generation jobs, recovers cleanly from failed steps and preserves usable outputs.

Success means an admin can complete each manual flow, an end-to-end scheduled dry run passes QA, and a separately approved real YouTube publication can be confirmed by its returned link. Do not report this as achieved until verified.

## 3. Audience and operating model

- **Primary user:** One owner/admin; no public accounts or multi-user roles in the MVP.
- **Audience of produced content:** Viewers seeking Christian prayer, faith, hope, encouragement, healing, protection, gratitude and strength.
- **Platform:** YouTube Shorts for automatic and on-demand publication; TikTok and other platforms are manually posted using downloaded assets and metadata.
- **Hosting:** Beast Ubuntu VPS; Node.js backend and simple web client.
- **Source control:** GitHub stores code, brand identity, reusable agent instructions, workflows and sanitized sample data. Never commit live media, private credentials, production logs, personal data or runtime indexes.
- **Runtime source of truth:** Server-side files and JSON indexes on Beast.

## 4. MVP user journeys and requirements

### 4.1 Admin login

- Show a login screen for the single administrator.
- Protect all library, generation, download, publishing and deletion endpoints with server-side authentication.
- Use a securely hashed password and secure, HttpOnly session cookie with logout and expiration; protect mutations against CSRF and check origin.
- No signup, user invitations, role matrix or client-side-only permission checks.
- Rate-limit failed login attempts; keep secrets outside the public repository.

### 4.2 Asset library — create, read, update, delete

- Upload approved images, video clips and licensed/owned music from the frontend using authenticated requests.
- Backend validates file type, size, extension/content and storage limits, assigns an asset ID, saves the file on Beast, extracts basic technical details and atomically updates `assets.json`.
- Library view allows listing, previewing, filtering/searching by tags/type/theme/mood, and seeing relevant rights information.
- Admin may update descriptive metadata (filename/display name, description, tags, mood, theme, rights/source, etc.) without breaking internal references.
- Deletion requires confirmation, checks for active job use and removes the stored file and index entry safely. Historical video records should retain a descriptive snapshot of used assets when necessary.
- Original uploaded assets are **never automatically deleted**.

### 4.3 Manual video generation

- Frontend has a **Generate Video** page and an optional prayer theme/topic field.
- Backend starts one job using the shared pipeline: choose theme → original prayer script → select suitable approved assets → narration → assemble video → captions → QA → save outputs and metadata.
- Show job state and errors. Allow viewing drafts in **Videos**.
- Manual output defaults to **draft/ready**, with **no automatic publication**.
- Do not start a second generation job while one is executing.

### 4.4 Video library and distribution

- **Videos** page lists drafts, ready outputs, published outputs and failed jobs, with previews and statuses.
- Provide downloads of MP4, SRT and transcript; provide copyable title, description and hashtags/keywords suitable for manual posting.
- Admin can select a QA-passed video and choose **Publish to YouTube**. This requires an explicit button action; display final status and YouTube URL.
- A publication failure must not delete or overwrite generated files.
- Other-platform publication stays manual; no TikTok autopost integration in MVP.

### 4.5 Daily generation and publication

- One scheduler triggers daily at **05:00 Europe/London** (timezone/DST-aware).
- Scheduled and manual generation use the **same** pipeline and QA gates.
- Check idempotency by scheduled date/job and publication status; single-job locking prevents concurrent jobs.
- **Publishing defaults OFF**, including after installation. Enabling unattended publication requires explicit owner approval following tested dry runs.
- When enabled and a scheduled video passes QA, publish once to YouTube via a configured adapter, initially intended to use Buffer **subject to verifying actual YouTube Shorts API support and account permissions** before implementation.
- Store remote publication ID/URL, timestamps, errors and retries/status checks. Never blindly retry a possibly successful external publish.

## 5. Video and creative specifications

- Original, respectful Christian prayers and suitable supporting imagery; follow the modular files under `brand/`, `instructions/` and `workflows/` when they exist.
- Target **45–60 seconds** and **9:16 vertical video** intended for Shorts; validate the current platform requirements before release.
- Visual style: cinematic Christian-themed digital artwork, emotionally powerful scenes, dramatic clouds/storms, golden spiritual light, thoughtful variety; avoid unrelated stock imagery, embedded logos and text.
- Match assets to themes (e.g., protection, healing, hope, gratitude, strength). Avoid unnecessary recent repeats.
- Narration via **ElevenLabs REST API**; assembly and timed captions via **FFmpeg**.
- Produce readable burned-in captions plus a separate SRT file, transcript, title, description and hashtags.
- Quality checks before marking ready: required files exist; video/audio decode; dimensions/aspect ratio/duration within target; audible narration; captions align; no empty text or invalid asset rights.
- Manual review or blocking remains appropriate for uncertain rights, quality or publication eligibility.

## 6. Simple technical architecture

```text
Browser admin UI (no permanent project data)
       | HTTPS authenticated API requests
       v
Node.js backend on Beast
  |-- Auth/session + protected API routes
  |-- Asset CRUD and index
  |-- One shared video-generation pipeline
  |     |-- Codex for planning, scripting and creative decisions
  |     |-- ElevenLabs REST for voiceover
  |     |-- FFmpeg for assembly and captions
  |-- QA + job lock + publication state
  |-- YouTube publishing adapter (Buffer if verified)
  |-- 05:00 Europe/London scheduler
  |-- Retention/cleanup
  +-- Server-only filesystem / JSON
```

Frontend pages: **Login**, **Assets**, **Generate**, **Videos**. Keep UI focused; no complex dashboard or agent framework. Use server-side APIs for all changes and data reads. Use HTTPS and appropriate access controls for the externally reachable app.

### 6.1 Server-only storage

Illustrative layout (actual root configurable):

```text
/var/lib/too-raw-media/
  data/
    assets.json
    videos.json
    history.json
  assets/
    images/
    music/
    clips/
  videos/
    completed/
    temporary/
  logs/
```

Store code and non-sensitive example schemas in the GitHub repository; **never** sync live runtime storage into GitHub. Use atomic JSON writes, validated schemas and a lock/queue suitable for a single running instance. Back up live metadata and important original media outside the cleanup directory.

### 6.2 Minimal JSON records

`assets.json` stores at least: `id`, `type`, `filename`, `path`, `description`, `tags[]`, `theme`, `mood`, `rights`, optional `source/license`, technical properties (size/type/duration/dimensions where relevant), `createdAt` and `lastUsedAt`.

`videos.json` stores at least: `id`, `title`, `theme`, `description`, `hashtags[]`, `videoPath`, `captionsPath`, `transcriptPath`, `assetIds[]`, `durationSeconds`, `status`, `keep`, `source` (manual/scheduled), `createdAt`, and publication fields (`platform`, `remoteId`, `youtubeUrl`, `publishedAt`, `error`, attempt/confirmation state).

`history.json` (or equivalent) stores job/run and publication attempts to support audit, idempotency and recovery. Paths are **server-relative** and served only via validated authenticated endpoints. IDs, not mutable filenames, are stable references.

## 7. Cleanup and retention

| Data | Default rule |
| --- | --- |
| Uploaded original images, music and clips | Keep until admin deletes |
| Generated MP4 files | Remove after **7 days** unless `keep=true` |
| Caption files, transcripts and small video metadata | Retain, subject to owner deletion/retention settings |
| Temporary successful generation files | Remove immediately after successful finalization |
| Failed-job intermediate files | Remove after **24 hours** |
| Application logs | Keep **7 days**, redact secrets |
| Publishing history and YouTube URL | Retain after local MP4 cleanup |

Cleanup runs on the server, not the client. Do not touch files belonging to active generation/publication; use safe path checks and atomically update indexes. Allow admin confirmation for manual deletion. Enforce a `Keep` flag for generated videos that must remain available.

**Decision to reconfirm before activating automatic cleanup:** the previously proposed 7-day MP4 retention policy has not yet been individually confirmed by the owner; treat it as a configurable default proposal until approved.

## 8. API surface (indicative)

- `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me`
- `GET /api/assets`, `POST /api/assets`, `PATCH /api/assets/:id`, `DELETE /api/assets/:id`
- `GET /api/videos`, `GET /api/videos/:id`, authenticated media/attachment download endpoints
- `POST /api/generation`, `GET /api/jobs/:id`
- `POST /api/videos/:id/publish`, `PATCH /api/videos/:id` (including Keep)
- `GET /api/system/status` for owner-visible schedule/publishing status

The backend checks ownership, validation, rate limits and current job/publishing state for every operation. Do not accept arbitrary filesystem paths supplied by the browser.

## 9. Safety and operational requirements

- No secrets in Git; use server environment configuration and least-privilege credentials.
- Rights-aware media selection; exclude files without suitable permission to publish.
- Guard uploads against path traversal, malformed files and oversized requests. Authenticated downloads must never expose arbitrary local files.
- Prevent duplicate daily jobs/posts; check remote result before retries after uncertain responses.
- A failed upload, generation, caption check or publish must be visible and recoverable without corrupting JSON or deleting originals.
- Log structured job outcomes without secret values; keep enough metadata to explain failures.
- Explicit separate approval is required for production deployment and for activating unattended publishing.
- Verify Buffer-to-YouTube Shorts compatibility, authorization and platform limits before choosing the final publishing implementation.
- Verify end-to-end runs, admin security, cleanup exclusions, failures and restores before production.

## 10. MVP acceptance checklist

- [ ] Admin can sign in and sign out; API refuses unauthenticated CRUD and media access.
- [ ] Upload, list, preview, search, edit and delete assets with correct server-side metadata.
- [ ] Asset rights/metadata are available to the generation agent.
- [ ] Manual generation creates a QA-passed **draft**, not a public post.
- [ ] Video list previews completed work and downloads MP4/SRT/transcript; metadata can be copied.
- [ ] Admin-selected publishing works once, records confirmed YouTube URL and has safe failure handling.
- [ ] Scheduled 05:00 Europe/London dry run produces a valid output and does not publish while disabled.
- [ ] Explicitly approved automatic publishing works with idempotency and confirmed result.
- [ ] Overlapping jobs and repeat publication attempts are blocked.
- [ ] Cleanup preserves uploaded originals, `keep=true` videos and publication records.
- [ ] Runtime data survives app restart; tested backup/restore works.
- [ ] Secrets and live assets are absent from GitHub; release verification and explicit production approval recorded.

## 11. Out of scope for MVP

Public user accounts, multi-user roles, a database, vector search, a large CMS, multiple concurrent workers, fully automatic TikTok/Instagram publishing, and complex analytics or approval workflow engines.

## 12. Delivery outline and unresolved decisions

1. **Foundation:** server storage, schemas, login, secured API, frontend shell and CRUD.
2. **Production:** script planning, asset selection, ElevenLabs, FFmpeg, captions and QA.
3. **Review:** draft list, downloads, metadata, Keep flag, status and recoverable errors.
4. **Publishing:** verify Buffer/YouTube API path; implement manual publish, then scheduled dry runs.
5. **Operations:** retention, backups, test coverage, safe recovery; explicitly approve deployment and separately activate automatic posting.

**Open decisions to validate before implementation/release:** publishing adapter/API support; specific limits on upload/storage; admin credential provisioning; which backup destination to use; and final approval for automatic 7-day cleanup. Do not silently treat proposed defaults as production approval.

---

**Source of scope:** project discussion through 2026-10-28 and the repository's existing documentation scaffold. Repository references to TikTok automatic posting are outdated relative to this PRD and should be updated in a separate scoped change.
