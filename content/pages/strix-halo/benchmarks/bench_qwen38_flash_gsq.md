---
title: Qwen3.8-Flash-Next GSQ-RCO：MTP 开关基准测试（strix-llama.cpp）
date: 2026-10-27
tags: [ai, llm, amd, gfx1151, benchmark, mtp, qwen]
summary: 在 AMD AI MAX+ 395 上对比 Qwen3.8-Flash-Next GSQ-RCO 的 MTP 开关，tg128 从 25.94 提升到 45.18 t/s（1.74×），接受率 73.7%。
---

# Qwen3.8-Flash-Next GSQ-RCO：MTP 开关基准测试（strix-llama.cpp）

日期：2026-10-27

## 测试环境

| 项目 | 值 |
|---|---|
| 项目 | `~/strix-llama.cpp`，master `c8912d7f5`，`build-hip`（Ninja Release） |
| 构建 | HIP/ROCm 10.1，`GGML_HIP=ON`、`GPU_TARGETS=gfx1151` |
| 主机 | AMD Ryzen AI MAX+ 395 w/ Radeon 8060S |
| GPU | `ROCm0`，gfx1151，128 GB 统一内存（报告 VRAM 122880 MiB） |
| 主模型 | `~/llm/qwen3.8-flash-next-gsq-rco/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`（2 分片） |
| 主模型规格 | 77.86 GiB，176.94 B 参数（A3B MoE），IQ3_S 3.4375 bpw，架构 `qwen4exp` |
| MTP 草稿 | `mtp-Qwen3.8-Flash-Next-hc-Q8_0.gguf`（3.9 GB，Q8_0，`nextn_predict_layers = 1`） |

主模型本身不含 nextn/MTP 张量；MTP 必须使用外挂草稿模型。

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
  -m ~/llm/qwen3.8-flash-next-gsq-rco/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf \
  -c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on \
  --parallel 1 -dev ROCm0 --host 127.0.0.1 --port 18099
```

MTP 开启（追加）：

```bash
  --spec-type draft-mtp \
  --spec-draft-model ~/llm/qwen3.8-flash-next-gsq-rco/mtp-Qwen3.8-Flash-Next-hc-Q8_0.gguf \
  --spec-draft-ngl all --spec-draft-device ROCm0
```

草稿长度使用默认 `--spec-draft-n-max 3`、`p-min 0.0`。

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
| 关 | 1 | 1005.71 | 509.10 | 4893.81 | 25.95 | — |
| 关 | 2 | 794.88 | 644.13 | 4897.81 | 25.93 | — |
| 关 | 3 | 798.87 | 640.90 | 4894.66 | 25.95 | — |
| 开 | 1 | 1318.35 | 388.36 | 2793.85 | 45.46 | 87/118 (73.7%) |
| 开 | 2 | 880.66 | 581.38 | 2851.84 | 44.53 | 87/118 (73.7%) |
| 开 | 3 | 880.19 | 581.69 | 2787.53 | 45.56 | 87/118 (73.7%) |

## 汇总

| 指标 | MTP 关 | MTP 开 | 变化 |
|---|---:|---:|---:|
| pp512 均值（3 次） | 598.04 ± 77.05 t/s | 517.14 ± 111.53 t/s | −13.5% |
| pp512 稳态（第 2–3 次） | 642.51 t/s | 581.54 t/s | −9.5% |
| tg128 均值 | 25.943 ± 0.011 t/s | 45.183 ± 0.566 t/s | **+74.2%（1.74×）** |
| 生成 128 token 平均耗时 | 4895.4 ms | 2811.1 ms | 1.74× 更快 |
| 平均接受率 | — | 73.7% | draft 平均长度 3.17 |

## llama-bench 交叉验证（MTP 关）

```
| model                          |    size   |  params | backend | ngl | fa | dev   |  test |             t/s |
| qwen4exp A3B IQ3_S - 3.4375bpw | 77.86 GiB | 176.94B | ROCm    | 999 |  1 | ROCm0 | pp512 | 737.73 ± 82.29 |
| qwen4exp A3B IQ3_S - 3.4375bpw | 77.86 GiB | 176.94B | ROCm    | 999 |  1 | ROCm0 | tg128 |   26.00 ± 0.09 |
```

`tg128` 与 server 口径一致（差约 0.2%）；server 的 `pp512` 比 `llama-bench` 低约 13–19%
（测量路径及 MoE 提示内容差异），因此 MTP 开关对比以同一 server 口径为准。

## 结论与说明

- MTP 开启后 tg128 从 25.94 t/s 提升到 45.18 t/s，约 **1.74 倍**；
  草稿接受率 **73.7%**，draft 平均长度 3.17。
- 接受率明显低于 27B GSQ 的 97.9%，因此加速比也更低（1.74× vs 2.36×）。
- 两模式第 1 次 pp 均为冷启动低值（77.86 GiB 模型首次逐页触盘/升频），第 2–3 次进入稳态；
  稳态下 MTP 开启 pp 仍低约 **9.5%**，与之前 27B 测试的 ~8–10% 一致，MTP 的收益集中在 decode。
- server 日志在两种模式下均出现 `resolve_fused_ops` 警告：部分 fused op
  （DeepSeek V4 HC pre/post、f16 MoE activations、GDN output norm 等）回退到 CPU
  （"usually due to missing support"）。两模式相同，不影响开关对比；绝对吞吐可能受此影响。
- `HIP_LAUNCH_BLOCKING` 未设置（与仓库 CI benchmark 口径一致）。两种模式下贪心生成样本
  均为可读文本，未观察到异步执行导致的输出损坏。
- 测试结束后无残留 `llama-server` / `llama-bench` 进程。

## 原始数据

- `/tmp/bench_flashnext_mtp.json`：逐次 timings JSON
- `/tmp/bench_flashnext_mtp-off.log`、`/tmp/bench_flashnext_mtp-on.log`：两次 server 运行日志
- `/tmp/bench_flashnext_mtp.py`：复现脚本
