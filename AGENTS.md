# AGENTS.md

Guidance for AI coding agents (Codex, Cursor, Copilot, Gemini CLI, Claude Code, etc.) working in this repository. Human-facing docs are in [README.md](README.md).

## What this project is

YouTube Channel Transcript Fetcher bulk-exports the transcripts (subtitles / captions) of every video on a YouTube channel into one LLM-ready plain-text file. It has a CLI (`youtube_channel_export`), a local FastAPI web UI (`youtube_channel_web`), view-count and age filters, and proxy support for YouTube IP blocks.

It is a fork of [jdepoix/youtube-transcript-api](https://github.com/jdepoix/youtube-transcript-api) (MIT). The upstream single-video library lives in `youtube_transcript_api/` (`_api.py`, `_transcripts.py`, `formatters.py`, `proxies.py`, ...). Everything this fork adds lives in `youtube_transcript_api/channel/`.

## Commands

```bash
# Install (Poetry is the source of truth; CI uses Poetry 2.x)
poetry install --with channel,test,dev

# Tests
poetry run pytest youtube_transcript_api

# Format and lint (CI runs both with --check / as-is)
poetry run ruff format youtube_transcript_api
poetry run ruff check youtube_transcript_api

# Coverage gate (must stay at 100% for the upstream library)
poetry run coverage run -m pytest youtube_transcript_api
poetry run coverage report -m --fail-under=100

# Run
poetry run youtube_channel_export "https://www.youtube.com/@ChannelName" --max-videos 3 -o smoke.txt
poetry run youtube_channel_web   # serves http://127.0.0.1:8080
```

`channel`, `test` and `dev` are **Poetry dependency groups, not pip extras**. `pip install -e ".[channel]"` does not install yt-dlp, FastAPI or uvicorn. With plain pip, install them explicitly: `pip install -e . yt-dlp fastapi "uvicorn[standard]"`.

Python 3.9 to 3.14 is supported (CI matrix). Don't use syntax newer than 3.9 without `from __future__ import annotations`.

## Architecture (`youtube_transcript_api/channel/`)

Data flows one way: scrape → filter → fetch → export.

| Module | Role |
|--------|------|
| `urls.py` | Parse/validate channel URLs: `@handle`, `/channel/UC...`, uploads playlist `list=UU...`. `/c/` and `/user/` are rejected. |
| `scraper.py` | List channel uploads with yt-dlp; enrich metadata (views, dates), optionally via `youtube_data_api.py` when `YOUTUBE_API_KEY` is set |
| `filter.py` | Age gate, view-count percentile cutoff, view floor, max-videos cap |
| `fetcher.py` | Threaded transcript download via `YouTubeTranscriptApi`, with proxy and 429 handling |
| `export.py` | Plain-text formatting (`compact` default, `verbose`) |
| `pipeline.py` | `run_pipeline()` orchestrates the four stages and progress reporting |
| `models.py` | Config dataclasses (`ScrapeConfig`, `FilterConfig`, `ExportConfig`) and exceptions |
| `proxy.py`, `env.py` | Proxy config from env/CLI; `.env` / `.env.local` loading |
| `cli.py` | `youtube_channel_export` entry point |
| `web/app.py`, `web/static/index.html` | FastAPI app + single-page UI; jobs run in a child process (`worker.py`) to keep the server's memory low |

## Conventions and gotchas

- Tests live in `youtube_transcript_api/test/`. Channel tests use fixtures in `test/assets/channel/` and must not hit the network. Mock yt-dlp and HTTP instead.
- Coverage `--fail-under=100` applies to the upstream library. `channel/*` is omitted from coverage but still has tests, so add or update them with any change.
- Never commit secrets. `YOUTUBE_API_KEY`, `WEBSHARE_PROXY_*` and `HTTP(S)_PROXY` belong in `.env.local` (gitignored). See `.env.example`.
- The web UI binds to `127.0.0.1` only and has no auth. Keep it that way unless auth is added.
- Proxy credentials are read server-side from env, never sent from the browser.
- `IpBlocked` / `RequestBlocked` / HTTP 429 errors come from YouTube rate limiting, not bugs. They are expected from cloud IPs.
- Keep upstream modules close to upstream so future merges from jdepoix/youtube-transcript-api stay easy.
