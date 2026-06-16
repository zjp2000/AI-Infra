---
status: Active
maintainer: pacoxu
last_updated: 2026-05-21
tags: blog, kubernetes, ai-infrastructure, 博客
canonical_path: zh/docs/blog/README.md
---

# AI-Infra 博客文章

较旧的文章已归档至 [docs/archive-blog](../archive-blog/README.md)。

本目录包含关于 AI 基础设施、Kubernetes 调度及相关主题的博客文章。

## 2026-05-21：Kubernetes Scheduler 演进主线：从 Predicates/Priorities 到 Scheduling Framework，再到 Workload-Aware Scheduling

- [Kubernetes Scheduler 演进主线：从 Predicates/Priorities 到 Scheduling Framework，再到 Workload-Aware Scheduling (中文)](./2026-05-21/2026-05-21-kubernetes-scheduler-framework-evolution_zh.md)

一篇以架构视角回顾 `kube-scheduler` 如何从内置的 predicates/priorities 演进为插件框架，进而迈向工作负载级调度的中文文章：

- **三段式演进**：将调度器历史框架为 `Predicates/Priorities + Extenders` → `Scheduling Framework` → `Workload-Aware Scheduling`
- **以框架为中心**：使用 `KEP-624` 和经典扩展点图作为核心解释工具，而非逐条列举发布说明
- **为什么 `v1.16-v1.20` 很重要**：解释为什么该时期可作为实用的"框架形成阶段"来写作，同时仍区别于官方的 `v1.19` 稳定里程碑
- **为什么这仍然重要**：将 `profiles`、队列行为和 `v1.35/v1.36` 工作负载感知调度连接到原始框架设计

## 2026-05-13：推理编排方案如何选择？AIBrix、Kthena、Dynamo、llm-d、KServe、vLLM Production Stack 与 SGLang/RBG

- [推理编排方案如何选择？AIBrix、Kthena、Dynamo、llm-d、KServe、vLLM Production Stack 与 SGLang/RBG (中文)](./2026-05-13/2026-05-13-how-to-choose-inference-orchestration-stacks_zh.md)
- [How to choose an inference orchestration stack: AIBrix, Kthena, Dynamo, llm-d, KServe, vLLM Production Stack, and SGLang/RBG](./2026-05-13/2026-05-13-how-to-choose-inference-orchestration-stacks.md)

对早期推理编排主题的双语更新，重新围绕堆栈层级而非项目名称来组织：

- **不要比较不同层级**：在比较项目之前，先区分运行时、服务栈、控制面和工作负载原语
- **最新公开进展**：更新 `KServe + llm-d`、`AIBrix`、`Kthena`、`Dynamo/Grove`、`vLLM Production Stack` 和 `SGLang/RBG` 截至 2026-05-13 的当前公开定位
- **基础设施优先的决策指南**：为 Volcano、KServe、vLLM、NVIDIA、原生 Kubernetes 和 SGLang 用户提供实用的选择路径
- **PD 仍然有条件性**：保留警告——prefill/decode 分离是系统权衡，并非自动获胜

## 2026-05-11：NVIDIA 推理编排主线拆解：Dynamo、Grove、KAI Scheduler 与 GPU DRA Driver

- [NVIDIA 推理编排主线拆解：Dynamo、Grove、KAI Scheduler 与 GPU DRA Driver (中文)](./2026-05-11/2026-05-11-dynamo-grove-kai-dra-ecosystem-zh.md)

从更广泛的 NVIDIA 开源全景聚焦到围绕 Dynamo 的编排路径的中文生态分析：

- **并非四个竞争项目**：将 `Dynamo → Grove → KAI Scheduler → GPU DRA Driver` 框架为从服务意图到硬件拓扑的纵向架构
- **Grove 为何重要**：将 Grove 解释为多角色分离式推理的工作负载抽象层，而不仅仅是另一个控制器
- **KAI 为何重要**：突出拓扑感知调度、层级化 PodGroups、DRA 支持和 fairshare 作为 AI 集群的执行层
- **DRA 为何现在重要**：将上游 Kubernetes 1.35/1.36 的 DRA 进展连接到 NVIDIA 的 `ComputeDomain` 和未来的 GPU 共享语义
- **NIXL、NCCL 和 AICR 的定位**：将 `NIXL/NCCL` 定位为数据移动和 GPU 通信层，`AICR/GPU Operator` 定位为集群基线和运维支持，而非编排核心
- **LWS 和 vLLM 的定位**：将 `LWS + Volcano` 定位为官方社区原生路径，并指出当团队已标准化使用 `Koordinator` 时可扮演类似调度角色；`vLLM` 仍然是引擎/数据面，而非与 Grove/KAI/DRA 同层

## 2026-04-28：如何理解 Hugging Face、私有 Hugging Face、MatrixHub、Harbor + Dragonfly + ModelPack 与 ModelExpress

- [如何理解 Hugging Face、私有 Hugging Face、MatrixHub、Harbor + Dragonfly + ModelPack 与 ModelExpress (中文)](./2026-04-28/2026-04-28-understanding-model-distribution-stack_zh.md)

将公共模型中心、私有模型中心、OCI 优先的制品栈、节点级 P2P 分发和运行时权重传输整合到统一模型分发视图的中文架构文章：

- **分层框架**：区分公共模型源、企业治理、节点分发和运行时加速
- **场景驱动对比**：解释何时优先选择 MatrixHub，何时选择 Harbor + ModelPack + Dragonfly，以及何时 ModelExpress 成为正确答案
- **一站式图表**：提供涵盖 Hugging Face、ModelScope、MatrixHub、Harbor、Dragonfly、ModelPack 和 ModelExpress 的 Mermaid 概览

## 2026-04-23：Kubernetes v1.36 DRA 整体设计

- [Kubernetes v1.36 DRA 的整体设计：从请求入口到调度、状态与拓扑 (中文)](./2026-04-23/2026-04-23-kubernetes-v1.36-dra-ai-infra_zh.md)

基于 Kubernetes v1.36 DRA KEP 图的中文架构分析：

- **架构图**：按请求入口、资源建模、调度与绑定、状态、可观测性和管理边界组织 DRA
- **官方 DRA 博客跟进**：现在也融入了上游 2026-05-07 DRA 博客，并将最近的 v1.33 → v1.34 → v1.36 进展关联到同一篇文章
- **拓扑焦点**：突出为什么 GPU/NIC、NUMA、PCIe、fabric 和超节点布局重塑了调度器问题
- **成熟度与下一步**：添加 v1.37 关注列表加上 KEP 和开放问题限制表格，将实际的 v1.36 能力与 DRA+WAS 及其他仍在开放的设计方向区分开来

## 2026-04-15：Kubernetes v1.34 PSI Metrics Beta（AI-Infra 落地）

- [Kubernetes v1.34 PSI Metrics Beta：AI-Infra 该如何落地 (中文)](./2026-04-15/2026-04-15-kubernetes-v1.34-psi-metrics-beta-ai-infra_zh.md)

一篇关注在 KEP-4205 进入 Beta 后如何在 AI 集群中落地 PSI 指标的中文实践指南：

- **从内核到平台信号**：将 Linux PSI 映射到 kubelet Summary API 和 `/metrics/cadvisor` 采集路径
- **可操作的可观测性**：突出六个 PSI Prometheus 指标并提出节点/工作负载告警起点
- **AI 导向的闭环**：将 PSI 压力连接到调度和扩缩容决策，而非仅停留在仪表盘层面

## 2026-04-15：VolumeGroupSnapshot + CBT for AI Checkpoints

- [用 VolumeGroupSnapshot + CBT 改善 AI Checkpoint 备份与恢复 (中文)](./2026-04-15/2026-04-15-volume-group-snapshot-and-cbt-for-ai-checkpoints_zh.md)

结合 KEP-3476 和 KEP-3314 用于 AI 训练备份和恢复的中文实现说明：

- **一致的多卷恢复点**：使用 VolumeGroupSnapshot 实现跨 PVC 组的崩溃一致性检查点
- **增量备份路径**：引入变更块跟踪 (CBT) 以避免重复的全量卷传输
- **生产部署框架**：包括驱动能力检查、分阶段采用和恢复优先的验证指南

## 2026-04-06：GPU Pod 冷启动优化：AI 工作负载的性能突破

- [GPU Pod 冷启动优化：AI 工作负载的性能突破 (中文)](./2026-04-06/2026-04-06-gpu-pod-cold-start-optimization_zh.md)

一篇将 PR #281 中的 AI 启动关键路径模型扩展到生产场景的中文实践深入分析：

- **关键路径 + 经济学**：将启动分解与租户级预热池成本建模相结合
- **模型加载优先**：在微调度器调优之前，优先考虑 Run:ai Model Streamer 和缓存编排 (Fluid)
- **容量感知扩缩容**：将大规模集群调度模式与多租户 AI 工作负载下的冷启动预算控制联系起来
- **不仅是加速**：引入规避路径（VPA 和 v1.35 中的原地 Pod 容器重启）以减少不必要的完整冷启动

## 2026-04-01：Kubernetes Swap Status in 2026

- [Kubernetes Swap Status in 2026: kubelet behavior, memory pressure, and what is next](./2026-04-01/2026-04-01-kubernetes-swap-status-kubelet-and-memory-pressure.md)
- [Kubernetes Swap 现状（2026）：kubelet 行为、内存压力与场景化落地 (中文)](./2026-04-01/2026-04-01-kubernetes-swap-status-kubelet-memory-pressure_zh.md)

以实用运维视角提供的 Kubernetes swap 支持现状总结：

- **当前状态**：KEP-2400 已实现并关闭；NodeSwap 在 1.34 中 GA
- **当前 kubelet 行为**：`NoSwap` 与 `LimitedSwap`，Linux + cgroup v2 路径
- **运维焦点**：可观测性端点、`kubectl top --show-swap` 和内存压力调优边界
- **路线图缺口**：工作负载控制的 swap、swap 感知调度和 swap 感知驱逐，分别在 #5359/#5424/#5433 中跟踪
- **场景导向指南**：将会议资料转化为具体的落地场景（成本密度、内存尖峰、控制面保护、细粒度策略探索）

## 2026-03-24：维护扫描（过期资源 + 安全升级）

在此次清理中实施：

- 标准化博客文件名为 `YYYY-MM-DD-topic-name(_lang).md`，以减少交叉引用和搜索索引中的歧义
- 从顶层 README 中移除重复的博客长列表，将完整归档保留在此文件中作为规范索引
- 移除 Agones 文章及所有相应的索引引用

待跟进的升级建议：

- 在 CI 中为 `docs/blog/**` 添加轻量级 markdown 链接检查器
- 每季度运行元数据刷新（`last_updated`、规范链接和过期的外部引用）

## 2026-03-24：vLLM v0.18.0 发布解读：AI-Infra 视角的重点变化

- [vLLM v0.18.0 发布解读：AI-Infra 团队值得重点关注什么？ (中文)](./2026-03-24/2026-03-24-vllm-v0.18.0-ai-infra-highlights_zh.md)

从 AI 基础设施运维视角对 vLLM v0.18.0 发布说明的中文分析：

- **服务接口扩展**：新的 `--grpc` 服务模式和 `vllm launch render` 用于无 GPU 预处理
- **内存层级升级**：更智能的 KV offloading、FlexKV 后端和多 KV 组 offload 支持
- **大规模 MoE 服务**：Elastic EP Milestone 2 与 NIXL-EP 和更快的专家加载路径
- **运维风险门控**：已知的 Qwen3.5 + FP8 KV cache 在 B200 上的精度问题，以及依赖/默认行为变更
- **发布清单**：为平台团队提供的具体灰度与升级检查项

## 2026-03-13：NVIDIA AICR：把 GPU 集群安装经验固化成可复现 Recipe

- [NVIDIA AICR 中文介绍](./2026-03-13/2026-03-13-nvidia-aicr-introduction_zh.md)

基于项目官方仓库和最近合并的 AI-Infra PR 编写的 NVIDIA AI Cluster Runtime (AICR) 中文介绍：

- **AICR 是什么**：一个配方驱动的工具，用于优化、验证和可复现的 GPU 加速 Kubernetes 集群设置
- **为什么重要**：将脆弱的手册转化为版本锁定、可审计的 Helm 和 GitOps 部署制品
- **核心工作流**：`snapshot` 捕获实时状态，`recipe` 定义目标，`validate` 检查漂移，`bundle` 渲染可部署输出
- **如何定位**：AICR 位于 GPU Operator 之上，补充 DRA，专注于集群基线管理而非设备分配
- **当前范围**：早期阶段但前景广阔，支持 EKS、GKE、Kind、H100/GB200、Ubuntu、Kubeflow 和 Dynamo

## 2026-03-13：【2020】拜耳作物科学以 15,000 节点的 GKE 集群为未来播种

- [Bayer Crop Science and a 15K-Node GKE Cluster (中文)](./2026-03-13/2026-03-13-bayer-gke-15000-nodes_zh.md)

关于 Google Cloud 2020 年 11 月 17 日拜耳作物科学将 GKE 扩展到 15,000 个工作节点的案例研究的中文翻译与回顾：

- **为什么 1K 节点不够**：基于路由的网络和 VPC 路由规模成为旧集群设计的实际天花板
- **关键产品变更**：VPC 原生 GKE 与 Alias IP 消除了每节点路由依赖，解锁了更大集群
- **区域集群改进**：自定义子网模式为大型区域部署实现了更精确的每可用区 IP 规划
- **成果**：单集群扩展到 **15,000 个工作节点**和 **50,000 个 Pod**
- **2026 年仍然相关**：提醒网络和 IP 规划通常成为第一个真正的规模瓶颈

## 2026-02-25：贡献开源的 ROI：LF Research 2025 年调查报告解读

- [贡献开源的 ROI (中文)](../archive-blog/2026-02-25/2026-02-25-opensource-contribution-roi_zh.md)

分析 Linux Foundation Research 2025 年开源 ROI 调查的中文博客，涵盖上游开源贡献的商业案例：

- **私有 Fork 的隐性成本**：组织花费数十万维护 fork，需承担持续的合并成本、延迟的安全补丁和重复的工程投入
- **上游贡献的三大核心收益**：更快的安全响应、加速的开发速度、更强的人才吸引和保留
- **量化 ROI**：平均成本效益比 2-5 倍；按定量模型，贡献组织实现约 **6 倍 ROI**
- **AI 基础设施背景**：调查结果如何适用于 Kubernetes、vLLM、DRA/NRI 和其他快速发展的开源生态系统
- **实用建议**：上游优先策略、专项贡献时间、开源参与绩效激励

## 2026-02-14：关注 AAIF 进展：Agentic AI Foundation 及其旗下活动预告

- [AAIF Introduction (中文)](../archive-blog/2026-02-14/2026-02-14-aaif-introduction_zh.md)

对 Agentic AI Foundation (AAIF) 及其即将举办的重点活动的全面介绍：

- **AAIF 概览**：一个中立的开放基金会，确保 AI Agent 能力透明协作地演进
- **AGNTCon + MCPCon 2026**：10 月 22-23 日，加州圣何塞——年度旗舰会议，从 MCP 扩展到整个开放智能体 AI 生态系统
- **MCP Dev Summit North America 2026**：4 月 2-3 日，纽约市——面向使用 Model Context Protocol 推进 AI Agent 的开发者的顶级聚会
- AAIF 的重要性：开放标准、社区协作、开源支持
- 如何参与 AAIF 社区

## 2026-01-15：推理平台实践与 AI 成本优化

- [Inference Cost Optimization (中文)](../archive-blog/2026-01-15/2026-01-15-inference-cost-optimization_zh.md)

AI 推理平台 (MaaS - Model as a Service) 成本优化策略的全面指南：

- 五大优化策略：
  - **减少无效 Token**：提示模板管理、输出长度控制、结构化输出约束、Agent 治理、RAG 质量提升
  - **加速计算**：动态批处理、并行化、内核优化、推测解码、量化
  - **提高利用率**：基于 backlog 的自动扩缩容、混合部署、峰谷策略
  - **降低资源成本**：硬件选择、部署形态优化、竞价实例、下一代硬件 (CXL, AI-native 存储)
  - **质量-成本策略**：智能路由、多级降级、分层 SLO 与预算
- 集群规模优化分支：小型 (1-8 GPU)、中型 (8-64 GPU)、大型 (64+ GPU)
- 成本可观测性与实施路线图
- 引用 NVIDIA Inference Context Memory Storage Platform 和行业最佳实践
- 链接到详细的推理文档：缓存、prefill-decode 分离、模型生命周期、AIBrix 平台

## 2026-01-12：KubeCon + CloudNativeCon Europe 2026 主题推荐

- [KubeCon EU 2026 Recommendations (中文)](../archive-blog/2026-01-12/2026-01-12-kubecon-eu-2026-recommendations_zh.md)

KubeCon + CloudNativeCon Europe 2026 的全面指南，包含精选的 AI 基础设施会议推荐：

- CNCF 官方公告翻译
- 会议日程链接（主会议和联合活动）
- 四个推荐的 AI 基础设施会议：
  - BYD 的百万任务规模实践与 Argo Workflows
  - KV-Cache 教程：在 Kubernetes 上构建 AI 感知的 LLM 路由
  - AI 推理工作负载的自适应路由
  - 重新定义 LLM 推理 SLI/SLO
- 会议详情含时间、地点、演讲者和中文描述
- 会议日程和注册链接

## 2026-01-07：Ambient Global Compute — Orchestrating the Non-Elastic Cloud

- [Ambient Global Compute 中文博客](../archive-blog/2026-01-07/2026-01-07-ambient-global-compute_zh.md)

基于 Jago Macleod 的 KubeCon 演讲，关于使用 Kubernetes 编排非弹性云的全面博客：

- 基础设施摆锤：从托管到虚拟化到弹性云到非弹性云
- 无限容量的云计算黄金时代
- 三大变革驱动力：硬件碎片化、区域扩展、GPU 稀缺
- CapEx 的回归与运维逆转
- Ambient Compute 四大支柱：工作负载编排、Kueue、优先级感知、全局调度
- 利用率与延迟冲突的解决
- 两种实用模式：全局批处理计算机和固定硬件上的弹性平台
- 用于全局容量调度的 MultiKueue
- KubeCon NA 2025 演讲参考

## 2025-12-22：OCI Is Quietly Taking Over Everything

- [OCI Taking Over Everything (English)](../archive-blog/2025-12-22/2025-12-22-oci-taking-over-everything_en.md)
- [OCI 正在悄悄占领一切 (中文)](../archive-blog/2025-12-22/2025-12-22-oci-taking-over-everything_zh.md)

全面分析 OCI（开放容器倡议）如何在 AI 时代成为镜像、Helm 图表、AI 模型和 WASM 的统一分发 backbone，引用 KubeCon Atlanta 见解：

- OCI 演进：从容器镜像到通用制品分发
- 为什么 AI 工作负载放大了对 OCI 的需求（规模、治理、统一）
- Kubernetes v1.35：OCI Image Volumes 默认启用 (Beta)
- ModelPack：使 AI 模型成为 OCI 一等公民
- Harbor v2.14.0：增强的 CNAI 模型集成
- Docker Model Runner：使用 OCI 统一推理引擎 (llama.cpp, vLLM)
- ORAS：OCI 制品的瑞士军刀
- WASM 制品仓库：下一个统一拼图
- 行业信号：Bitnami 政策变更、Docker Hardened Images (DHI)
- 性能演进：containerd v2.2 Rebase Snapshot
- 企业实用采用路线图

## 2025-12-22：vLLM 2025 Retrospective & 2026 Roadmap

- [vLLM 2025 Retro & 2026 RoadMap (English)](../archive-blog/2025-12-22/2025-12-22-vllm-2025-retro-2026-roadmap_en.md)
- [vLLM 2025 年度回顾与 2026 年路线图 (中文)](../archive-blog/2025-12-22/2025-12-22-vllm-2025-retro-2026-roadmap_zh.md)

基于 vLLM Office Hours #38 的 vLLM 2025 年成就和 2026 年愿景全面总结：

- vLLM 项目概况：65K+ stars，800+ PRs/月，2000+ 贡献者
- 2025 年增长亮点：Q1/Q2 增长 80%，Q3 增长 30%
- API 演进：Agentic AI 和 RL 支持与原生框架集成
- 模型生态系统：100+ 架构、Transformers 后端、SOTA 视觉模型
- 引擎改造：V1 核心架构与混合分配器和 KV 连接器
- 硬件生态系统：TPU、Ascend、Neuron、Gaudi、OpenVINO 支持
- 分布式能力：DeepSeek 2.2k tok/s 案例、vLLM Router 发布
- 2026 年焦点：稳定性、准确性、性能、前沿模型、硬件稳定性

## 2025-12-17：GPU Fault Detection and Self-Healing in Kubernetes

- [GPU 故障检测与自愈实践指南 (中文)](../archive-blog/2025-12-17/2025-12-17-gpu-fault-detection_zh.md)

面向 AI 基础设施工程师和 SRE 的实践指南，涵盖 Kubernetes 集群中 GPU 硬件故障的检测、诊断和自动恢复：

- 四种 GPU 故障类型：显卡掉线、链路故障、内存错误、驱动故障
- 检测方法：DCGM Exporter、Node Problem Detector、NVIDIA Debug Guidelines
- 三层故障语义：NodeCondition、DeviceCondition、WorkloadCondition
- 渐进式修复策略：6 级升级 (L0-L6)
- 故障感知调度：GPU 健康评分及与 Kueue/Volcano 的集成
- 作业级归因：用于公平计费的 DCGM HPC 作业映射
- 生产部署阶段和最佳实践
- 引用字节跳动 Volcano、Microsoft AKS 和 NVIDIA 的实践

## 2025-12-17：DRANET — Community-Driven Kubernetes Network Driver

- [DRANET 介绍 (中文)](../archive-blog/2025-12-17/2025-12-17-dranet-kubernetes-network-driver_zh.md)

关于 DRANET（动态资源分配网络）的全面博客，结合 KubeCon NA 2025 主题演讲和研究论文内容：

- DRANET 概述：基于 DRA 的网络驱动，具有拓扑感知资源管理
- 新生态系统的出现：AI/ML、HPC 和 Telco 驱动网络创新
- DRA 作为通用语言：统一资源分配框架
- 思想汇聚：从 CNI Spec (2015) 到 Network Drivers (2025)
- 拓扑彩票问题：传统网络中的性能瓶颈
- 解决方案：使用 DRA + NRI 的拓扑感知调度
- 性能结果：NCCL 基准测试最高提升 59.6%
- 高性能网络的可组合架构
- 项目捐赠给 kubernetes-sigs 组织
- IEEE LCN 2025 论文和 KubeCon NA 2025 主题演讲参考

## 2025-12-15：ByteDance's Solution for Ultra-Large-Scale Kubernetes Clusters

- [ByteDance Large-Scale K8s (English)](../archive-blog/2025-12-15/2025-12-15-bytedance-large-scale-k8s.md)
- [字节跳动超大规模 K8s 方案 (中文)](../archive-blog/2025-12-15/2025-12-15-bytedance-large-scale-k8s_zh.md)

字节跳动超大规模 Kubernetes 集群解决方案的全面概述：

- **KubeBrain**：支持 20,000+ 节点集群的替代元数据存储系统
- **KubeAdmiral**：下一代多集群编排和调度引擎
- **Gödel Scheduler**：大规模集群的统一调度架构
- **Katalyst**：提高利用率和 QoS 的资源管理系统
- 多集群与单集群的权衡
- KubeCon China 2025 演讲和技术博客参考
- 开源项目和 GitHub 仓库

## 2025-12-15：From SQL on CPUs to Inference on GPUs

- [SQL to Inference Evolution (English)](../archive-blog/2025-12-15/2025-12-15-sql-to-inference.md)
- [SQL 到推理演进 (中文)](../archive-blog/2025-12-15/2025-12-15-sql-to-inference_zh.md)

基于 Robert Nishihara (Anyscale) 在 PyTorchCon 2025 上的演讲，关于 AI 数据处理变革的全面博客：

- 新工作负载：从 SQL on CPUs 到 inference on GPUs 的 AI 数据处理演进
- 数据类型范式转变：从表格数据到多模态数据（图像、视频、音频、文本、传感器）
- PARK 栈：PyTorch + AI + Ray + Kubernetes 作为新的 AI 基础设施标准
- vLLM + Ray 的共同演进：为什么几乎所有 RL 框架都使用 Ray 作为编排器
- Ray + Kubernetes 集成：AI 工作负载的互补优势
- RL/后训练架构和编排需求
- 在研究实验室、科技公司和云提供商的真实采用案例
- 与定义 Web 时代的 LAMP 栈的对比

## 2025-12-15：Multi-Tenancy Isolation in AI Infra Era

- [Multi-Tenancy Isolation (English)](../archive-blog/2025-12-15/2025-12-15-multi-tenancy-isolation.md)
- [多租户隔离性方案探讨 (中文)](../archive-blog/2025-12-15/2025-12-15-multi-tenancy-isolation_zh.md)

AI 基础设施时代 Kubernetes 多租户隔离方案的全面指南：

- 公有云与私有云在训练和推理方面的考量
- 跨维度的对比表：用例、资源、成本、调度、数据、SLO、安全、挑战
- 容器安全隔离策略：集群隔离、节点池、沙箱运行时 (gVisor, Kata)
- Agent Sandbox 预热池模式和安全策略
- 面向敏感 AI 工作负载的机密容器
- 网络和存储隔离最佳实践
- 使用 CNCF MCM Radar 2024Q3 的多集群应用管理
- 多租户频谱：从命名空间到专用集群

## 2025-12-08：GKE 65,000 Node Support — Benchmarking AI Workloads at Scale

- [GKE 65K Nodes (中文)](../archive-blog/2025-12-08/2025-12-08-gke-65k-nodes_zh.md)

Google Cloud 关于 GKE 支持 65,000 节点用于 AI 工作负载的博客文章的全面翻译：

- 65K 节点集群架构与设计
- 大规模集群的调度器优化
- 混合工作负载支持：50K 训练 Pod + 15K 推理 Pod
- 基于抢占机制的工作负载隔离
- 故障恢复能力和 StatefulSet 保证
- 性能基准：Pod 启动时间和 API 服务器性能
- 超大规模的控制面优化
- VPC 原生网络的网络优化
- AI 训练和推理用例
- KubeCon NA 主题演讲参考："Kubernetes in the Second Decade"
- 社区贡献和上游改进

## 2025-12-05：How the Kubernetes Community Operates — Entry Points in the AI Era

- [Kubernetes Community Operations (English)](../archive-blog/2025-12-05/2025-12-05-kubernetes-community-operations.md)
- [Kubernetes 社区运作方式 (中文)](../archive-blog/2025-12-05/2025-12-05-kubernetes-community-operations_zh.md)

关于 Kubernetes 社区结构以及 AI 时代参与入口的全面指南：

- 社区结构：CNCF、Steering Committee、SIGs/WGs、Subprojects
- 贡献者阶梯：从非成员到 SIG Chair
- 截至 2024 年底当前的 SIGs、WGs 和 Committees
- AI/ML 工作组：Batch、Serving、Device Management、AI Gateway、AI Integration
- 新贡献者入门资源
- AI/ML 特定入口和机会
- 社区统计：97.8k 贡献者、463 万次贡献、8.6k 评审员

## 2025-12-03：Ant Group Large-Scale Cluster — 50% Memory Reduction at 20K Nodes

- [Ant Group Large-Scale K8s (English)](../archive-blog/2025-12-03/2025-12-03-ant-group-large-scale-k8s.md)
- [蚂蚁大规模集群经验 (中文)](../archive-blog/2025-12-03/2025-12-03-ant-group-large-scale-k8s_zh.md)

蚂蚁集团 20,000+ 节点大规模 Kubernetes 集群经验的全面概述：

- Etcd 拆分实践 (2022)：运维时间从 1-2 小时减少到 10 分钟
- 数字智能时代的大规模 Kubernetes 服务突破
- API Server 内存优化：零侵入架构减少 50% 内存
- KoM (Kubernetes on Mesh) 网关用于统一流量管理
- 资源分组策略：Pod、Config、Event、Default 组
- 性能提升：CPU -30%、ETCD 存储 -20%、吞吐量 +40%
- 容器交付优化：应用启动速度提升 95%
- E2E 诊断与自愈，L1 拦截率 80%+

## 2025-12-02：KCD Hangzhou — Observability Optimization at Scale

- [KCD Hangzhou Observability (English)](../archive-blog/2025-12-02/2025-12-02-kcd-hangzhou-observability.md)
- [KCD 杭州可观测性优化 (中文)](../archive-blog/2025-12-02/2025-12-02-kcd-hangzhou-observability_zh.md)

涵盖 KCD Hangzhou + OpenInfra Days China 2025 和 KubeCon NA 2025 最热门可观测性主题的博客：

- 小红书 (RED) 大规模指标监控优化
- 10 倍查询加速，节省数万 CPU 核心
- 基于 vmagent 的采集层重构
- 高可用改进和跨云多活部署
- 计算下推与预聚合实现查询加速
- OpenAI 的 Fluent Bit 优化：一行代码释放 30,000 CPU 核心
- 使用 Linux Perf 的性能分析洞察

## 2025-12-01：Kubernetes Safe Upgrade and Rollback

- [Safe Upgrade and Rollback (English)](../archive-blog/2025-12-01/2025-12-01-safe-upgrade-rollback.md)
- [安全升级与回滚 (中文)](../archive-blog/2025-12-01/2025-12-01-safe-upgrade-rollback_zh.md)

基于 Google Cloud 博客和 KubeCon NA 2025 主题演讲的 Kubernetes 安全升级和回滚能力全面指南：

- 模拟版本 (`--emulation-version`) 在 Kubernetes 1.31+ 中可用
- 最低兼容版本 (`--min-compatibility-version`) 在 1.35+ 中
- KEP-4330：兼容版本
- GKE 可靠性：99.98% 升级成功率
- 升级就绪的三个阶段
- 安全升级和回滚流程的最佳实践

## 2025-12-01：Inference Orchestration Solutions

- [Inference Orchestration (English)](../archive-blog/2025-12-01/2025-12-01-inference-orchestration.md)
- [推理编排 (中文)](../archive-blog/2025-12-01/2025-12-01-inference-orchestration_zh.md)

当前开源推理编排解决方案及融合趋势概述：

- 工作负载解决方案：dual LWS (llm-d)、Serving Group (Kthena)、StormService (AIBrix)、Dynamo Grove/LWS、RBG
- 生态系统中的融合趋势
- PD 分离何时真正提供价值
- AIConfigurator 用于配置优化
- 对新部署和现有部署的建议

## 2025-12-01：AWS 10K Node EKS Ultra Scale Clusters

- [AWS 10K Node Clusters (English)](../archive-blog/2025-12-01/2025-12-01-aws-10k-node-clusters.md)
- [AWS 万节点集群 (中文)](../archive-blog/2025-12-01/2025-12-01-aws-10k-node-clusters_zh.md)

继 Google 的 130K 节点集群之后，关于 AWS EKS 超大规模优化的内容：

- 社区改进：Kubernetes v1.33 read/list cache, Karpenter
- AWS 特定：QLDB journal for etcd, BoltDB on tmpfs
- 镜像加速：SOCI Snapshotter 用于延迟加载
- AI 工作负载：LWS + vLLM, CoreDNS 自动扩缩容
- 性能 SLO：get/write 1s, list 30s, 500 pods/second

## 2025-11-28：Agent Sandbox — Secure AI Agents on Kubernetes

- [Agent Sandbox (English)](../archive-blog/2025-11-28/2025-11-28-agent-sandbox.md)
- [Agent Sandbox (中文)](../archive-blog/2025-11-28/2025-11-28-agent-sandbox_zh.md)

关于 Kubernetes SIG Apps 项目 Agent Sandbox 的全面指南，用于安全的 AI Agent 执行：

- 项目介绍和 Sandbox CRD
- gVisor (GKE) 集成状态
- Kata Containers 集成状态
- SandboxWarmPool 实现亚秒级启动延迟
- AI Agent、开发环境和 Notebook 的用例
- 行业趋势和未来方向

## 2025-11-26：JobSet In-Place Restart — 92% Faster Recovery

- [JobSet In-Place Restart (English)](../archive-blog/2025-11-26/2025-11-26-jobset-in-place-restart.md)
- [JobSet In-Place Restart (中文)](../archive-blog/2025-11-26/2025-11-26-jobset-in-place-restart_zh.md)

关于 JobSet 利用 Kubernetes In-Place Container Restart（共同演进主题）的博客：

- 共同演进概念：Kubernetes 功能赋能生态系统
- In-Place Container Restart 能力（1.34 中的 KEP-5307，1.35 中的 KEP-5532）
- 实际效果：在 5000 节点上重启时间从 2m10s 降至 10s（加速 92%）
- 对分布式训练、作业依赖和资源效率的好处
- 集成考量和未来路线图

## 2025-11-26：cgroup v2 Migration Guide

- [cgroup v2 Migration Guide (English)](../archive-blog/2025-11-26/2025-11-26-cgroup-v2.md)
- [cgroup v2 Migration Guide (中文)](../archive-blog/2025-11-26/2025-11-26-cgroup-v2_zh.md)

面向 Kubernetes 用户的 cgroup v2 迁移全面指南：

- Kubernetes 1.31 维护模式和 1.35 弃用公告
- cgroup v1 与 v2 的区别和技术改进
- 历史时间线和内核/控制器演进
- cgroup v2 层级结构和控制器详情（CPU、内存、IO、PSI）
- 迁移指南及 runc (1.3.2+) 和 crun (1.23+) 推荐
- cgroup v1 环境的 kubeadm 升级警告

## 2025-11-25：Topology-Aware Scheduling

- [Topology-Aware Scheduling (English)](../archive-blog/2025-11-25/2025-11-25-topology-aware-scheduling.md)
- [Topology-Aware Scheduling (中文)](../archive-blog/2025-11-25/2025-11-25-topology-aware-scheduling_zh.md)

面向 AI 工作负载的拓扑感知调度全面指南：

- 当前拓扑调度背景 (Device Plugin, Kueue, Volcano)
- GPU + NIC 协调的 DRA 拓扑管理
- Kubernetes 1.34 中的 DRAConsumableCapacity 特性
- 从 Device Plugin 迁移到 DRA 的挑战
- KubeCon NA 2025 见解和资源

## 2025-11-25：Gang Scheduling

- [Gang Scheduling Blog (English)](../archive-blog/2025-11-25/2025-11-25-gang-scheduling.md)
- [Gang Scheduling Blog (中文)](../archive-blog/2025-11-25/2025-11-25-gang-scheduling_zh.md)
- [上游：Kubernetes v1.35 WAS 介绍](https://kubernetes.io/blog/2025/12/29/kubernetes-v1-35-introducing-workload-aware-scheduling/)

Kubernetes v1.35 中到来的 Gang 调度和工作负载感知调度的全面概述：

- Workload API (Alpha)
- Gang Scheduling (Alpha)
- Opportunistic Batching (Beta)
- Kubernetes 1.36 路线图
- AI/ML 工作负载的实际用例

## 贡献

要添加新博客文章：

1. 创建以日期命名的目录：`YYYY-MM-DD/`
2. 添加博客文章为 `YYYY-MM-DD-topic-name.md`（英文）
3. 可选添加带后缀的翻译版本，如 `YYYY-MM-DD-topic-name_zh.md`
4. 遵循现有文章使用的元数据格式
5. 确保所有 markdown 通过 `markdownlint` 验证
6. 更新此 README，添加文章链接
