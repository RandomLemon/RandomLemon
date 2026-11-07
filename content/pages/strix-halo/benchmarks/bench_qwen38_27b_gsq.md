---
title: Qwen3.8-27B GSQ-RCO：MTP 开关基准测试（strix-llama.cpp）
date: 2026-10-27
tags: [ai, llm, amd, gfx1151, benchmark, mtp, qwen]
summary: 在 AMD AI MAX+ 395 上对比 Qwen3.8-27B GSQ-RCO 的 MTP 开关，tg128 从 16.63 提升到 39.18 t/s（2.36×），接受率 97.9%。
---

# Qwen3.8-27B GSQ-RCO：MTP 开关基准测试（strix-llama.cpp）

日期：2026-10-27

## 测试环境

| 项目 | 值 |
|---|---|
| 项目 | `~/strix-llama.cpp`，master `c8912d7f5`（无预编译产物，本次新建 `build-hip`） |
| 构建 | Ninja Release，HIP/ROCm 10.1，`GGML_HIP=ON`、`GPU_TARGETS=gfx1151` |
| 主机 | AMD Ryzen AI MAX+ 395 w/ Radeon 8060S |
| GPU | `ROCm0`，gfx1151，128 GB 统一内存（报告 VRAM 122880 MiB） |
| 模型 | `~/llm/qwen3.8-27b-gsq-rco/Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf` |
| 模型规格 | 11.28 GiB，27.32 B 参数，IQ3_S 3.4375 bpw，架构 `qwen35`，`nextn_predict_layers = 1`（MTP 头内嵌，无需外挂 draft 模型） |

## 测试方法

- 该树的 `llama-bench` 没有 `--spec-*` 参数，无法开启 MTP，因此 MTP 开/关两组统一使用
  `llama-server` 的 `/completion` 内部 `timings`（纯模型计算时间，不含 HTTP 开销）。
- 每次请求使用 `/tokenize` 得到的**精确 512 个提示 token**，`cache_prompt=false`
  （每次完整重新 prefill），`n_predict=128`、`ignore_eos=true`、贪心采样
  （`temperature=0, top_k=1`，`seed=1234`）。
- 每个模式先做一次 16-token 预热并丢弃，再连续测 3 次。
- 共同参数：`-c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on --parallel 1 -dev ROCm0`，KV cache f16。
- `HSA_OVERRIDE_GFX_VERSION=11.5.1`（gfx1151）。

MTP 关闭：

```bash
~/strix-llama.cpp/build-hip/bin/llama-server \
  -m ~/llm/qwen3.8-27b-gsq-rco/Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf \
  -c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on \
  --parallel 1 -dev ROCm0 --host 127.0.0.1 --port 18099
```

MTP 开启（仅追加）：

```bash
  --spec-type draft-mtp
```

该参数让 server 从主模型内嵌的 nextn 头创建 MTP draft context；草稿长度使用默认
`--spec-draft-n-max 3`，`p-min 0.0`。

请求体：

```json
{
  "prompt": "<512 token IDs>",
  "n_predict": 128,
  "ignore_eos": true,
  "cache_prompt": false,
  "temperature": 0.0,
  "top_k": 1,
  "seed": 1234,
  "stream": false
}
```

## 逐次结果

| MTP | 第几次 | prompt 耗时 (ms) | pp512 (t/s) | 生成耗时 (ms) | tg128 (t/s) | draft 接受 |
|---|---:|---:|---:|---:|---:|---:|
| 关 | 1 | 1435.07 | 356.78 | 7639.09 | 16.63 | — |
| 关 | 2 | 1336.63 | 383.05 | 7637.46 | 16.63 | — |
| 关 | 3 | 1348.14 | 379.78 | 7636.60 | 16.63 | — |
| 开 | 1 | 1510.89 | 338.87 | 3225.76 | 39.37 | 94/96 (97.9%) |
| 开 | 2 | 1513.47 | 338.29 | 3270.18 | 38.84 | 94/96 (97.9%) |
| 开 | 3 | 1456.33 | 351.57 | 3227.92 | 39.34 | 94/96 (97.9%) |

## 汇总

| 指标 | MTP 关 | MTP 开 | 变化 |
|---|---:|---:|---:|
| pp512 均值 | 373.20 ± 14.32 t/s | 342.91 ± 7.50 t/s | −8.1% |
| tg128 均值 | 16.628 ± 0.003 t/s | 39.184 ± 0.301 t/s | **+135.6%（2.36×）** |
| 生成 128 token 平均耗时 | 7637.7 ms | 3241.3 ms | 2.36× 更快 |
| 平均接受率 | — | 97.9% | draft 平均长度 3.94 |

## llama-bench 交叉验证（MTP 关）

```
| model                          |    size   | params | backend | ngl | fa | dev   |  test |             t/s |
| qwen35 27B IQ3_S - 3.4375 bpw  | 11.28 GiB | 27.32B | ROCm    | 999 |  1 | ROCm0 | pp512 | 400.28 ± 14.15 |
| qwen35 27B IQ3_S - 3.4375 bpw  | 11.28 GiB | 27.32B | ROCm    | 999 |  1 | ROCm0 | tg128 |   16.63 ± 0.03 |
```

`tg128` 与 server 口径完全一致；server 的 `pp512` 比 `llama-bench` 低约 7%
（两者测量路径不同），因此 MTP 开关对比以同一 server 口径为准。

## 结论与说明

- MTP 开启后 tg128 从 16.63 t/s 提升到 39.18 t/s，约 **2.36 倍**；
  草稿接受率约 **97.9%**。
- MTP 开启时 pp512 实测下降约 **8.1%**：该实现为主模型内嵌 MTP 头创建了独立
  draft context，prompt 阶段存在额外开销；本次未对该成因做隔离测试。
  MTP 的收益集中在 decode。
- MTP 关闭时，模型加载日志把 `blk.64.*`（nextn 层）列为未使用张量并跳过；
  MTP 开启时这些张量参与 draft context，行为符合预期。
- `HIP_LAUNCH_BLOCKING` 未设置（与仓库 CI benchmark 口径一致）。两种模式下
  贪心生成样本均为可读文本，未观察到异步执行导致的输出损坏。
- 测试结束后无残留 `llama-server` / `llama-bench` 进程。

## 原始数据

- `/tmp/bench_strix_mtp.json`：逐次 timings JSON
- `/tmp/bench_strix_mtp-off.log`、`/tmp/bench_strix_mtp-on.log`：两次 server 运行日志
- `/tmp/bench_strix_mtp.py`：复现脚本
