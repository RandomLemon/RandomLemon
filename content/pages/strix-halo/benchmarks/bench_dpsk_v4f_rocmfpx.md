# DeepSeek V4 Flash Vision Exp ROCmFPX：DSpark 开关基准测试（lucebox）

日期：2026-10-27

## 测试环境

| 项目 | 值 |
|---|---|
| 项目 | `~/lucebox`（`Luce-Org/lucebox`），HEAD `48f49bb1` |
| 引擎 | 预编译 `server/build/luce_server`，HIP 后端，`gfx1151`，ROCm 10.1 |
| 主机 | AMD Ryzen AI MAX+ 395 w/ Radeon 8060S |
| GPU | `hip:0`，gfx1151，128 GB 统一内存（报告 VRAM 122880 MiB） |
| 主模型 | `~/llm/deepseek-v4-flash-vision-exp-dspark-rocmfpx/DeepSeek-V4-Flash-Vision-Exp-ROCMFPX-MIX-STRIX.gguf`（93 GB，ROCmFPX MIX Strix） |
| DSpark 草稿 | `DeepSeek-V4-Flash-0731-DSpark-draft-Q4RMFP4-denseF16.gguf`（10 GB，Q4RMFP4 denseF16） |
| 视觉 projector | 未加载（本次仅测文本；`mmproj` 文件存在但未传入） |

## 测试方法

- 通过 `--draft <DSpark-draft.gguf>` 开启 DSpark；关闭时不传 `--draft`，其余参数完全一致。
- 每次请求使用 server 返回的 `usage.timings` 作为模型侧计时来源。
- Prompt 经逐次校准，**精确 512 个 prompt token**（包含 chat template）；`max_tokens=128`、
  `temperature=0`、`stream=false`。
- 先执行 1 次相同参数的预热并丢弃，再连续测 3 次；每次都完整重新 prefill
  （`cache_hit=false`、`cached_prefix_tokens=0`）。
- 负载使用项目标准的确定性计数任务：给出参考文本后要求按行输出整数 `1..1000`，
  保证生成达到 128-token 上限。
- 环境变量：`HSA_OVERRIDE_GFX_VERSION=11.5.1`、`GGML_HIP_ENABLE_UNIFIED_MEMORY=1`、
  `LUCE_DS4_SPARSE_DECODE_FLASH=1`。

共同参数：

```bash
env HSA_OVERRIDE_GFX_VERSION=11.5.1 \
    GGML_HIP_ENABLE_UNIFIED_MEMORY=1 \
    LUCE_DS4_SPARSE_DECODE_FLASH=1 \
./server/build/luce_server <主模型.gguf> \
  --model-name ds4-bench \
  --port 18099 \
  --target-device hip:0 \
  --max-ctx 4096 \
  --chunk 512 \
  --disk-prefix-cache off \
  --prefix-cache-slots 0 \
  --prefill-cache-slots 0 \
  --cache-type-k q4_0 \
  --cache-type-v q4_0 \
  --ds4-fused-decode \
  --ds4-fused-verify-f16-kv \
  --ds4-expert-top-k 6 \
  --ds4-prefill sparse
```

DSpark 开启时追加：

```bash
  --draft ~/llm/deepseek-v4-flash-vision-exp-dspark-rocmfpx/DeepSeek-V4-Flash-0731-DSpark-draft-Q4RMFP4-denseF16.gguf
```

请求形状（OpenAI Chat Completions）：

```json
{
  "model": "ds4-bench",
  "messages": [
    {"role": "system", "content": "<system message>"},
    {"role": "user", "content": "<reference padding + counting task>"}
  ],
  "max_tokens": 128,
  "temperature": 0.0,
  "stream": false
}
```

## 逐次结果

| DSpark | 第几次 | prefill 耗时 (ms) | pp512 (t/s) | decode 耗时 (ms) | tg128 (t/s) | draft 接受率 |
|---|---:|---:|---:|---:|---:|---:|
| 关 | 1 | 1943.7 | 263.42 | 5552.9 | 23.1 | — (AR) |
| 关 | 2 | 1913.2 | 267.61 | 5554.9 | 23.0 | — (AR) |
| 关 | 3 | 1940.8 | 263.81 | 5554.3 | 23.0 | — (AR) |
| 开 | 1 | 1922.8 | 266.28 | 2730.2 | 46.9 | 100% |
| 开 | 2 | 1876.5 | 272.85 | 2723.8 | 47.0 | 100% |
| 开 | 3 | 1877.1 | 272.76 | 2728.3 | 46.9 | 100% |

## 汇总

| 指标 | DSpark 关 | DSpark 开 | 变化 |
|---|---:|---:|---:|
| pp512 均值 | 264.95 ± 2.32 t/s | 270.63 ± 3.77 t/s | +2.1%（噪声内） |
| tg128 均值 | 23.033 ± 0.058 t/s | 46.933 ± 0.058 t/s | **+103.8%（2.04×）** |
| 128 token decode 平均耗时 | 5554.0 ms | 2727.4 ms | 2.04× 更快 |
| 平均接受率 | — | 100% | — |

## 校验

- 每次运行：`prompt_tokens=512`、`prefilled_tokens=512`、`completion_tokens=128`、
  `cache_hit=false`、`cached_prefix_tokens=0`。
- `spec_decode_ran`：关闭组 `false`；开启组 `true`。关闭组 `accept_rate=0.0`。
- 6 次运行的输出 SHA-256 完全一致：
  `0f785a7ffa406498aafb14553966eaed0f52220fed0f7cc016b66921d104d194`
  （开关两组、组内 3 次均逐 token 一致）。

## 结论与说明

- DSpark 开启后 tg128 从 23.03 t/s 提升到 46.93 t/s，约 **2.04 倍**；
  pp512 基本不变（+2.1%，处于噪声范围），符合 DSpark 只作用于 decode 的预期。
- 本次计数负载高度可预测，DSpark 接受率为 **100%**，属于**高接受率上界**场景；
  不代表普通对话/散文吞吐（项目文档给出的混合负载约 27.9 t/s、散文约 25 t/s）。
- `--ds4-prefill sparse` 是项目推荐的批量 prefill 近似路径，仍实际处理全部 512 个
  prompt tokens；两组使用同一路径。
- `LUCE_DS4_SPARSE_DECODE_FLASH=1` 为实验性选项，两组一致启用（可能改变生成的 token，
  但本次开关两组输出 SHA 完全一致）。
- 测试结束后无残留 `luce_server` 进程。

## 原始数据

- `/tmp/bench_dspark.json`：逐次 `usage.timings`、接受率、输出 SHA
- `/tmp/bench_dspark_dspark-off.log`、`/tmp/bench_dspark_dspark-on.log`：两次 server 运行日志
- `/tmp/bench_dspark.py`：复现脚本（含 prompt 校准与请求逻辑）
