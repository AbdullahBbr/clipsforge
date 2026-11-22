# ClipForge — OpusClip-style rebuild: investigation + implementation plan

Date: 2026-11-21. Owner: Abdullah. Working name: **ClipForge** (own code,
own brand — not a copy of OpusClip's name, logo or assets).

## 1. What OpusClip actually is (investigation)

### REA findings (already done, 2026-11-21)
- `www.opus.pro` is a **Webflow marketing site**, not the app. REA
  `analyze-javascript-application` on the homepage + 4 JS files:
  706 findings, 246 modules, 0 JS routes, 0 parse failures.
  Parent Evidence `ev_876c6ccb9f7afd42edaec39a580981709467670db140db723d62c312c950ce8d`.
  Full report: `~/workspace/opus-pro/opus-pro-rea-report.md`.
- Homepage flow (inline scripts, REA did not analyse them): paste-a-YouTube-link
  form gated by Cloudflare Turnstile → hidden iframe to
  `clip.opus.pro/lp-embed` → login via postMessage `CLIP_LOGGED_IN/OUT` →
  UTM params kept in sessionStorage → Google One Tap →
  `clip.opus.pro/auth/callback`. No Opus API on the marketing site.
- **Hopper / Ghidra: not applicable, not used.** There is no native binary
  to decompile — OpusClip is a web app. `rea doctor` also shows Hopper,
  Ghidra, IDA, JADX, Kuna all missing on this machine. Using them here
  would be the wrong tool; the honest route is public docs + static web
  analysis + rebuilding the *behaviour*, not decompiling anything.

### Public product surface (opus.pro/pricing, /api, /mcp — read 2026-11-21)
- Plans: Free $0, Starter $15/mo, Pro $29/mo, Business Custom.
- Core pipeline: import long video (local file up to 10–30 GB, YouTube URL,
  public mp4 URL, auto-import from verified YouTube) → AI curation
  (ClipAnything: any genre, not just podcasts; topic/prompt/timeframe,
  clip lengths 0–1m … 10–15m, virality score, auto hook) → captions
  (animated templates, multi-language transcription, speaker colours,
  censor, emoji/keyword highlights) → reframe (9:16 / 1:1 / 16:9, moving
  object tracking, dynamic layout, screenshare/gameplay layouts) →
  edit (text & timeline editing, B-roll, voice-over, speech enhancement,
  pause/filler removal, eye-contact correction, transitions, music,
  text overlay, upscaling) → dubbing + AI sound effects → brand kit
  (templates, intro/outro, fonts, vocabulary) → export (MP4 up to 4K,
  bulk, Premiere/DaVinci XML) → publish (post to social, scheduler,
  multi-profile, title/description/hashtag generator, analytics).
- REST API: `POST https://api.opus.pro/api/clip-projects`, Bearer `opk_`
  key, body `{videoUrl, curationPref:{range:{startSec,endSec},
  clipDurations:[[0,90]], topicKeywords, genre, skipCurate},
  importPref:{sourceLang}, brandTemplateId}`. SDKs: Node, Python, Go,
  PHP, Ruby, Java. Public OpenAPI + llms.txt.
- MCP: `https://mcp.opus.pro/mcp`, OAuth (no API key), 29 tools prefixed
  `opusclip_`. Most-used: `submit_project`, `list_clips` (virality-ranked),
  `get_transcript` (per-word timing), `get_editing_script` /
  `apply_editing_script` (trims, caption fixes, retiming),
  `create_censor_job`, `export_clip` (HD/4K/Premiere XML),
  `create_social_copy_job`, `schedule_publish`. Clients: Claude, Cursor,
  ChatGPT, VS Code. Claimed: 1B+ clips, ~90s for 10 clips.
- What was **not** established: the clipping model itself (ClipAnything /
  ReframeAnything internals), virality-score formula, and any server code.
  All of that lives behind login on `clip.opus.pro` and is proprietary.
  The rebuild below therefore re-implements the *observable behaviour*
  with local, explainable methods — it does not copy OpusClip code.

## 2. Rebuild scope — what Abdullah asked for

1. Long video → clips
2. Animated captions
3. YouTube / Twitch / link → clips maker
4. AI video dubbing
5. (skipped in his list — no #5)
6. Social scheduler
7. Editor
8. Thumbnail generator
9. MCP

Honest limits for v1 (local, no paid cloud keys):
- Clip selection is transcript + loudness + hook scoring, not a trained
  virality model. It finds good moments; it will not match OpusClip's
  scoring exactly.
- Reframe is centre-crop to 9:16 with optional face-centre bias when a
  face is detectable; no full object tracking in v1.
- Dubbing is translate-transcript + TTS voice-over mixed over the clip
  (edge-tts when online; offline fallback = original audio + translated
  captions). No voice cloning.
- Scheduler queues posts and builds the publish package (video + title /
  description / hashtags + time). Actually posting to YouTube/TikTok/
  Instagram needs the user's own OAuth accounts connected — v1 ships
  the queue + YouTube upload as an optional, credentials-gated step,
  and never pretends a post went out when it did not.
- No user accounts / billing / teams in v1. Single-user local app.

## 3. Tech stack (chosen for Abdullah's machine)

- **Backend: Python 3.12 + FastAPI** — same as PeakCut, which he already
  runs on Windows (`pip install -r requirements.txt`,
  `python -m studio.server`). No Node build step.
- **Video: ffmpeg** (already installed here, v8.1.2), **yt-dlp** for
  YouTube/Twitch links (PeakCut's Android-client fallback + cookies
  handling reused where it fits).
- **Speech: faster-whisper, tiny/base model, local** — transcript with
  word timings for captions + clip scoring. No API key.
- **Captions: ASS subtitles burned in with ffmpeg** — word-by-word
  highlight (karaoke style), 3 templates (Bold, Karaoke, Minimal).
- **Thumbnails: ffmpeg frame grab + Pillow** text overlay, 1280×720.
- **Dubbing: edge-tts (online, free) + ffmpeg mix**; translation via
  deep-translator when online, else captions-only fallback.
- **Data: SQLite + files on disk** (jobs, clips, schedules, thumbnails).
- **Frontend: plain HTML/CSS/JS single page** in `web/` — upload/paste
  link, job list, clip player, simple timeline editor (trim, caption
  edit, style pick), scheduler form, thumbnail preview. No framework.
- **MCP: Python `mcp` package, stdio server** (`mcp_server.py`) exposing
  8 tools mirroring OpusClip's most-used ones: `clip_video`,
  `list_clips`, `get_transcript`, `caption_clip`, `reframe_clip`,
  `make_thumbnail`, `dub_clip`, `schedule_post`.
- **Jobs: background worker thread** in the FastAPI app, same pattern as
  PeakCut `studio/jobs.py` — create job → transcribe/score → render →
  done/failed with a plain-language error.

## 4. Project layout (in `~/workspace/clipforge/`)

```
clipforge/
  README.md            setup + how to run (Windows + Linux)
  requirements.txt     fastapi, uvicorn, faster-whisper, yt-dlp, edge-tts,
                       deep-translator, Pillow, mcp, python-multipart
  .env.example
  studio/
    __init__.py  config.py  db.py  server.py   # FastAPI app, /api/*, static web/
    media.py     # yt-dlp / Twitch fetch, ranged audio, probe
    transcribe.py# faster-whisper word timings
    score.py     # clip scoring: transcript density, hooks, loudness
    captions.py  # ASS generation, 3 templates, word highlight
    render.py    # ffmpeg clip render 1080x1920, captions, reframe
    dub.py       # translate + edge-tts + mix
    thumbs.py    # thumbnail frames + Pillow overlay
    schedule.py  # SQLite scheduler queue, due-checker
    editor.py    # trim / caption-edit / re-render operations
    mcp_server.py# MCP stdio server, 8 tools
  web/
    index.html  app.js  style.css
  tests/
    test_smoke.py  # API health, thumbnail, caption ASS generation
  data/  (created at runtime, gitignored)
```

API (v1):
- `POST /api/jobs` {url|file, max_clips, clip_length, lang} → job
- `GET /api/jobs`, `GET /api/jobs/{id}` → status + clips
- `GET /api/clips/{id}/video`, `.../thumbnail`
- `POST /api/clips/{id}/edit` {start,end,caption_style,caption_text}
- `POST /api/clips/{id}/dub` {target_lang}
- `POST /api/clips/{id}/thumbnail` {title}
- `POST /api/schedule` {clip_id, platform, publish_at, title, description}
- `GET /api/schedule` → queue

## 5. Build order

1. Scaffold + config + DB + server + web shell (job create/list).
2. Media fetch (YouTube/Twitch/link/file) + probe — reuse PeakCut
   lessons (ranged fetch, Android client fallback, friendly errors).
3. Transcribe + score + clip selection.
4. Render 1080×1920 + animated captions (3 templates).
5. Editor (trim / caption edit / re-render) + thumbnails.
6. Dubbing.
7. Scheduler queue (+ YouTube upload stub, credentials-gated).
8. MCP server (8 tools) + smoke tests.
9. End-to-end test on a short public video, fix bugs, zip + setup docs.

## 6. Test plan

- Smoke: server starts, `/api/health` 200, ASS + thumbnail generation
  without a video.
- E2E: short public YouTube video (Rick Astley, as used for PeakCut —
  known-good) → job → ≥1 clip at 1080×1920 with burned captions →
  thumbnail 1280×720 → edit (trim 2s) re-renders → dub to Spanish
  (online) or falls back cleanly offline → schedule entry created →
  MCP `clip_video` + `list_clips` work over stdio.
- Every failure must surface a plain-language error, never a bare 500.
