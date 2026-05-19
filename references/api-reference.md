# LLM Wiki Clip Server API Reference

The LLM Wiki Tauri desktop app embeds a HTTP daemon (`clip_server.rs`, powered by `tiny_http`) on port 19827 by default. It was originally built for the Chrome Web Clipper extension but serves as the primary programmatic interface for any external tool.

## Base URL

```
http://127.0.0.1:19827
```

All endpoints use JSON for request/response bodies.

---

## Endpoints

### GET /status

Health check — verify the app is running and the clip server is responsive.

**Request:** No body.

**Response:**
```json
{
  "ok": true
}
```

**Usage:**
```bash
curl -s http://127.0.0.1:19827/status
```

---

### GET /projects

List all knowledge base projects known to the app.

**Request:** No body.

**Response:**
```json
{
  "ok": true,
  "projects": [
    {
      "current": true,
      "name": "My Knowledge Base",
      "path": "/Users/user/Documents/wiki-projects/my-kb"
    },
    {
      "current": false,
      "name": "Work Notes",
      "path": "/Users/user/Documents/wiki-projects/work"
    }
  ]
}
```

The `current: true` field marks the currently active project in the app UI.

**Usage:**
```bash
curl -s http://127.0.0.1:19827/projects
```

---

### GET /project

Get the current project's path.

**Request:** No body.

**Response:**
```json
{
  "ok": true,
  "path": "/Users/user/Documents/wiki-projects/my-kb"
}
```

**Usage:**
```bash
curl -s http://127.0.0.1:19827/project
```

---

### POST /project

Switch the current project in the app.

**Request:**
```json
{
  "path": "/Users/user/Documents/wiki-projects/my-kb"
}
```

**Response:**
```json
{
  "ok": true
}
```

**Usage:**
```bash
curl -s -X POST http://127.0.0.1:19827/project \
  -H 'Content-Type: application/json' \
  -d '{"path":"/Users/user/Documents/wiki-projects/my-kb"}'
```

---

### POST /clip

Import content into a project. This is the main endpoint — it writes the file and auto-triggers the ingest pipeline.

**Request:**
```json
{
  "title": "Article Title",
  "url": "https://example.com/article",
  "content": "# Article Title\n\nFull markdown content here...",
  "projectPath": "/Users/user/Documents/wiki-projects/my-kb"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| title | string | Yes | Article title. Used to generate the filename slug. |
| url | string | No | Original source URL. Stored in frontmatter for reference. |
| content | string | Yes | Markdown content of the article. |
| projectPath | string | Yes | Absolute path to the target project directory. |

**Response:**
```json
{
  "ok": true,
  "path": "raw/sources/article-title-20260519.md"
}
```

**What happens server-side:**
1. Generates a URL-safe slug from the title (kebab-case, max 50 chars)
2. Appends date: `{slug}-{YYYYMMDD}.md`
3. Writes to `{projectPath}/raw/sources/` with clip frontmatter:
   ```markdown
   ---
   title: Article Title
   url: https://example.com/article
   date: 2026-05-19
   source: clip
   ---
   
   # Article Title
   
   Full markdown content here...
   ```
4. Enqueues an IngestTask in `{projectPath}/.llm-wiki/ingest-queue.json`
5. Auto-triggers the two-step chain-of-thought ingest pipeline:
   - Step 1: Source → generates a wiki source summary
   - Step 2: Source → generates entities and concepts

**Deduplication:** If a file with the same name already exists, a numeric suffix is appended (-2, -3, ...).

**Usage:**
```bash
curl -s -X POST http://127.0.0.1:19827/clip \
  -H 'Content-Type: application/json' \
  -d '{"title":"My Article","url":"https://example.com","content":"# My Article\n\nContent here.","projectPath":"/path/to/project"}'
```

For content with special characters, use the Python approach (see SKILL.md).

---

### GET /clips/pending

Drain and return any pending clips from the internal queue.

**Request:** No body.

**Response:**
```json
{
  "ok": true,
  "clips": []
}
```

---

## Response Format

All endpoints use a consistent envelope:

```json
{
  "ok": true,
  // ... endpoint-specific fields
}
```

On error:
```json
{
  "ok": false,
  "error": "Description of what went wrong"
}
```

## Error Scenarios

| Scenario | HTTP Status | Response |
|----------|-------------|----------|
| App not running | Connection refused | (no response) |
| Invalid JSON body | 400 | `{"ok":false,"error":"..."}` |
| Missing required field | 400 | `{"ok":false,"error":"..."}` |
| Project path not found | 200 | `{"ok":false,"error":"..."}` |

## Port Configuration

Default port: 19827

If the user has configured a different port in LLM Wiki settings, update `clipServerUrl` in `~/.llm-wiki-import.json` accordingly.

## File System Structure

After a successful clip, the project directory looks like:

```
<projectPath>/
├── .llm-wiki/
│   ├── ingest-queue.json    # Current queue state ([] when all done)
│   └── ingest-cache.json    # History of processed items
├── raw/
│   └── sources/
│       └── article-slug-20260519.md   # The clipped source file
└── wiki/
    ├── overview.md
    ├── log.md
    ├── sources/
    │   └── article-slug-20260519.md   # Generated source summary
    ├── entities/
    │   └── ...                         # Extracted entities
    └── concepts/
        └── ...                         # Extracted concepts
```

## Ingest Queue Format

`ingest-queue.json` contains an array of IngestTask objects:

```json
[
  {
    "projectId": "uuid-here",
    "filePath": "raw/sources/article-slug-20260519.md",
    "status": "pending",
    "retryCount": 0,
    "addedAt": 1778685859381
  }
]
```

Status values: `pending` → `processing` → (consumed/removed)

On failure: `retryCount` increments (max 3 retries), then the task is removed.

## Ingest Cache Format

`ingest-cache.json` records all successfully processed items:

```json
{
  "entries": {
    "article-slug-20260519.md": {
      "hash": "sha256-hash-of-content",
      "timestamp": 1778685859381,
      "filesWritten": [
        "wiki/sources/article-slug-20260519.md",
        "wiki/entities/some-entity.md",
        "wiki/concepts/some-concept.md"
      ]
    }
  }
}
```
