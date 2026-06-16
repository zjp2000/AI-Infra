---
status: Active
maintainer: pacoxu
last_updated: 2025-10-30
tags: training, parallelism, distributed-training, pytorch, data-parallel,
  sharded-data-parallel, context-parallel
canonical_path: zh/docs/training/parallelism.md
---

# 分布式训练的并行策略

## 概述

训练大规模 AI 模型需要将计算分布到多个 GPU 和节点上。不同的并行策略在内存效率、计算开销和实现复杂度之间提供不同的权衡。本指南涵盖现代分布式训练中使用的三种基本并行技术。

## 关键并行策略

### 1. 数据并行（DP）

**数据并行**是最直接的并行策略，模型在所有设备上复制，每个设备处理训练数据的不同子集。

#### 数据并行的工作原理

- 每个 GPU 维护模型的完整副本
- 训练数据被拆分为批次并分发到各个 GPU
- 每个 GPU 上独立计算前向和反向传播
- 使用 AllReduce 在所有 GPU 间同步（平均）梯度
- 所有 GPU 上的模型参数以相同方式更新

#### PyTorch 中的数据并行实现

```python
import torch
import torch.nn as nn
from torch.nn.parallel import DistributedDataParallel as DDP

# 初始化分布式进程组
torch.distributed.init_process_group(backend='nccl')

# 创建模型并移动到 GPU
model = MyModel().cuda()

# 使用 DDP 包装
model = DDP(model, device_ids=[local_rank])

# 训练循环 - DDP 处理梯度同步
for data, target in dataloader:
    optimizer.zero_grad()
    output = model(data)
    loss = criterion(output, target)
    loss.backward()  # 梯度自动同步
    optimizer.step()
```

#### 数据并行特征

- **内存：** 每个 GPU 存储完整模型 + 优化器状态 + 梯度
- **通信：** 用于梯度同步的 All-reduce 操作
- **可扩展性：** 受限于模型大小能否适配单个 GPU 内存
- **效率：** 高吞吐量，最小通信开销

#### 数据并行的最佳使用场景

- 适合单个 GPU 内存的模型
- 大批次大小的高吞吐量训练
- 简单的分布式训练设置

### 2. 分片数据并行（SDP）/ 全分片数据并行（FSDP）

**分片数据并行**（也称为**全分片数据并行**或**FSDP**）通过在 GPU 间分片模型参数、梯度和优化器状态来扩展数据并行，显著降低每个 GPU 的内存使用。

#### 分片数据并行的工作原理

- 模型参数在所有 GPU 间分片（分区）
- 在前向传播期间：
  - 每个 GPU 通过 all-gather 收集其需要的参数
  - 计算前向传播
  - 使用后丢弃参数
- 在反向传播期间：
  - 每个 GPU 再次收集参数用于梯度计算
  - 计算梯度并在 GPU 间规约
  - 每个 GPU 仅保留其分片的梯度
- 优化器状态也被分片，进一步减少内存

#### PyTorch 中的 FSDP 实现

```python
import functools
import torch
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP
from torch.distributed.fsdp import ShardingStrategy
from torch.distributed.fsdp.wrap import size_based_auto_wrap_policy

# 初始化分布式进程组
torch.distributed.init_process_group(backend='nccl')

# 自动分层分片的包装策略
auto_wrap_policy = functools.partial(
    size_based_auto_wrap_policy,
    min_num_params=1e8  # 包装具有 >100M 参数的层
)

# 使用 FSDP 创建模型
model = FSDP(
    MyModel(),
    sharding_strategy=ShardingStrategy.FULL_SHARD,
    auto_wrap_policy=auto_wrap_policy,
    device_id=torch.cuda.current_device(),
)

# 训练循环 - FSDP 处理 all-gather 和 reduce-scatter
for data, target in dataloader:
    optimizer.zero_grad()
    output = model(data)
    loss = criterion(output, target)
    loss.backward()
    optimizer.step()
```

#### 分片策略

- **FULL_SHARD：** 参数、梯度和优化器状态均分片（最大内存节省）
- **SHARD_GRAD_OP：** 仅梯度和优化器状态分片
- **NO_SHARD：** 等效于标准 DDP（不分片）
- **HYBRID_SHARD：** 节点内分片，节点间复制

#### 分片数据并行特征

- **内存：** 显著降低——每个 GPU 存储 1/N 的参数（N = GPU 数量）
- **通信：** 更频繁的 all-gather 和 reduce-scatter 操作
- **可扩展性：** 可以训练远大于单个 GPU 内存的模型
- **效率：** 通信开销略高于 DDP

#### 分片数据并行的最佳使用场景

- 不适合单个 GPU 内存的非常大的模型
- 每个设备 GPU 内存有限的训练环境
- 多节点训练场景

### 3. 上下文并行（CP）

**上下文并行**是一种专门为具有长序列长度的 transformer 模型设计的并行策略。它将序列维度分区到多个 GPU 上，使得训练上下文长度超过单个 GPU 内存容量成为可能。

#### 上下文并行的工作原理

- 输入序列沿序列维度在 GPU 间分割
- 注意力机制被修改以处理跨 GPU 通信
- 对于自注意力，每个 GPU：
  - 计算其序列分区的注意力
  - 与其他 GPU 通信以交换用于完整注意力计算的键/值
- 层输出根据下一层的需求被收集或保持分片状态

#### 关键概念

**Transformer 的序列并行：**

```text
序列长度：32K tokens
4 个 GPU 进行上下文并行

GPU 0: Tokens [0:8K]
GPU 1: Tokens [8K:16K]
GPU 2: Tokens [16K:24K]
GPU 3: Tokens [24K:32K]

每个 GPU 计算其分区上的注意力，同时与其他 GPU 交换
键值信息以获得完整的注意力分数。
```

#### PyTorch 中的上下文并行实现（简化版）

```python
import torch
import torch.distributed as dist

def context_parallel_attention(
    query, key, value, cp_group, cp_rank, cp_world_size
):
    """
    上下文并行自注意力。

    参数：
        query, key, value: [batch, seq_len_local, hidden_dim]
        cp_group: 上下文并行进程组
        cp_rank: 上下文并行组内的 rank
        cp_world_size: 上下文并行组的大小
    """
    batch_size, seq_len_local, hidden_dim = query.shape

    # 从所有 GPU 收集所有键和值
    key_all = [torch.empty_like(key) for _ in range(cp_world_size)]
    value_all = [torch.empty_like(value) for _ in range(cp_world_size)]

    dist.all_gather(key_all, key, group=cp_group)
    dist.all_gather(value_all, value, group=cp_group)

    # 拼接得到完整序列
    key_full = torch.cat(key_all, dim=1)  # [batch, seq_len, hidden_dim]
    value_full = torch.cat(value_all, dim=1)

    # 为本地查询计算完整序列的注意力
    # 注意：compute_attention 是您注意力实现的占位符
    # （例如缩放点积注意力或 flash attention）
    attention_output = compute_attention(query, key_full, value_full)

    return attention_output
```

#### 上下文并行特征

- **内存：** 激活内存按序列分区比例减少
- **通信：** 注意力层中键/值的 All-gather
- **可扩展性：** 支持使用非常长的上下文长度进行训练
- **效率：** 通信开销随序列长度增加

#### 上下文并行的最佳使用场景

- 训练具有非常长的上下文长度（>8K tokens）的 transformer
- 序列长度是内存瓶颈的模型
- 长上下文 LLM（32K、64K、128K+ tokens）

## 组合多种并行策略

现代大规模训练通常组合多种并行策略以最大化效率和规模。常见组合包括：

### 3D 并行（DP + TP + PP）

```text
数据并行（DP）：跨数据复制
张量并行（TP）：分片模型层
流水线并行（PP）：跨流水线阶段拆分模型
```

### 4D 并行（DP + TP + PP + CP）

```text
为长上下文模型添加上下文并行到 3D 并行：
- DP：4 路（4 个数据副本）
- TP：8 路（模型在 8 个 GPU 间分片）
- PP：4 路（模型拆分为 4 个流水线阶段）
- CP：2 路（序列在 2 个 GPU 间分割）
总计：256 个 GPU（4 × 8 × 4 × 2）
```

### 示例：PyTorch TorchTitan 配置

```python
# 来自 torchtitan 的 LLaMA 3 训练示例配置
parallelism_config = {
    "data_parallel": 4,           # 4 路数据并行
    "tensor_parallel": 8,          # 8 路张量并行
    "pipeline_parallel": 4,        # 4 路流水线并行
    "context_parallel": 2,         # 2 路上下文并行
    "enable_fsdp": True,           # 使用 FSDP 进行数据并行
    "fsdp_sharding_strategy": "FULL_SHARD",
}
```

## 性能考量

### 内存效率比较

对于在 N 个 GPU 上训练的具有 M 个参数的模型：

| 策略 | 每 GPU 内存 | 通信量 |
|----------|----------------|---------------------|
| DP | M + Optimizer + Gradients | 仅梯度 |
| FSDP | M/N + Optimizer/N + Gradients/N | 参数 + 梯度 |
| CP | 按序列分片减少 | 注意力中的 K-V |

### 通信模式

- **DP：** 反向传播期间的 All-reduce（环或树拓扑）
- **FSDP：** All-gather（前向）+ reduce-scatter（反向）
- **CP：** 注意力层中的 All-gather，输出中的 reduce-scatter

### 何时使用每种策略

| 使用场景 | 推荐策略 |
|----------|---------------------|
| 模型适合单个 GPU | DP（DDP） |
| 模型太大无法放入单个 GPU | FSDP 或模型并行 |
| 长上下文长度（>8K） | CP + FSDP |
| 数十亿参数模型 | FSDP + TP + PP |
| 最大吞吐量 | 大批次大小的 DP |

## 工具和框架

### PyTorch 原生

- [`torch.nn.parallel.DistributedDataParallel`](https://pytorch.org/docs/stable/generated/torch.nn.parallel.DistributedDataParallel.html)：标准 DDP 实现
- [`torch.distributed.fsdp`](https://pytorch.org/docs/stable/fsdp.html)：全分片数据并行（FSDP）
- [`torch.distributed.tensor.parallel`](https://pytorch.org/docs/stable/distributed.tensor.parallel.html)：张量并行 API

### 参考实现

- [`PyTorch TorchTitan`](https://github.com/pytorch/torchtitan)：PyTorch 的面向大型模型训练的参考实现，包含所有并行策略。提供 DP、FSDP、TP、PP 和 CP 的生产级示例。
- [`Megatron-LM`](https://github.com/NVIDIA/Megatron-LM)：NVIDIA 的具有先进并行技术的框架
- [`DeepSpeed`](https://github.com/microsoft/DeepSpeed)：Microsoft 的优化库，包含 ZeRO（类似于 FSDP）

### 训练 Operator

- [`Kubeflow Training Operator`](https://github.com/kubeflow/training-operator)：支持分布式 PyTorch 训练的 Kubernetes 原生 Operator
- [`Volcano`](https://github.com/volcano-sh/volcano)：用于多 GPU 任务的批处理调度，具有 gang scheduling

## 学习资源

### 官方文档

- [PyTorch Distributed Overview](https://pytorch.org/tutorials/beginner/dist_overview.html)
- [FSDP Tutorial](https://pytorch.org/tutorials/intermediate/FSDP_tutorial.html)
- [Distributed Training with PyTorch](https://pytorch.org/tutorials/beginner/ddp_series_intro.html)

### 研究论文

- **FSDP/ZeRO：**
  ["ZeRO: Memory Optimizations Toward Training Trillion Parameter Models"](https://arxiv.org/abs/1910.02054)（arXiv:1910.02054）
- **Megatron-LM：**
  ["Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism"](https://arxiv.org/abs/1909.08053)（arXiv:1909.08053）
- **序列并行：**
  ["Reducing Activation Recomputation in Large Transformer Models"](https://arxiv.org/abs/2205.05198)（arXiv:2205.05198）

### 示例项目

- [PyTorch TorchTitan Examples](https://github.com/pytorch/torchtitan/tree/main/docs)：包含所有并行策略的生产级示例
- [PyTorch FSDP Examples](https://github.com/pytorch/examples/tree/main/distributed/FSDP)：简单的 FSDP 使用示例
- [Megatron-LM Training Scripts](https://github.com/NVIDIA/Megatron-LM/tree/main/examples)：高级并行配置

## 最佳实践

### 选择并行配置

1. **从数据并行（DDP）开始：** 如果模型适合 GPU 内存
2. **添加 FSDP：** 当模型内存超过单个 GPU 容量时
3. **考虑上下文并行：** 对于序列长度 >8K tokens
4. **使用张量/流水线并行：** 对于非常大的模型（>100B 参数）

### 优化技巧

- **梯度累积：** 无需更多内存即可增加有效批次大小
- **混合精度（FP16/BF16）：** 减少内存并增加吞吐量
- **激活检查点：** 以计算换内存
- **通信重叠：** 尽可能使通信与计算重叠

### 调试分布式训练

- 先在单个 GPU 上测试，然后扩展到多 GPU
- 使用较小的模型变体进行更快的迭代
- 监控 GPU 内存使用和通信时间
- 验证梯度同步正确性
- 使用 NCCL_DEBUG=INFO 检查挂起的进程

## 路线图

### 即将推出的功能

- **自动并行选择：** 框架级的并行策略选择优化
- **异构训练：** 混合 GPU 类型和内存配置
- **动态并行：** 基于工作负载特征的自适应并行

### 社区贡献

欢迎对并行策略文档和示例做出贡献！请通过提交 issue 或 pull request 分享您的经验、基准测试和最佳实践。

---

**注意：** 本文档基于 PyTorch 2.x 并参考了 [PyTorch TorchTitan](https://github.com/pytorch/torchtitan) 项目。部分内容描述了需要仔细调优才能用于生产的高级技术。请始终根据您的特定工作负载和硬件验证配置。
