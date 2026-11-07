---
title: Qwen3.8-27B ROCmFP4：MTP 开关基准测试
date: 2026-10-27
tags: [ai, llm, amd, gfx1151, benchmark, mtp, qwen]
summary: 在 AMD AI MAX+ 395 上对比 Qwen3.8-27B ROCmFP4 的 MTP 开关，tg128 从 14.10 提升到 52.82 t/s（3.75×），接受率 98.8%。
---

# Qwen3.8-27B ROCmFP4：MTP 开关基准测试

日期：2026-10-27

## 测试环境

| 项目 | 值 |
|---|---|
| 主机 | AMD Ryzen AI MAX+ 395 w/ Radeon 8060S |
| GPU | gfx1151（Radeon 8060S，128 GB 统一内存，报告 VRAM 122880 MiB） |
| ROCm | `/opt/rocm/core-10.1` |
| 构建 | `~/ROCmFPX/build-strix-rocmfp4`（build 7966b1db7 (12197)） |
| 目标模型 | `~/llm/qwen3.8-27b-rocmfp4/Qwen3.8-27B-Uncensored-ROCmFP4-STRIX_LEAN.gguf`（13.58 GiB，26.90 B 参数） |
| MTP 草稿模型 | `~/llm/qwen3.8-27b-rocmfp4/mtp-Qwen3.8-27B-Q4_0.gguf` |

## 测试方法

- 该树的 `llama-bench` 没有 `--spec-*` 参数，无法开启 MTP，因此 MTP 开/关两组统一使用
  `llama-server` 的 `/completion` 内部 `timings`（纯模型计算时间，不含 HTTP 开销）。
- 每次请求使用 `/tokenize` 得到的**精确 512 个提示 token**，`cache_prompt=false`
  （每次完整重新 prefill），`n_predict=128`、`ignore_eos=true`、贪心采样
  （`temperature=0, top_k=1`，`seed=1234`）。
- 每个模式先做一次 16-token 预热并丢弃，再连续测 3 次。
- 共同参数：`-c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on --load-mode dio -fit off --parallel 1 -dev ROCm0`，KV cache f16。
- 环境变量：`HSA_OVERRIDE_GFX_VERSION=11.5.1`、`GGML_HIP_ENABLE_UNIFIED_MEMORY=1`。

MTP 关闭：

```bash
~/ROCmFPX/build-strix-rocmfp4/bin/llama-server \
  -m ~/llm/qwen3.8-27b-rocmfp4/Qwen3.8-27B-Uncensored-ROCmFP4-STRIX_LEAN.gguf \
  -c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on \
  --load-mode dio -fit off --parallel 1 -dev ROCm0 --jinja \
  --host 127.0.0.1 --port 18099
```

MTP 开启（在上一命令基础上追加，沿用 `start_qwen_3.8_27b.sh` 的配置）：

```bash
  --spec-type draft-mtp \
  --spec-draft-model ~/llm/qwen3.8-27b-rocmfp4/mtp-Qwen3.8-27B-Q4_0.gguf \
  --spec-draft-ngl 99 --spec-draft-device ROCm0 \
  --spec-draft-n-max 6 --spec-draft-n-min 0 --spec-draft-p-min 0.0
```

## 逐次结果

| MTP | 第几次 | prompt 耗时 (ms) | pp512 (t/s) | 生成耗时 (ms) | tg128 (t/s) | draft 接受 |
|---|---:|---:|---:|---:|---:|---:|
| 关 | 1 | 1414.19 | 362.04 | 9001.72 | 14.11 | — |
| 关 | 2 | 1417.75 | 361.14 | 9000.95 | 14.11 | — |
| 关 | 3 | 1429.67 | 358.12 | 9023.33 | 14.07 | — |
| 开 | 1 | 1545.41 | 331.30 | 2402.24 | 52.87 | 108/109 (99.1%) |
| 开 | 2 | 1458.07 | 351.15 | 2407.44 | 52.75 | 108/110 (98.2%) |
| 开 | 3 | 1465.07 | 349.47 | 2403.30 | 52.84 | 108/109 (99.1%) |

## 汇总

| 指标 | MTP 关 | MTP 开 | 变化 |
|---|---:|---:|---:|
| pp512 均值 | 360.43 ± 2.05 t/s | 343.98 ± 11.01 t/s | −4.6% |
| tg128 均值 | 14.098 ± 0.020 t/s | 52.822 ± 0.060 t/s | **+274.7%（3.75×）** |
| 平均接受率 | — | 98.8% | draft 平均长度 6.68 |
| 生成 128 token 平均耗时 | 9008.66 ms | 2404.32 ms | 3.75× 更快 |

说明：MTP 不参与 prefill，pp 差异属于冷启动/噪声级别；MTP 开组第 1 次为冷启动，
后两次 pp 为 351.15 / 349.47 t/s，相对关组均值约 −2.8%。

## llama-bench 交叉验证（MTP 关）

```
| model                          |       size |     params | backend    | ngl |  fa | dev   |  lm |  test |                  t/s |
| qwen35 27B ROCmFP4 Strix Lean  |  13.58 GiB |    26.90 B | ROCm,Vulkan | 999 |   1 | ROCm0 | dio | pp512 |       378.68 ± 10.93 |
| qwen35 27B ROCmFP4 Strix Lean  |  13.58 GiB |    26.90 B | ROCm,Vulkan | 999 |   1 | ROCm0 | dio | tg128 |         14.13 ± 0.04 |
```

`tg128` 与 server 口径一致（差 0.2%）；`pp512` 高约 4–5%（工具测量路径差异），
因此 MTP 开关对比以同一 server 口径为准。

## 结论

- MTP 开启后 tg128 从 14.10 t/s 提升到 52.82 t/s，约 **3.75 倍**。
- 草稿接受率约 **98.8%**，`n-max 6` 下平均每轮接受 6.68 个 token。
- pp512 基本不变（略低，处于冷启动/噪声范围），符合 MTP 仅作用于 decode 的预期。
- 测试结束后无残留 `llama-server` / `llama-bench` 进程。

## 原始数据

- `/tmp/bench_qwen38_mtp.json`：逐次 timings JSON
- `/tmp/bench_mtp-off.log`、`/tmp/bench_mtp-on.log`：两次 server 运行日志
- `/tmp/bench_mtp.py`：复现脚本
