---
name: llm-wiki-import
description: Import or save content into wiki/knowledge base systems. Use when user wants to move articles, web pages, files, clipboard content, or notes INTO a personal knowledge management tool — especially LLM Wiki, but also generic wikis or AI-powered knowledge systems. Trigger on intent indicators like "import to wiki", "save to knowledge base", "add to my wiki", "clip article", "migrate notes", "backup to wiki". Watch for explicit mentions (LLM Wiki, Tauri wiki) AND generic wiki/knowledge base language. Works across English and Chinese — 导入、保存、迁移、知识库、剪贴板. Focus on the act of bringing external content INTO a knowledge system. Not for deploying wiki software, creating new wiki projects, organizing existing notes, or general web browsing.
version: 3.1
---

# LLM Wiki Content Import

Import articles, files, and web content into LLM Wiki via the built-in clip server HTTP API.

## When to Use

- User asks to import an article, file, or content into LLM Wiki
- User wants to add something to their knowledge base managed by LLM Wiki
- User asks to clip a web page into their wiki
- User wants to save content for AI-powered search and retrieval
- User mentions "import to wiki", "clip to wiki", "add to knowledge base"

## How It Works

LLM Wiki embeds a tiny_http-based HTTP daemon (`clip_server.rs`) originally built for the Chrome Web Clipper extension. This same API is the best programmatic import path — a single `POST /clip` call handles everything:

1. Writes the Markdown file to `raw/sources/` with proper frontmatter
2. Enqueues it in the ingest queue
3. Auto-triggers the two-step chain-of-thought ingest pipeline (source → entities → concepts)

No manual file writes, no queue manipulation, no app restart needed.

---

## First-Time Setup

Before using this skill for the first time, check if the user has already configured it:

```bash
cat ~/.llm-wiki-import.json 2>/dev/null
```

If the file exists and contains `"configured": true`, skip setup and proceed to Import Steps.

If the file does NOT exist, walk the user through this one-time setup:

### Setup Step 1: Check if LLM Wiki is running

```bash
curl -s http://127.0.0.1:19827/status
```

- Returns `{"ok":true,...}` → app is running, proceed to Step 2.
- Connection refused → ask the user to open LLM Wiki app, then retry.
- If the user's clip server runs on a different port, note it for the config.

### Setup Step 2: Verify projects exist

```bash
curl -s http://127.0.0.1:19827/projects
```

Returns:
```json
{"ok":true,"projects":[{"current":true,"name":"My Project","path":"/path/to/project"},...]}
```

- If projects exist → tell the user which ones are available, proceed to Step 3.
- If no projects → ask the user to create a project in LLM Wiki first, then come back.

### Setup Step 3: Test import

Send a tiny test clip to verify end-to-end:

```bash
curl -s -X POST http://127.0.0.1:19827/clip \
  -H 'Content-Type: application/json' \
  -d '{"title":"Setup Test","url":"","content":"# Setup Test\n\nThis is a test clip to verify the import pipeline.","projectPath":"<path-from-step-2>"}'
```

- Returns `{"ok":true,"path":"raw/sources/setup-test-YYYYMMDD.md"}` → success, proceed to Step 4.
- Returns error or fails → see Troubleshooting section below.

### Setup Step 4: Save configuration

Write the config file:

```bash
cat > ~/.llm-wiki-import.json << 'EOF'
{
  "clipServerUrl": "http://127.0.0.1:19827",
  "configured": true
}
EOF
```

Tell the user: "Setup complete! You can now import content anytime. Which project to import into will be chosen dynamically each time — no need to update this config when you add new projects."

---

## Import Steps

### Step 1: Read config and check app status

```bash
# Read config (fails gracefully if missing → triggers setup)
CLIP_URL=$(python3 -c "import json; c=json.load(open('$HOME/.llm-wiki-import.json')); print(c['clipServerUrl'])" 2>/dev/null || echo "http://127.0.0.1:19827")

# Check app is running
curl -s "$CLIP_URL/status"
```

If status check fails, ask the user to open LLM Wiki app.

### Step 2: Get available projects and select target

```bash
curl -s "$CLIP_URL/projects"
```

Returns all projects with their paths. The `current: true` project is the active one.

**Always use `projectPath` in the clip request** — this is the primary mechanism. You do NOT need to call `POST /project` unless the user wants to change the app UI's active project.

How to select the target project:
- If the user specifies a project name → find its `path` from the list by name
- If the user says "current" or doesn't specify → use the `current: true` entry's path
- If the user wants to switch the app's active project (rare) → call `POST /project`

**Project name matching:** Match case-insensitively. If no exact match, look for partial matches and ask the user to confirm. Example: user says "work" → matches "Work Notes".

This design means:
- New projects appear automatically (no config update needed)
- Switching between projects is seamless (just pass the right path)
- Deleted projects disappear from the list naturally

### Step 3: Acquire content

See "Content Acquisition Methods" below.

### Step 4: Send to LLM Wiki via POST /clip

```bash
curl -s -X POST "$CLIP_URL/clip" \
  -H 'Content-Type: application/json' \
  -d '{"title":"Article Title","url":"https://source-url.com","content":"# Article Title\n\nFull markdown content...","projectPath":"/path/to/project"}'
```

For content with special characters (quotes, newlines), use Python to build the payload:

```python
import json, subprocess

clip_url = "http://127.0.0.1:19827"  # read from config in practice
payload = json.dumps({
    "title": "Article Title",
    "url": "https://source-url.com/article",
    "content": "# Article Title\n\nContent with 'quotes' and \"double quotes\"...",
    "projectPath": "/path/to/project"
})

result = subprocess.run(
    ["curl", "-s", "-X", "POST", f"{clip_url}/clip",
     "-H", "Content-Type: application/json",
     "-d", payload],
    capture_output=True, text=True
)
print(result.stdout)
# Expected: {"ok":true,"path":"raw/sources/slug-YYYYMMDD.md"}
```

The clip server handles:
- Generating a URL-safe slug from the title (kebab-case, max 50 chars)
- Appending date: `{slug}-{YYYYMMDD}.md`
- Writing to `{projectPath}/raw/sources/` with clip frontmatter
- Auto-enqueueing for ingest (pending → processing → done)
- Deduplication: if file exists, appends numeric suffix (-2, -3, ...)

### Step 5: Verify processing

The ingest pipeline starts automatically after POST /clip returns. LLM processing takes 30-90 seconds depending on content length and model speed.

```bash
# Check if queue is drained (empty array = all done)
cat "<projectPath>/.llm-wiki/ingest-queue.json"
# [] = all consumed
# entries with "processing" = still running
# entries with "failed" = error occurred
```

**Verification shortcut — check the cache, not just the queue:** an empty queue does NOT prove your clip was processed. Look for your source file's key in `<projectPath>/.llm-wiki/ingest-cache.json` (structure is `{"entries": {"<filename>": {"hash", "timestamp", "filesWritten": [...]}}}`). If the latest cache timestamp is days old, the ingest worker is stalled — see Troubleshooting.

For detailed results, check the ingest cache:
```bash
cat "<projectPath>/.llm-wiki/ingest-cache.json"
# Each entry has: hash, timestamp, filesWritten (array of generated wiki paths)
```

Wait between checks using a separate `terminal("sleep 30")` call — never use `time.sleep()` inside execute_code.

**If ingest is stuck at "processing" after 3 minutes:** Check ingest-cache.json for the entry. If it shows no `filesWritten`, the ingest likely failed after exhausting retries (3 attempts). You can re-POST the same clip to retry. If it keeps failing, the content may be too long or the LLM model may be hitting output limits — suggest splitting the content into smaller clips.

---

## Content Acquisition Methods

Choose based on the source type:

**Decision tree:** If the user provides a URL → Web articles. If the user provides a file path → Local files. If the user says "clipboard" or "paste" → Clipboard. If the user pastes text directly → Plain text. If unclear → ask: "Is this from a URL, a file, or would you like to paste the content?"

### Title derivation
If the user doesn't specify a title: use the first `# heading` in the content, or the filename for local files, or ask the user. The `url` field is optional — pass an empty string `""` if there's no source URL.

### Local files
Read with `read_file`, pass content to POST /clip.

### Web articles
1. `browser_navigate` to the URL
2. Extract content via `browser_console`:
   ```js
   // Try the main content container first
   (document.querySelector('article') || document.querySelector('main') || document.body).innerText
   ```
3. If the page requires login or blocks automated access, ask the user to copy-paste the content.

### Clipboard (user pasted content)
```bash
pbpaste > /tmp/article.txt  # macOS
# or: xclip -selection clipboard -o > /tmp/article.txt  # Linux
```
Then `read_file` and POST /clip.

### Plain text / user-provided content
Use directly as the `content` field in POST /clip.

### Site-specific extraction tips

These are common patterns — adapt based on the actual page structure:

| Site type | Extraction selector |
|-----------|-------------------|
| Blog/article | `article` or `main` element |
| Documentation | `.content` or `article` |
| Twitter/X post | The `browser_navigate` snapshot already contains the full tweet text (page title + text nodes), the quoted tweet, and top replies — capture from there. `[data-testid="tweetText"]` often returns NOT FOUND without login; fallback to `document.body.innerText`. X posts are short, so expand into structured markdown (source attribution, quoted-tweet context, notable replies) to give the ingest pipeline enough substance |
| Paywalled sites | Try browser first; if blocked, ask user for text |

Always prefer `browser_console` JS extraction over `browser_snapshot` — snapshot truncates at ~8000 chars and misses full article content.

---

## Clip Server API Reference

Full API documentation is in `references/api-reference.md`. Summary:

| Method | Path | Purpose |
|--------|------|---------|
| GET | /status | Health check |
| GET | /project | Get current project path |
| POST | /project | Switch current project |
| GET | /projects | List all projects |
| POST | /clip | Import content (main endpoint) |

All responses use `{"ok": boolean, ...}` envelope.

---

## Configuration File

Location: `~/.llm-wiki-import.json`

```json
{
  "clipServerUrl": "http://127.0.0.1:19827",
  "configured": true
}
```

Only two fields:
- `clipServerUrl` — the clip server address (default: `http://127.0.0.1:19827`)
- `configured` — whether first-time setup has been completed

Project paths are NOT stored here — they are fetched dynamically from `GET /projects` every time. This way, new projects are automatically recognized without any config changes.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Connection refused on /status | App not running | Open LLM Wiki app, wait a few seconds, then retry /status |
| /status works but /projects empty | No projects created | Create a project in LLM Wiki UI first |
| POST /clip returns error | Invalid JSON or missing field | Ensure title, content, and projectPath are provided |
| POST /clip returns project not found | Wrong projectPath | Re-fetch projects with GET /projects and use exact path |
| Queue stuck at "processing" | LLM model timeout or error | Wait 2-3 minutes; the app auto-retries up to 3 times. See Step 5 for detailed recovery |
| POST /clip returns ok but ingest never fires (queue stays `[]`, no new entry in ingest-cache.json) | App-side ingest worker silently stalled — the app can keep serving /clip for days while the pipeline is dead. Diagnose by comparing the latest timestamp in ingest-cache.json; if it is days old, the worker is stalled | Restart the LLM Wiki app. On startup it re-scans `raw/sources/` and reprocesses the backlog. Do NOT re-POST as a retry — it writes a duplicate `-2` file without triggering ingest; delete duplicates |
| "Generation failed" in logs | Model output exceeded limits | Content may be too long; try splitting into smaller clips |
| 404 on all endpoints | Wrong port | Check if user changed the default port in LLM Wiki settings |

Check app process: `ps aux | grep llm-wiki` (process name is `llm-wiki`)

---

## Pitfalls

- Clip server only works when LLM Wiki app is running — always check `/status` first
- Do NOT manually write to `raw/sources/` — external file writes do NOT trigger the ingest pipeline
- Do NOT use `time.sleep()` inside `execute_code` to wait for processing — use separate `terminal("sleep N")` calls
- `browser_snapshot` truncates long content — always use `browser_console` JS extraction for full articles
- Some sites block automated browsing (captcha, login wall) — fall back to asking the user to copy-paste
- The ingest pipeline can fail if content is too short (< 100 chars) or triggers model output limits — provide meaningful content
- For content with special characters (quotes, unicode), use the Python subprocess approach rather than inline shell strings
- Python subprocess.run(["curl", ..., "-d", large_json]) can silently return empty stdout for large payloads (>2KB). Write payload to a temp file with `write_file` and use `curl -d @/tmp/payload.json` instead — this is far more reliable than passing JSON as a subprocess argument
- An empty ingest queue proves nothing by itself — a stalled worker processes nothing and the queue stays `[]`. The real success signal is a NEW entry with `filesWritten` in ingest-cache.json
- Re-POSTing a clip to "retry" a stalled pipeline creates duplicate `-2`/`-3` source files — restart the app instead, then delete duplicates

## Fallback: Direct Filesystem Write

If the clip server is unavailable (app not running and user declines to start it):

1. Write a Markdown file to `<projectPath>/raw/sources/`
2. The file will be picked up when the app restarts (the restore queue converts processing → pending entries)
3. This is less reliable — strongly prefer the clip server API
