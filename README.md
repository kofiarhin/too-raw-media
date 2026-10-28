# Too Raw Media

Autonomous Christian prayer video production for TikTok.

## MVP
- User uploads images, video clips and approved music via a simple portal.
- A local asset index stores metadata and tags.
- Beast triggers the worker daily at 05:00 Europe/London.
- Codex drafts a new prayer, ElevenLabs produces narration, FFmpeg creates a 9:16 captioned video, automated QA checks it, and Buffer publishes after publishing is explicitly enabled.
- Daily jobs record history and avoid duplicate publication.

## Instructions
Start at [`instructions/INDEX.md`](instructions/INDEX.md) to find the right rule file. Edit one file to change that behavior without rewriting the entire ChatGPT Project instructions.

- `AGENTS.md`: Codex operating and safety rules.
- `creative.md`: backward-compatible pointer to modular creative rules.
- `data/assets.example.json`: example asset metadata; production data lives on Beast.
- `.agents/skills/video-generator/SKILL.md`: proposed video generation workflow.

## Ownership
GitHub stores code and technical instructions; IdeaHub stores durable project context; Linear tracks execution; Beast is runtime truth.

**Status:** documentation scaffold only. No worker, portal, API integration, deployment or publishing is implemented.
