---
title: Karpathy llm-wiki 方法笔记（2026-04 讨论记录）
date: 2026-09-06
tags: [meta, wiki, 方法论, karpathy, llm-wiki]
---

## 背景 / 问题
2026-04 对照 Karpathy 的 llm-wiki 方案（gist）审视本团队/本人 wiki（EvolveKB / mywiki）的做法，记录采纳要点、明确放弃项与立场分歧。

## 采纳的要点（Karpathy llm-wiki 主张）
- **原始层不可变**：raw 素材只进不覆盖；改写产物放上层，不回写原素材。
- **index.md + log.md**：用 index 组织入口、log 记录变更轨迹。
- **被推翻的结论留下**：结论被更新/推翻时保留原文痕迹，不静默删除。
- **入库改写先确认**：把外部素材改写进 wiki 前，先与原作者确认，避免曲解原意。

## 明确未纳入（没折入）
以下 Karpathy 方案中的概念/文件**不**采用：`SCHEMA.md`、复制状态 lint、`verified`、`qmd`。

## 分歧（我们 vs Karpathy）
- Karpathy 主张：一份素材可以反复改写、展开成 **10–15 页**的深度整理。
- 我们主张：**原子笔记**——一条一主题，**不反复重写**同一素材；需要新内容就开新条目，避免旧条目越改越臃肿、失去原子性。

## 出处
- https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f
