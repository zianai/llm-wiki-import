# 审查结论待合并（2026-09-16 实测回归）

背景：今日经 clip server 完整导入一篇 X 长文（syndication API → CDP proxy 切片 → POST /clip → ingest 验证通过），以下是实测暴露的缺口。只做增量修改，不要重写无关段落，不要 commit（工作区已有未提交的 v3.1 改动）。

## 1. SKILL.md — 「Site-specific extraction tips」表格 Twitter/X 行

补充 X 长文（article）处理路径：

- syndication API（`cdn.syndication.twimg.com/tweet-result?id=<ID>`）响应含 `article` key 时，正文是 X 长文：该接口只返回 `title` + `preview_text`（约100字）+ `rest_id`，拿不到全文
- 长文正文地址可直接拼：`https://x.com/i/article/<rest_id>`，走 CDP proxy（登录态 Chrome）打开后台 tab
- 渲染后按 `document.body.innerText` 以约 3800 字符切片抓取拼接
- 收尾截断规则：推文发布超过约 24 小时时页面没有时间戳行，按时间戳正则截断会失败；改用 `text.rfind('<作者名>\n@<handle>\nFollow')` 截掉尾部作者资料卡（rfind 而非 find，正文开头也会出现作者名）
- 详细提取纪律（切片拼接、死路清单）在 x-content-extraction skill，此处一行指引即可

## 2. references/api-reference.md — ingest-queue.json 条目结构

实测（2026-09-16，v0.1.0）queue 条目字段如下，写入 API 文档：

```json
{
  "id": "ingest-<ms>-<rand>",
  "projectId": "<uuid>",
  "sourcePath": "raw/sources/<slug>-<YYYYMMDD>.md",
  "folderContext": "",
  "status": "pending | processing | failed",
  "addedAt": <epoch-ms>,
  "error": null,
  "retryCount": 0
}
```

## 3. SKILL.md — Step 5（Verify processing）补充

一次 `POST /clip` 会在 queue 里出现 **两条** 条目（source → entities 两步流水线链式触发），同一 clip 看到 2 条是正常现象，不是重复提交；两条都被消费、cache 出现带 `filesWritten` 的新条目才算完成。
