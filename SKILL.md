---
name: llm-wiki-import
description: 导入到 wiki：原样投递就绪内容并确认 LLM Wiki 投递状态。
version: 3.11.1
---

# LLM Wiki 原样投递

仅在用户已要求入库、内容已就绪时使用。只负责传输、投递参数、状态确认和异常上报；不提取、不改写、不补事实、不拆分内容。实验任务也只按上游已给定的输入、项目和入口清单执行。

## 标准投递路径

用户指定“应用原生解析文件”时走下方原生入口；其余就绪内容按以下六步投递，无须事前 UI 确认：

1. **查状态**：读取 `~/.llm-wiki-import.json` 的 `clipServerUrl`，未配置则用 `http://127.0.0.1:19827`，赋给 `CLIP_URL`；执行 `curl -s "$CLIP_URL/status"`。端点不可达则报告并建议启动应用；首次使用按下方初始化保存配置。
2. **选项目**：执行 `curl -s "$CLIP_URL/projects"`；按用户指定名称取返回的 `path`，未指定用唯一的 `current: true`；歧义才询问。`title` 取用户指定值／正文首个 `#` 标题／文件名，`url` 用已交付来源 URL，无则 `""`。执行下方独立去重步骤：只读查目标 `raw/sources/` 同 URL，命中则警告后继续，不改正文。
3. **落盘 payload**：先用 `execute_code` 的 `json.load` 读取并留存目标 `.llm-wiki/ingest-cache.json` 基线及提交时间；文件不存在可按空基线，读取/解析失败不能当空。将 `title`、`url`、原样 `content`、`projectPath` 编码为 JSON，用 `write_file` 写到本次独立的临时 payload 文件。JSON 编码是传输封装，不是加工。
4. **POST 一次**：执行 `curl -s -X POST "$CLIP_URL/clip" -H 'Content-Type: application/json' -d @/tmp/payload.json`（替换为本次实际路径）。保留返回的 `ok`、精确 `path` 和错误；`ok: true` 只表示受理，不自动再发一次。
5. **等待**：独立调用 `terminal("sleep 30")`，随后检查 cache；可在约 3 分钟窗口内按此间隔复查，不在 `execute_code` 内使用 `time.sleep()`。
6. **验证本文件 cache**：用 `execute_code`、`json.load` 重读 cache，按返回源路径定位记录（常见规则：去掉 `raw/sources/` 前缀，保留子目录）。相对基线新增或更新、且 `filesWritten` 非空，即报告“投递成功（cache 已确认）”，附项目、源路径和 `filesWritten`。不要求读取 UI 终态；约 3 分钟仍无证据，按异常上报段检查并有界跟进一次后收尾，不重投。

## 一、投递

### 传输封装不等于加工

- **传输编码 ≠ 加工**：JSON 对引号、换行、反斜杠和中文的编码，及 payload 落盘，都是为使 App 解码后收到同一份 `content`；不能因此拒绝封装。App 自行添加的存储元数据不由本 skill 改写。
- 保留原文字符、换行、代码、链接及已交付说明，不 trim、不改换行格式、不追加标题到正文。只从正文读取标题来确定 `title` 参数，不修改正文。文件读入时避免隐式换行转换。
- **首选 `write_file` 落盘 JSON → 裸 `curl -d @文件`**。有过 >2KB JSON 经 Python subprocess 参数传递后 stdout 静默为空的事故；heredoc／内联解释器提交也触发过审批门并在无人应答时卡住。不要退回长内联 JSON 或 Python subprocess 提交。
- 临时 payload 使用本次独立路径，勿覆盖其他任务文件。可读回 JSON 核对解码后的 `content` 与输入一致；JSON 文本中的转义形式不必与正文表面相同。
- 返回空 stdout／网络超时不能据此断言未受理，更不能补 POST 探测；转入只读状态排查。

### 原生文件入口保护

用户要求“直接用 LLM Wiki 解析原文件”时，使用应用支持的原生文件入口提交该文件；不把 HTML/PDF 转成 Markdown 冒充原生解析，不擅自改图片处理配置。原生入口无需 clip-server 初始化。

从该入口可用的项目列表选择目标，先做同 URL 去重提醒并记录 cache 基线；导入一次后记录实际源路径，使用同样的 cache 成功判据。入口不可用或源身份无法确定时报告限制，不静默换成 POST，不猜测成功。原生入口中的项目选择不构成 POST 路径的 UI 前置要求。

### 首次初始化（已有 configured: true 则跳过）

1. 查 `/status`，确认实际可用端点；不可达则建议启动应用并报告状态。
2. 查 `/projects`；没有项目则提示先建项目，不发送内容。
3. 可选测试只使用测试项目，**禁止向活跃库投合成测试稿**；无测试项目则跳过，以首次真实投递验证。
4. 用 `write_file` 保存 `~/.llm-wiki-import.json`：`{"clipServerUrl":"http://127.0.0.1:19827","configured":true}`，地址须替换为实际验证值；只保存端点和配置标记，不固定项目。初始化完成不等于内容已投递成功。

## 二、投递参数与去重

| 参数 | 确定规则 |
|---|---|
| `title` | 用户指定 → 正文首个一级标题 → 文件名；均无则向上游索取，不凭空概括标题 |
| `url` | 用户或上游已提供的来源 URL；无则空字符串，不自行提取或展开 |
| `content` | 就绪内容原封不动，不再选择加工档位 |
| `projectPath` | 每次查 `/projects`；用户指定名称按不区分大小写匹配，只有部分匹配或多个候选时澄清；未指定用唯一 `current: true`，不硬编码项目路径 |

`projectPath` 选择写入目标，不保证后台处理；`POST /project` 只更新服务端项目标记，**不证明前端切换**。保留此限制即可，不要求 POST 前操作或核验 UI；无 cache 证据时先排查项目不匹配。

### 独立步骤：投递前只读去重

选定项目后、提交前，使用 `search_files` 或固定字符串搜索，在目标 `raw/sources/` 查相同 URL；已交付的短链/展开形式可一起查，不为去重新增提取。URL 为空则注明无法按 URL 检查。

命中时告知“同 URL 已有来源：<实际路径>；本次按请求继续投递”，**警告后继续，不拦截、不改内容、不删除旧文件**。搜索受阻则说明未完成去重检查并继续。记录服务器实际新源路径，不预设重复文件的后缀；用户主动取消除外。

## 三、投递状态确认

验证使用 `execute_code` 中的 Python `json.load`；等待始终用独立 `terminal("sleep 30")`。例如将 `cache_path` 设为目标项目的 `.llm-wiki/ingest-cache.json` 后读取：

```python
import json
with open(cache_path, encoding="utf-8") as f:
    cache = json.load(f)
```

- 提交前留存基线，提交后只匹配本次返回的源身份；常见结构为 `cache["entries"][源身份]`。例如 `raw/sources/folder/a.md` 对应 `folder/a.md`，不能仅按标题或 basename 匹配。
- 本源记录相对基线**新增或更新**，且 `filesWritten` 为非空列表，就是本 skill 的投递成功判据。不增加摘要正文、frontmatter 回链或全局队列为空等门槛。
- App「原始资料」界面的已摄取标记来自前端快照（sources-view 挂载时读一次，之后只在 dataVersion 变化时重拉）；外部 POST 投递完成后该标记可能仍显示「未摄取」，这不是投递失败——以本源 cache 记录为准，并向用户说明刷新面板或重启 App 即可更新显示，不据此重投。
- 只有 POST `ok`：报告“已受理，尚无本源 cache 证据”；存在旧记录但未更新，不算本次成功。读不到 cache、JSON 解析失败或版本格式未知，均明确报告证据不足，不能当空文件或成功。
- App 若额外暴露可读的本源终态，可一并报告；没有则 cache 即实际最高可验证级，不把 CLI 读不到的 UI 终态设为必达目标。成功口径仅指投递及 cache 确认，不承诺语义正确、摘要完整或全部后台任务完成；若另有失败/警告记录也如实附上。
- 若 `execute_code` 被审批门阻塞，可用可用的只读 `search_files`／`read_file` 获取本源记录；片段不足以判断新旧或 `filesWritten` 时报告未验证，不以工具受阻为由重投。

## 四、异常上报与停止条件

1. **约 3 分钟仍无 cache 证据**：只读检查项目匹配、目标 LLM 配置、worker/队列状态及现有错误；不修改队列、缓存或内容，不读取密钥值来写报告。不要轮询会取走待处理条目的 `/clips/pending`。
2. **有界跟进一次**：可建议用户重启 LLM Wiki；随后独立 `terminal("sleep 30")` 并重读一次状态/cache。仍无证据就报告现状收尾，不无限轮询，不自动重启、Retry/Resume、重新触发 ingest 或重新导入。
3. **报告内容**：目标项目、实际源路径（未获得则明说）、POST 受理情况、本源 cache 证据、已知错误、未确认状态和重启建议；不把排查时间到点说成已确定失败。
4. **无重复 POST 红线**：禁止为探测恢复或空 stdout 自动重投。用户显式要求重投/重跑除外：执行该已授权请求一次，记录为新投递，不宣称旧任务因此已修复。
5. 参数无效、应用未启动、目标不存在等明确错误直接报告；文件读取或 JSON 编码问题在发出 POST 前解决。内容修订、补充或拆分全部退回 `wiki-draft-composition`，本 skill 不处理。

接口字段和源身份示例见 [API 参考](references/api-reference.md)；版本机制与实验勘误见 [共享参考](references/ingest-pipeline-behavior.md)。常规投递无需加载实验记录；加工与产物质量审查不属于本 skill。
