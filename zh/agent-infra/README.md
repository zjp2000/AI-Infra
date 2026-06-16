---
status: Active
maintainer: pacoxu
last_updated: 2026-05-26
tags: ai-agents, agentic-workflow, kubernetes, mcp, agent-platforms
canonical_path: zh/agent-infra/README.md
---

# AI Agent 平台与框架

本文档全面概述了云原生生态系统中的 AI Agent 平台、框架和基础设施项目。AI Agent 是能够感知环境、做出决策并采取行动以实现特定目标的自主软件系统，通常使用大语言模型（LLM）作为其核心推理引擎。

## 目录

- [概述](#概述)
- [原生 AI Agent 套件](#原生-ai-agent-套件)
- [Kubernetes 原生 Agent 平台](#kubernetes-原生-agent-平台)
- [Agent Infra 生态图谱（2026）](#agent-infra-生态图谱2026)
- [Agent 开发框架](#agent-开发框架)
- [Agent 沙箱基础设施](#agent-沙箱基础设施)
- [Agent 基础设施组件](#agent-基础设施组件)
- [CNCF 和生态系统倡议](#cncf-和生态系统倡议)
- [学习主题](#学习主题)
- [路线图](#路线图)
- [参考](#参考)

## 概述

AI Agent 代表了 AI 基础设施的下一次演进，从简单的模型推理走向能够执行以下操作的自主系统：

- 执行复杂的多步骤工作流
- 使用工具和 API 与外部系统交互
- 跨对话维护记忆和上下文
- 基于目标和约束做出决策
- 从反馈中学习和适应

### AI Agent 的四阶段演进

根据 **杨强教授**（香港科技大学）的观点，AI Agent（智能体）预计将在 2026 年及以后经历四个不同的阶段：

1. **第一阶段：人类定义目标和规划**
   - 目标和执行计划均由人类定义
   - Agent 遵循预定的工作流
   - 当前最常见的实现阶段

2. **第二阶段：AI 辅助规划**
   - 目标由人类定义
   - 规划和执行策略由 AI 辅助
   - AI 帮助优化工作流，但人类保持控制

3. **第三阶段：AI 从过程数据中学习规划**
   - AI 观察人类工作流和过程数据
   - 自动学习并生成规划策略
   - 减少常规任务中的人工干预

4. **第四阶段：完全自主 Agent**
   - 最终阶段：目标和规划均由 LLM 内部定义
   - Agent 以真正的自主性运行
   - 自我导向的决策和执行

**当前状态**：大多数 AI Agent 目前在阶段 1 和阶段 2 运行。

**经济价值洞察**：根据行业观察，agent 创造真正经济价值的瓶颈不在于模型能力本身，而在于：

- **环境**：将模型部署到多样化的企业环境中
- **教育**：培训组织有效使用 agent 系统

即使没有进一步的模型改进，通过将现有模型部署到各种企业场景中，也可以实现显著的经济效益。重点应放在集成、部署基础设施和组织准备度上。

参考：[Agent 演进理论 - 微信公众号文章](https://mp.weixin.qq.com/s/NUx4n5j0ftxzZ0Sz29RjOQ)（2026 年访问）

### Agent 领域分类

AI Agent 领域包括：

1. **原生 Agent 套件**：特定云提供商的平台（如火山引擎）
2. **Kubernetes 原生方案**：专为 K8s 环境设计的项目（如 KAgent、AgentCube）
3. **开发框架**：用于构建 agent 的库和 SDK（如 AgentScope、LangChain）
4. **基础设施组件**：沙箱、网关、MCP 服务器

## 原生 AI Agent 套件

### 火山引擎原生 AI Agent 套件（字节跳动）

<a href="https://www.volcengine.com/">火山引擎</a> 提供了一个全面的原生 AI Agent 套件，架构如下：

**架构组件：**

- **API 网关**：流量接入入口
- **AI Agent Core**：中央编排，具有三个关键能力：
  - 丰富的应用模版
  - 极致弹性 按需付费
  - AI 应用全链路可观测

**核心能力：**

1. **极致弹性**：
   - 帮助分层的 vCPU 弹性
   - 满足 agent 资源需求
   - 从零扩展到数千个实例

2. **支持 MCP**：
   - 支持 Model Context Protocol
   - Agent 可以使用 MCP 与生态系统集成
   - 连接到数据源和现有工具

3. **记忆管理**：
   - 提供即时 RAG 功能
   - 支持私有领域知识
   - 长期记忆召回

4. **全链路观测**：
   - 端到端的 agent 监控
   - 跟踪开发者调用链、token 消耗和框架开销
   - 对开发者透明，不影响业务服务性能

**三层架构：**

1. **模型**：
   - 火山方舟模型
   - 自建模型

2. **Tools**：
   - MCP Hub
   - OS Agent
   - Sandbox

3. **Memory**：
   - 知识平台
   - 长期记忆

参考：火山引擎 AI Agent 平台展示了行业趋势——提供具有弹性、协议支持（MCP）、记忆管理和全面可观测性的集成 agent 基础设施。

## Kubernetes 原生 Agent 平台

### KAgent

<a href="https://github.com/kagent-dev/kagent">`KAgent`</a>：**CNCF Sandbox 项目**

KAgent 是一个 Kubernetes 原生平台，用于构建和部署 AI agent。它使用 Kubernetes 自定义资源提供声明式的 agent 管理方法。

**关键特性：**

- Kubernetes 原生 agent 编排
- 声明式 agent 配置
- 内置 agent 生命周期管理
- 与 Kubernetes RBAC 和安全策略集成

**状态**：CNCF Sandbox（2024）

**使用场景：**

- 多租户 agent 部署
- 企业级 agent 编排
- 云原生 agent 工作流

### Volcano AgentCube

<a href="https://github.com/volcano-sh/agentcube">`AgentCube`</a>

AgentCube 是 Volcano 生态系统的一部分，在 Kubernetes 上提供 agent 编排能力，专注于批处理工作负载和资源管理。

**关键特性：**

- 与 Volcano 调度集成
- 批处理 agent 执行
- GPU 感知的 agent 调度
- 基于队列的 agent 管理

**维护者**：Volcano 社区（CNCF Graduated 项目）

### Volcano Kthena

<a href="https://github.com/volcano-sh/kthena">`Kthena`</a>

Kthena 是一个 Volcano 生态系统项目，用于 Kubernetes 原生 LLM 推理编排和管理。虽然主要专注于推理工作负载，但 Kthena 为需要 LLM 后端的 agent 系统提供了高级调度能力。

**关键特性：**

- LeaderWorkerSet 集成，用于分布式推理拓扑
- 网络拓扑感知调度（Volcano v1.14+）
- 面向分布式工作负载的角色级 gang scheduling
- 带版本控制的 ModelServing 生命周期管理
- 路由器可观测性和调试能力

**维护者**：Volcano 社区（CNCF Graduated 项目）

### Kubernetes SIG Agent Sandbox

<a href="https://github.com/kubernetes-sigs/agent-sandbox">`agent-sandbox`</a>

官方 Kubernetes SIG 项目，提供安全的沙箱环境，用于运行具有强隔离保证的 AI agent。

**关键特性：**

- agent 代码的安全执行环境
- agent 与主机系统之间的强隔离
- 与 Kubernetes 安全策略的集成
- 支持代码解释器和工具执行

**状态**：Kubernetes SIG 项目（官方）

参见：[Agent Sandbox Documentation](../docs/kubernetes/isolation.md#6-agent-sandbox-kubernetes-sig-project)

### Agent Substrate

<a href="https://github.com/agent-substrate/substrate">`agent-substrate/substrate`</a>

Agent Substrate 是一个基于 Kubernetes 的运行时和控制平面，用于类 agent 工作负载。它不是将一个 Pod 绑定到一个长寿命会话，而是将大量有状态"actor"映射到较小的就绪 worker 池上，并使 Kubernetes 控制平面远离热路径。

**关键特性：**

- Actor/worker 多路复用，实现更高密度
- 有状态会话的挂起/恢复生命周期
- 实时分配和流量路由到活跃 worker
- 基于 gVisor 的 OCI 容器执行，框架无关兼容

**状态**：非常早期的开发阶段；尚未准备好投入生产

**使用场景：**

- 大部分空闲的编码或工具使用 agent 的大型群组
- 需要亚秒级重新激活的有状态 agent 会话
- 希望在底层使用 Kubernetes 原语但需要比每会话 Pod 编排更低延迟的平台

### Agent Infra Sandbox

<a href="https://github.com/agent-infra/sandbox">`agent-infra/sandbox`</a>

社区项目，为 AI agent 提供沙箱基础设施，专注于安全和隔离。

**关键特性：**

- 轻量级沙箱运行时
- 多种隔离后端
- 与 agent 框架集成
- 资源限制和监控

### OpenKruise Agents

<a href="https://github.com/openkruise/agents">`openkruise/agents`</a>

OpenKruise 扩展，为 Kubernetes 中的应用生命周期管理提供 agent 能力。

**关键特性：**

- 应用感知的 agent 操作
- 与 OpenKruise 工作负载控制器集成
- agent 工作负载的高级部署策略
- agent 的就地更新支持

### ArgoCD Agent

<a href="https://github.com/argoproj-labs/argocd-agent">`argocd-agent`</a>

ArgoCD Agent 为 GitOps 工作流提供基于 agent 的部署能力，实现跨多个集群的安全可扩展应用交付。

**关键特性：**

- 基于 agent 的集群注册
- 安全的基于拉取的部署模型
- 多集群 GitOps 管理
- 与 ArgoCD 生态系统集成
- 减少控制平面开销

**状态**：ArgoProj Labs（实验性）

**使用场景：**

- 多集群 GitOps 部署
- 边缘集群管理
- 受限环境中的安全部署
- 大规模集群群组管理

### Agent Infra 生态图谱（2026）

基于"Agent Infra 生态图谱"会议总结（KCD Beijing / vLLM 社区），当前沙箱/运行时生态系统可以总结为：

- **K8s 标准层 — `kubernetes-sigs/agent-sandbox`**：SIG Apps 官方项目，包含 `Sandbox` CRD、`SandboxWarmPool` 和深度 GKE 集成。
- **Google `Agent Substrate`**：低意见运行时，将许多 agent actor 多路复用到较小的 worker 池上，支持挂起/恢复和实时路由。
- **阿里巴巴 `OpenSandbox`**：四种沙箱类型（`Coding`、`GUI`、`Exec`、`RL`）加上为异构 agent 工作负载设计的三层隔离架构。
- **OpenKruise AIO Sandbox**：生命周期管理、E2B 协议兼容性以及一体化容器设计，简化框架集成。
- **字节跳动 `DeerFlow 2.0`**：子 agent 编排，带有独立沙箱模式，实现细粒度隔离和并行调度。
- **腾讯云 Agent Runtime**：针对生产吞吐量优化，声称约 100 ms 冷启动和 10 万级并发。

这些项目指示了一条收敛路径：**标准化的 CRD 控制平面 + 持久的 actor/会话运行时 + 可插拔沙箱运行时后端 + 基于预热池或快照的延迟控制**。

#### Agent 沙箱选型更新（2026-04-22）

最近的 agent 沙箱项目应按层次进行比较，而非作为直接的 1:1 替代。一个实用的平台通常结合 agent 面向 API、Kubernetes 生命周期控制器和一个或多个隔离运行时：

| 层次 | 候选项目 | 主要角色 | 选型说明 |
| ----- | ------------------ | ------------ | --------------- |
| Agent 面向沙箱平台 | [OpenSandbox](https://github.com/alibaba/OpenSandbox), [CubeSandbox](https://github.com/TencentCloud/CubeSandbox), [E2B](https://github.com/e2b-dev/e2b), [Daytona](https://github.com/daytonaio/daytona), [Sandbox0](https://github.com/sandbox0-ai/sandbox0) | SDK/API、沙箱生命周期、命令/文件/浏览器执行、模板 | 使用 OpenSandbox 作为广泛的自托管默认方案；评估 CubeSandbox 用于高密度 E2B 兼容微虚拟机池；嵌入前检查 Daytona 的 AGPLv3 影响。 |
| Kubernetes 生命周期 API | [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox), [agent-sandbox/agent-sandbox](https://github.com/agent-sandbox/agent-sandbox), [volcano-sh/agentcube](https://github.com/volcano-sh/agentcube) | CRD/控制器、预热池、稳定沙箱身份、调度钩子 | `kubernetes-sigs/agent-sandbox` 是中立的 K8s API 层；`agent-sandbox/agent-sandbox` 添加了 REST/MCP 和 E2B 兼容性；AgentCube 仍处于提案/早期设计阶段。 |
| 隔离运行时 | [gVisor](https://github.com/google/gvisor), [Kata Containers](https://github.com/kata-containers/kata-containers), [containerd/nerdbox](https://github.com/containerd/nerdbox), [Firecracker](https://github.com/firecracker-microvm/firecracker), [Kuasar](https://github.com/kuasar-io/kuasar) | 不可信代码的运行时边界 | 从 gVisor 开始以获取密度和运维简便性；使用 Kata 实现 VM 级租户边界和 GPU 路径；将 Firecracker 视为低级原语，除非团队拥有控制平面。 |
| 本地编码 agent 沙箱 | [Cleanroom](https://github.com/buildkite/cleanroom), [Brood Box](https://github.com/stacklok/brood-box), [microsandbox](https://github.com/superradcompany/microsandbox), [BoxLite](https://github.com/boxlite-ai/boxlite), [Matchlock](https://github.com/jingkaihe/matchlock), [Shuru](https://github.com/superhq-ai/shuru), [sandbox-runtime](https://github.com/anthropic-experimental/sandbox-runtime) | 开发者机器隔离、仓库策略、凭证代理、diff 审查 | 当威胁模型是编码 agent 访问仓库时，优先选择 Cleanroom/Brood Box；使用 microsandbox/BoxLite 获取可嵌入的本地微虚拟机 API；使用 sandbox-runtime 获取无 VM 隔离的 OS 策略护栏。 |
| 暂缓/避免 | [BinSquare/ERA](https://github.com/BinSquare/ERA), [autohandai/sandbox-core](https://github.com/autohandai/sandbox-core), [SmolVM](https://github.com/CelestoAI/SmolVM), [Gondolin](https://github.com/earendil-works/gondolin), [Pyro](https://github.com/danievanzyl/pyro), `opencapsule/opencapsule` | 早期阶段或可用性不明确 | ERA 已归档/弃用；`opencapsule/opencapsule` 在检查时在 GitHub 上无法公开解析；其余项目值得跟踪或借鉴思路，但需要更深入的成熟度验证。 |

`agent-substrate/substrate` 最接近**持久 agent 会话运行时**层，而非通用 agent 框架或沙箱 SDK。

- 最接近的架构邻居：如果您想要一个具有持久会话的 Kubernetes 原生运行时，则选择 `agent-sandbox/agent-sandbox`，但其重心是 REST/MCP 沙箱生命周期管理和 E2B 兼容性。
- 最接近的 agent 面向 API 替代方案：如果您想要直接向应用代码暴露会话/沙箱 API、持久卷、预热池以及凭证或网络控制，则选择 `Sandbox0` 和 `E2B`。
- 更高级的替代方案：如果您想要将 agent 定义和操作为 Kubernetes 资源，而不是优化运行时层的 actor 多路复用和挂起/恢复延迟，则选择 `kagent`。

默认栈推荐：

1. **Agent API**：OpenSandbox 作为广泛的自托管 API/SDK 接口。
2. **Kubernetes 生命周期**：`kubernetes-sigs/agent-sandbox`，用于 `Sandbox`、`SandboxTemplate`、`SandboxClaim` 和 `SandboxWarmPool`。
3. **默认运行时**：gVisor，用于仅 CPU 的高密度不可信代码。
4. **高风险运行时**：Kata 或 CubeSandbox，用于更强的微虚拟机边界。
5. **本地开发**：Cleanroom 或 Brood Box，用于仓库范围的出口控制、主机端凭证和运行后变更审查。

## Agent 开发框架

### LangChain DeepAgents

<a href="https://github.com/langchain-ai/deepagents">`deepagents`</a>

DeepAgents 是 LangChain 的框架，用于构建复杂的多 agent 系统，通过更深的思维链处理和规划增强推理能力。

**关键特性：**

- 通过更深的思维链处理增强推理
- 高级规划和任务分解
- 与 LangChain 生态系统集成
- 多 agent 协调模式
- 高级记忆和状态管理
- 支持复杂的 agent 工作流

**维护者**：LangChain AI

**状态**：活跃开发中

### AgentScope

<a href="https://github.com/agentscope-ai/agentscope">`AgentScope`</a>

AgentScope 是一个灵活全面的 agent 开发框架，支持多 agent 系统。

**关键特性：**

- 多 agent 编排
- 内置通信协议
- 支持各种 LLM 后端
- 基于 Python 的开发体验

**维护者**：AgentScope AI 社区

### Dapr Agents

<a href="https://github.com/dapr/dapr-agents">`dapr-agents`</a>

将 agent 能力与 Dapr（分布式应用运行时）集成，提供云原生 agent 原语。

**关键特性：**

- 用于 agent 的 Dapr 构建块
- 服务到服务的 agent 调用
- 用于 agent 记忆的状态管理
- 用于 agent 通信的发布/订阅模式

**状态**：Dapr 生态系统的一部分

### Coze Studio

<a href="https://github.com/coze-dev/coze-studio">`Coze Studio`</a>

用于构建对话式 AI agent 的开发环境，提供可视化工具。

**关键特性：**

- 可视化 agent 设计界面
- 预构建的 agent 模板
- 与流行 LLM 提供商集成
- 测试和调试工具

### Open-AutoGLM

<a href="https://github.com/zai-org/Open-AutoGLM">`Open-AutoGLM`</a>

基于 GLM（通用语言模型）的开源自主 agent 框架。

**关键特性：**

- 自主任务执行
- 工具使用和 API 调用
- 中文语言优化
- 与智谱 AI 模型集成

### Spring AI Alibaba

<a href="https://github.com/alibaba/spring-ai-alibaba">`spring-ai-alibaba`</a>

Spring 生态系统集成，用于使用阿里云服务构建 AI agent。

**关键特性：**

- 用于 agent 的 Spring Boot 集成
- 阿里云服务连接器
- Java/Kotlin agent 开发
- 企业级可靠性模式

### Google ADK-Go

<a href="https://github.com/google/adk-go">`adk-go`</a>

Google 的面向 Go 的 Agent 开发工具包，提供 Go 原生的 agent 开发能力。

**关键特性：**

- Go 原生 agent 框架
- Google Cloud 集成
- 高性能 agent 运行时
- 结构化 agent 模式

## Agent 沙箱基础设施

Agent 沙箱为运行不可信代码的 AI agent 提供安全、隔离的执行环境——LLM 生成的 Python、shell 命令、文件操作和浏览器自动化。设计可扩展的沙箱需要选择正确的架构模式和底层技术。

### 两种架构模式

<a href="https://browser-use.com/posts/two-ways-to-sandbox-agents">Browser Use</a> 描述了保持 agent 代码与基础设施密钥隔离的两种基本模式：

**模式 1：隔离工具**——agent 循环在您的后端运行；只有危险操作（代码执行、终端）在通过 HTTP 访问的单独沙箱中运行。易于采用，但 agent 仍然与 API 后端共享资源。

**模式 2：隔离 Agent**——整个 agent 运行在零密钥的沙箱中。无状态**控制平面**持有所有凭证并代理每个敏感操作：

- **LLM 调用**：控制平面拥有完整对话历史并代理到 LLM 提供商；沙箱仅发送新消息
- **文件同步**：沙箱请求预签名上传/下载 URL；从不持有云凭证
- **计费和成本上限**：在控制平面边界强制执行

模式 2 使 agent 完全可丢弃，并独立扩展每一层。关键见解：*"您的 agent 应该没有什么值得偷，也没有什么值得保留。"*

### 以 OpenClaw 为中心的云原生部署

[OpenClaw](https://openclawai.github.io/) 是一个已经包含沙箱抽象和后端插件的开源 agent 框架。在实践中，它很好地映射到 Kubernetes 风格的控制平面/数据平面分离：

- **Agent 控制平面**（API、会话状态、策略、认证）作为常规 Deployments/Services 运行在 Kubernetes 上。
- **执行沙箱平面**（工具调用、shell/代码执行、浏览器/文件访问）在由沙箱后端选择的隔离 worker 中运行。

OpenClaw 当前的沙箱后端选项包括 `DockerSandboxBackend`、`SSHSandboxBackend` 和 `OpenShellSandboxBackend`，在其后端配置中具有 `mirror` 和 `remote` 执行模式。此模型与 `agent-sandbox` CRD 或任何为每个 agent 会话分配一个隔离运行时的内部沙箱 operator 兼容。

#### 最小云原生形态

1. **Ingress/API 层**：Agent 网关（`Deployment` + `Service` + `HorizontalPodAutoscaler`）
2. **调度器/队列层**：将任务路由到沙箱池（Kubernetes Queue、内部队列或工作流引擎）
3. **沙箱运行时层**：每个会话/任务一个沙箱，具有严格配额、网络策略和文件系统边界
4. **可观测性层**：将 `agent_step`、`tool_call` 和 `sandbox_id` 一起追踪，用于调试和成本归因

### Agent 运行时选择：Pod 还是 VM？

简短回答：将**控制平面运行在 Pod 中**，并根据风险级别为**沙箱工作节点**选择 Pod 运行时 vs VM 运行时。

| 运行时选择 | 最佳适用场景 | 隔离强度 | 启动/成本特征 | GPU 路径 |
| -------------- | -------- | ------------------ | -------------------- | -------- |
| Pod（`runc`） | 可信内部工具，低风险 | 低 | 最快，最便宜 | 原生 |
| Pod + `gVisor` RuntimeClass | 需要中等兼容性的不可信代码 | 中 | 快速，低开销 | 受限/取决于栈 |
| Pod + Kata RuntimeClass（微虚拟机隔离） | 多租户不可信工作负载 | 高 | 较慢，较高开销 | 直通设置良好 |
| 每 agent/租户专用 VM | 最强边界，合规敏感场景 | 非常高 | 最慢，最昂贵 | 完全 VM 控制 |

#### 实用建议

1. 从 **Kubernetes Pods 用于控制平面**和**沙箱工作节点使用 RuntimeClass**（`gVisor` 用于 CPU 密集型不可信任务，`Kata` 用于更严格的隔离）开始。
2. 仅对高风险租户、受监管数据或在需要内核隔离/合规要求硬 VM 边界时使用 **VM 池**。
3. 保持 agent 运行时**临时性**（预热池 + TTL 清理），仅持久化明确的工作（日志、输出、检查点）。

示例 `Sandbox` 运行时选择：

```yaml
apiVersion: agents.x-k8s.io/v1alpha1
kind: Sandbox
metadata:
  name: openclaw-session-123
spec:
  podTemplate:
    spec:
      runtimeClassName: kata-qemu
      containers:
      - name: openclaw-worker
        image: ghcr.io/example/openclaw-worker:latest
        resources:
          limits:
            cpu: "2"
            memory: "4Gi"
```

### 沙箱技术全景

| 技术 | 冷启动 | 隔离 | Python | 镜像构建 |
| ---------- | --------- | --------- | ------ | ----------- |
| 容器（Docker） | ~50ms | 共享内核 | 完整 | Dockerfile |
| Firecracker 微虚拟机 | ~125ms（+快照 <1s） | 独立内核 | 完整 | ext4 rootfs |
| Kata Containers | ~125ms | 独立内核 | 完整 | OCI/Docker |
| WASM（monty） | 0.06ms | WASI | 有限 | 自定义 |
| Unikernel（Unikraft） | <100ms | 最小内核 | 改进中 | 自定义 |

核心权衡由 高策（gaocegege）在 <a href="https://gaocegege.com/Blog/genai/unikernel-agent">Agent sandbox 可能的选型以及 unikernel 的机会</a>（2026）中分析：

- **e2b**（<a href="https://github.com/e2b-dev/e2b">GitHub</a>）：使用 Firecracker 微虚拟机。模板是完整的 VM 快照，在 1 秒内恢复（Intel <8ms，AMD <3ms）。镜像构建将 Dockerfile 转换为 ext4 rootfs。使用轻量级 best-of-k 调度器而非 Kubernetes。
- **k7**（<a href="https://github.com/Katakate/k7">GitHub</a>）：使用带有 Firecracker VMM 的 Kata Containers。直接处理 OCI/Docker 镜像——无需 ext4 转换。与 Kubernetes/k3s 集成。Kata 的抽象阻止直接使用 Firecracker 快照。
- **monty**（<a href="https://github.com/pydantic/monty">GitHub</a>）：基于 WASM 的 Python 子集解释器，实现 0.06ms 冷启动，但仅支持 Python 的有限子集（无 `class`，有限 stdlib）。
- **Unikraft**（<a href="https://unikraft.org/">网站</a>）：Unikernel 框架，将应用 + 最小内核编译为单个镜像。极小的攻击面和快速启动。在 v0.19（2025 年 5 月）中添加了有限的多进程支持。Browser Use 在生产中使用 Unikraft 微虚拟机进行零密钥 agent 隔离。

### 镜像分发优化

对于大型镜像，拉取通常是主要的延迟来源——而非容器/VM 启动本身。Modal 的**基于 FUSE 的延迟加载**方法从优先级缓存链（内存 → 本地 SSD → 区域缓存 → CDN → 对象存储）按需提供文件，极大地缩短了启动时间。类似技术：<a href="https://github.com/containerd/stargz-snapshotter">estargz</a>、<a href="https://github.com/dragonflydb/dragonfly">Dragonfly</a>。

### 选型指南

对于生产 AI-Infra 平台，避免在不首先决定层次边界的情况下选择单个"沙箱项目"：

1. **平台 API**：根据自托管、协议兼容性和许可选择 OpenSandbox、CubeSandbox、E2B、Daytona 或 Sandbox0。
2. **Kubernetes API**：当平台需要声明式生命周期、预热池和稳定沙箱身份时，使用 `kubernetes-sigs/agent-sandbox`。
3. **运行时边界**：将风险级别映射到 `RuntimeClass` 或 VM 池：gVisor 用于密度，Kata 用于 VM 隔离，专用 VM/微虚拟机池用于受监管租户。
4. **开发者工作台**：与集群沙箱分开评估 Cleanroom 或 Brood Box，因为本地编码 agent 需要仓库 diff 审查、凭证转发和默认拒绝的出口策略。

有用的决策规则是：**OpenSandbox 或 CubeSandbox 是平台选择；Agent Sandbox 是 Kubernetes 生命周期选择；gVisor、Kata、nerdbox 和 Firecracker 是运行时选择**。

### 参考

- <a href="https://browser-use.com/posts/two-ways-to-sandbox-agents">Browser Use: Building Secure, Scalable Agent Sandbox Infrastructure</a>
- <a href="https://gaocegege.com/Blog/genai/unikernel-agent">高策: Agent sandbox 可能的选型以及 unikernel 的机会</a>
- <a href="https://github.com/e2b-dev/e2b">e2b (Firecracker sandbox)</a>
- <a href="https://github.com/alibaba/OpenSandbox">OpenSandbox</a>
- <a href="https://github.com/TencentCloud/CubeSandbox">CubeSandbox</a>
- <a href="https://github.com/containerd/nerdbox">containerd/nerdbox</a>
- <a href="https://github.com/buildkite/cleanroom">Cleanroom</a>
- <a href="https://github.com/stacklok/brood-box">Brood Box</a>
- <a href="https://github.com/superradcompany/microsandbox">microsandbox</a>
- <a href="https://github.com/Katakate/k7">k7 (Kata sandbox)</a>
- <a href="https://unikraft.org/">Unikraft</a>
- <a href="../docs/kubernetes/isolation.md#6-agent-sandbox">详细实施指南：Kubernetes Isolation 文档中的 Agent Sandbox</a>

---

## Agent 基础设施组件

### Model Context Protocol（MCP）

**状态**：已采用（CNCF Tech Radar 2025）

Model Context Protocol（MCP）是一个新兴的 agent 间通信和上下文共享标准。根据 CNCF Tech Radar 2025，MCP 和 Llama Stack 处于"Adopt"位置。

**关键概念：**

- 在 agent 之间共享上下文的标准协议
- 用于上下文管理的服务器-客户端架构
- 与现有数据源和工具的集成
- 支持 agent 生态系统

**使用 MCP 的项目：**

- 火山引擎 Native AI Agent Kit（MCP Hub、MCP Server）
- 各种 agent 框架正在采用 MCP 以实现互操作性

### Agent-to-Agent（A2A）/ Agent Communication Protocol（ACP）

下一代 agent 协作协议：

- **A2A**：直接的 agent 间通信模式
- **ACP**：面向多 agent 系统的标准化通信协议

**状态**：新兴标准，正在各种项目中进行评估

### Kube-Agentic-Networking

<a href="https://github.com/kubernetes-sigs/kube-agentic-networking">`kube-agentic-networking`</a>

Kubernetes SIG 项目，用于定义 Kubernetes 中 agent 和工具的网络策略和治理。

**关键特性：**

- agent 通信的网络策略
- 工具访问控制和治理
- Agent 到工具的安全边界
- 与 Kubernetes NetworkPolicy API 的集成

## CNCF 和生态系统倡议

### CNCF Agentic System Initiative

<a href="https://github.com/cncf/toc/issues/1746">CNCF TOC Issue #1746</a>

CNCF 技术监督委员会正在探索一项 Agent System 倡议，以协调云原生生态系统中的努力。

**重点领域：**

- agent 接口的标准化
- agent 的安全和隔离
- 多 agent 编排模式
- 与 CNCF 项目的集成

**状态**：讨论中（2024-2025）

### WG AI Integration

<a href="https://github.com/kubernetes/community/blob/master/wg-ai-integration/charter.md">Kubernetes WG AI Integration Charter</a>

专注于将 AI 能力集成到 Kubernetes 中的工作组，包括 agent 工作负载。

**范围：**

- Kubernetes 中的 AI 工作负载模式
- Agent 生命周期管理
- 与 Kubernetes API 的集成
- Kubernetes 上 AI agent 的最佳实践

### KubeEdge Sedna

<a href="https://github.com/kubeedge/sedna">`Sedna`</a>

Sedna 是一个 KubeEdge 子项目，提供边云协同 AI 能力，包括联邦学习、增量学习和具有类 agent 协调的边缘推理。

**关键特性：**

- AI 工作负载的边云协同
- 跨边缘节点的联邦学习
- 增量学习和模型更新
- 边云联合推理
- 持续学习能力

**状态**：CNCF Incubating（KubeEdge 项目）

**使用场景：**

- 边缘 AI agent 部署
- 跨边缘设备的分布式学习
- 带云协调的边缘模型推理
- IoT 和边缘计算场景

**架构：**

- GlobalManager：云端控制平面
- LocalController：边缘侧 agent 组件
- Workers：执行 AI 任务（训练、推理、评估）
- 跨边云的数据集/模型管理

### CNCF Tech Radar 2025 - Agentic AI Platforms

根据 **CNCF Tech Radar 2025（Q3）**，Agent AI 领域分类如下：

**Adopt：**

- **Model Context Protocol (MCP)**：大多数用例的可靠选择
- **Llama Stack**：面向 agent 平台的成熟技术

**Trial：**

- **Agent2Agent (A2A)**：值得探索以满足特定需求
- **agentgateway**：实验性但有前景
- **Haystack**：多模态 agent 框架
- **autogen**：Microsoft 的 agent 框架

**Assess：**

- **kagent**：需要仔细评估的 CNCF 项目
- **kgateway**：正在评估中的 agent 网关

**Hold：**

- **LangChain**：成熟但不太适合 Kubernetes 原生用例
- **crewAI**：流行但对云原生部署来说过于复杂

参考：CNCF Tech Radar 提供了对技术成熟度和采用建议的开发者感知。

### 相关视频资源

- **YouTube**：<a href="https://www.youtube.com/watch?v=WvpDBJVjIbI">Kubernetes and AI Agents</a>
- **Blog**：<a href="https://jimmysong.io/blog/kubernetes-ai-oss-solo/">Kubernetes AI OSS Solo</a> by Jimmy Song

### DeepSeek AI Agent 策略

主要 AI 模型提供商正在将 agent 作为其平台的核心：

- <a href="https://x.com/deepseek_ai/status/1995452641430651132">DeepSeek Agent Announcement</a>
- 模型越来越多地将 agent 能力作为一等公民设计
- 从纯推理向 agent 推理模式的转变

## 学习主题

### 核心概念

1. **Agent 架构**：
   - 感知：Agent 如何理解其环境
   - 推理：使用 LLM 进行决策
   - 行动：工具使用和 API 调用
   - 记忆：短期和长期上下文

2. **Agent 编排**：
   - 单 agent vs 多 agent 系统
   - Agent 协作模式
   - 任务分解和规划
   - 错误处理和恢复

3. **Agent 安全**：
   - 沙箱环境（模式 1：隔离工具 vs 模式 2：隔离 agent）
   - 技术选择：Firecracker、Kata Containers、Unikernel、WASM
   - 资源限制和配额
   - 工具访问控制
   - 审计日志
   - 参见 [Agent 沙箱基础设施](#agent-沙箱基础设施) 和 [隔离指南](../docs/kubernetes/isolation.md#6-agent-sandbox)
   - 运行时基准测试：[runc vs gVisor vs Kata vs VM](./runtime-benchmark.md)

### 基础设施模式

1. **部署模型**：
   - Serverless agent（缩零）
   - 长运行 agent 服务
   - 批处理 agent 执行
   - 事件驱动 agent 调用

2. **资源管理**：
   - Agent 的 CPU/内存分配
   - Agent 工作负载的 GPU 调度
   - 弹性扩展策略
   - 成本优化

3. **可观测性**：
   - Agent 性能指标
   - Token 消耗跟踪
   - 工具调用追踪
   - 决策路径可视化

### 协议和标准

1. **Model Context Protocol (MCP)**：
   - 服务器实现
   - 客户端集成
   - 上下文共享模式
   - 工具注册

2. **Agent 通信**：
   - A2A 协议模式
   - 消息格式和模式
   - 状态同步
   - 错误处理

## 路线图

### 短期（2026 Q1-Q2）

- [ ] 扩展每个主要 agent 平台的文档
- [ ] 创建在 Kubernetes 上部署 agent 的实用指南
- [ ] 记录 MCP 实现模式
- [x] 添加 agent 安全最佳实践和沙箱基础设施指南

### 中期（2026 Q3-Q4）

- [ ] 多 agent 系统模式和架构
- [ ] Agent 可观测性和监控指南
- [ ] 与 AI Gateway 方案的集成模式
- [ ] Agent 工作负载的成本优化策略

### 长期（2027+）

- [ ] Agent 原生 Kubernetes 扩展
- [ ] 高级多 agent 编排
- [ ] Agent 市场和生态系统
- [ ] 标准化工作和协议演进

## 参考

### 官方文档

- [KAgent Documentation](https://github.com/kagent-dev/kagent)
- [Volcano AgentCube](https://github.com/volcano-sh/agentcube)
- [Volcano Kthena](https://github.com/volcano-sh/kthena)
- [OpenClaw](https://openclawai.github.io/)
- [OpenClaw Sandboxing](https://openclawai.github.io/docs/deploying/sandboxing/)
- [Kubernetes SIG Agent Sandbox](https://github.com/kubernetes-sigs/agent-sandbox)
- [Agent Substrate](https://github.com/agent-substrate/substrate)
- [Agent Infra Sandbox](https://github.com/agent-infra/sandbox)
- [Kube-Agentic-Networking](https://github.com/kubernetes-sigs/kube-agentic-networking)
- [LangChain DeepAgents](https://github.com/langchain-ai/deepagents)
- [ArgoCD Agent](https://github.com/argoproj-labs/argocd-agent)
- [KubeEdge Sedna](https://github.com/kubeedge/sedna)
- [NVIDIA NeMo Agent Toolkit (Code Sandbox)](https://docs.nvidia.com/nemo/agent-toolkit/latest/reference/sandbox.html)

### 社区资源

- [CNCF Agentic System Initiative](https://github.com/cncf/toc/issues/1746)
- [WG AI Integration Charter](https://github.com/kubernetes/community/blob/master/wg-ai-integration/charter.md)
- [CNCF Tech Radar 2025](https://radar.cncf.io/)
- [Agent 演进理论 - 微信公众号文章](https://mp.weixin.qq.com/s/NUx4n5j0ftxzZ0Sz29RjOQ)
- [Browser Use: Building Secure, Scalable Agent Sandbox Infrastructure](https://browser-use.com/posts/two-ways-to-sandbox-agents)
- [高策: Agent sandbox 可能的选型以及 unikernel 的机会](https://gaocegege.com/Blog/genai/unikernel-agent)

### 相关文档

- [AI Gateway & Agentic Workflow](../README.md#-3-ai-gateway--agentic-workflow)
- [Agent Sandbox and Isolation](../docs/kubernetes/isolation.md#6-agent-sandbox)
- [Runtime Benchmark: runc vs gVisor vs Kata vs VM](./runtime-benchmark.md)
- [Memory and Context Management](../docs/inference/memory-context-db.md)
- [LLM Inference Platforms](../docs/inference/README.md) - 了解 Kaito、AIBrix 和 OME 等为 agent 系统提供推理后端的模型部署平台

---

**注意**：这是一个快速发展的领域。许多项目处于早期阶段或实验性。在采用之前，请务必验证项目成熟度和生产就绪性。部分内容由 AI 辅助生成——在生产环境中使用前请核实细节。
