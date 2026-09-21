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

The `current: true` field marks the clip server's current project. A programmatic server-side switch alone does not establish that the frontend has opened that project; see the version-specific processing notes below.

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

Set the clip server's current project path. In the implementation linked below, this updates server-side state; it does not itself open the project in the frontend.

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

Import content into a project. This endpoint writes the raw file and publishes a pending clip. Frontend enqueueing and ingest depend on the active project, LLM configuration and worker state; an ok response confirms acceptance, not generated wiki pages.

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
4. Publishes the clip for the frontend watcher, which enqueues eligible clips in the ingest queue.
5. When the project and worker are ready, the two-step ingest pipeline runs:
   - Step 1: Analyze the source and produce analysis used by generation.
   - Step 2: Generate wiki files, including the source summary and applicable entity/concept pages.

Two queue records for one clip were observed in prior runs. This does not establish a fixed record count or a one-to-one mapping between queue records and the two LLM stages.

**Deduplication:** If a file with the same name already exists, a numeric suffix is appended (-2, -3, ...).

**Usage:**
```bash
curl -s -X POST http://127.0.0.1:19827/clip \
  -H 'Content-Type: application/json' \
  -d '{"title":"My Article","url":"https://example.com","content":"# My Article\n\nContent here.","projectPath":"/path/to/project"}'
```

实际投递使用 SKILL.md 的文件式路径：JSON 编码后用 `write_file` 落盘，再 `curl -d @文件`；不使用长内联 JSON 或 Python subprocess 提交。传输编码不改变解码后的 content，不属于内容加工。

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

After clip acceptance and verified generation, an example project layout is:

```
<projectPath>/
├── .llm-wiki/
│   ├── ingest-queue.json    # Current queue state (empty does not prove success)
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

`ingest-queue.json` contains an array of IngestTask objects. Entry structure as observed on 2026-09-16 (app v0.1.0):

```json
[
  {
    "id": "ingest-<ms>-<rand>",
    "projectId": "<uuid>",
    "sourcePath": "raw/sources/<slug>-<YYYYMMDD>.md",
    "folderContext": "",
    "status": "pending",
    "addedAt": 1778685859381,
    "error": null,
    "retryCount": 0
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| id | string | Task ID in the form `ingest-<epoch-ms>-<random>` |
| projectId | string | UUID of the target project |
| sourcePath | string | Source file path, relative to the project root |
| folderContext | string | Folder context hint (empty string for clipped content) |
| status | string | One of `pending`, `processing`, `failed` |
| addedAt | number | Epoch milliseconds when the task was enqueued |
| error | string \| null | Failure message; `null` until an attempt fails |
| retryCount | number | Failed-attempt counter |

Status values: `pending` → `processing` → (consumed/removed); `failed` marks an entry that errored.

Retry and retention behavior is version-dependent. In the fixed source revision linked below, exhausted tasks remain failed and restored work can require Resume; do not assume that disappearance from the queue means success or that restarting resumes all work automatically.

## Ingest Cache Format

`ingest-cache.json` 记录缓存的生成结果。本 skill 的投递成功判据是：本次源文件对应的记录相对提交前基线新增或更新，且 `filesWritten` 为非空列表。该口径不等于摘要完整、内容正确或所有后台任务结束：

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

For the documented source-identity format, normalize the POST response path and remove its `raw/sources/` prefix to obtain the `entries` key, retaining subdirectories. For example, `raw/sources/folder/article.md` maps to `folder/article.md`, not just `article.md`. Confirm the running version's mapping if its structure differs. A missing cache before the first submission is an empty baseline; a read/parse error is not.

验证时用 `execute_code` 的 `json.load` 读取，与提交前基线比较；等待用独立 `terminal("sleep 30")`。按上述本源 cache 判据即可报告“投递成功（cache 已确认）”，不把摘要正文、frontmatter 回链或 UI 终态设为必达门槛。若 App 暴露额外可读终态或警告则如实附报；否则 cache 是实际最高可验证级。摘要/派生页的事实与质量检查属于独立审查，不扩张本 skill 的成功条件。详见 SKILL.md“投递状态确认”。

## Version-specific processing and recovery

The following mechanisms were reviewed at upstream commit `e8082119649e6a8e1cf85eaf289adcabfdf39d4e`; this does not establish that the installed app uses the same implementation:

- [clip-watcher.ts](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/clip-watcher.ts): the watcher enqueues clips for the frontend's current project when an LLM is usable. Passing another projectPath does not by itself guarantee background processing.
- [clip_server.rs](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src-tauri/src/clip_server.rs): reading `/clips/pending` drains pending clips; do not poll it as a diagnostic substitute for the app watcher. POST /project updates the server-side project marker, not proof of a frontend switch.
- [ingest-queue.ts](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/ingest-queue.ts): restored work may require Resume, and exhausted failures are retained. 这是机制说明，不是本 skill 自动恢复授权；本 skill 只读排查、建议重启并有界跟进一次，不自动 Resume/Retry、触发 ingest 或重投。
- [source-identity.ts](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/source-identity.ts), [ingest-cache.ts](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/ingest-cache.ts), and [ingest.ts](https://github.com/nashsu/llm_wiki/blob/e8082119649e6a8e1cf85eaf289adcabfdf39d4e/src/lib/ingest.ts): source identity, artifact verification and task completion are distinct. Non-empty filesWritten can lack the source summary, and cache precedes optional embeddings.

`POST /project` 不证明前端切换，但不是投递前 UI 核验要求；没有 cache 证据时才把项目不匹配作为首要嫌疑。约 3 分钟后排查项目、LLM 配置和 worker，建议重启并跟进一次，仍无证据就报告现状收尾。不要发明端点或将未知格式视为空 queue/cache。禁止重复 POST 探测；用户显式要求重投时只执行其已授权的新投递，并单独记录。技术机制不改变 SKILL.md 的操作边界。
