# reels-assets

Public host for reel/Shorts videos + `reels_schedule.json`, read by **n8n on the VPS** to auto-publish
to Instagram and YouTube. Posting runs on the server, so it works even when the local machine is off/offline.

## Flow
1. Local: `sync_reels.py --push` (in the obs reels project) stages pending reels from `queue.json` /
   `yt_queue.json` → copies mp4s into `reels/` → writes dated entries to `reels_schedule.json` → pushes here.
2. VPS n8n (cron, BRT):
   - **Reels Auto-Publish (IG)** `0 14 * * *` → picks the earliest `platform=instagram, status=ready, date<=today`,
     creates a Graph API `REELS` container from the raw GitHub video_url, polls `status_code=FINISHED`, publishes,
     then writes `status=published` + `media_id` back here.
   - **YT Shorts Auto-Publish** `0 15 * * *` → refreshes OAuth, resumable-uploads the video to YouTube
     (public `#Shorts`), writes `status=published` + `video_id` back here.
3. Missed a day (VPS/API down)? The entry stays `ready` with a past date → the next run catches it up (1/run).

## reels_schedule.json entry
```json
{ "date":"YYYY-MM-DD", "time":"14:00", "platform":"instagram|youtube", "slug":"...",
  "video_url":"https://raw.githubusercontent.com/ninja-automacoes/reels-assets/main/reels/<file>.mp4",
  "caption":"... (IG)", "title":"... (YT)", "desc":"... (YT)", "status":"ready|published" }
```

## Workflows (n8n.ninjadasautomacoes.com)
- Reels Auto-Publish (IG) — manual run: `GET /webhook/reels-run-now`
- YT Shorts Auto-Publish — manual run: `GET /webhook/yt-run-now`

Local launchd posters are **disabled** (`~/Library/LaunchAgents.disabled/`) — the VPS is the sole poster.
