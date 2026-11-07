---
title: DeepSeek V4 Flash Vision Exp IQ2_XXS：DSpark 开关基准测试（strix-llama.cpp）
date: 2026-10-27
tags: [ai, llm, amd, gfx1151, benchmark, dspark, deepseek]
summary: 在 AMD AI MAX+ 395 上对比 DeepSeek V4 Flash Vision Exp IQ2_XXS 的 DSpark 开关，tg128 提升 20.7%，pp512 基本不变。
---

# DeepSeek V4 Flash Vision Exp IQ2_XXS：DSpark 开关基准测试（strix-llama.cpp）

日期：2026-10-27

## 前提：DSpark 草稿转换

`~/llm/deepseek-v4-flash-vision-exp-iq2-xxs` 只有 3 个主模型分片和 mmproj，**没有 DSpark 草稿**。
strix 的 `--spec-type draft-dspark` 需要 `dflash` 架构的 GGUF；本机唯一的 DSpark 草稿：

```
~/llm/deepseek-v4-flash-vision-exp-dspark-rocmfpx/
  DeepSeek-V4-Flash-0731-DSpark-draft-Q4RMFP4-denseF16.gguf
```

是 lucebox 格式（`general.architecture = deepseek4-dflash-draft`），strix 直接加载报：

```
unknown model architecture: 'deepseek4-dflash-draft'
```

因此创建了仅修改元数据/张量命名的转换副本
`/tmp/DeepSeek-V4-Flash-DSpark-draft-strix.gguf`，权重数据原样复制：

- `general.architecture` → `dflash`；键前缀 `deepseek4-dflash-draft.` → `dflash.`
- 张量重命名：
  - `dflash.fc.weight` → `fc.weight`
  - `dflash.hidden_norm.weight` → `enc.output_norm.weight`
  - `dflash.dspark.markov.w1` → `markov_w1.weight`
  - `dflash.dspark.markov.w2` → `markov_w2.weight`
  - `dflash.dspark.confidence.weight` → `conf_proj.weight`
  - `dflash.dspark.confidence.bias` → `conf_proj.bias`
- 补充 strix 所需元数据：
  - `target_layers = [40, 41, 42]`
  - `attention.sliding_window = 128`
  - `attention.compress_ratios = [0, 0, 0]`
  - `expert_weights_norm = true`
  - `swiglu_clamp_exp = [10, 10, 10]`
  - `context_length = 131072`
  - `sample_from_anchor = true`
  - 目标模型 tokenizer 与 `tokenizer.ggml.mask_token_id = 128799`
  - `expert_gating_func = 4`（`SQRT_SOFTPLUS`）

转换后加载成功。server 日志确认：

```
adding speculative implementation 'draft-dspark'
n_max=5, n_min=0, p_min=0.00
block_size=5, mask_token_id=128799, n_extract=3, sample_from_anchor=true
```

未修改 strix 源码或模型目录。

## 测试环境

| 项目 | 值 |
|---|---|
| 项目 | `~/strix-llama.cpp`，master `c8912d7f5`，`build-hip`（Ninja Release） |
| 构建 | HIP/ROCm 10.1，`GGML_HIP=ON`、`GPU_TARGETS=gfx1151` |
| 主机 | AMD Ryzen AI MAX+ 395 w/ Radeon 8060S |
| GPU | `ROCm0`，gfx1151，128 GB 统一内存（报告 VRAM 122880 MiB） |
| 主模型 | `~/llm/deepseek-v4-flash-vision-exp-iq2-xxs/DeepSeek-V4-Flash-Vision-Exp-UD-IQ2_XXS-00001-of-00003.gguf`（3 分片，约 86 GB） |
| 模型架构 | `deepseek4`，43 层，256 experts / top-6，Vision-Exp |
| DSpark 草稿 | `/tmp/DeepSeek-V4-Flash-DSpark-draft-strix.gguf`（10 GB，Q4RMFP4 denseF16） |

## 测试方法

- 通过 `--spec-type draft-dspark --spec-draft-model <draft>` 开启 DSpark；关闭时不传这些参数，其余完全一致。
- 每次请求使用 `/tokenize` 得到的**精确 512 个 prompt token**，`cache_prompt=false`、
  `n_predict=128`、`ignore_eos=true`、贪心采样（`temperature=0, top_k=1`，`seed=1234`）。
- 先做一次 16-token 预热并丢弃，再连续测 3 次。
- 计时取 server 返回的 `timings`，pp = `prompt_n / prompt_ms`，tg = `predicted_n / predicted_ms`。
- 共同参数：`-c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on --parallel 1 -dev ROCm0`，KV cache f16。
- 环境：`HSA_OVERRIDE_GFX_VERSION=11.5.1`，ROCm 10.1 运行库。

DSpark 关闭：

```bash
~/strix-llama.cpp/build-hip/bin/llama-server \
  -m ~/llm/deepseek-v4-flash-vision-exp-iq2-xxs/DeepSeek-V4-Flash-Vision-Exp-UD-IQ2_XXS-00001-of-00003.gguf \
  -c 4096 -b 2048 -ub 512 -t 16 -ngl 999 -fa on \
  --parallel 1 -dev ROCm0 --host 127.0.0.1 --port 18099
```

DSpark 开启（追加）：

```bash
  --spec-type draft-dspark \
  --spec-draft-model /tmp/DeepSeek-V4-Flash-DSpark-draft-strix.gguf \
  --spec-draft-n-max 5 \
  --spec-draft-ngl all \
  --spec-draft-device ROCm0
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

| DSpark | 第几次 | prompt 耗时 (ms) | pp512 (t/s) | 生成耗时 (ms) | tg128 (t/s) | draft 接受 |
|---|---:|---:|---:|---:|---:|---:|
| 关 | 1 | 2832.32 | 180.77 | 7588.88 | 16.74 | — |
| 关 | 2 | 2292.18 | 223.37 | 7575.17 | 16.77 | — |
| 关 | 3 | 2580.49 | 198.41 | 7559.38 | 16.80 | — |
| 开 | 1 | 2369.58 | 216.07 | 6319.92 | 20.10 | 98/138 (71.0%) |
| 开 | 2 | 3038.60 | 168.50 | 6554.90 | 19.37 | 98/138 (71.0%) |
| 开 | 3 | 2359.38 | 217.01 | 5980.60 | 21.24 | 98/138 (71.0%) |

## 汇总

| 指标 | DSpark 关 | DSpark 开 | 变化 |
|---|---:|---:|---:|
| pp512 均值 | 200.85 ± 21.40 t/s | 200.53 ± 27.74 t/s | −0.2%（噪声内） |
| tg128 均值 | 16.767 ± 0.033 t/s | 20.235 ± 0.938 t/s | **+20.7%（1.21×）** |
| 生成 128 token 平均耗时 | 7574.5 ms | 6285.1 ms | 1.21× 更快 |
| 平均接受率 | — | 71.0% | draft 平均长度 4.50 |

## 校验与说明

- 每次运行：`prompt_n=512`、`predicted_n=128`、`cache_prompt=false`；
  DSpark 关闭组 `draft_n` 为空，开启组 `draft_n=138`、`draft_n_accepted=98`。
- 两组生成样本（观测到的 160 字符前缀）逐字符一致；DSpark 验证未改变贪心输出。
- pp 波动较大（168–223 t/s）：86 GB 主模型 + 10 GB 草稿接近 120 GB 统一内存上限，
  prefill 受内存/时钟影响明显；tg 较稳定。两组 pp 均值几乎相同，符合 DSpark 只作用于 decode。
- 接受率 71.0%（自然文本续写），加速 1.21×。draft 为 3 层、256 experts / top-6、Q4RMFP4，
  相对 IQ2_XXS 主模型每步开销较高，因此低于高接受率计数负载的理论收益。
- 与之前 lucebox 的 DSpark 46.9 t/s 不可直接比较：那是“1..1000 计数”高可预测负载、
  接受率 100%，且引擎/目标量化不同。
- 测试结束后无残留 `llama-server` 进程。

## 原始数据

- `/tmp/bench_dspark_strix.json`：逐次 timings、接受数
- `/tmp/bench_dspark_strix_dspark-off.log`、`/tmp/bench_dspark_strix_dspark-on.log`：两次 server 日志
- `/tmp/bench_dspark_strix.py`：复现脚本
- `/tmp/convert_dspark_strix.py`：lucebox → strix DSpark 草稿转换脚本
- `/tmp/DeepSeek-V4-Flash-DSpark-draft-strix.gguf`：转换后的草稿
