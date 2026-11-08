---
title: DeepSeek V4.1 Flash GSQ_RCO IQ2_XXS：双机测试（llama.cpp）
date: 2026-10-27
tags: [ai, llm, amd, gfx1151, benchmark, deepseek]
summary: 在两台 AMD AI MAX+ 395 上测试运行 DeepSeek V4.1 Flash GSQ_RCO IQ2_XXS。
---

# 双机 deepseek v4.1 flash 初步测试

- **服务类型**: llama.cpp `llama-server` v0.9.0 Vulkan
- **模型**: `DeepSeek-V4.1-Flash-1017-GSQ-RCO-IQ2_XXS`
- **量化**: GSQ_RCO IQ2_XXS / 2.0625 bpw
- **上下文窗口 (n_ctx)**: 262144
- **并发槽位**: 4
- **测试日期**: 2026-11-08

## 部署基座

`llama.cpp` v0.9.0, Vulkan backend, rpc with TCP

两台机器使用usb4 thunderbolt网络连接，使用iperf3测速约为8.9Gbps。使用llama.cpp的rpc能力实现双机部署，tensor切分为1：1，每台机器约占用69-72GB显存，运行时观察到核显负载不高，温度也不到50度。雷电网络传输速度在3MB/s上行 1MB/s下行左右（Host端）。雷电网络连接将MTU设置为15000。

## 测试方法

所有测试通过 HTTP `/completion` 接口调用，关键参数：

| 参数 | 值 |
|---|---|
| `cache_prompt` | `false`（禁用 prompt 缓存，避免命中污染） |
| `temperature` | `0.0` |
| `ignore_eos` | `true`（确保生成足量 token） |
| `stream` | `false` |

- **tgN**：读取响应 `timings.predicted_per_second`，设置 `n_predict=N`。
- **ppN**：先用 `/tokenize` 逐词校准 prompt 到精确 N tokens，再读取 `timings.prompt_per_second`，设置 `n_predict=1`。
- 每项多次运行，剔除首跑暖机差异后取中位数。

## 测试结果

### 汇总

| 指标 | 目标 Tokens | 各次测量 (tok/s) | 中位 (tok/s) |
|---|---|---|---|
| **tg** | 128 | 16.996, 16.797 | **~16.9** |
| **tg** | 4096 | 16.14, 16.19 | **~16.17** |
| **pp** | 512 | 223.94, 241.35, 239.24 | **~239** |
| **pp** | 4096 | 214.57, 220.95, 221.64 | **~220.95** |

### 详细数据

#### tg128（生成 128 tokens）

| 运行 | 生成 tokens | tg (tok/s) | 墙钟 (s) |
|---|---|---|---|
| 0 | 128 | 16.996 | 7.761 |
| 1 | 128 | 16.797 | 7.799 |

#### tg4096（生成 4096 tokens）

| 运行 | 生成 tokens | tg (tok/s) | 墙钟 (s) |
|---|---|---|---|
| 0 | 4096 | 16.14 | 253.93 |
| 1 | 4096 | 16.19 | 253.25 |

#### pp512（处理 512 tokens prompt）

| 运行 | prompt_n | pp (tok/s) | 墙钟 (s) |
|---|---|---|---|
| 0 | 512 | 223.94 | 2.367 |
| 1 | 512 | 241.35 | 2.158 |
| 2 | 512 | 239.24 | 2.176 |

#### pp4096（处理 4096 tokens prompt）

| 运行 | prompt_n | pp (tok/s) | 墙钟 (s) |
|---|---|---|---|
| 0 | 4096 | 214.57 | 19.13 |
| 1 | 4096 | 220.95 | 18.58 |
| 2 | 4096 | 221.64 | 18.52 |

## 分析

### 序列长度对性能的影响

| 指标 | 128/512 | 4096 | 变化 |
|---|---|---|---|
| tg | ~16.9 tok/s | ~16.17 tok/s | **−4.3%** |
| pp | ~239 tok/s | ~220.95 tok/s | **−7.6%** |

- **tg 衰减极小**：从 128 到 4096 tokens，生成速率仅下降 4.3%。说明在 IQ2_XXS 量化下，decode 阶段瓶颈仍是权重读取带宽，KV cache 从 128 增长到 4096 的额外开销占比很小。
- **pp 下降 7.6%**：长 prompt 下 attention 的 $O(n^2)$ 分量开始显现，但尚未主导整体 prefill 时间，因此下降仍属温和。
- **pp/tg 比**：128/512 场景 ≈14:1，4096 场景 ≈13.7:1，比值几乎不变，说明该模型在此硬件上 prefill 相对 decode 的优势曲线平缓。

### 稳定性

- 同一配置多次运行差异普遍 < 2%，tg 波动最小（< 1.2%），服务在测试期间表现稳定。
- pp 首跑略低（223.94 vs 后续 ~240），为暖机效应；剔除后取中位。

## 注意事项

1. **测量路径含开销**：以上为 HTTP 接口端到端 timings，包含网络传输与 JSON 序列化开销，与服务器端 `llama-bench` 直接对比会有偏差。
2. **batch 配置未知**：`/props` 未暴露 `n_batch` / `n_ubatch`，无法确认 prefill 批大小是否为 512/512，这可能影响 pp 绝对值。
3. **建议对标方式**：若需与官方基准严格对比，建议在服务器端运行：
   ```bash
   llama-bench -m <gguf_path> -p 512 -n 128 -r 3
   llama-bench -m <gguf_path> -p 4096 -n 4096 -r 3
   ```