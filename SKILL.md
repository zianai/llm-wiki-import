---
name: llm-wiki-import
description: Import articles/content into LLM Wiki (Tauri desktop app) via the clip server API. Covers project discovery, content acquisition, API-based import with auto-ingest trigger, and verification.
version: 2.0
---

# LLM Wiki Article Import

Import articles/content into LLM Wiki (Tauri desktop app) via the clip server HTTP API.

## When to Use
- User asks to import an article, file, or content into LLM Wiki
- User wants to add something to their knowledge base managed by LLM Wiki

## Key Discovery: Clip Server API

LLM Wiki embeds a tiny_http-based HTTP daemon (clip_server.rs) for the Chrome Web Clipper extension. This same API is the best programmatic import path:

- Base URL: `http://127.0.0.1:19827`
- Key endpoint: `POST /clip` — saves a Markdown file to `raw/sources/` AND auto-triggers the ingest pipeline
- Other endpoints: `GET /status`, `GET /project`, `GET /projects`, `POST /project`

The clip server handles:
1. Writing file to `raw/sources/` with proper frontmatter
2. Enqueuing in the ingest queue with correct IngestTask format
3. Auto-triggering the two-step chain-of-thought ingest pipeline

**This means: no need to manually write files, no need to manipulate ingest-queue.json, no need to restart the app.**

## Prerequisites

1. LLM Wiki app must be running (check: `curl -s http://127.0.0.1:19827/status`)
2. If not running: `open -a 'LLM Wiki'`, wait a few seconds

## Import Steps

### 1. Identify target project

```bash
curl -s http://127.0.0.1:19827/projects
```

Returns: `{"ok":true,"projects":[{"current":true,"name":"...","path":"..."},...]}`

Known projects:
- "create" -> `/Users/a/Documents/Obsidian/daily/create/`
- "Renzhi" -> `/Users/a/Documents/Obsidian/LLM Wiki/Renzhi/`

The `current:true` project is the default target. To switch:
```bash
curl -s -X POST http://127.0.0.1:19827/project -H 'Content-Type: application/json' -d '{"path":"/path/to/project"}'
```

### 2. Acquire content

See "Content Acquisition Methods" section below.

### 3. Send to LLM Wiki via POST /clip

```python
import json, subprocess

payload = json.dumps({
    "title": "Article Title",
    "url": "https://original-source-url.com/article",
    "content": "# Article Title\n\nFull markdown content here...",
    "projectPath": "/Users/a/Documents/Obsidian/daily/create"
})

# Escape single quotes in payload for shell
escaped = payload.replace("'", "'\\''")
result = subprocess.run(
    f"curl -s -X POST http://127.0.0.1:19827/clip -H 'Content-Type: application/json' -d '{escaped}'",
    shell=True, capture_output=True, text=True
)
print(result.stdout)
# Expected: {"ok":true,"path":"raw/sources/slug-YYYYMMDD.md"}
```

Or via terminal tool:
```bash
curl -s -X POST http://127.0.0.1:19827/clip \
  -H 'Content-Type: application/json' \
  -d '{"title":"...","url":"...","content":"...","projectPath":"/Users/a/Documents/Obsidian/daily/create"}'
```

The clip server will:
- Generate a slug from the title (kebab-case, max 50 chars)
- Append date: `{slug}-{YYYYMMDD}.md`
- Write to `{projectPath}/raw/sources/` with clip frontmatter
- Auto-enqueue for ingest (status: pending -> processing -> done)
- Dedup: if file exists, appends numeric suffix (-2, -3, ...)

### 4. Verify processing

After POST /clip returns `{"ok":true}`, the ingest pipeline starts automatically. Check progress:

```bash
# Wait 30-60 seconds for LLM processing, then:
cat <project_path>/.llm-wiki/ingest-queue.json
# [] = all consumed; "processing" = still running; "failed" = error occurred
```

For detailed results, check `ingest-cache.json` — each entry has `hash`, `timestamp`, `filesWritten`.

Use `terminal("sleep 30")` as a SEPARATE call (not inside execute_code) to wait without timeout risk.

## Content Acquisition Methods (by source type)

- **Local files**: Read directly with `read_file`, pass content to POST /clip
- **Web articles (WeChat)**: `browser_navigate` to load, `browser_console` JS to extract. WeChat blocks curl.
- **X/Twitter articles**: `browser_navigate` to tweet URL, extract via `browser_console` JS
- **General web**: `browser_navigate` first, fallback to browser extraction
- **Clipboard**: If user pasted content, `terminal("pbpaste > /tmp/article.txt")` then read_file

### X/Twitter Article Extraction

1. `browser_navigate` to tweet URL (works without login for articles)
2. Extract via `browser_console`:
   ```js
   document.querySelector('article[data-testid="tweet"]').innerText
   ```
3. Use `browser_console` JS extraction — `browser_snapshot` truncates at ~8000 chars

### WeChat Article Extraction

1. `browser_navigate` to the WeChat URL
2. Immediately extract via `browser_console`:
   ```js
   (document.querySelector('#js_content') || document.querySelector('.rich_media_content')).innerText
   ```
3. If page expired/captcha: ask user to open in browser, Cmd+A, Cmd+C, then `pbpaste > /tmp/article.txt`

## Clip Server API Reference

| Method | Path | Purpose | Request Body |
|--------|------|---------|-------------|
| GET | /status | Health check | - |
| GET | /project | Current project path | - |
| POST | /project | Set current project | `{"path":"..."}` |
| GET | /projects | List all projects | - |
| POST | /projects | Bulk-set project list | `{"projects":[{name,path}]}` |
| GET | /clips/pending | Drain pending clips | - |
| POST | /clip | Save a web clip | `{"title","url","content","projectPath"}` |

All responses use `{"ok": boolean, ...}` envelope.

## Verification: How to Confirm Processing

1. **Check ingest-queue.json** — `[]` means all consumed
2. **Check ingest-cache.json** — entries keyed by source filename with `filesWritten` arrays
3. **Count wiki pages**: `find <project_path>/wiki -name '*.md' -not -path '*/.obsidian/*' | wc -l`
4. **Spot-check**: read a generated entity/concept page

### ingest-cache.json structure
```json
{
  "entries": {
    "source-filename.md": {
      "hash": "sha256...",
      "timestamp": 1778685859381,
      "filesWritten": ["wiki/sources/...", "wiki/entities/...", "wiki/concepts/..."]
    }
  }
}
```

## Pitfalls

- Clip server only works when LLM Wiki app is running — check `/status` first
- For articles with single quotes in content, properly escape the JSON payload for shell
- Do NOT manually write to `raw/sources/` and `ingest-queue.json` — use POST /clip instead. Manual file writes do NOT auto-trigger ingest (no file watcher for external writes)
- Do NOT use `time.sleep()` inside `execute_code` to wait for processing — use separate `terminal("sleep N")` calls
- `browser_snapshot` truncates long articles — always use `browser_console` JS extraction
- WeChat pages are single-use: navigating back triggers captcha
- LLM ingest can fail if content is too short (< 100 chars) or causes model to exceed output limits
- App process name: `llm-wiki` (check with `ps aux | grep llm-wiki`)

## Fallback: Direct Filesystem Write

If clip server is unavailable (app not running and user declines to start it):
1. Write Markdown file to `<project_path>/raw/sources/`
2. The file will be picked up when the app restarts (restoreQueue converts processing->pending)
3. This is less reliable — prefer the clip server API
