---
status: Active
maintainer: pacoxu
last_updated: 2025-10-31
tags: pre-training, moe, distributed-training, pytorch, torchtitan,
  deepseekv3, llama4, amd-mi325
canonical_path: zh/docs/training/pre-training.md
---

# 预训练大语言模型

## 概述

大规模预训练大语言模型（LLM）需要复杂的分布式训练技术，特别是对于混合专家（MoE）架构。本指南涵盖 MoE 预训练的独特挑战、性能优化策略以及使用 DeepseekV3-671B 和 Llama4 等最先进模型的扩展实验。

## MoE 预训练的挑战

混合专家模型由于其稀疏激活模式和动态路由机制，引入了与稠密模型相比独特的挑战。

### 1. 内核效率

MoE 模型面临显著的内核效率挑战：

- **专家微批次 → 小规模 GEMM：**
  - 每个专家处理一小部分 token
  - 导致许多小型 GEMM 操作，而非大型批次
  - 由于计算单元利用不足，GPU 利用率低下

- **Permute 开销：**
  - Token 路由需要昂贵的排列操作
  - 内存密集型操作，增加延迟
  - 难以与计算重叠

- **分组/融合内核：**
  - 多个专家计算可以分组在一起
  - 内核融合减少内存带宽需求
  - 需要专门的实现（例如 FP8 分组 GEMM）

- **填充稀疏内核：**
  - 专家间的负载不均衡需要填充
  - 填充 token 上的计算浪费
  - 效率与负载均衡之间的权衡

### 2. 并行

MoE 架构需要多种并行策略的精心协调：

- **FSDP × PP × EP 的交互产生气泡：**
  - 全分片数据并行（FSDP）：分片模型参数
  - 流水线并行（PP）：跨设备分区模型层
  - 专家并行（EP）：跨设备分发专家
  - 这些策略之间的交互产生流水线气泡
  - 气泡降低 GPU 利用率和吞吐量

- **负载均衡挑战：**
  - 专家间的 token 分布不均
  - 一些专家可能空闲，而其他专家过载
  - 动态负载均衡增加协调开销

### 3. 通信

MoE 训练中的通信模式在大规模下占主导地位：

- **EP ranks 之间的 All2all 在大规模下占主导：**
  - 专家间 token 路由的全到全通信
  - 带宽密集型操作，随规模增长
  - 可能成为大规模训练的主要瓶颈

- **通信-计算重叠：**
  - 隐藏通信延迟的机会有限
  - Token 路由依赖阻止完全重叠
  - 需要仔细的调度和流水线化

### 4. 路由与稳定性

路由机制引入了训练稳定性方面的挑战：

- **负载不均衡：**
  - 热门专家收到不成比例的 token 数量
  - 利用不足的专家无法有效学习
  - 需要负载均衡损失项和辅助损失

- **Token 丢弃：**
  - 容量约束强制丢弃 token
  - 信息丢失和训练不稳定
  - 可能导致路由崩溃（所有 token 流向少数专家）

- **训练动态：**
  - 路由器的学习需要仔细调优
  - 探索-利用权衡
  - 可能需要课程学习或逐步增加容量

## MoE 预训练加速

### TorchTitan 在 AMD MI325 上

Meta 的 TorchTitan 框架为大规模模型提供优化的预训练基础设施，特别针对 AMD MI325 GPU 进行了加速。

#### AMD GPU 架构优势

AMD MI325 GPU 为 MoE 预训练提供独特优势：

- **高内存带宽：**
  - 对通信密集型的 MoE 工作负载至关重要
  - 高效处理 all-to-all 操作

- **矩阵核心优化：**
  - 针对 FP8 和 BF16 优化的 GEMM 内核
  - 支持分组 GEMM 操作

- **大容量片上内存：**
  - 减少全局内存访问延迟
  - 更好地处理小型专家批次

#### 并行策略设计

有效的 MoE 预训练需要仔细的并行策略设计：

- **专家并行（EP）度：**
  - 跨多个 GPU 分发专家
  - 减少每个 GPU 的内存，支持更大的模型
  - 需要用于 token 路由的 all-to-all 通信

- **数据并行（DP）维度：**
  - 跨 DP 组复制专家
  - 增加吞吐量而不产生 all-to-all 开销
  - 内存与通信之间的权衡

- **流水线并行（PP）阶段：**
  - 对非常大的模型进行分层分区
  - 需要仔细的气泡最小化
  - 交错调度可提高效率

#### Primus-Turbo 优化

Primus-turbo 为 AMD GPU 提供了一套内核级优化：

- **FP8 分组 GEMM：**
  - 将多个专家 GEMM 批量处理到单个内核中
  - 减少内核启动开销
  - 提高小批次的 GPU 利用率

- **FP8 GEMM 内核：**
  - 使用 FP8 算术的低精度训练
  - 更高的吞吐量，保持精度
  - 减少内存带宽需求

- **Aiter ASM FAv3（Flash Attention v3）：**
  - 针对 AMD GPU 的手工调优汇编级 Flash Attention v3 实现
  - 减少注意力计算的内存占用
  - 长序列训练的更快注意力
  - "Aiter" 指汇编代码中 AMD 优化的迭代器模式

- **DeepEP（深度专家并行）：**
  - 具有层次路由的高级专家并行
  - 减少 all-to-all 通信开销
  - 跨专家组的更好负载均衡

## DeepseekV3 和 Llama4 预训练

### DeepseekV3-671B 扩展

DeepseekV3-671B 是最大的开放 MoE 模型之一，需要复杂的扩展策略。

#### 模型架构

- **671B 总参数：**
  - 稠密层：所有 token 共享
  - MoE 层：具有专家路由的稀疏激活
  - 典型配置：每个 token 激活 8 个专家

- **多头潜在注意力：**
  - 针对长上下文的高效注意力机制
  - 与标准注意力相比减少内存占用

- **无辅助损失负载均衡：**
  - 无需辅助损失的新型负载均衡（传统 MoE 使用辅助损失来鼓励均衡的专家使用，这可能损害模型质量）
  - 提高训练稳定性和收敛性
  - 消除了辅助损失权重的超参数调优

#### DeepseekV3 的集群设置

推荐用于 DeepseekV3-671B 训练的集群配置：

- **硬件需求：**
  - 1024+ 个 AMD MI325 GPU（128+ 个节点 × 8 GPU）
  - 高带宽互连（RoCE v2 或 InfiniBand）
  - 用于检查点的分布式存储（10+ TB）

- **并行配置：**
  - 专家并行：16-32（分发专家）
  - 数据并行：32-64（用于吞吐量）
  - 流水线并行：4-8（用于内存效率）
  - 总计 GPU：EP × DP × PP

- **训练超参数：**
  - 全局批次大小：4M-8M tokens
  - 序列长度：4K-8K tokens
  - 学习率：1e-4 到 3e-4，使用余弦衰减
  - 预热步数：2000-4000

#### 性能指标

预期的性能特征：

- **训练吞吐量：**
  - 目标：每个 GPU 50-100 TFLOPs
  - MFU（模型 FLOPs 利用率）：40-55%
  - Tokens per second：全集群 1M-2M

- **通信开销：**
  - All-to-all：总时间的 10-20%
  - 梯度同步：5-10%
  - 总通信：15-30%

### Llama4-Scout 扩展

Llama4-scout 作为一个较小的测试模型，用于在完整 Llama4 预训练之前验证扩展规律和训练基础设施。

#### 模型配置

- **Scout 模型大小：**
  - 小：7B-13B 参数（验证）
  - 中：34B-70B 参数（扩展测试）
  - 完整：405B+ 参数（生产）

- **架构特性：**
  - 分组查询注意力（GQA）
  - RMSNorm 用于层归一化
  - SwiGLU 激活函数
  - 旋转位置嵌入（RoPE）

#### 实验设置

推荐的工作流程：

1. **小规模验证（7B-13B）：**
   - 8-64 个 GPU 用于快速迭代
   - 验证数据管道和训练稳定性
   - 测试检查点/恢复机制
   - 基准测试每 GPU 吞吐量

2. **中规模测试（34B-70B）：**
   - 128-512 个 GPU 用于扩展验证
   - 测量通信开销
   - 验证 FSDP/DP/PP 策略
   - 测试容错和恢复

3. **全规模预训练（405B+）：**
   - 1024+ 个 GPU 用于生产训练
   - 持续监控 ETTR（有效训练时间比）和 MFU
   - 实现所有容错机制
   - 定期检查点验证

#### 性能优化

Llama4 预训练的关键优化策略：

- **梯度检查点：**
  - 在反向传播期间重计算激活
  - 用计算换内存
  - 对大型模型至关重要

- **Flash Attention：**
  - 内存高效的注意力实现
  - 支持更长的序列训练
  - 将激活内存减少 5-10 倍

- **混合精度训练：**
  - BF16 用于激活和权重
  - FP32 用于优化器状态
  - FP8 用于特定操作（AMD MI325）

- **序列打包：**
  - 将多个序列打包到单个批次中
  - 最大化 GPU 利用率
  - 减少填充开销

## 预训练最佳实践

### 基础设施设置

1. **网络拓扑：**
   - 使用高带宽互连（每个 GPU 200+ Gbps）
   - 支持 RDMA 的 NIC 用于低延迟通信
   - 叶脊或胖树拓扑用于可扩展性

2. **存储架构：**
   - 用于数据集的分布式文件系统（Lustre、CephFS、JuiceFS）
   - 基于 NVMe 的本地存储用于检查点
   - 分层存储：热（NVMe）→ 温（SSD）→ 冷（HDD）

3. **监控和可观测性：**
   - GPU 指标（利用率、温度、内存）
   - 网络指标（带宽、丢包率）
   - 训练指标（loss、吞吐量、MFU）
   - 实时仪表板和告警

### 训练配置

1. **批次大小选择：**
   - 从适合内存的最大批次开始
   - 使用梯度累积增加有效批次
   - 监控梯度噪声尺度以获得最佳批次大小

2. **学习率调度：**
   - 预热：在总步数的 1-2% 内线性增加
   - 衰减：余弦或线性衰减到峰值 LR 的 10%
   - 峰值 LR：按 sqrt(batch_size) 缩放或网格搜索

3. **检查点策略：**
   - 早期频繁检查点（每 100-500 步）
   - 后期较少检查点（每 1000-5000 步）
   - 保留最后 N 个检查点和周期性里程碑检查点
   - 定期验证检查点（加载并验证）

### 调试和故障排除

1. **Loss 发散：**
   - 降低学习率
   - 增加梯度裁剪阈值
   - 检查数据质量问题
   - 审查数值稳定性（FP16 溢出）

2. **训练缓慢：**
   - 分析 GPU 利用率并识别瓶颈
   - 检查通信开销（理想情况下 <30%）
   - 验证数据加载未阻塞
   - 审查并行策略（可能需要重新平衡）

3. **OOM（内存不足）：**
   - 启用梯度检查点
   - 减少微批次大小
   - 增加模型并行度
   - 使用优化器状态卸载

4. **通信瓶颈：**
   - 增加 EP 度以减少 all-to-all 大小
   - 计算与通信重叠
   - 使用层次化 all-to-all 进行大规模扩展
   - 检查网络配置和带宽

## 项目和框架

### 预训练框架

- <a href="https://github.com/pytorch/torchtitan">`TorchTitan`</a>：Meta 的面向 Llama 模型的预训练框架。原生 PyTorch，针对大规模分布式训练进行了优化。

- <a href="https://github.com/NVIDIA/Megatron-LM">`Megatron-LM`</a>：NVIDIA 的用于训练大型 transformer 模型的框架。支持各种并行策略和优化。

- <a href="https://github.com/microsoft/DeepSpeed">`DeepSpeed`</a>：Microsoft 的深度学习优化库。包含用于内存效率的 ZeRO 优化器。

- <a href="https://github.com/allenai/OLMo">`OLMo`</a>：Allen AI 的开放语言模型训练框架。专注于可复现性和透明度。

### MoE 实现

- <a href="https://github.com/huggingface/transformers">`Hugging Face Transformers`</a>：
  包含各种模型的 MoE 实现（Mixtral、Switch Transformer）。

- <a href="https://github.com/google-research/t5x">`T5X`</a>：Google 的用于训练 T5 和 Switch Transformer 模型的框架。

- <a href="https://github.com/microsoft/tutel">`Tutel`</a>：Microsoft 的优化 MoE 库，具有高效的专家并行。

### AMD 特定工具

- <a href="https://github.com/ROCm/rocm">`ROCm`</a>：AMD 的面向 GPU 计算的开放软件平台。MI325 训练需要。

- <a href="https://github.com/ROCm/RCCL">`RCCL`</a>：ROCm Communication Collectives Library。AMD 的多 GPU 通信 NCCL 等价物。

- <a href="https://github.com/ROCm/hipify">`Hipify`</a>：用于将 CUDA 代码转换为 AMD GPU 的 HIP 的工具。对移植基于 CUDA 的训练代码很有用。

## 研究和资源

### 学术论文

- **MoE 架构：**
  - ["Mixture of Experts: A Survey"](https://arxiv.org/abs/2407.06204)（arXiv:2407.06204）
  - ["Switch Transformers: Scaling to Trillion Parameter Models with Simple and Efficient Sparsity"](https://arxiv.org/abs/2101.03961)（arXiv:2101.03961）

- **预训练优化：**
  - ["Efficient Large-Scale Language Model Training on GPU Clusters Using Megatron-LM"](https://arxiv.org/abs/2104.04473)（arXiv:2104.04473）
  - ["ZeRO: Memory Optimizations Toward Training Trillion Parameter Models"](https://arxiv.org/abs/1910.02054)（arXiv:1910.02054）

### 技术博客

- **DeepSeek：**
  - [DeepSeek-V3 Repository](https://github.com/deepseek-ai/DeepSeek-V3)（包含技术报告和实现）

- **Meta：**
  - [Training Llama 3 at Scale](https://ai.meta.com/blog/meta-llama-3/)

- **AMD：**
  - [MI300 Series](https://www.amd.com/en/products/accelerators/instinct/mi300.html)

## 路线图（进行中的研究）

### 新兴技术

- **稀疏升级：**
  将稠密预训练模型转换为 MoE 模型

- **Soft MoE：**
  专家的加权组合而非硬路由

- **专家专业化：**
  鼓励多样化专家学习的技术

### 基础设施改进

- **自适应路由：**
  基于训练动态和负载的动态路由

- **层次化专家：**
  用于更好扩展的多级专家层次结构

- **通信优化：**
  用于减少延迟的新型 all-to-all 算法

## 快速开始

### 先决条件

- 具有 AMD MI300/MI325 或 NVIDIA H100/H200 GPU 的 Kubernetes 集群
- 已安装 ROCm 6.0+ 或 CUDA 12.0+
- 支持分布式训练的 PyTorch 2.4+
- 高带宽网络（200+ Gbps InfiniBand 或 RoCE v2）

### 快速开始：TorchTitan

```bash
# 克隆 TorchTitan 仓库
git clone https://github.com/pytorch/torchtitan.git
cd torchtitan

# 安装依赖
pip install -r requirements.txt

# 配置训练（编辑 toml 文件）
vim train_configs/llama3_8b.toml

# 启动分布式训练
torchrun --nproc_per_node=8 train.py \
  --config train_configs/llama3_8b.toml
```

### 示例：MoE 预训练配置

```toml
# 用于 MoE 模型的 TorchTitan 配置
[model]
model_type = "moe"
num_layers = 32
hidden_size = 4096
num_attention_heads = 32
num_experts = 8
experts_per_token = 2

[parallelism]
expert_parallel = 4
data_parallel = 8
pipeline_parallel = 2

[training]
global_batch_size = 4_000_000
sequence_length = 4096
learning_rate = 3e-4
warmup_steps = 2000

[checkpointing]
checkpoint_interval = 1000
checkpoint_dir = "/mnt/checkpoints"
```

## 性能基准测试

### DeepseekV3-671B 性能

配置：1024 个 AMD MI325 GPU，EP=16，DP=64，PP=1

- **吞吐量：** 1.8M tokens/秒
- **MFU：** 48%
- **通信开销：** 22%
- **ETTR：** 94%

### Llama4-405B 性能

配置：2048 个 NVIDIA H100 GPU，DP=64，PP=8，FSDP

- **吞吐量：** 3.2M tokens/秒
- **MFU：** 52%
- **通信开销：** 18%
- **ETTR：** 96%

---

**注意：** 本文档涵盖大规模模型的先进预训练技术。内容基于近期研究论文、技术报告和行业最佳实践。实施细节请参考官方框架文档。
