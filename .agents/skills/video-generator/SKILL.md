# Video Generator Skill

## Purpose

Plan and produce one vertical cinematic Christian prayer video from approved local assets; this is a **proposed workflow guide**, not an implemented executable skill.

## Inputs

Original prayer script, ElevenLabs narration, rights-approved assets/music, metadata index and a server-controlled output directory.

## Steps

1. Check the selected original files exist and licensing covers the intended destination (YouTube Shorts for automatic/explicit publishing).
2. Match visuals to each part of the prayer, varying recent asset selections.
3. Render a 9:16, ideally 1080×1920, H.264 MP4 at 30 fps via FFmpeg with balanced narration/music and tasteful cinematic movement.
4. Burn in synchronized captions inside safe margins and save a separate SRT and transcript.
5. Validate duration (target 45–60 seconds), visual/audio decoding, caption timing, rights, complete metadata and output paths.
6. Persist output and QA state on Beast. Manual jobs are **drafts**; publication is a separate guarded action. Scheduled publication is disabled until separately approved.

## Outputs

MP4, SRT, transcript, title, description, hashtags, asset IDs, rights evidence and QA/job status.

No blind retries after uncertain YouTube posting, no automatic deletion of original assets. Other platforms are manual download/post targets.
