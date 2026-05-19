# llm-wiki-import

A Hermes Agent skill for importing articles, web pages, and local files into [LLM Wiki](https://github.com/nicepkg/llm-wiki) via the built-in clip server API.

## What It Does

LLM Wiki is a Tauri desktop app that turns Markdown files into an AI-searchable knowledge base. It embeds a small HTTP server (clip server) for the Chrome Web Clipper extension. This skill uses that same API to let Hermes Agent import content programmatically — a single `POST /clip` call handles file creation and auto-triggers the AI ingest pipeline.

Features:
- First-time setup with guided configuration
- Dynamic project detection (new knowledge bases are auto-recognized)
- Web page content extraction via browser automation
- Local file import
- Clipboard content import
- Ingest progress verification

## Requirements

- [Hermes Agent](https://github.com/nicepkg/hermes) installed and configured
- [LLM Wiki](https://github.com/nicepkg/llm-wiki) desktop app installed and running
- LLM Wiki clip server accessible (default: `http://127.0.0.1:19827`)

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

On first use, the agent will guide you through setup:
1. Checks if LLM Wiki is running
2. Lists your existing projects
3. Sends a test clip to verify the pipeline
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
4. Acquires content from the specified source (web, file, clipboard)
5. Sends to `POST /clip` with title, content, URL, and target project path
6. Clip server writes file and auto-triggers AI ingest
7. Agent verifies processing completed via ingest queue

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Connection refused" | Open LLM Wiki app first |
| "No projects found" | Create a project in LLM Wiki UI |
| Ingest stuck at "processing" | Wait 2-3 min; auto-retries up to 3 times |
| "Generation failed" | Content may be too long; try smaller clips |
| Wrong port | Update `clipServerUrl` in `~/.llm-wiki-import.json` |

## License

MIT
