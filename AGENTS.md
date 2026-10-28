# Too Raw Media — Codex Instructions

## Goal

Implement the scoped MVP in [docs/PRD.md](docs/PRD.md): a secure server-based Christian prayer video factory with admin media CRUD, manual drafts/downloads and optionally scheduled **YouTube Shorts** publishing. Read `README.md` and `instructions/INDEX.md`, then load only relevant instructions. Apply `.agents/skills/video-generator/SKILL.md` for video creation.

## Operating rules

- Codex is the selected coding executor; Beast is the intended Ubuntu VPS runtime. Tracked implementation follows Linear.
- Prefer a small Node.js frontend/backend, FFmpeg and server-side JSON/filesystem. No database or agent framework for the MVP.
- Frontend is a client only; Beast owns original assets, `assets.json`, `videos.json`, `history.json`, renders, logs and secure credentials.
- One authenticated admin; protect **every** file and mutation endpoint. Never trust browser-supplied filesystem paths or access checks.
- Never commit secrets, original licensed media, user data, production indexes, logs or rendered videos to this public repository.
- Use atomic JSON writes, safe path checks, one-job locking, backups and idempotent job/publication records.
- Never blindly retry a possibly successful YouTube publication: reconcile provider state first.
- Publishing and scheduled automatic cleanup default **off** until the applicable verification and explicit approvals. Production deployment and unattended publication activation require separate approval.
- Uploaded originals are never removed by automatic cleanup. Retain generated videos marked Keep.
- Preserve Post Generator unchanged. Do not confuse the new prayer-video product with IdeaHub's historical Too Raw Media urban photography record.
- Report implementation, verification, commit, merge, deployment and publishing status only when independently evidenced.

## Planned shared generation flow

Manual request or daily **05:00 Europe/London** schedule → original 45–60 second prayer via Codex → approved asset selection → ElevenLabs REST narration → FFmpeg 9:16 vertical video + timed captions → quality gates → draft/ready outputs. Manual jobs stay drafts; only explicitly initiated admin publishing or a separately enabled scheduled job can publish to **YouTube Shorts**.

Buffer remains a candidate publisher until YouTube Shorts API and account support are verified. TikTok and other platforms are manual download/post destinations in this MVP.

## Instructions

Follow `instructions/INDEX.md` and the relevant modular files. Use the approved license/rights metadata and avoid repeating recently used content. The product PRD wins if legacy content guidance still mentions automatic TikTok publishing.
