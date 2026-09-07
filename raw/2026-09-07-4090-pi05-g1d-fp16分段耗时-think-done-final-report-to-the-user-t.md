---
id: 20260907-4090-pi05-g1d-fp16分段耗时-think-done-final-report-to-the-user-think-测完了-用-un
title: "4090_pi05_g1d_fp16分段耗时 <think>Done. Final report to the user.</think> ✅ 测完了。用 Un"
type: raw
summary:
tags: []
status: draft
created: 2026-09-07
updated: 2026-09-07
source: "slash:/note"
confidence: medium
review_after: 2026-09-21
links: []
superseded_by:
refuted_because:
---

4090_pi05_g1d_fp16分段耗时 <think>Done. Final report to the user.</think> ✅ 测完了。用 UniRT 自带的 `pi05_llm` 分阶段计时（GPU1 空闲卡，warmup 10 + 50 次，分布很稳），结果如下： ## 4090 分段耗时（pi05 G1D fp16，3 相机，prefix 968，10 步去噪） | 阶段 | p50 (ms) | mean (ms) | p95 (ms) | 占比 | |---|---|---|---|---| | h2d images | 0.30 | 0.31 | 0.34 | 0.5% | | **embed_image**（3 cam SigLIP，单次 VM 调用） | **5.65** | 5.66 | 5.73 | 8.6% | | embed_language（200 tok） | 0.03 | 0.03 | 0.04 | ~0 | | stitch prefix | 0.02 | 0.02 | 0.03 | ~0 | | **prefill**（968 tok，gemma 2B） | **27.22** | 27.22 | 27.27 | 41% | | **denoise_loop_kv**（10 步图内循环） | **32.20** | 32.33 | 33.80 | 49% | | ↳ 每步 | 3.22 | 3.23 | — | | | d2h actions | 0.07 | 0.08 | 0.12 | ~0 | | **end-to-end** | **65.61** | **65.74** | **67.15** | **15.2 帧/s** | **变体**： - host-loop（denoise_step_kv + host Euler）：每步 3.41ms，e2e p50 68.56ms——比图内循环慢 ~3ms（每步 host 往返） - apexinfer Session 业务路径（含分词/图像归一化/host 数学）：e2e p50 **65.77ms**，与分段和一致，编排开销≈0 ## 横向对比（同机同 G1D 权重） | 栈 | vit | prefill | denoise×10 | E2E | |---|---|---|---|---| | **UniRT fp16（本测）** | 5.65 | 27.2 | 32.2 | **65.6 ms / 15.2Hz** | | TRT bf16 引擎（trtexec） | 5.39 | 28.7 | 34.1 | ~68.2 ms | | FlashRT FP8（⚠️2 views 全融合） | — | — | — | 23.1 ms / 43.3Hz | | Thor TRT bf16（8/28 基线） | 31.6 | 67.1 | 131.7 | 240.7 ms | ## 结论 1. **UniRT fp16 ≈ TRT bf16 引擎**（e2e 还略优 4%），TVM 栈在 4090 上没有明显劣势； 2. **4090 比 Thor 快 ~3.7×**； 3. 瓶颈清晰：denoise 49% + prefill 41%。还有三个没开的优化开关：编译期 `--trt`（TRT BYOC）、`--cuda-graph`、`--fp8` 量化——FlashRT FP8 的 23ms 说明 FP8+融合在这卡上空间很大； 4. 注意：未锁频（无 sudo），但 warmup 后 p95≈mean，数据可信。
