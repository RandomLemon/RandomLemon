---
title: Qwen3.6-35B-A3B ROCmFP4：MTP 开关基准测试（ROCmFPX）
date: 2026-10-27
tags: [ai, llm, amd, gfx1151, benchmark, mtp, qwen]
summary: 在 AMD AI MAX+ 395 上对比 Qwen3.6-35B-A3B ROCmFP4 的 MTP 开关，tg128 从 63.46 提升到 112.36 t/s（1.77×），接受率 93.0%。
---

# Qwen3.6-35B-A3B ROCmFP4：MTP 开关基准测试（ROCmFPX）

日期：2026-10-27

## 测试环境

| 项目 | 值 |
|---|---|
| 项目 | `~/ROCmFPX`，`build-strix-rocmfp4`（HIP/ROCm 10.1，build 7966b1db7 (12197)） |
| 主机 | AMD Ryzen AI MAX+ 395 w/ Radeon 8060S |
| GPU | `ROCm0`，gfx1151，128 GB 统一内存（报告 VRAM 122880 MiB） |
| 模型 | `~/llm/qwen3.6-35b-a3b-rocmfp4/Qwen3.6-35B-A3B-MTP-ROCmFP4-STRIX-embF16-headQ6.gguf` |
| 模型规格 | 18.49 GiB，35.51 B 参数，架构 `qwen35moe`，256 experts / top-8，`nextn_predict_layers = 1`（MTP 头内嵌，无外挂 draft 文件） |

视觉 projector（`mmproj-F32.gguf`）未加载；本次仅测文本。

## 测试方法

- MTP 开启：`--spec-type draft-mtp`，使用主模型内嵌的 nextn/MTP 头创建 draft context；
  `--spec-draft-n-max 3 --spec-draft-n-min 0 --spec-draft-p-min 0.0`。
- MTP 关闭：不传上述 speculative 参数，其余完全一致。
- 每次请求使用 `/tokenize` 得到的**精确 512 个 prompt token**，`cache_prompt=false`
  （每次完整重新 prefill），`n_predict=128`、`ignore_eos=true`、贪心采样
  （`temperature=0, top_k=1`，`seed=1234`）。
- 每个模式先做一次 16-token 预热并丢弃，再连续测 3 次。
- 计时取 server 返回的 `timings`：pp = `prompt_n / prompt_ms`，tg = `predicted_n / predicted_ms`。
- 共同参数：`-c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on --load-mode dio --parallel 1 -dev ROCm0`，KV cache f16。
- 环境：`HSA_OVERRIDE_GFX_VERSION=11.5.1`、`GGML_HIP_ENABLE_UNIFIED_MEMORY=1`。

MTP 关闭：

```bash
~/ROCmFPX/build-strix-rocmfp4/bin/llama-server \
  -m ~/llm/qwen3.6-35b-a3b-rocmfp4/Qwen3.6-35B-A3B-MTP-ROCmFP4-STRIX-embF16-headQ6.gguf \
  -c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on \
  --load-mode dio --parallel 1 -dev ROCm0 \
  --host 127.0.0.1 --port 18099
```

MTP 开启（追加）：

```bash
  --spec-type draft-mtp \
  --spec-draft-n-max 3 \
  --spec-draft-n-min 0 \
  --spec-draft-p-min 0.0
```

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
| 关 | 1 | 399.70 | 1280.95 | 2004.50 | 63.36 | — |
| 关 | 2 | 395.17 | 1295.66 | 1999.82 | 63.51 | — |
| 关 | 3 | 399.99 | 1280.05 | 1999.86 | 63.50 | — |
| 开 | 1 | 464.67 | 1101.86 | 1130.63 | 112.33 | 93/100 (93.0%) |
| 开 | 2 | 416.30 | 1229.88 | 1129.70 | 112.42 | 93/100 (93.0%) |
| 开 | 3 | 412.88 | 1240.08 | 1130.52 | 112.34 | 93/100 (93.0%) |

## 汇总

| 指标 | MTP 关 | MTP 开 | 变化 |
|---|---:|---:|---:|
| pp512 均值 | 1285.55 ± 8.76 t/s | 1190.61 ± 77.03 t/s | −7.4%（稳态约 −4.1%） |
| tg128 均值 | 63.456 ± 0.085 t/s | 112.361 ± 0.051 t/s | **+77.1%（1.77×）** |
| 生成 128 token 平均耗时 | 2001.4 ms | 1130.3 ms | 1.77× 更快 |
| 平均接受率 | — | 93.0% | draft 平均长度 3.74 |

## llama-bench 交叉验证（MTP 关）

```
| model                        |    size   | params | backend     | ngl | fa | dev   |  lm |  test |             t/s |
| qwen35moe 35B.A3B ROCmFP4    | 18.49 GiB | 35.51B | ROCm,Vulkan | 999 |  1 | ROCm0 | dio | pp512 | 1371.10 ± 101.71 |
| qwen35moe 35B.A3B ROCmFP4    | 18.49 GiB | 35.51B | ROCm,Vulkan | 999 |  1 | ROCm0 | dio | tg128 |    66.38 ± 0.19 |
```

`llama-bench` 的 tg 比 server 口径高约 4.4%，pp 高约 6.2%，属于两条测量路径的固定差异；MTP 开关对比使用同一 server 口径。

## 结论与说明

- MTP 开启后 tg128 从 63.46 t/s 提升到 112.36 t/s，约 **1.77 倍**；
  接受率 **93.0%**，draft 平均长度 3.74。
- MTP 开启时 pp 均值低约 7.4%，其中第 1 次为冷启动（1101.86 t/s），
  后两次约 1230–1240 t/s，相对关闭组稳态约 −4.1%。MTP 的收益集中在 decode。
- server 日志提示 `Qwen MTP strict verification is disabled`
  （可用 `--spec-mtp-strict-qwen` 开启严格校验）。本次两组生成样本一致，
  但未强制校验逐 token 等价。
- 模型加载日志显示 MTP 头来自主模型自身：
  `creating MTP draft context against the target model`；
  关闭组不加载 nextn 头。
- 测试结束后无残留 `llama-server` 进程。

## 原始数据

- `/tmp/bench_qwen36_35b_mtp.json`：逐次 timings 与接受数
- `/tmp/bench_qwen36_35b_mtp-off.log`、`/tmp/bench_qwen36_35b_mtp-on.log`：两次 server 日志
- `/tmp/bench_qwen36_35b_mtp.py`：复现脚本
