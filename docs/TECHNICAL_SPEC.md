# Too Raw Media — Detailed MVP Technical Specification

**Status:** Proposed implementation design, not evidence of a working application  
**Date:** 2026-10-28  
**Source of product requirements:** [PRD](PRD.md)  
**Implementation owner:** Codex (chosen coding executor)  
**Deployment target:** Beast (Ubuntu VPS)  
**Authority:** If a design recommendation here conflicts with the PRD, the PRD controls scope. Operational approvals remain separate.

## 1. Purpose and boundaries

Build a private, owner-operated content factory for cinematic Christian prayer videos. An authenticated browser manages uploaded images, clips and licensed music, starts generation, previews and downloads completed output, and explicitly publishes a ready video to YouTube Shorts. A 05:00 **Europe/London** scheduled run uses the same pipeline and may publish unattended **only after explicit activation**.

Production data stays on Beast. The client is not a source of truth and does not store a persistent project library. The application must remain small: **one admin, one server process, one active generation job, JSON and filesystem storage, no relational/NoSQL database, no agent framework.**

**Not part of MVP:** public users, team roles, TikTok/Instagram auto-publication, multi-worker execution, multi-tenant storage, advanced analytics, in-app video editor, image generation inside the worker, multiple remote publishing providers at once.

**Do not infer implementation:** the repository currently contains documentation and instructions, not a verified service, UI, scheduler or integration.

## 2. Requirement traceability

| ID | Requirement | PRD area | Acceptance evidence |
|---|---|---|---|
| AUTH-01 | Single-admin login and protected APIs | 4.1 | Auth tests and browser login/logout check |
| ASSET-01 | Upload/list/preview/search/edit/delete media | 4.2 | CRUD tests with real temporary files |
| ASSET-02 | Rights-aware, agent-readable metadata | 4.2, 5 | Invalid rights excluded from selection |
| GEN-01 | Optional topic, manual draft generation | 4.3 | E2E request produces downloadable draft |
| GEN-02 | Shared production and QA pipeline | 4.3, 4.5 | Same service called by API and timer |
| VIDEO-01 | MP4/SRT/transcript and copyable metadata | 4.4 | Browser preview and authenticated download tests |
| PUB-01 | Explicit admin-triggered YouTube publication | 4.4 | Confirmed remote ID/URL with safe retry behavior |
| PUB-02 | Disabled-by-default daily 05:00 UK schedule | 4.5 | DST and dry-run tests |
| STORE-01 | All files/JSON on Beast, no client persistence | 6 | Restart and API tests |
| CLEAN-01 | Retention, Keep, originals protected | 7 | Dry-run cleanup and fixture tests |
| OPS-01 | Backups, idempotency, failure recovery | 9 | Restart/crash and simulated publisher-failure tests |

## 3. Runtime design

```text
Admin browser (HTTPS)
  └─ React/Vite TypeScript UI (suggested implementation choice)
       └─ same-origin /api/ authenticated requests
             └─ Node.js server (TypeScript + Express suggested)
                  ├─ session login + CSRF + request validation
                  ├─ assets API + stream/preview/download
                  ├─ generation API + job-status read
                  ├─ videos API + publishing actions
                  ├─ JSON repository + safe file store
                  ├─ single-job coordinator
                  │    └─ Codex script/shot planning
                  │         └─ ElevenLabs REST narration
                  │              └─ FFmpeg render and captions
                  │                   └─ ffprobe/QA
                  ├─ publisher adapter (Buffer candidate; verify first)
                  ├─ UK daily scheduler
                  └─ retention and backup tasks
```

**Architectural choice:** one Node.js application can serve built client assets and `/api` from the same origin; development can use Vite proxy. Avoid microservices and Redis. Bind internal service to localhost behind existing HTTPS reverse-proxy only after authorized infrastructure setup.

Use a typed shared package for API records and validation, but avoid an abstraction-heavy repository pattern. The architecture is proposed; validate against Beast's real registered workspace before implementation.

## 4. Suggested repository layout (not yet created)

```text
too-raw-media/
  README.md
  AGENTS.md
  docs/
    PRD.md
    TECHNICAL_SPEC.md
  brand/                  # future modular source-of-truth docs
    identity.md
    audience.md
    voice-and-tone.md
    visual-style.md
  instructions/           # current reusable creative instructions
    INDEX.md
    image-generation.md
    prayer-writing.md
    asset-selection.md
    video-production.md
    publishing.md         # proposed
  workflows/              # proposed, model-agnostic operating steps
    generate-assets.md
    generate-video.md
    daily-publishing.md
  .agents/skills/video-generator/SKILL.md
  data/
    assets.example.json
    videos.example.json   # proposed, sanitized
  client/
    package.json
    src/
      App.tsx
      api/client.ts
      pages/Login.tsx
      pages/Assets.tsx
      pages/Generate.tsx
      pages/Videos.tsx
      components/
  server/
    package.json
    src/
      index.ts
      config.ts
      routes/{auth,assets,generation,videos,system}.ts
      services/{asset-store,json-store,job-coordinator,script-planner,elevenlabs,ffmpeg,quality-check,publisher,cleanup,backup}.ts
      jobs/daily.ts
      auth/session.ts
  packages/shared/src/schemas.ts
  scripts/
    backup.sh
    verify-environment.sh
  tests/
    auth.test.ts
    assets.test.ts
    generation.test.ts
    publishing.test.ts
    cleanup.test.ts
  package.json
  .env.example
  .gitignore
```

Root npm workspaces are an implementation recommendation, not a confirmed existing setup. Do not create additional frameworks or storage layers unless tests show need.

## 5. Server-owned storage and safety

Configure one canonical **absolute data root**, suggested `/var/lib/too-raw-media`:

```text
/var/lib/too-raw-media/
  data/
    assets.json
    videos.json
    history.json
    settings.json                 # non-secret feature flags only
    backups/                     # protected metadata snapshots
  assets/
    images/
    clips/
    music/
  videos/
    completed/<videoId>/
       video.mp4
       captions.srt
       transcript.txt
       metadata.json             # optional per-video export
    temporary/<jobId>/
  logs/
  locks/
```

This is **runtime data outside the public Git repository**. Production secrets (session key, password hash, ElevenLabs token, publishing tokens) must live in secured environment configuration, never `settings.json`, frontend bundles, Git or Codex prompts.

**File safety contract:**

1. Every stored filename is server-generated (UUID or safe random ID); user display names live as metadata only.
2. Resolve every requested path from a known server-side record and allowlist root; reject `..`, symlink escapes and user-provided absolute paths.
3. Validate detected MIME/signature, size, content, dimensions/duration and extension; transcode or reject unsupported content. Upload to staging; validate first; then atomically rename into library. Clean up failed staging files.
4. JSON writes use validation and atomic temp-file+rename in the same filesystem, guarded by one writer/lock; maintain versioned snapshots/backups. Never write half a record.
5. One process/instance is the MVP assumption; refuse startup of a second writer or provide a well-tested cross-process lock. Recover stale locks only after proving owner process is gone.
6. Filesystem operations use least-privilege service account permissions, restrictive umask, controlled capacity/quotas and safe streaming. No direct public static route to asset originals or video paths.
7. On failure between file placement and index write, reconcile or quarantine the orphan rather than silently losing metadata. On deletion, remove references atomically and retain an audit event.

**Backups:** scheduled and before destructive data migrations, back up JSON and uploaded originals to a separate, access-controlled storage destination; test recovery. Backup target and retention require decision before live setup. Do not assume local same-disk backup protects a VPS outage.

## 6. Data model and validation

Use JSON files with **versioned envelopes** (proposed change from the PRD's minimal illustrative arrays), e.g. `{ "version": 1, "items": [] }`. The existing `data/assets.example.json` currently uses `{ "version": 1, "assets": [] }`; implementation must either preserve that structure or migrate examples and runtime together. **Recommendation:** keep `assets`, `videos` and `runs` collection keys respectively, with a version number.

### 6.1 AssetRecord

```ts
type AssetRecord = {
  id: string;                 // immutable UUID
  type: "image" | "clip" | "music";
  filename: string;           // original display filename, not physical path
  storageKey: string;         // server-generated relative storage key
  mimeType: string;
  sizeBytes: number;
  description: string;
  tags: string[];
  theme?: string;
  mood?: string;
  width?: number; height?: number; durationSeconds?: number;
  rights: {
    status: "approved" | "pending" | "rejected";
    source: string;
    licenseReference?: string;
    allowedPlatforms: string[]; // e.g. ["youtube", "tiktok"]
  };
  createdAt: string; updatedAt: string; lastUsedAt: string | null;
};
```

Match the existing sample `rights.status` and `rights.source`; normalize legacy `type:"video"` vs `type:"clip"` during implementation if any data exists. **Never** treat missing license evidence as approved.

### 6.2 VideoRecord

```ts
type VideoRecord = {
  id: string;
  source: "manual" | "scheduled";
  scheduledDate?: string;            // local YYYY-MM-DD idempotency key
  theme: string;
  title: string;
  description: string;
  hashtags: string[];
  assetIds: string[];
  assetSnapshot: Array<{ id: string; description: string; rightsSource?: string }>;
  videoPath: string | null;
  captionsPath: string | null;
  transcriptPath: string | null;
  durationSeconds: number | null;
  status: "draft" | "generating" | "ready" | "qa_failed" | "publishing" | "published" | "publish_unknown" | "publish_failed";
  qa: { passed: boolean; checks: Record<string, boolean>; issues: string[] };
  keep: boolean;
  publication?: {
    platform: "youtube";
    provider: string;             // only verified adapter may be enabled
    requestKey: string;
    remoteId?: string;
    youtubeUrl?: string;
    requestedAt: string;
    confirmedAt?: string;
    lastError?: string;
  };
  createdAt: string;
  updatedAt: string;
};
```

When MP4 expires set `videoPath=null` and retain published metadata; UI shows `Expired locally` and disables the download. **Status denotes generation/publication outcome; local file availability is separate.** Repeated publication attempts do not create a second `publication` for an already-confirmed video.

### 6.3 RunRecord / history.json

```ts
type RunRecord = {
  id: string;
  source: "manual" | "scheduled";
  scheduledDate?: string;
  videoId?: string;
  stage: "queued" | "planning" | "narrating" | "rendering" | "quality_check" | "ready" | "publishing" | "completed" | "failed" | "cancelled";
  status: "pending" | "running" | "succeeded" | "failed" | "cancelled";
  startedAt?: string; endedAt?: string;
  attempts: number;
  errorCode?: string;
  safeErrorMessage?: string;
};
```

Indexes are small and read/write under one coordinator. Validate every record at startup, reject unsupported schema versions, never truncate corrupt data automatically. Preserve immutable IDs and explicit UTC ISO-8601 timestamps; scheduling keys use **Europe/London local date**, not UTC date.

### 6.4 settings.json

`{ "schemaVersion":1, "publishingEnabled":false, "cleanupEnabled":false, "timezone":"Europe/London" }`. Both switches default **false** on missing/invalid settings. Approval records and credentials are not expressed as an easily editable browser boolean; production activation is a separately authorized operational step.

## 7. Authentication and browser security

**One admin only; no sign-up.** Provision admin password out of band, store only a memory-hard salted hash (Argon2id preferred; bcrypt acceptable with safe parameters). Rate-limit failed logins and use constant-time comparison as appropriate.

- Authenticate with server-managed session IDs and `HttpOnly; Secure; SameSite=Strict` cookies under HTTPS; rotate on login, expire on logout and idle/absolute timeout. Session state can live in process memory for MVP, but restarts require login again. No localStorage tokens.
- Require authenticated access on all API routes except login and generic minimal health; sensitive metadata/media access must not be public.
- Validate exact Origin and CSRF tokens for state-changing requests; enforce restrictive CORS (same-origin), frame-ancestors CSP, security headers and request body/upload limits.
- Return generic invalid-login errors, redact secrets/authorization headers from all logs, no pass-through of raw Codex or external API output to browser.
- Protect against command injection with subprocess argument arrays and strict allowed binary/options; timeouts, process tree termination and output limits apply to Codex/FFmpeg.
- Admin deletion requires a confirmation dialog identifying affected files. Keep server-side checked reference counts/job locks; UI confirmation alone is not authorization.

## 8. API design (version 1)

All JSON responses are `{ "data": ..., "error": null }` or `{ "data": null, "error": { "code": string, "message": string } }`. Never return stack traces, secrets or absolute paths. Provide stable `id`, `createdAt`, `updatedAt`. Paginate lists with bounded `limit`/`cursor` (or simple page/pageSize). Use 400 invalid input, 401 unauthorized, 403 forbidden, 404 unknown ID, 409 conflicting job/state, 413 oversized upload, 422 invalid media, 429 login throttled, 503 unavailable provider.

| Method | Endpoint | Request | Successful result |
|---|---|---|---|
| POST | `/api/auth/login` | `{password}` | 200 session cookie + CSRF token |
| GET | `/api/auth/me` | cookie | admin identity/session status |
| POST | `/api/auth/logout` | CSRF | 204 and invalidated session |
| GET | `/api/assets` | `type,tag,theme,mood,q,cursor,limit` | filtered AssetRecord page |
| POST | `/api/assets` | multipart `file` + metadata | 201 AssetRecord |
| GET | `/api/assets/:id` | — | AssetRecord |
| PATCH | `/api/assets/:id` | editable metadata only | AssetRecord |
| DELETE | `/api/assets/:id` | CSRF + confirmation | 204 or 409 if locked/in use |
| GET | `/api/assets/:id/preview` | — | authenticated streamed media |
| POST | `/api/generation` | `{theme?:string}` | 202 `{jobId}` or 409 |
| GET | `/api/jobs/:id` | — | stage, progress, safe error, videoId |
| GET | `/api/videos` | `status,source,cursor,limit` | video page |
| GET | `/api/videos/:id` | — | VideoRecord |
| PATCH | `/api/videos/:id` | `{keep:boolean}` | updated record |
| GET | `/api/videos/:id/preview` | HTTP range supported | authenticated video stream |
| GET | `/api/videos/:id/download/:kind` | kind `mp4,srt,transcript,metadata` | attachment or 410 if expired |
| POST | `/api/videos/:id/publish` | CSRF, explicit admin click | 202/confirmed state, 409/503 |
| GET | `/api/system/status` | — | publishingEnabled, cleanupEnabled, job summary, next schedule |

**Publisher action:** reject when QA failed, already published, currently publishing, or provider has unresolved `publish_unknown` state. The manual publish button is owner initiation, but the production publishing integration must still be configured/verified first. Do not expose a casual API `publishingEnabled=true` toggle.

**Examples:**

```json
POST /api/generation
{"theme":"protection"}
```

```json
HTTP 202
{"data":{"jobId":"job_123"},"error":null}
```

Server validates and bounds string lengths; actual IDs are generated, never supplied by the client.

## 9. Frontend UX

**Login:** password field, loading/error, logout. Redirect unauthenticated access to login; server independently protects endpoints.

**Assets:** upload control with accepted media types and size guidance; preview cards/table; search by description/tag; filter type/mood/theme; detail/edit form for tags, description, mood, theme, rights/source; delete confirmation. Show *pending rights* visibly and exclude from production.

**Generate:** optional theme field; Generate Video button; show busy state, currently running stage, failures and link to completed draft. 409 means another job is active; no accidental duplicate click.

**Videos:** sortable list with statuses and date, portrait preview, metadata copy, downloadable MP4/SRT/transcript and a Keep switch. An explicit **Publish to YouTube** action displays final status and confirmed URL; disable during unknown publication state. Expired local files show retained metadata and publication link. Failures show non-secret actionable errors.

Keep design accessible: labelled controls, keyboard interaction, status `aria-live`, focus-managed dialogs, caption legibility and responsive vertical previews. Avoid expanding MVP into analytics/editor dashboards.

## 10. Generation pipeline

One `generateVideo({source, theme?, scheduledDate?})` path for both manual and scheduled jobs.

1. **Acquire single-job lock.** Validate request, capacity, credentials/tool readiness; create RunRecord. For scheduled jobs claim unique `scheduledDate` in Europe/London.
2. **Plan content.** Read brand and relevant modular instructions. Ask Codex to produce structured output with original prayer, theme, title, description, hashtags and suggested visual beats. Validate JSON schema; never execute arbitrary returned shell commands. Compare topic/script fingerprint with recent history, retry bounded planning if duplicate.
3. **Choose assets.** Read server `assets.json`; only rights-approved, suitable-platform files. Score theme/mood/tags, exclude missing originals and reduce repeated use. Fail safely if insufficient cleared material; do not invent media.
4. **Narrate.** ElevenLabs REST with configured voice and bounded timeout; save voice audio in per-job temporary directory. Track usage and errors without logging tokens.
5. **Time captions.** Use transcript and measured audio duration; produce valid SRT cue intervals and a safe-margin burned-in caption layer. Timing must be checked against audio rather than assuming a fixed speaking rate.
6. **Render.** FFmpeg builds 9:16 1080x1920 H.264 output at 30fps with slow motion/pans/crops, smooth cuts, controlled music level and spoken audio priority. Ensure no clip/image extends outside actual audio/video duration.
7. **QA.** ffprobe checks streams/dimensions/frame rate/duration, FFmpeg decode check, nonzero audio, SRT cue monotonicity and caption-area policy; require asset rights and metadata completeness. Proposed duration gate targets 45–60s, with policy-configurable tolerance subject to PRD.
8. **Finalize.** Atomically move complete output into `videos/completed/<videoId>`, save video metadata, link run to video, then clear successful temp files and unlock.
9. **Publish separately if eligible.** Manual remains draft/ready; scheduled publishes only if activation enabled and verified QA pass. Publication failure never removes completed output.

**Errors:** each stage emits safe error code, stage and timestamp. Never mark ready when QA fails. Cancel/timeout terminates processes and releases lock after safe reconciliation. On restart, detect interrupted `running` records, mark them interrupted/failed or requiring review, and never auto-resume a state-changing publish.

**Agent isolation:** Codex may plan scripts/shot sequences via its authenticated CLI, but must not obtain raw server secrets or unrestricted write/delete authority; backend uses typed calls and validated paths. Actual CLI authentication, noninteractive operation and required quotas must be tested on Beast before implementation acceptance.

## 11. Publisher contract and idempotency

```ts
interface Publisher {
  validateConfiguration(): Promise<void>;
  publish(input: {
    videoId: string;
    requestKey: string;
    mediaPath: string;
    title: string;
    description: string;
    hashtags: string[];
  }): Promise<{ state: "confirmed" | "pending" | "unknown"; remoteId?: string; url?: string }>;
  getStatus(remoteIdOrRequestKey: string): Promise<{ state: "confirmed" | "failed" | "unknown"; url?: string }>;
}
```

This is an **internal proposed interface**, not a claim about Buffer's public API. **Integration gate:** confirm Buffer currently supports creating and confirming YouTube Shorts through the needed API plan/channel or select another approved adapter; verify OAuth scopes, token storage, media upload/polling, visibility, AI disclosure, caption limits, rate limits and resulting Shorts URL. If not proven, do not silently switch provider or claim auto-publishing works.

- Write a durable `requestKey` / publish-attempt record **before** the external request.
- `publishing` → `published` only with confirmed remote ID/URL or reliable provider confirmation.
- Timeout or ambiguous response → `publish_unknown`; lock that video from further publish until remote status is checked. Never blindly retry.
- If provider explicitly confirms no remote publication, transition to `publish_failed` and allow an explicit admin retry or governed scheduled recovery policy.
- A single scheduled date may have one published post, enforced by durable state and a lock, even if the worker restarts.

## 12. Daily scheduling

Use a single Beast scheduler invocation at **05:00 Europe/London**. Prefer a timezone-aware OS scheduler or an equivalent tested Node scheduling mechanism; specific systemd unit/timer paths must be derived from the real Beast environment and approved independently.

- Europe/London date drives idempotency, automatically handling GMT/BST.
- Scheduler calls the same coordinator as manual generation.
- If another job is running, do not start a second; record `skipped_busy` for that occurrence and alert/admin-visible result. Do not silently post twice later.
- Scheduled job may generate a draft during dry run while `publishingEnabled=false`. It must not perform a remote publish.
- A missed schedule after downtime does not auto-publish on boot unless a separately approved catch-up policy exists; default: log missed run and resume next day.
- Changes to production timer/service require explicit Beast operational approval.

## 13. Retention and cleanup

All cleanup is **server-side**, initially disabled pending approval. Run a periodic dry-run first; log candidate paths/IDs without deleting anything.

| Class | Proposed default | Hard protection |
|---|---|---|
| Uploaded originals | no expiration | Never automatically remove |
| Generated MP4 | 7 days after creation (configurable) | `keep=true`, active job, pending/unknown publish |
| Captions, transcripts, metadata | retain | Preserve source records and download status |
| Successful temporary render files | remove at finalization | Only after output and metadata are verified |
| Failed/interrupted temporary files | 24 hours | Never during active/recovery inspection |
| Application logs | 7 days | Redact secrets |
| Publication history/YouTube URL | retain | Not removed by MP4 expiry |

The PRD specifies a proposed **7-day MP4 rule** but expressly requires owner confirmation before activation; do not mistake this spec for that approval. When MP4 is deleted, set video record `videoPath=null` and display expired locally, preserving publication data. Cleanup should use a tombstone/state transition and safe retry/reconciliation after crashes; never follow untrusted symbolic links.

Manual asset deletion: require confirmation and reject if in active use; keep snapshots of historical video rights/asset attribution. Define whether deletion of an original referenced by completed videos is allowed after active use ends; default is allow with warning that future rerenders are unavailable.

## 14. Environment configuration

Suggested names only (do not commit values):

```text
PORT
DATA_ROOT
ADMIN_PASSWORD_HASH
SESSION_SECRET
ELEVENLABS_API_KEY
ELEVENLABS_VOICE_ID
PUBLISHER_PROVIDER
PUBLISHER_ACCESS_TOKEN
YOUTUBE_CHANNEL_ID
CODEX_COMMAND
FFMPEG_PATH
FFPROBE_PATH
PUBLISHING_ENABLED=false
CLEANUP_ENABLED=false
TZ=Europe/London
```

Production toggles must be validated against authorized activation; environment flags alone are not proof of permission. `.env.example` may list names and placeholders, never real credentials. Log tool version summaries and health without secret values.

## 15. Verification and delivery

### Unit tests

- Input schema validation, MIME and path traversal rejection, asset rights filters, JSON atomic/recovery behavior, one-lock coordinator, schedule local-date key and DST.
- Status transition table, dedupe/script fingerprint, publisher unknown state and no-repeat logic.
- Cleanup Keep/original safeguards, interrupted job recovery, backup/restore.
- Auth session timeout, CSRF/origin protection, login throttling, unauthenticated preview/download denial.

### Integration tests

- Multipart upload + metadata edit/delete with temp directory; real FFmpeg fixture with short synthesized/test audio and known images; SRT output and ffprobe checks.
- Mock Codex/ElevenLabs and publisher failure/retry cases without spending quota.
- End-to-end dry run with **publishing off** and a scheduled invocation at a controlled timezone/date.
- Browser tests for login, asset CRUD, Generate, Videos preview/download/Keep and explicit publish confirmation.

### Production readiness (not automatic)

1. Validate repository instructions, build/test/lint/type checks and all fixtures.
2. Confirm Beast workspace, tooling, disk quota, HTTPS, credentials, service user, backup destination and recovery.
3. Verify real ElevenLabs generation and approved media rights.
4. Independently verify real publisher API support and safe remote status checks before enabling publishing.
5. Request **explicit production deployment approval**, separately from **unattended publishing activation approval**. Manual YouTube action must remain guarded and verified.
6. Approve retention/cleanup separately, verify dry-run candidates and restore workflow.
7. Record evidence in GitHub and tracked execution state in Linear; only then claim verified operation.

**No changes to Post Generator. No production services, firewall, DNS, timers, publication or secrets are authorized by this document.**

## 16. Suggested delivery slices

1. **Foundation:** TypeScript project skeleton, shared validation, JSON store, test harness.
2. **Admin library:** secure login/session, assets API and frontend CRUD.
3. **Generation:** scripted content contract, rights selection, ElevenLabs adapter, FFmpeg/SRT/QA, job persistence.
4. **Review:** Videos UI, preview/download, copyable metadata, Keep and error status.
5. **Publication:** verified YouTube provider adapter, manual action and idempotency/reconciliation tests.
6. **Schedule and operations:** 05:00 UK daily run, safe retry/skip, backup/retention/dry runs and release verification.

Every tracked implementation slice needs a Linear issue. Do not mark Done without evidence. A documentation spec alone does not start implementation.

## 17. Open decisions and assumptions

| Item | Current treatment |
|---|---|
| Buffer support for YouTube Shorts API | **Unverified**. Block production integration until confirmed. |
| Final upload/file/disk limits | Choose only after Beast capacity check; values not fixed here. |
| Single-admin password provisioning and reset | Implement safely after owner approves credentials process; no self-service sign-up. |
| Backups destination/retention | Not selected; required for live readiness. |
| Seven-day MP4 auto-cleanup | Proposed; requires separate explicit owner approval to activate. |
| Site URL, domain and reverse proxy | Not yet approved; do not change DNS/Nginx/public ports. |
| Codex noninteractive server execution | Must be validated with approved tooling; do not assume available. |
| Existing server runtime | Not verified; no deployment claim. |
| Historical name overlap with photography account | Distinct documented records; do not merge identities without evidence. |

---

**Design note:** This technical spec adds implementation recommendations (TypeScript/React/Express, schemas, endpoints, error states) to the product PRD. These choices are *proposals*, not decisions previously locked by the owner. Implementers should preserve the agreed product behavior and raise material changes through the appropriate approval process.
