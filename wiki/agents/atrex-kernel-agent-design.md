---
title: Atrex Kernel Agent 核心设计与 4090 实测复现
date: 2026-09-07
tags: [agents, gpu-kernel, atrex, aka, llm-optimization, sandbox, git-worktree]
---

## 背景 / 问题

alibaba/atrex-kernel-agent（AKA，Apache 2.0，arXiv 2607.14541）是一个 GPU kernel 自动优化 Agent 系统：输入一个算子（SOL-ExecBench 或 Atrex-Bench 布局），通过"编码 Agent + GPU profiling + 正确性/性能门禁 + Git 隔离"的编排式工作流，迭代优化成生产级 kernel。曾帮 Qwen3.8 拿下 SOL-ExecBench FlashInfer 算子优化榜第一。

核心问题：LLM 生成的 kernel 能不能上生产？AKA 的答案是单靠 LLM 不行，必须加机械验收 + 隔离 + 证据链。

## 结论

AKA 的价值不在某个 kernel 写得多快，而在**把不可信的 Agent 输出变成可信的生产流程**的工程约束。在 RTX 4090 上完整复现验证：addmul 算子 5-trial episode，V0 基线 0.818x → v1 晋升 1.366x（+66.96%），机械门禁正确拒绝了 4 个无效尝试。

## 关键实现

### 七大设计原则

1. **机械控制（最核心）**：终止和验收由代码决定，不由 Agent 自我评估。Agent 只能提议+实现；supervisor 独占预算、状态转移、沙箱执行、门禁、回滚、晋升、打包。
2. **权责边界**：Agent 只能在隔离 worktree 改代码，不能自我晋升/改 incumbent/换 evaluator 输入/用本地 GPU；supervisor 不生成优化代码。
3. **Git 即状态机**：每个优化版本 = 私有分支/worktree 里的一个 episode。HEAD 永远是现任最优 kernel。只有"严格通过正确性 + 确实更快"才 squash-promote；失败也作为证据提交。跨 episode 状态靠 Git + 结构化 memory + journal，支持 Agent 崩溃后接力恢复。
4. **分层优化**：早期 fast episode（5 个 plan→implement→evaluator 试验，无 profiling/ABBA，低成本广撒网）；后期 full episode（profiler 证据 + ABBA 同分配交叉验证才允许晋升）。
5. **评测器完整性**：ground truth 不可变 + 全 workload 验证，防 Agent 改测试框架作弊或只赢部分 shape。
6. **执行隔离**：GPU 工作全过 tools/sandbox.py；远程场景在强制断网 Bubblewrap 命名空间跑，只可见指定 GPU；健康探测失败→保留现场+自动恢复。
7. **GPU Wiki 知识飞轮**：优化 trace + Agent 会话 → 确定性抽取 → 语义蒸馏 → Wiki Gate（唯一写入者，schema/来源/冲突审查）→ 结构化 JSON 知识库 → 查询（bridge agent 只做意图转换，检索由确定性代码执行）→ 支撑下轮优化。双存储隔离：kernel_wiki（经验，排序检索，零命中给兜底样本）vs hardware_wiki（事实，精确检索，fail-loud——"返回不相关的峰值 FLOPS 会静默污染所有利用率计算"）。

### 4090 复现环境（实测可用）

| 组件 | 位置/版本 |
|---|---|
| AKA 仓库 | `~/xiaoning.zhao/aka/atrex-kernel-agent`（git clone --depth 1） |
| Python venv | `~/aka-venv`（python3.12 + torch 2.5.1+cu121） |
| SOL-ExecBench | `~/xiaoning.zhao/aka/SOL-ExecBench`（NVIDIA 官方，pip install -e .） |
| Claude CLI | `~/.npm-global/bin/claude`（npm config set prefix ~/.npm-global 后全局装） |
| Local gateway | `/tmp/start_aka_gateway.sh`（port 8760） |

运行命令：
```bash
# 启动 local gateway（沙箱执行入口，PATH 必须含 aka-venv/bin）
setsid nohup /tmp/start_aka_gateway.sh &

# 启动优化 campaign
python orchestrator/optimize.py \
  --op-dir ~/xiaoning.zhao/aka/aka_op_addmul \
  --platform RTX4090 --sandbox-hardware local \
  --sandbox-url http://127.0.0.1:8760 \
  --framework pytorch --agent-cli claude \
  --max-iters 6 --optimization-mode leaderboard \
  --workspace ~/xiaoning.zhao/aka/aka-runs
```

### SOL 算子 definition.json 正确 schema

与直觉的裸 PyTorch 布局不同，需要：axes（const/var 类型）+ inputs/outputs（shape 用字符串 axis 名，dtype 枚举）+ reference 是 definition 内嵌代码字符串（run() 返回值风格，非 DPS）+ workload.jsonl 带 uuid/axes 具体值/inputs random spec/tolerance。

```json
{
  "name": "addmul",
  "axes": {"M": {"type": "const", "value": 4096}, "N": {"type": "const", "value": 4096}},
  "inputs": {"a": {"shape": ["M", "N"], "dtype": "float16"}, ...},
  "outputs": {"out": {"shape": ["M", "N"], "dtype": "float16"}},
  "reference": "import torch\n\ndef run(a, b, c):\n    return a * b + c\n"
}
```

## 坑

1. **CUPTI 版本冲突**：sol-execbench 默认拉 cupti-python 13.x（要 libcupti.so.13），与 torch cu121 栈（libcupti.so.12）冲突。修复：`pip install cupti-python==12.8.0`（pip 会报 sol-execbench 依赖不满足，忽略即可）。
2. **AKA 上游 bug（已本地 patch，可提 PR）**：`tools/sandbox.py` 的 `EVALUATION_INPUT_PATHS` allowlist 漏了 `config.json` → 沙箱内 sol-execbench 不跑 reference 基准 → speedup=0 → V0 误判失败（报 "missing reference speedup"）。修复：allowlist 加一行 `"config.json"`。
3. **gateway 进程易死**：nohup + ssh 断开会被杀。必须用 `setsid` + 独立启动脚本 + disown；且 gateway 的 PATH 必须含 aka-venv/bin（沙箱命令用裸 `python`）。
4. **venv 无 ensurepip**：Ubuntu 24.04 的 python3.12 venv 可能缺 ensurepip，用 `curl bootstrap.pypa.io/get-pip.py` 装 pip。
5. **端口 8000 可能被占**：local gateway 默认文档示例用 8000，实际机器可能被占，换 8760 等空闲端口。

## 验证

- V0 基线：2/2 workloads PASSED，performance_score 0.818x（手动跑 `test_kernel.py` 确认）
- 5-trial fast episode 完整跑完：T1 addcmul 融合 +67% 被晋升；T2-T5（Triton/autotune/Inductor/128blocks）全部被门禁正确拒绝并留档 journal
- 最终 v1：performance_score 1.366x（+66.96%），4096² fp16 228→158µs，8192² 1161→601µs
- 产物：`aka-runs/kernel_opt_aka_op_addmul_pytorch_rtx4090/` 下 kernel.py、memory/v0|v1.json、submission.json
- CAMPAIGN EXIT=0，token 消耗 8.38M

### 对 ai_infer 栈的借鉴

| AKA 机制 | 对应借鉴 |
|---|---|
| 机械验收 | 对拍门禁（cosine 0.95/0.99）已是此思路；可补"Agent 辅助优化必须过 apex_cli golden + bench 门禁才准合入" |
| Git worktree 隔离 + HEAD=incumbent | kernel 优化轮次直接套：每候选一个 worktree，过门禁才 promote |
| ABBA 同分配验证 | bench 已有 cuda_event sync，缺 ABBA 交叉消噪——<5ms 延迟差异对比值得加 |
| GPU Wiki 飞轮 + 双存储 | aiinfer-kb 的 lessons（≈kernel_wiki）+ map（≈hardware_wiki）雏形已有；缺自动挖掘和 fail-loud 事实库纪律 |
| Evaluator 不可变 + 全 workload | 对拍 golden bin 应加只读保护 + 全 shape 验证 |
