# Asset Selection — Too Raw Media

The authenticated owner uploads, views, edits and deletes images, video clips and music through a simple frontend. **Beast owns every original file and the live `assets.json` index**; `data/assets.example.json` is a sample, not inventory.

Choose assets using description, tags, theme, mood, rights/source, dimensions or duration and last-used information. Match the prayer's story, vary scenes and music, and avoid unnecessary repeats. Use only media whose rights record clearly permits the intended destination, including YouTube Shorts. “Royalty-free” by itself is not sufficient permission. If permission is missing, fail closed.

No agent may assume a file is available or delete originals through rendering cleanup. Changes to asset metadata and files must go through the authenticated server API; deletion needs confirmation and must not disrupt an active job. If live inventory is unavailable, report that and perform planning only.
