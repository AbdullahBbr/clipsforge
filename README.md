# ClipForge

Long video in, vertical clips out — a simpler, own-code rebuild of the
*behaviour* of an OpusClip-style app. Single-user, local only. No
accounts, no billing, no teams, and no OpusClip code, name, logo or assets.

What it does: paste a YouTube / Twitch link (or upload a video) →
transcript + scoring picks the best moments → renders 1080×1920 clips
with animated captions (3 templates) → edit (trim, caption text, style)
→ thumbnails 1280×720 → AI dubbing (translate + TTS, with a translated-
captions fallback when offline) → social scheduler queue → MCP server
with 8 tools.

Clip selection is transcript + loudness + hook scoring — an explainable
local method, not a trained virality model. Reframe is a centre crop to
9:16. Dubbing uses online edge-tts; it never voice-clones.

## Run it

### Windows
1. Install Python 3.12 (tick "Add Python to PATH") and ffmpeg
   (`winget install Gyan.FFmpeg`).
2. `pip install -r requirements.txt`
3. `python -m studio.server`
4. Open http://127.0.0.1:8765

### Linux (this VM)
```
python3 -m venv .venv --system-site-packages   # or plain: python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m studio.server
# open http://127.0.0.1:8765
```
The speech model (faster-whisper, ~150 MB) downloads once from
Hugging Face on first transcription. YouTube's own captions are used
first when a video has them, so many YouTube jobs need no download.

## MCP server

```
python -m studio.mcp_server        # stdio, for Claude / Cursor / etc.
```
Tools: `clip_video`, `list_clips`, `get_transcript`, `caption_clip`,
`reframe_clip`, `make_thumbnail`, `dub_clip`, `schedule_post`.

Example MCP client config (Claude Code):
```json
{"mcpServers": {"clipforge": {"command": "python", "args": ["-m", "studio.mcp_server"], "cwd": "/path/to/clipforge"}}}
```

## YouTube keeps refusing

If YouTube blocks downloads ("sign in to confirm you're not a bot"),
put a `cookies.txt` exported from your browser in the `data/` folder,
or set `CLIPFORGE_YT_COOKIES=/path/to/cookies.txt`. Uploading the video
file always works instead.

## Scheduler / posting — read this

Scheduling only **queues** a post (clip + title + description +
hashtags + time). Nothing is posted automatically. YouTube upload
needs your own OAuth credentials (`CLIPFORGE_YOUTUBE_CLIENT_SECRETS`);
without them a due post stays queued with a note saying why.
ClipForge never pretends a post went out when it did not.

## Known limits (v1)

- Long videos (>15 min): audio is fetched for scoring, then each clip's
  video is fetched as a section. The section cut lands on a keyframe,
  so captions on long-video clips can be off by about a second.
- Dubbing needs the internet (translation + TTS). Offline, you get
  translated captions over the original audio instead — the result
  says so. For fully offline translation, install Argos Translate
  (`pip install argostranslate`, then `argospm install translate-en_es`
  for each language pair you want); ClipForge uses it first when it is
  present, and falls back to the free online services otherwise. The
  free online translators are rate-limited per shared network, so on a
  busy shared IP they may say "try again in a few minutes" — that is
  the service, not a bug.
- If the server restarts (or an MCP client disconnects) while a job is
  running, the job is picked up again on the next server start instead
  of staying stuck at 0%.
- No object tracking in reframe; it is a centre crop.
- Tests: `python -m pytest tests/ -x -q` (or `python tests/test_smoke.py`).

## Layout

```
studio/  config, db, media (yt-dlp), transcribe (faster-whisper),
         score, captions (ASS), render (ffmpeg), thumbs (Pillow),
         dub, editor, schedule, jobs, server (FastAPI), mcp_server
web/     plain HTML/CSS/JS frontend
tests/   smoke tests
data/    jobs, clips, database (created at runtime)
```
