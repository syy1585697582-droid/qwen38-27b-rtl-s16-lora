# Qwen3.8-27B · RTL LoRA s16（GGUF 适配器）

一个面向 **SystemVerilog/Verilog RTL 生成** 的文本层 LoRA 适配器，叠加在 `Qwen/Qwen3.8-27B` 上使用（推理时配合 unsloth 的 Q6_K GGUF 基座）。
A text-layer LoRA adapter for spec-to-RTL generation, applied on top of `Qwen/Qwen3.8-27B` (served with the unsloth UD-Q6_K GGUF base via llama.cpp).

## 文件

| 文件 | 字节 | SHA-256 |
|---|---|---|
| `adapter-s016.gguf` | 76,561,984 | `b3be572618a92d5588eeee5e9db8edaec576b2e914839ee1e0435ee2f31ed145` |

## 基座

| 项 | 值 |
|---|---|
| 声明基座 | `Qwen/Qwen3.8-27B`（BF16 revision `1d4bf0f2ff6012fd82039f2fa52739d0dd7c60c0`） |
| 推理用 GGUF | [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF) @ `4ca720788d1e01f1bff70c033e0d0028fd02e502`，文件 `Qwen3.8-27B-UD-Q6_K.gguf`（21,983,677,344 bytes，SHA-256 `c9c206812fbe4ac7b76a729e25928b63f2ae89d37f69da7a71c20aec763cd436`） |

历史材料未证明推理用 GGUF 量化时采用了上述精确 BF16 revision；推理权重以所列 GGUF revision 和 SHA-256 为准。本仓库仅包含适配器，基座需另行下载。

## 用法（llama.cpp）

```bash
llama-server -m Qwen3.8-27B-UD-Q6_K.gguf --lora adapter-s016.gguf \
  -ngl 999 -c 32768 -np 1 --jinja --reasoning off --reasoning-format deepseek
```

训练与推理都使用 thinking-off 模式。W7900D / gfx1100 上，`-c 32768`、无 MTP 时实测显存约 22.3 GiB；加入 `--spec-type draft-mtp` 后稳态为 23,532 MiB（约 22.98 GiB，2026-10-28 测量）。

## 训练

| 项 | 内容 |
|---|---|
| 方法 | PEFT LoRA：r=16，alpha=32，dropout=0.05，bias=none；19,136,512 个可训练参数；GGUF 共 224 个张量 |
| 目标层 | full-attention 的 q/k/v/o 与 linear-attention 的 out_proj；vision 塔与 MTP 头未训练 |
| 超参 | AdamW，lr 5e-7，weight decay 0.01，梯度裁剪 1.0，warmup ratio 0.05，micro-batch 1，梯度累积 4，max_length 4096，seed 42 |
| 检查点 | 计划 64 次优化器更新；本文件是第 16 次更新（64 次样本曝光） |
| 监督目标 | 单轮对话；题面与空思考前缀被掩码，只监督最终答案（Verilog 代码）直到 EOS |

## 训练数据来源

数据池共 204 条互不相同的记录（`train.jsonl` SHA-256 `6c332ac2d73f0050c8d4234ab26ed49dd65c0e9124e62f3ab9e68340ca1e3b7e`）。本检查点实际只见过其中 64 条：32 条来自“旧”组（16 条组合逻辑、16 条时序逻辑），32 条来自“新”组（覆盖全部 23 个规格合同）。

- **“旧”组（池中 128 条）**：项目内部 V5 数据集（2048 条，64 个手工设计的行为族）。
  - 规格由项目的 Python 模板生成；RTL 答案为“AI 编码助手编写的 RTL + 独立实现的 Python 行为模型”。
  - 每条都经 Vivado 2024.2 仿真，并与独立 Python oracle 的 192 组向量比对，4 种负控全部检出。
- **“新”组（池中 76 条）**：项目自建的 23 个时序规格，其中 8 个由项目自写，15 个由 Claude Sonnet 起草、经项目修订，并配有独立 Python oracle。
  - 答案来源：
    - 教师模型 gpt-5.6-terra/high 等：gold 23 条、合法替代实现 21 条；
    - xiaomi mimo-v2.6-pro 验证通过的样本 15 条；
    - 首答失败、修复后通过的样本 16 条；
    - 项目早期自研模型的首答 1 条。
  - 入库条件：同时满足以下三点——
    1. 通过独立 oracle 的公开与留出两套周期轨迹；
    2. 能区分 2 个变异体；
    3. 通过 Vivado 综合。
- **不使用公开数据集**。
  - 我们用 8-word shingle 包含率和归一化代码精确匹配，对照 VerilogEval v2（156 题）、RTLLM v2.1（50 题）和 3 道竞赛官方示例题做了筛查：精确匹配为 0，代码相似度最大值分别为 0.194、0.262、0.073。
  - 这只是筛查，不能证明语义上互相独立；基座在预训练时是否见过这些基准，我们无从得知。

训练数据本身不随本仓库发布。

## 许可

本适配器以 Apache-2.0 发布，与基座 Qwen3.8-27B 的许可一致（见 `LICENSE`）。
