---
status: Active
maintainer: pacoxu
last_updated: 2026-03-25
tags: repository, structure, documentation, organization, 仓库结构
canonical_path: zh/STRUCTURE.md
---

# 仓库结构

本文档描述了 AI-Infra 仓库在遵循内容管理最佳实践重构后的组织和结构。

## 设计原则

仓库结构遵循以下原则：

1. **单一真相来源**：每个主题只出现在 ONE 个规范文件中
2. **集中化资源**：所有图片和图表存储在 `diagrams/` 中
3. **清晰层级**：按领域组织内容（kubernetes、inference、training）和专用的 `agent-infra/` 轨道
4. **元数据**：每个 markdown 文件包含状态、维护者和标签
5. **相对链接**：所有内部引用使用相对路径

## 目录结构

```text
AI-Infra/
├── README.md                          # 主入口与学习路径（英文）
├── README.zh-CN.md                    # 主入口与学习路径（中文）
├── RoadMap.md                         # 未来主题与规划
├── STRUCTURE.md                       # 本文件——仓库组织指南
├── diagrams/                          # 集中化图片存储
├── docs/                              # 所有文档内容
│   ├── kubernetes/                    # Kubernetes 与调度主题
│   ├── inference/                     # LLM 推理与服务主题
│   ├── training/                      # AI 模型训练主题
│   ├── observability/                 # 可观测性内容
│   ├── blog/                          # 博客归档与索引
│   └── archive-blog/                  # 历史博客文章归档
├── agent-infra/                       # Agent Infra 文档与基准测试资源
│   ├── README.md                      # 智能体平台、沙箱与生态系统
│   ├── runtime-benchmark.md           # runc/gVisor/Kata/VM 基准测试
│   └── scripts/
│       └── agent-sandbox-benchmark/   # 可复现的基准测试脚本
├── scripts/                           # 工具与生成脚本
└── .github/                           # GitHub 自动化和说明

目录/文件数量经常变化；避免硬编码总数。
```

## 内容组织

### 主入口

- **README.md**：提供以下内容的主学习路径：
  - AI 基础设施全景概览
  - 跨三个层级的结构化学习路径
  - 链接到所有详细文档
  - 即将推出的主题和路线图

### 领域类别

#### 1. Kubernetes (`docs/kubernetes/`)

**目的**：核心 Kubernetes 概念与 AI 工作负载优化

**规范文件**：

- `README.md` - Kubernetes 主题导航中心
- `learning-plan.md` - 三阶段结构化学习路径
- `pod-lifecycle.md` - 详细的 Pod 生命周期和 PLEG
- `pod-startup-speed.md` - 启动优化策略
- `scheduling-optimization.md` - 大规模调度模式
- `isolation.md` - 安全与多租户隔离
- `dra.md` - 动态资源分配 API 参考
- `nri.md` - 节点资源接口参考

**交叉引用**：链接到 `../inference/` 和 `../training/` 以获取相关主题

#### 2. 推理 (`docs/inference/`)

**目的**：LLM 推理引擎、服务平台和优化

**规范文件**：

- `README.md` - 推理平台全景（AIBrix, OME, Kthena, KServe, llm-d, Dynamo）
- `aibrix.md` - AIBrix 平台特性与架构
- `pd-disaggregation.md` - Prefill-Decode 分离实现
- `caching.md` - KV 缓存和前缀缓存技术
- `memory-context-db.md` - AI 智能体的内存管理
- `large-scale-experts.md` - MoE 架构与部署
- `model-lifecycle.md` - 模型生命周期管理策略
- `ome.md` - OME Kubernetes Operator 特性

**交叉引用**：链接到 `../kubernetes/` 以获取基础设施主题

#### 3. 训练 (`docs/training/`)

**目的**：分布式训练、容错和 ML 管道

**规范文件**：

- `README.md` - 训练挑战与最佳实践概述
- `kubeflow.md` - Kubeflow 训练算子与 Trainer V2
- `argocd.md` - 训练管理的 GitOps 工作流

**交叉引用**：链接到 `../kubernetes/` 以获取调度和资源

#### 4. Agent Infra (`agent-infra/`)

**目的**：AI 智能体平台、框架和基础设施

**规范文件**：

- `README.md` - 智能体平台、MCP 协议、Kubernetes 原生解决方案和生态系统快照概述
- `runtime-benchmark.md` - 运行时基准测试与可复现性指南

**交叉引用**：链接到 `../docs/kubernetes/isolation.md` 以获取沙箱相关内容，
链接到 `../docs/inference/memory-context-db.md` 以获取智能体内存相关内容

#### 5. 可观测性 (`docs/observability/`)

**目的**：AI 工作负载的监控、遥测和诊断

**规范文件**：

- `README.md` - 可观测性方法与参考概述

#### 6. 博客 (`docs/blog/`)

**目的**：按时间排序的技术博客和案例研究文章

**规范文件**：

- `README.md` - 完整博客索引与维护说明
- `YYYY-MM-DD-topic-name*.md` - 博客文章（扁平化，日期前缀命名）

### 资源 (`diagrams/`)

所有文档文件引用的图片均存储在此处：

- 可视化图表（全景图、架构图）
- 流程图（Pod 生命周期）
- 参考图片

**引用模式**：从文档文件中使用 `../../diagrams/image-name.svg`（或 `.png`）

## 元数据格式

每个 markdown 文件包含 YAML 前置元数据头：

```yaml
---
status: Active | Draft | Deprecated
maintainer: pacoxu
last_updated: YYYY-MM-DD
tags: comma, separated, tags
canonical_path: path/from/repo/root.md
---
```

**字段**：

- `status`：文档的当前状态
- `maintainer`：主要作者/维护者
- `last_updated`：最近一次重大更新的日期
- `tags`：涵盖的主题（便于搜索）
- `canonical_path`：完整路径（用于存根引用）

## 内容指南

### 单一真相来源

每个主要主题有一个规范位置：

- **Pod 生命周期** → `docs/kubernetes/pod-lifecycle.md`（规范）
  - 其他文件通过相对链接引用此文
  - 其他地方不重复解释

- **Prefill-Decode 分离** → `docs/inference/pd-disaggregation.md`（规范）
  - 在其他文件中提及并链接到规范版本
  - 实现细节仅出现在规范文件中

### 创建存根引用

当提及某个主题但详细内容在别处时，使用存根模式：

```markdown
## 主题名称

有关主题名称的全面覆盖，请参见[主题名称](../category/topic.md)。

关键点：
- 简要要点
- 简要要点
```

**最多 3-4 行**加上链接到规范版本。

### 添加新内容

1. **确定类别**：kubernetes、inference、training、observability、blog、archive-blog 或 agent-infra
2. **创建文件**在相应的 `docs/` 子目录或 `agent-infra/` 中
3. **添加元数据头**，包含状态、维护者、日期、标签
4. **更新父级 README.md** 以链接到新内容
5. **更新主 README.md**（如果添加了重要的新主题）
6. **对所有内部链接使用相对路径**
7. **将图片放在** `diagrams/` 目录中

### 链接模式

**从主 README 到文档**：

```markdown
[Topic](./docs/category/topic.md)
```

**文档文件之间（同一目录）**：

```markdown
[Topic](./topic.md)
```

**文档文件之间（不同目录）**：

```markdown
[Topic](../other-category/topic.md)
```

**从文档到图片**：

```markdown
![Description](../../diagrams/image.png)
```

## 维护

### 常规任务

1. **更新时间戳**：在做出重大编辑时更改 `last_updated`
2. **验证链接**：定期运行 `markdown-link-check`
3. **检查重复**：搜索重复内容并整合
4. **更新标签**：保持标签相关且一致
5. **检查 markdown**：在提交前运行 `markdownlint`

### 质量标准

- **技术准确性**：与官方文档核对
- **链接完整性**：所有内部链接必须正确解析
- **图片引用**：所有图片必须存在于 `diagrams/` 中
- **元数据完整性**：每个文件必须具有完整的元数据
- **存根纪律**：避免重复内容；链接到规范版本

## 迁移说明

### 重构中变更的内容

1. **目录结构**：
   - 旧：`kubernetes/`、`inference/`、`training/` 在根目录
   - 新：`docs/kubernetes/`、`docs/inference/`、`docs/training/`

2. **图片**：
   - 旧：分散在各类别目录中
   - 新：集中存储在 `diagrams/` 中

3. **文件重命名**：
   - `kubernetes/kubernetes.md` → `docs/kubernetes/learning-plan.md`

4. **所有文件新增**：
   - 元数据头
   - 规范路径标记
   - 更新的相对链接

### 向后兼容性

重构后旧链接已失效。更新任何外部引用：

- `./kubernetes/pod-lifecycle.md` → `./docs/kubernetes/pod-lifecycle.md`
- `./ai-infra-landscape.svg` → `./diagrams/ai-infra-landscape.svg`

## 未来增强

计划中的仓库结构改进：

1. **存根页面**：为经常引用的主题创建重定向
2. **主题索引**：所有内容的基于标签的自动索引
3. **变更日志**：跟踪文档随时间的演进
4. **贡献指南**：添加内容的详细指南
5. **搜索集成**：在文档中启用全文搜索

## 问题与反馈

关于仓库结构的问题或改进建议：

- 在仓库中开启 issue
- 联系维护者：pacoxu
- 参考 `.github/copilot-instructions.md` 获取详细指南
