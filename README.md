# llm-wiki-import

A Hermes Agent skill for delivering ready content into [LLM Wiki](https://github.com/nashsu/llm_wiki) via the clip server API or a requested native file-import route.

## What It Does

本 skill 只将就绪内容原样投递到 LLM Wiki。POST ok 表示受理；本源 cache 记录相对提交前新增或更新且 filesWritten 非空，即为“投递成功（cache 已确认）”。不要求 CLI 读取 UI 终态，不承诺摘要或内容质量。

Features:
- First-time setup with guided configuration
- Dynamic project detection (new knowledge bases are auto-recognized)
- Delivery of prepared web/pasted content
- Native local file import when requested
- 本源 cache 验证、只读排查和有界异常上报

Extraction belongs to `x-content-extraction` / `web-access`; optional composition and fact-checking belong to `wiki-draft-composition`. `wiki-output-review` provides separate read-only checks of generated facts, provenance and links. Delivery success is not a content-quality judgment; a ready draft does not itself authorize importing it.

## Requirements

- [Hermes Agent](https://github.com/nicepkg/hermes) installed and configured
- [LLM Wiki](https://github.com/nashsu/llm_wiki) desktop app installed and running
- For API delivery, LLM Wiki clip server accessible (default: `http://127.0.0.1:19827`)

## Installation

Copy the skill directory to your Hermes skills folder:

```bash
cp -r llm-wiki-import ~/.hermes/skills/productivity/
```

Or clone directly:

```bash
git clone https://github.com/YOUR_USERNAME/llm-wiki-import.git ~/.hermes/skills/productivity/llm-wiki-import
```

## Usage

After installation, the skill activates automatically when you ask Hermes to import content into LLM Wiki.

### First Time

On first API use, the agent will guide you through setup (native file import skips clip-server setup):
1. Checks if LLM Wiki is running
2. Lists your existing projects
3. Optionally tests in a test project; otherwise verifies the first real import without a synthetic clip
4. Saves configuration to `~/.llm-wiki-import.json`

### Import Examples

```
# Import a web article
"帮我把这篇文章导入 LLM Wiki: https://example.com/article"

# Import a local file
"把这个文件导入到知识库: ~/Downloads/notes.md"

# Import pasted content
"把这段内容存到 LLM Wiki: [paste content]"

# Import to a specific project
"把这篇文章导入到我的 Work Notes 项目"
```

### Multi-Project Support

The skill dynamically fetches your project list from the API every time. When you create a new project in LLM Wiki, it's automatically available — no config changes needed.

If you have multiple projects, specify which one to use:
- "导入到 [project name]" → imports to that specific project
- No specification → imports to the currently active project

## Configuration

Config file: `~/.llm-wiki-import.json`

```json
{
  "clipServerUrl": "http://127.0.0.1:19827",
  "configured": true
}
```

Only two fields:
- `clipServerUrl` — clip server address (change if you use a custom port)
- `configured` — marks setup as complete

Project paths are NOT stored in config — they are fetched from the API dynamically.

## Project Structure

```
llm-wiki-import/
├── SKILL.md                    # Main skill instructions (agent reads this)
├── README.md                   # This file
├── LICENSE                     # MIT
├── references/
│   └── api-reference.md        # Full clip server API documentation
```

## How It Works

1. Agent checks `~/.llm-wiki-import.json` for config
2. If not configured → runs first-time setup
3. Fetches project list from `GET /projects`
4. Receives ready content from the user or extraction/composition skills
5. 投递前只读查同 URL，命中则警告并继续；write_file 落盘 JSON 后用 curl -d @文件提交一次，用户指定原生文件入口时不替换路线
6. 独立 sleep 后用 execute_code/json.load 验证本源新增或更新的 cache 记录及非空 filesWritten
7. 约 3 分钟无证据则排查项目匹配、LLM 配置和 worker，建议重启并有界跟进一次后报告；不自动重投或重触发 ingest

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Connection refused" | Open LLM Wiki app first |
| "No projects found" | Create a project in LLM Wiki UI |
| Ingest stuck at "processing" | 只读排查，建议重启，有界跟进一次后报告；不自动恢复或重投 |
| "Generation failed" | Record the actual error; hand necessary content revision/splitting to the composition skill |
| Wrong port | Update `clipServerUrl` in `~/.llm-wiki-import.json` |

## License

MIT
