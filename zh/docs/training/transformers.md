---
status: Active
maintainer: pacoxu
last_updated: 2025-10-31
tags: transformers, huggingface, pytorch, model-definitions, standardization
canonical_path: zh/docs/training/transformers.md
---

# Transformers：标准化跨 PyTorch 生态系统的模型定义

## 概述

<a href="https://github.com/huggingface/transformers">`Transformers`</a> 来自 Hugging Face，是跨 PyTorch 生态系统的事实标准模型定义库。在 GitHub 上拥有超过 400,000 个依赖项目，Transformers 处于模型发布周期的中心位置，为训练框架、推理引擎、预训练工具和本地部署方案提供标准化接口。

Transformers 目前与绝大多数模型提供商合作，是整个 AI 基础设施领域中一致模型定义的基础。

## Transformers 在生态系统中的位置

### 中心枢纽架构

Transformers 充当连接多类工具的中央标准化层：

```text
┌─────────────────────────────────────────────────────────────────┐
│                    微调框架                                       │
│  Unsloth, TRL, Axolotl, PyTorch Lightning, LLaMA-Factory,       │
│                         MaxText                                  │
└────────────────────────────────┬────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────┐
│                                                                  │
│                         transformers                             │
│            （标准化模型定义）                                       │
│                                                                  │
└────────────┬────────────────────┬──────────────────┬────────────┘
             │                    │                  │
┌────────────▼──────────┐ ┌──────▼─────────┐ ┌─────▼────────────┐
│  预训练工具             │ │ 推理引擎         │ │ 本地部署          │
│  • torchtitan         │ │ • vLLM          │ │ • LLaMA.cpp      │
│  • NVIDIA NEMO        │ │ • SGLang        │ │ • MLX            │
│  • Nanotron           │ │ • TensorRT      │ │ • candle         │
│                       │ │                  │ │                  │
└───────────────────────┘ └─────────────────┘ └──────────────────┘
```

### 微调框架

Transformers 为主要的微调工具提供基础：

- <a href="https://github.com/unslothai/unsloth">`Unsloth`</a>：超快速微调，带内存优化
- <a href="https://github.com/huggingface/trl">`TRL`</a>（Transformer Reinforcement Learning）：RLHF 和偏好学习
- <a href="https://github.com/axolotl-ai-cloud/axolotl">`Axolotl`</a>：精简微调工具包
- <a href="https://github.com/Lightning-AI/pytorch-lightning">`PyTorch Lightning`</a>：高层训练框架
- <a href="https://github.com/hiyouga/LLaMA-Factory">`LLaMA-Factory`</a>：高效的 LLM 微调框架
- <a href="https://github.com/google/maxtext">`MaxText`</a>：Google 的高性能 LLM 训练

### 推理引擎

主要推理引擎依赖 Transformers 模型定义：

- <a href="https://github.com/vllm-project/vllm">`vLLM`</a>：使用 PagedAttention 的快速 LLM 推理
- <a href="https://github.com/sgl-project/sglang">`SGLang`</a>：面向 LLM 的结构化生成语言
- <a href="https://github.com/NVIDIA/TensorRT-LLM">`TensorRT-LLM`</a>：NVIDIA 优化的推理引擎
- <a href="https://github.com/huggingface/text-generation-inference">`TGI`</a>：Hugging Face 的生产级推理方案

### 预训练工具

预训练框架基于 Transformers 标准构建：

- <a href="https://github.com/pytorch/torchtitan">`torchtitan`</a>：PyTorch 原生的大规模预训练
- <a href="https://github.com/NVIDIA/NeMo">`NVIDIA NEMO`</a>：构建和定制 AI 模型的一体化平台
- <a href="https://github.com/huggingface/nanotron">`Nanotron`</a>：面向预训练 transformer 的 3D 并行

### 本地部署

本地部署工具利用 Transformers 模型格式：

- <a href="https://github.com/ggerganov/llama.cpp">`LLaMA.cpp`</a>：纯 C/C++ 推理，支持量化
- <a href="https://github.com/ml-explore/mlx">`MLX`</a>：Apple Silicon 优化的 ML 框架
- <a href="https://github.com/huggingface/candle">`candle`</a>：基于 Rust 的极简 ML 框架

## Transformers 第 5 版：重点领域

Transformers v5 代表了一次重大演进，在多个维度上进行了有针对性的改进：

### 1. 性能赋能：针对目标加速的一致定义

**目标：** 提供一致的模型定义，使整个生态系统能够实现特定硬件的优化。

- **标准化接口：** 模型组件的通用 API（注意力、嵌入、MLP 层）
- **优化钩子：** 定义良好的扩展点，供硬件供应商注入优化内核
- **加速器支持：** 与 CUDA、ROCm、XPU、NPU 和其他加速器的无缝集成
- **性能原语：** 高效实现，作为进一步优化的基线

**影响：**

- 推理引擎可以优化特定操作，同时保持模型兼容性
- 硬件供应商可以针对单一标准进行优化
- 性能改进自动惠及所有下游框架

### 2. 建模工具包：使用通用 API 快速集成模型

**目标：** 以最少的代码重复实现新模型架构的快速集成。

- **模块化架构：** 通用模式的可复用组件（注意力机制、归一化层、激活函数）
- **配置系统：** 用于模型定制的灵活配置类
- **AutoModel 类：** 基于配置自动检测和加载模型
- **向后兼容：** 模型更新的平滑迁移路径
- **测试框架：** 全面的模型验证测试套件

**影响：**

- 新模型可在数天内（而非数周）添加
- 减少了跨相似架构的代码重复
- 更易于跨模型系列的维护和错误修复
- 社区贡献变得更加便捷

### 3. 代码即产品：可读、清晰、易懂的代码

**目标：** 维护高代码质量标准，使 Transformers 成为可靠的参考实现。

**原则：**

- **简单胜于巧妙：** 优先选择清晰直接的实现
- **标准化：** 一致的命名约定、代码结构和文档模式
- **文档：** 全面的文档字符串、教程和示例
- **类型注解：** 完整的类型提示，以获得更好的 IDE 支持和错误检测
- **代码审查：** 对所有贡献进行严格的审查流程
- **测试：** 广泛的测试覆盖以保障可靠性

**影响：**

- 工程师可以快速理解和调试模型实现
- 在整个生态系统中更易于识别和修复错误
- 作为学习模型架构的教育资源
- 建立生产部署中的信任

### 4. 训练原语：Trainer API 和 Accelerate

**目标：** 提供简化分布式训练同时保持灵活性的高级 API。

#### Trainer API

Trainer API 抽象了常见的训练模式：

```python
from transformers import Trainer, TrainingArguments

training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=8,
    gradient_accumulation_steps=4,
    learning_rate=5e-5,
    warmup_steps=500,
    logging_steps=100,
    save_steps=1000,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
)

trainer.train()
```

**功能特性：**

- 自动混合精度训练
- 梯度累积和检查点
- 分布式训练支持（DDP、FSDP）
- 与日志工具的集成（TensorBoard、Weights & Biases）
- 评估和指标计算
- 早停和学习率调度

#### Accelerate

<a href="https://github.com/huggingface/accelerate">`Accelerate`</a> 提供了一个轻量级的分布式训练封装：

```python
from accelerate import Accelerator

accelerator = Accelerator()
model, optimizer, dataloader = accelerator.prepare(
    model, optimizer, dataloader
)

for batch in dataloader:
    optimizer.zero_grad()
    outputs = model(**batch)
    loss = outputs.loss
    accelerator.backward(loss)
    optimizer.step()
```

**优势：**

- 编写一次，随处运行（CPU、GPU、TPU、多节点）
- 分布式训练的最小代码修改
- 自动设备放置和梯度同步
- 与 DeepSpeed、FSDP 和其他后端的集成
- 开箱即用的混合精度支持

**影响：**

- 降低了分布式训练的入门门槛
- 跨不同硬件配置的一致行为
- 简化的调试和开发工作流
- 最少量样板代码的生产级训练循环

### 5. 模型无关的实用工具

**目标：** 提供跨所有模型架构工作的工具，无需模型特定的定制。

**关键工具：**

- **Tokenizers：** 跨模型的快速一致的分词
- **AutoClasses：** 自动的模型/分词器/配置检测
- **Pipeline API：** 常见任务的高级接口
- **Model Hub 集成：** 从 Hugging Face Hub 无缝加载
- **量化工具：** 模型压缩工具
- **ONNX 导出：** 跨框架互操作性
- **剪枝和蒸馏：** 模型压缩技术

**影响：**

- 简化的从业者工作流
- 跨不同模型的一致体验
- 更轻松的模型比较和基准测试
- 新模型的集成开销降低

## Transformer 架构：训练与推理

### 训练过程

训练期间，transformer 架构以并行的 teacher-forcing 方式运行：

```text
训练：时间步 = 1（所有操作在一个时间步内完成）

输入："<SOS> I love you very much <EOS>" → Ti amo molto <EOS>
                                              （目标/标签）

┌─────────────────────────────────────────────────────────────┐
│                        编码器堆栈                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  编码器输入 → 位置编码                                │   │
│  │  ↓                                                    │   │
│  │  多头注意力（自注意力）                                │   │
│  │  ↓                                                    │   │
│  │  Add & Norm → 前馈网络 → Add & Norm                   │   │
│  │  ↓                                                    │   │
│  │  [重复 Nx 次]                                         │   │
│  │  ↓                                                    │   │
│  │  编码器输出：(seq, d_model)                            │   │
│  └──────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────┘
                                │ 交叉注意力连接
┌───────────────────────────────▼─────────────────────────────┐
│                        解码器堆栈                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  解码器输入："<SOS> Ti amo molto"                    │   │
│  │  ↓                                                    │   │
│  │  输出嵌入 → 位置编码                                  │   │
│  │  ↓                                                    │   │
│  │  带掩码的多头自注意力                                  │   │
│  │  ↓                                                    │   │
│  │  Add & Norm                                          │   │
│  │  ↓                                                    │   │
│  │  多头交叉注意力（关注编码器）                            │   │
│  │  ↓                                                    │   │
│  │  Add & Norm → 前馈网络 → Add & Norm                   │   │
│  │  ↓                                                    │   │
│  │  [重复 Nx 次]                                         │   │
│  │  ↓                                                    │   │
│  │  解码器输出：(seq, d_model)                            │   │
│  └──────────────────────────────────────────────────────┘   │
│                            ↓                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Linear: (seq, d_model) → (seq, vocab_size)         │   │
│  └──────────────────────────────────────────────────────┘   │
│                            ↓                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Softmax: (seq, vocab_size)                          │   │
│  └──────────────────────────────────────────────────────┘   │
│                            ↓                                  │
│                  交叉熵损失                                    │
│          （与目标比较："Ti amo molto <EOS>"）                 │
└─────────────────────────────────────────────────────────────┘
```

**关键训练特征：**

- **Teacher Forcing：** 解码器接收正确的前一个 token 作为输入，而非自身的预测
- **并行处理：** 训练期间所有位置同时处理
- **掩码注意力：** 解码器的自注意力被掩码以防止关注未来位置
- **单个时间步：** 整个序列在一次前向传播中处理
- **交叉熵损失：** 跨所有位置同时计算损失

### 推理过程

推理期间，transformer 以自回归方式生成 token：

```text
推理：时间步 = 1

输入："<SOS> I love you very much <EOS>"
输出：Ti（第一个 token）

┌─────────────────────────────────────────────────────────────┐
│                    编码器（不变）                               │
│  一次性处理输入并缓存编码器输出                                │
└───────────────────────────────┬─────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────┐
│                         解码器                                │
│  输入："<SOS>"                                               │
│  ↓                                                           │
│  解码器输出 → Linear → Softmax                               │
│  ↓                                                           │
│  选择最大概率的 token："Ti"                                    │
└─────────────────────────────────────────────────────────────┘

推理：时间步 = 2

┌─────────────────────────────────────────────────────────────┐
│                编码器输出（从缓存中复用）                       │
└───────────────────────────────┬─────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────┐
│                         解码器                                │
│  输入："<SOS> Ti"（追加前一个输出）                           │
│  ↓                                                           │
│  解码器输出 → Linear → Softmax                               │
│  ↓                                                           │
│  选择第二个 token："amo"                                      │
└─────────────────────────────────────────────────────────────┘

推理：时间步 = N（持续到 <EOS>）
```

**关键推理特征：**

- **自回归生成：** 输出 token 一次生成一个
- **顺序处理：** 每个 token 依赖于所有之前的 token
- **KV 缓存：** 编码器输出和之前的解码器状态被缓存以避免重计算
- **贪心/采样：** 通过 argmax（贪心）或从分布中采样选择 token
- **可变长度：** 生成持续到序列结束 token 或最大长度

### 训练与推理的差异

| 方面 | 训练 | 推理 |
|--------|----------|-----------|
| **时间步** | 单次前向传播 | 多个顺序步骤 |
| **输入** | 完整目标序列 | 之前生成的 token |
| **解码器输入** | 真实标签（教师） | 模型预测 |
| **并行化** | 完全并行 | 顺序生成 |
| **速度** | 快（并行） | 较慢（顺序） |
| **内存** | 更高（存储所有梯度） | 更低（仅前向传播） |
| **输出** | 所有位置的损失 | 每个步骤一个 token |
| **目的** | 学习模型参数 | 生成新序列 |

### 常见优化

**训练优化：**

- **混合精度：** FP16/BF16 实现更快的计算
- **梯度累积：** 有效的更大批次大小
- **梯度检查点：** 以计算换内存
- **数据并行：** 在 GPU 间分发批次
- **模型并行：** 在 GPU 间拆分模型层

**推理优化：**

- **KV 缓存：** 避免重计算注意力键值
- **持续批处理：** 请求的动态批处理
- **量化：** INT8/INT4 减少内存并加速计算
- **Flash Attention：** 优化的注意力计算
- **推测解码：** 带验证的并行 token 生成

## 学习主题

### 模型架构基础

- **注意力机制：**
  - 自注意力和交叉注意力
  - 多头注意力
  - 缩放点积注意力
  - 位置级前馈网络

- **Transformer 变体：**
  - 仅编码器模型（BERT、RoBERTa）
  - 仅解码器模型（GPT、Llama、Mistral）
  - 编码器-解码器模型（T5、BART）
  - 现代架构（Llama 3/4、Qwen、Mixtral、DeepSeek）

- **位置编码：**
  - 正弦位置嵌入
  - 可学习位置嵌入
  - 旋转位置嵌入（RoPE）
  - ALiBi（线性偏置注意力）

### 训练技术

- **微调策略：**
  - 完整微调
  - 参数高效微调（PEFT）
  - LoRA（低秩适配）
  - QLoRA（量化 LoRA）
  - Adapter 层

- **训练稳定性：**
  - 层归一化
  - 学习率调度
  - 梯度裁剪
  - 权重衰减
  - 混合精度训练

- **数据处理：**
  - 分词策略
  - 词汇表构建
  - 数据增强
  - 课程学习

### 推理优化

- **生成策略：**
  - 贪心解码
  - 束搜索
  - Top-k 采样
  - Top-p（核）采样
  - 温度缩放

- **性能优化：**
  - 模型量化（GPTQ、AWQ、GGUF）
  - KV 缓存管理
  - Flash Attention 和其他注意力优化
  - 持续批处理
  - 推测解码

- **部署：**
  - 模型服务框架（vLLM、TGI、TensorRT-LLM）
  - 硬件考量（GPU、CPU、边缘设备）
  - 延迟 vs 吞吐量权衡

### 生态系统集成

- **Model Hub：**
  - 发布和共享模型
  - 模型版本控制
  - 模型卡片和文档
  - 许可证合规

- **工具集成：**
  - 将 Transformers 与 PyTorch 一起使用
  - 与训练框架的集成
  - 连接到推理引擎
  - 评估和基准测试

## 项目和资源

### 核心库

- <a href="https://github.com/huggingface/transformers">`Transformers`</a>：面向 PyTorch、TensorFlow 和 JAX 的最先进机器学习库
- <a href="https://github.com/huggingface/accelerate">`Accelerate`</a>：使用多 GPU、TPU、混合精度训练和使用 PyTorch 模型
- <a href="https://github.com/huggingface/tokenizers">`Tokenizers`</a>：快速可定制的文本分词库
- <a href="https://github.com/huggingface/peft">`PEFT`</a>：最先进的参数高效微调方法
- <a href="https://github.com/huggingface/optimum">`Optimum`</a>：transformer 的硬件加速和优化

### 训练框架

- <a href="https://github.com/huggingface/trl">`TRL`</a>：Transformer Reinforcement Learning
- <a href="https://github.com/unslothai/unsloth">`Unsloth`</a>：2-5 倍更快的 LLM 微调
- <a href="https://github.com/axolotl-ai-cloud/axolotl">`Axolotl`</a>：LLM 微调的首选方案
- <a href="https://github.com/hiyouga/LLaMA-Factory">`LLaMA-Factory`</a>：易用的 LLM 微调框架

### 推理工具

- <a href="https://github.com/vllm-project/vllm">`vLLM`</a>：高吞吐量和内存高效的推理引擎
- <a href="https://github.com/huggingface/text-generation-inference">`TGI`</a>：生产级推理容器
- <a href="https://github.com/sgl-project/sglang">`SGLang`</a>：前端和运行时协同设计的结构化生成
- <a href="https://github.com/NVIDIA/TensorRT-LLM">`TensorRT-LLM`</a>：NVIDIA 优化的推理库

### 文档和学习

- [Hugging Face Course](https://huggingface.co/learn/nlp-course/chapter1/1)：使用 Transformers 的综合 NLP 课程
- [Transformers Documentation](https://huggingface.co/docs/transformers/)：官方文档，包含教程和 API 参考
- [Attention is All You Need (Blog)](https://huggingface.co/blog/Esmail-AGumaan/attention-is-all-you-need)：transformer 架构的详细解释
- [PyTorch Conference: Transformers Standardization Talk](https://sched.co/27QDi)：Lysandre Debut & Arthur Zucker, Hugging Face - Transformers: Standardizing Model Definitions Across the PyTorch Ecosystem

## 最佳实践

### 模型开发

- **从现有模型开始：** 利用预训练模型并为您的任务进行微调
- **使用 AutoClasses：** 启用自动模型检测和配置
- **类型注解：** 维护类型提示以获得更好的 IDE 支持
- **文档：** 编写全面的文档字符串和示例
- **测试：** 为自定义组件添加单元测试

### 训练

- **梯度累积：** 在有限硬件上实现有效的更大批次大小
- **混合精度：** 启用 FP16/BF16 以实现更快的训练
- **检查点：** 定期保存模型并实现可恢复训练
- **监控：** 使用 TensorBoard 或 Weights & Biases 跟踪指标
- **验证：** 在留出数据集上定期评估

### 部署

- **量化：** 在适当时使用量化模型进行推理
- **批处理：** 利用批量推理提高吞吐量
- **硬件选择：** 根据延迟/吞吐量要求选择合适的硬件
- **监控：** 跟踪推理延迟、吞吐量和资源使用情况
- **版本控制：** 维护模型版本并跟踪配置

## 路线图（持续开发中）

### Transformers 核心

- **第 5 版增强：**
  - 改进的模块化和代码组织
  - 增强的性能原语
  - 更好的文档和示例
  - 扩展的模型覆盖范围

- **社区发展：**
  - 加强与模型提供商的合作
  - 更广泛的生态系统集成
  - 更多社区贡献的模型
  - 增强的工具和实用程序

### 未来生态系统集成

- **训练框架：**
  - 与分布式训练工具更深度的集成
  - 改进的 PEFT 方法
  - 更好的多模态支持

- **推理引擎：**
  - 与 vLLM、TGI 和其他引擎更紧密的集成
  - 标准化的优化接口
  - 跨框架兼容性改进

- **硬件支持：**
  - 扩展的加速器支持
  - 特定硬件优化
  - 更好的性能分析和调试工具

## 贡献

Transformers 项目欢迎社区贡献：

- **模型集成：** 添加新的模型架构
- **优化：** 改进现有模型的性能
- **文档：** 增强指南和示例
- **Bug 修复：** 报告和修复问题
- **社区支持：** 在讨论和论坛中帮助他人

详见 [Transformers Contributing Guide](https://github.com/huggingface/transformers/blob/main/CONTRIBUTING.md) 了解详细说明。

## 参考

### 官方资源

- [Transformers GitHub](https://github.com/huggingface/transformers)
- [Hugging Face Documentation](https://huggingface.co/docs)
- [Hugging Face Hub](https://huggingface.co/models)
- [Hugging Face Blog](https://huggingface.co/blog)

### 学术论文

- ["Attention is All You Need"](https://arxiv.org/abs/1706.03762)：原始 transformer 论文
- ["BERT: Pre-training of Deep Bidirectional Transformers"](https://arxiv.org/abs/1810.04805)
- ["Language Models are Few-Shot Learners"](https://arxiv.org/abs/2005.14165)：GPT-3 论文
- ["LLaMA: Open and Efficient Foundation Language Models"](https://arxiv.org/abs/2302.13971)

### 会议演讲

- [PyTorch Conference 2024: Transformers Standardization](https://sched.co/27QDi)：Lysandre Debut & Arthur Zucker, Hugging Face 演讲 - Transformers: Standardizing Model Definitions Across the PyTorch Ecosystem

### 博文

- [Attention is All You Need - Explained](https://huggingface.co/blog/Esmail-AGumaan/attention-is-all-you-need)：transformer 架构的全面解释，包含训练和推理细节

---

**注意：** 本文档反映了 Transformers 库和生态系统的当前状态。由于这是一个快速发展的领域，请参考官方文档以获取最新信息。
