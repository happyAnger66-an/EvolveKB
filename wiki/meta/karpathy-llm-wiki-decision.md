---
title: 决策：Karpathy llm-wiki 方法（采纳/放弃/分歧）
date: 2026-09-06
type: decision
tags: [meta, method, decision]
source: "raw/2026-09-06-karpathy-llm-wiki-2026-04-来源-https-gist-github-com-karpathy.md"
---

## 背景 / 问题

Karpathy 2026-04 提出一套「LLM 驱动 wiki」的方法（gist：https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f），与我们自己的知识库 EvolveKB 直接相关。需要明确一条决策：对他的方法哪些**采纳**、哪些**放弃（没折入）**、哪些**和他有分歧**。

本条目是 decision（决定记录），source 指回原始素材 raw 文件，不写成泛化的 topic 综述。

## 结论

### 采纳
- **原始层不可变**：入库的素材（raw/）不改写，只新增；本库以 `raw/` 前缀实现这一层。
- **index.md + log.md**：index 由 frontmatter 自动生成（不要手工重排）；log 记录每次 write/note（见本库 `log.md`）。
- **被推翻的结论留下当路标**：不删除旧结论，改用 `superseded_by` / `refuted_because` 字段标注；本库 raw 的 frontmatter 已含这两个字段。
- **入库改写先报计划再批准**：把素材改写/整理进库之前先给计划，批准后才动手。

### 放弃（没折入）
- SCHEMA.md 接管规范：不引入 SCHEMA.md 作为规范源。
- 复制状态 lint。
- 独立的 verified 状态。
- qmd（Quarto）格式。

### 分歧（与他不同）
- 他主张：一份素材改写成 10–15 页的成稿。
- 我们主张：原子笔记尽量小而单一，避免反复重写同一素材。

## 关键实现（EvolveKB 落地事实）

- 库根：`/home/zhangxa/docs/mywiki/EvolveKB`（git 仓库；orbit.json：`knowledge.stores.EvolveKB.dir`，defaultStore=EvolveKB）。
- `raw/` 前缀 = 原始层，human-only：本 agent 不能写入（knowledge_manage 写 raw/ 会失败），只读。
- `wiki/` 前缀 = 精炼层，本 agent 可写；不带前缀的路径默认落到 `wiki/` 下（如 `meta/...` → `wiki/meta/...`）。
- `/note` 命令会生成 `raw/YYYY-MM-DD-<slug>.md`，frontmatter 含：id、title、type: raw、summary、tags、status: draft、created/updated、source（如 "slash:/note"）、confidence、review_after、links、`superseded_by`、`refuted_because`。
- `log.md` 每次操作追加一行，形如 `## [YYYY-MM-DD] write | <标题>` 下接路径。
- 本决策条目即按上述约定落在 `wiki/meta/karpathy-llm-wiki-decision.md`，source 指回 raw 文件，构成 provenance 链：外部 gist → raw（/note）→ wiki decision。

## 坑

- `log.md` 里出现过 write 路径 ≠ 文件当前一定存在：本会话发现 log 记录了 `wiki/meta/karpathy-llm-wiki.md`，但该文件并不在 store 索引/工作树里。判断条目是否存在，以 `knowledge_manage list` / `read` 为准，不要信 log。
- `.git/objects` 是 zlib 压缩对象，直接 `read` 会得到乱码，别用它考古文件树。
- `knowledge_manage list` / `search` 只索引库内可识别条目；写完后要确认 commit（含 push）是否成功返回。
- 库带 git remote（会 push 到远端 GitHub）：任何 API key / secret / 不需要的内部主机名都不要写进条目（本条目不含凭据）。

## 验证

- 创建后 `knowledge_manage list` 应能看到 `wiki/meta/karpathy-llm-wiki-decision.md`。
- `knowledge_manage read wiki/meta/karpathy-llm-wiki-decision.md` 应能读回全文，且 frontmatter `source` 指向 raw 文件。
- 本次 write 返回 git commit 结果（commit hash 与 push 状态）。
