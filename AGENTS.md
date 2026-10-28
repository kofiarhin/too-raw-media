# Too Raw Media — Codex Instructions

## Goal
Build a small, secure MVP for daily cinematic Christian prayer videos on TikTok. Read `README.md` and `creative.md` before implementation. Apply `.agents/skills/video-generator/SKILL.md` when making videos.

## Work rules
- Codex is the chosen coding executor; Beast runs the service.
- Prefer Node.js, FFmpeg and JSON files. Do not introduce a database or agent framework for the MVP.
- Keep Post Generator untouched. Track active implementation in Linear.
- Never put secrets, real customer information, licensed originals, production logs or generated MP4 files in this public repository.
- Store original media and runtime `assets.json` / `history.json` on Beast, not in Git. Provide read-only sanitized asset-index snapshots to ChatGPT when authorized and available.
- Use atomic file writes and a single-job lock for runtime state. Check for prior work before retries or publication.
- Never auto-retry a potentially successful TikTok publication without first confirming status.
- Publishing must default off. Require explicit authorization before production deployment or turning on unattended publishing.
- Never claim a completed action without evidence. Tests and dry runs must verify output before activation.

## Daily flow (planned)
At 05:00 Europe/London: select a fresh topic, write an original 45–60 second prayer, synthesize narration via ElevenLabs REST, choose approved indexed assets, compose a 9:16 video with FFmpeg, verify script/audio/captions/format, publish via Buffer after API compatibility and permissions are verified, then store results and clean expired outputs.

## Asset selection
Match images and licensed music to a prayer's theme, emotion and story. Avoid repeated scenes and recently used tracks. Never infer TikTok reuse rights from a royalty-free label alone.
