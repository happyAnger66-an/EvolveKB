---
title: GPT-6 Astra：新特性、迁移参数与提示词调教要点
date: 2026-09-07
type: topic
tags: [llm, openai, gpt-6-astra, agent, api, prompting, migration]
source: "raw/2026-09-07-gpt-6-astra-使用文档.md"
---

## 背景 / 问题

OpenAI 官方文档《Using GPT-6 Astra》（`model: gpt-6-astra`，Responses API）给出该模型的新特性、行为特点、提示词调教模板与迁移清单。原始素材已由 `/note` 存入 `raw/2026-09-07-gpt-6-astra-使用文档.md`（raw 层 human-only 不可写，故本条为 wiki 精炼层，构成 provenance 链）。

需要留存的是：迁移时**必须改的参数**、模型**已知行为缺陷及对策**，以及对自建 agent 平台的设计参考。

## 结论

- 三个新特性全部面向长时间运行的 agent 工作流：**异步工具调用**、**回合中转向**、**会话中改推理强度且保缓存**。
- **工具调用必须走 Responses API**；Chat Completions 仍支持该模型，但不支持工具调用。
- **采样参数被移除**：`temperature`、`top_p`、`top_logprobs` 不再支持（Chat Completions 另需移除 `logprobs`；Responses 需从 `include` 移除 `message.output_text.logprobs`）。
- 不支持 `none` 推理档；原用 `none`/`minimal` 的应从 `low` 起步对比。
- EU 数据驻留：不支持 `service_tier: "fast"` / `"priority"`，须用 Standard；且 Fast mode 本身不含延迟 SLA。
- 成本口径改为「每任务成本」：单 token 更贵但输出 token 更少。
- 官方把对齐做成运行期可观测项：**失准监控（misalignment monitoring）**异步监测并告警。
- 模型行为缺陷被官方文档化并给出提示词对策（见下），说明落地仍高度依赖提示词工程。

## 关键实现

### 新特性用法

- **异步工具调用**：function/custom tool 上设 `async: true`，应用执行工具期间模型可继续推理、调其他工具、回答请求中独立的部分；结果按原 `call_id` 回填。工具执行与待办管理仍由应用负责。
- **回合中转向（mid-turn steering）**：模型工作期间可发额外用户指令（纠正/改需求）；WebSocket 连接下 Responses API 保留已完成工作，并在续跑中纳入更新。
- **会话中改推理强度**：加入 `configuration_update` 输入项升/降 reasoning effort，**不重写原 prompt 前缀**，因此不破坏缓存；生效至下一个 `configuration_update` 覆盖为止。请求级 `reasoning.effort` 应保持不变以保住缓存前缀。
- 继承 GPT-5.6 能力：computer use、Structured Outputs、streaming、Programmatic Tool Calling、多 agent 编排、prompt caching、persisted reasoning、compaction、pro mode。

### 迁移参数对照

```text
model                      -> "gpt-6-astra"
reasoning effort           -> Responses: reasoning.effort / Chat: reasoning_effort
                              原 none|minimal -> 从 low 起步
temperature, top_p, top_logprobs -> 删除
Chat Completions: logprobs       -> 删除
Responses include: message.output_text.logprobs -> 删除
prompt_cache_retention     -> prompt_cache_options.ttl = "30m"   # 从 GPT-5.5 及更早迁移时
service_tier fast|priority -> EU 数据驻留不可用，改 Standard
工具调用                    -> 必须 Responses API
```

Codex 可自动套用本指南（OpenAI Docs skill，仓库 `openai/skills`，路径 `skills/.curated/openai-docs`）：

```text
$openai-docs migrate this project to GPT-6 Astra
```

### 行为特点与官方对策（提示词模板要点）

| 行为 | 表现 | 对策 |
|---|---|---|
| 主动性/跟进 | 比旧模型更爱提问而非自行假设，可能在用户期望推进时停下 | 注入「偏向行动」：从指令与上文推断意图和范围，坚持到目标完成；可逆/只读/已授权动作不必请示 |
| 批准时机 | 易在干活前就阻塞求批准 | 要求「先完成上下文已授权、且让方案变得具体可审的工作，再请用户批准」；部署/写外部系统/合 PR/发布前先把前置工作做完，让批准成为最后一步 |
| 指令遵循 | 更强，但对 skills 与 `AGENTS.md` 等文件中的指令**更敏感**；含糊或冲突的 skill 指引会让它过早暂停 | 显式声明优先级：用户指令 > skill 指引；要求它在因 skill 暂停/转向时**指名并链接具体 SKILL.md、引用原文条款**，并区分「skill 明确要求」与「它自己的解读」 |
| 个性/文风 | 偏好列表、表格、Markdown；跨会话复现固定短语 | 指定文风与结构；官方给出「反 slop」禁用词表（如 `delve`、`foster`、`leverage`、`it's worth noting`、`Bottom Line:`、`In short:`、"这不是 X 而是 Y" 式对比框架、自造复合标签） |
| 子代理委派 | 可能委派不足 | 明确「能并行就委派」（root 与 subagent 均适用）；并要求 agent 间消息与最终答案保持可读（词/数字间留空格） |
| 测试验证 | 编码任务测试彻底，小任务会过度测试 | 校准：可逆低影响、与实现镜像的改动不写测试；跑与改动相称的测试，通过后仅在有新增改动/失败/未决疑虑时才扩大或重跑 |

### 对自建 agent 平台的设计参考（从本模型特性/缺陷反推）

1. **异步工具调用 + 回合中转向**直指长任务 agent 的两大痛点：工具执行期间 agent 空等、用户中途补指令导致已完成工作丢失。自建平台（如本环境的 Orbit 定时任务轮次）可参照：工具异步化按 `call_id` 回填；中途消息不重启轮次而是并入续跑。
2. **指令文件优先级与透明化**：平台加载多层指令（系统提示 / AGENTS.md / skills）时，应显式声明「用户指令 > skill 指引」，并要求 agent 因 skill 停下时指名引用具体文件与条款——既是排障手段，也是静默/冲突指令的审计手段。
3. **「先干活再请示」的批准交互**：把审批点放在「具体可审的结果」之后而非动作之前，可逆/只读操作免请示——可直接用作自建 agent 的审批流设计原则。
4. 行为调教模板（偏向行动/测试校准/反 slop 文风/委派提示）与具体模型无关，可移植到任何 LLM 的系统提示中。

## 坑

- **skill / `AGENTS.md` 成为提示注入面**：模型对这些文件更听话，官方「强烈建议」审计模型可访问的 skill 与指令文件。任何加载外部指令文件的 agent 平台都应把这些文件纳入安全审查。
- **迁移漏删参数会直接报错**：`temperature` / `top_p` / `top_logprobs` 属不支持参数，不是「被忽略」；Chat Completions 与 Responses 各自还有额外的 logprobs 相关项要清。
- **动态改 effort 用错层会丢缓存**：应通过 `configuration_update` 输入项改，若改请求级 `reasoning.effort` 会重写前缀、破坏 prompt 缓存。采纳前需查该特性的兼容性限制。
- **缓存计费口径变化**：迁移时要一并复核缓存边界与 cache-write 计费，不只是改字段名。
- **「老停下来求批准」不是 bug**：属已知行为，用主动性/批准时机提示词调，而不是反复重试。
- **Fast mode 无延迟 SLA**：即便在非 EU 场景可用，也不要把 Fast mode 当作延迟保证。

## 验证

- 迁移后逐项自检：`model` 已改；不支持参数已全部移除；工具调用走 Responses API；EU 驻留场景未使用 `service_tier: fast|priority`；缓存字段已换成 `prompt_cache_options.ttl`。
- 推理档位：原 `none`/`minimal` 的调用改 `low` 后需与旧结果对比再定档。
- 行为调教生效判据：出现「频繁求批准 / 过度测试 / 委派不足 / 文风套路化」时，套用对应模板后应可观测到行为变化；若因 skill 暂停，模型应能指名具体 SKILL.md 与条款（可用于排查静默或冲突指令）。
- 本条目为文档要点留存，未在本环境实跑该模型 API；`raw/` 原始素材只读，判断条目存在与否以 `knowledge_manage list` / `read` 为准。
