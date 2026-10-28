# Video Production — Too Raw Media

The **planned shared pipeline** (not yet implemented): manual admin request **or** Beast daily 05:00 Europe/London scheduler → Codex writes an original prayer and selects approved media → ElevenLabs REST creates narration → FFmpeg renders a 9:16, preferably 1080×1920 H.264 MP4 with balanced music, cinematic motion and timed captions → QA → store video, SRT, transcript, title, description, hashtags and rights/asset references on Beast.

**Manual generation creates a draft**. A ready video can be published to YouTube Shorts only by an explicit admin action; a scheduled video can be published only after unattended publishing has been separately activated. Buffer is a candidate adapter, not a verified integration. TikTok/other platforms require manual downloads/posts in this MVP.

Follow `.agents/skills/video-generator/SKILL.md`. Verify originality, timing (target 45–60 seconds), media license/platform permission, playable video/audio, subtitles, caption safe areas and duplicate prevention. Do not blindly re-post on uncertain publication outcomes. Use a shared single-job lock; save outputs even if publishing fails.

Publishing defaults **off**. Production deployment, unattended publishing and auto-cleanup activation must follow separate explicit approval/verification gates. See `docs/PRD.md`.
