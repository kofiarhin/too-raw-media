# Too Raw Media

A planned, lightweight **Christian prayer video factory**. It creates original cinematic 9:16 prayer videos from approved media and provides a simple private admin app to manage assets, generate videos, download outputs, and publish to YouTube Shorts.

> **Status:** Documentation and PRD only. No frontend, API, worker, scheduler, live integration, deployment or publishing has been verified or implemented.

## Product scope

- **Admin:** One password-protected owner account, with authenticated create/read/update/delete for images, video clips, music and descriptive metadata.
- **Frontend:** Login, Assets, Generate Video and Videos. Browser makes authenticated API requests; it does **not** own permanent data.
- **Runtime:** Node.js on Beast; all live originals, generated videos, JSON indexes, history and logs stay on the server. No database or agent framework for the MVP.
- **Video pipeline:** Codex helps plan/write prayers and select cleared assets; ElevenLabs REST produces narration; FFmpeg assembles vertical captioned MP4; automated QA validates output. Scheduled and manual generation share one pipeline.
- **Manual generation:** Optional theme; completed videos are drafts/ready for preview and download. Include MP4, SRT, transcript, title, description and hashtags for manual posting to TikTok or other platforms.
- **Publishing:** One 05:00 **Europe/London** daily schedule is planned for **YouTube Shorts**. Admin can also explicitly publish a ready video to YouTube. Publishing defaults **off** until verified dry runs and separate approval. Buffer is a candidate publisher **only if real API capabilities and permissions are verified**.
- **Reliability:** Single active job, duplicate-prevention, atomic JSON changes, publication status reconciliation and backups.
- **Cleanup:** Never automatically delete uploaded originals. Proposed retention: generated MP4 after 7 days unless marked Keep; failed temp files after 24 hours; logs after 7 days; retain captions, metadata and publication history. Seven-day auto-cleanup needs owner confirmation before activation.

## Documentation

- **[Product requirements](docs/PRD.md)** — canonical current MVP scope and acceptance criteria.
- **[Instruction index](instructions/INDEX.md)** — modular creative and production instructions.
- **[AGENTS.md](AGENTS.md)** — Codex, safety, execution and approval boundaries.
- **[Video-generator skill](.agents/skills/video-generator/SKILL.md)** — proposed workflow guidance, not executable implementation.
- **[Example asset index](data/assets.example.json)** — sanitized example, **not** live Beast inventory.
- **[Creative pointer](creative.md)** — backward-compatible index.

## Sources of truth

GitHub stores code, instructions and specs; IdeaHub records durable project knowledge; Linear tracks execution; Beast provides actual runtime evidence. No live state or secret belongs in this public repo.

**Reconciliation note (2026-10-28):** This README follows merged PRD #1 and supersedes the earlier TikTok-autopublishing description. A historical IdeaHub Instagram photography record with the same project name must be preserved and distinguished rather than silently overwritten.
