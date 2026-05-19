---
name: llm-wiki-import
description: Import articles, web pages, and local files into LLM Wiki (Tauri desktop app) via the clip server HTTP API. Use this skill whenever a user wants to import content into their LLM Wiki knowledge base, add articles to their wiki, save web pages for later, clip content from the browser, or manage LLM Wiki projects. Also use when the user mentions "knowledge base", "wiki import", "clip to wiki", or asks to save content for AI-powered retrieval — even if they don't explicitly say "LLM Wiki".
version: 3.0
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

Then clean up the test file:

```bash
# Optionally remove the test clip's source file and wiki output
# (the test clip is harmless to keep, but you can offer to clean it up)
```

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

How to select the target project:
- If the user specifies a project name → find its path from the list
- If the user says "current" or doesn't specify → use the `current: true` entry
- If the user wants a different project → switch with `POST /project`:
  ```bash
  curl -s -X POST "$CLIP_URL/project" \
    -H 'Content-Type: application/json' \
    -d '{"path":"/path/to/desired/project"}'
  ```

The project path is passed as `projectPath` in the clip request. You do NOT need to switch the current project — just pass the correct path.

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

For detailed results, check the ingest cache:
```bash
cat "<projectPath>/.llm-wiki/ingest-cache.json"
# Each entry has: hash, timestamp, filesWritten (array of generated wiki paths)
```

Wait between checks using a separate `terminal("sleep 30")` call — never use `time.sleep()` inside execute_code.

---

## Content Acquisition Methods

Choose based on the source type:

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
| Twitter/X post | `[data-testid="tweetText"]` |
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
| Connection refused on /status | App not running | Open LLM Wiki app, wait a few seconds |
| /status works but /projects empty | No projects created | Create a project in LLM Wiki UI first |
| POST /clip returns error | Invalid JSON or missing field | Ensure title, content, and projectPath are provided |
| Queue stuck at "processing" | LLM model timeout or error | Wait 2-3 minutes; the app auto-retries up to 3 times |
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

## Fallback: Direct Filesystem Write

If the clip server is unavailable (app not running and user declines to start it):

1. Write a Markdown file to `<projectPath>/raw/sources/`
2. The file will be picked up when the app restarts (the restore queue converts processing → pending entries)
3. This is less reliable — strongly prefer the clip server API
