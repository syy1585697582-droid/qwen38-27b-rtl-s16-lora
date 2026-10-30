# Qwen3.8-27B RTL LoRA s16

本仓库提供一个用于 Verilog/SystemVerilog 生成的 LoRA 适配器。输入端口定义和行为规格后，它与 Qwen3.8-27B 基座一起生成 RTL，适合放在编译、仿真和综合验证流程中使用。

仓库提供 GGUF 适配器、校验和及许可证，基座按下文链接另行下载。`s16` 表示第 16 次优化器更新，对应 64 次样本曝光，共使用 64 条不同记录。

## 下载并校验权重

推理需要两个文件。复现时请使用下表的固定版本，并校验 SHA-256。

| 项目 | 基座 GGUF | LoRA 适配器 |
|---|---|---|
| 来源 | [unsloth/Qwen3.8-27B-GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF) | 本仓库 |
| revision | `4ca720788d1e01f1bff70c033e0d0028fd02e502` | `9c9f252408ef9ee248b5d233d0fcb6da24e28874` |
| 文件 | `Qwen3.8-27B-UD-Q6_K.gguf` | `adapter-s016.gguf` |
| 大小 | 21,983,677,344 bytes | 76,561,984 bytes |
| SHA-256 | `c9c206812fbe4ac7b76a729e25928b63f2ae89d37f69da7a71c20aec763cd436` | `b3be572618a92d5588eeee5e9db8edaec576b2e914839ee1e0435ee2f31ed145` |

以下命令适用于安装了 `curl` 和 `sha256sum` 的 Linux 环境。两个权重文件合计约 22.1 GB，部署和推理还需要额外磁盘空间。

```bash
mkdir -p models

curl --fail --location --retry 3 \
  'https://huggingface.co/unsloth/Qwen3.8-27B-GGUF/resolve/4ca720788d1e01f1bff70c033e0d0028fd02e502/Qwen3.8-27B-UD-Q6_K.gguf' \
  -o models/Qwen3.8-27B-UD-Q6_K.gguf

curl --fail --location --retry 3 \
  'https://github.com/syy1585697582-droid/qwen38-27b-rtl-s16-lora/raw/9c9f252408ef9ee248b5d233d0fcb6da24e28874/adapter-s016.gguf' \
  -o models/adapter-s016.gguf

printf '%s\n' \
  'c9c206812fbe4ac7b76a729e25928b63f2ae89d37f69da7a71c20aec763cd436  models/Qwen3.8-27B-UD-Q6_K.gguf' \
  'b3be572618a92d5588eeee5e9db8edaec576b2e914839ee1e0435ee2f31ed145  models/adapter-s016.gguf' \
  | sha256sum -c -
```

两项校验都应显示 `OK`。如果网络无法访问 GitHub 原始文件域名，可用以下匿名 API 下载同一适配器，然后重新运行上面的校验命令：

```bash
curl --fail --location --retry 3 \
  -H 'Accept: application/vnd.github.raw+json' \
  'https://api.github.com/repos/syy1585697582-droid/qwen38-27b-rtl-s16-lora/contents/adapter-s016.gguf?ref=9c9f252408ef9ee248b5d233d0fcb6da24e28874' \
  -o models/adapter-s016.gguf
```

最近一次整文件核验于 2026-10-31 00:32（北京时间）在本地 Windows 完成：通过匿名 API 下载，TLS 校验开启，文件大小与 SHA-256 均与上表一致。

训练所用 BF16 基座为 `Qwen/Qwen3.8-27B`，声明的 revision 为 `1d4bf0f2ff6012fd82039f2fa52739d0dd7c60c0`。推理复现以表中的 GGUF revision 和 SHA-256 为准；量化前的精确 BF16 revision 尚待确认。

## 启动本地推理

项目使用 W7900D（gfx1100）和 lemonade b1326 / upstream f114f91 的 llama.cpp。安装该后端并将 `llama-server` 加入 PATH 后，运行下面的命令，在 `127.0.0.1:8000` 前台启动服务。

```bash
llama-server \
  -m models/Qwen3.8-27B-UD-Q6_K.gguf \
  --lora models/adapter-s016.gguf \
  -ngl 999 -c 32768 -np 1 \
  --host 127.0.0.1 --port 8000 \
  --alias qwen38-27b-q6k-s16 \
  --jinja --reasoning off --reasoning-format deepseek \
  --spec-type draft-mtp \
  -t 4 -tb 4 --poll 0 \
  --no-cache-prompt --cache-ram 0 --ctx-checkpoints 0
```

`--lora` 默认以 scale 1.0 加载适配器。训练与推理使用 thinking-off 模式。MTP 使用基座 GGUF 中的 `nextn` 层起草 token，再由主模型验证。上面的启动参数对应 lemonade b1326；迁移到其他版本时，应核对其命令行和适配器支持。

在另一个终端发送请求：

```bash
curl --fail --silent --show-error --noproxy '*' \
  http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  --data-binary @- <<'JSON'
{
  "model": "qwen38-27b-q6k-s16",
  "messages": [
    {"role": "system", "content": "Return only synthesizable Verilog code."},
    {"role": "user", "content": "Implement module TopModule(input a, input b, output y); y is the logical AND of a and b."}
  ],
  "temperature": 0,
  "top_p": 1.0,
  "max_tokens": 8192,
  "chat_template_kwargs": {"enable_thinking": false},
  "reasoning_effort": null
}
JSON
```

从响应的 `choices[0].message.content` 取得生成文本，提取其中的 RTL 后即可送入编译与仿真流程。

### 显存需求

2026-10-30 完整开发集显存测量使用 W7900D、32768 token 上下文、单 slot、MTP 和上述适配器。按约 1 Hz 读取分配给容器的 GPU 的 `mem_info_vram_used`，共取得 13045 个样本，最大值为 25,740,701,696 bytes，即 25.741 GB 或 23.973 GiB。

2026-10-28 的历史稳态记录为 23532 MiB（约 22.98 GiB）。显存容量按完整回归的采样峰值规划；更换上下文长度、后端或并发配置后，应重新测量。

## 训练方法

适配器从新的 LoRA 参数开始训练，只更新基座的文本层。单轮对话中的题面与空思考前缀被掩码，只监督最终 RTL 答案直到 EOS。

| 项目 | 设置 |
|---|---|
| 方法 | PEFT LoRA，r=16，alpha=32，dropout=0.05，bias=none |
| 目标层 | full-attention 的 q/k/v/o，linear-attention 的 out_proj；vision 和 MTP 参数冻结 |
| 参数量 | 19,136,512 个可训练参数；GGUF 含 224 个张量 |
| 优化器 | AdamW，学习率 5e-7，weight decay 0.01，梯度裁剪 1.0，warmup ratio 0.05 |
| 批次与随机种子 | microbatch 1，梯度累积 4，max_length 4096，seed 42 |
| 训练计划 | 64 次优化器更新，共 256 次样本曝光 |
| 发布检查点 | 第 16 次更新，共 64 次曝光，涉及 64 条不同记录 |

## 训练数据从哪里来

训练池包含 204 条不同的“题面—已验证 RTL 答案”记录，`train.jsonl` 的 SHA-256 为 `6c332ac2d73f0050c8d4234ab26ed49dd65c0e9124e62f3ab9e68340ca1e3b7e`。s16 完成前 16 次更新，使用其中 64 条：16 条既有组合逻辑、16 条既有时序逻辑，以及覆盖 23 个新增时序规格的 32 条记录。

既有数据组共 128 条，选自项目内部的 2048 条数据集，覆盖 64 个手工设计的行为族。规格由项目 Python 模板生成，RTL 答案由 AI 编码助手编写，并配有独立实现的 Python 行为模型。每条记录经 Vivado 2024.2 仿真，与独立模型的 192 组向量比对；验证过程检出了四种刻意注入的错误。

新增数据组共 76 条，围绕 23 个自建时序规格生成。8 个规格在项目内形成，另 15 个由 Claude Sonnet 会话起草，经项目修订并配独立 Python 行为模型。答案来源如下：

| 来源 | 记录数 |
|---|---:|
| 教师模型生成的参考实现 | 23 |
| 教师模型生成的合法替代实现 | 21 |
| Xiaomi MiMo-v2.6-pro 通过验证的答案 | 15 |
| 首答失败后修复通过的答案 | 16 |
| 项目早期微调模型通过验证的首答 | 1 |

教师与修复记录包含 gpt-5.6-terra、gpt-5.6-sol 等模型。

新增样本使用独立行为模型生成公开与留出两套周期轨迹，以 Vivado 仿真检查 RTL 的逐周期输出。每个规格还构造两个错误变异体，用于确认验证器能够检出错误。通过两套轨迹比对、变异体检查和 Vivado 综合后，将题面与最终答案组成训练记录。留出轨迹用于训练样本验收。

对全部 204 条记录进行归一化代码精确匹配与连续 8 词片段筛查，在 VerilogEval v2（156 题）、RTLLM v2.1（50 题）和 3 道竞赛示例题中的精确匹配均为 0，代码相似度最大值分别为 0.194、0.262、0.073。

数据范围：训练池全部来自上述项目数据，公开 RTL 数据集及第三方基准用于对照筛查。筛查覆盖文本与代码片段；语义等价及基座预训练重叠情况未评估。Claude 会话的精确版本和 3 条修复答案的生成型号未记录。训练数据保存在项目内部，公开仓库提供适配器推理所需的权重与说明。

## 适用范围与许可

该适配器用于从接口和行为规格生成 RTL，已在 AMD/ROCm 上完成项目开发回归。检查点选择和调试采用开发数据，评价环节覆盖编译、功能仿真与综合。使用生成的 RTL 时，应针对目标硬件验证位宽、复位、周期行为和时序约束。

适配器按 [Apache-2.0](LICENSE) 发布，基座同样声明 Apache-2.0。适配器校验和见 [SHA256SUMS](SHA256SUMS)。
