---
status: Active
maintainer: pacoxu
last_updated: 2026-04-14
tags: inference, kubernetes, agentic, networking, governance
canonical_path: zh/docs/inference/kube-agentic-networking.md
---

# kube-agentic-networking：Kubernetes 上的 Agentic 网络治理

[`kube-agentic-networking`](https://github.com/kubernetes-sigs/kube-agentic-networking) 是一个 Kubernetes SIG 项目，专注于 Agent 工作负载的网络策略和治理。在实践中，它针对 Agent 平台中的一个常见问题：Agent 必须调用许多工具和 API，但平台团队仍需要清晰的边界、最小权限控制和可审计性。

## 为何重要

Agent 系统与传统微服务不同：

- **动态出站模式**：工具调用通常在运行时决定，而非在部署时固定
- **跨域访问**：一个工作流可能触及模型端点、MCP 工具、外部 API 和内部服务
- **治理要求**：企业环境需要对 Agent 行为进行策略控制、可追溯性和租户隔离

`kube-agentic-networking` 通过将 Agent 感知的网络治理引入 Kubernetes 原生工作流来解决这些需求。

## 在 AI 基础设施栈中的角色

在本仓库的 AI 基础设施模型中，`kube-agentic-networking` 位于以下之间：

- **Agent 运行时/编排器**（Dify、AgentScope、LangGraph 风格系统）
- **网关和路由层**（AI 网关、推理路由）
- **工具后端和外部 API**（MCP 服务器、内部平台、SaaS）

它通过关注网络策略边界和治理态势来补充 API 网关和 Agent 协议（例如 MCP/A2A）。

## 典型用例

- **多租户 Agent 平台**：强制命名空间/团队边界，默认情况下一个团队的 Agent 不能到达另一个团队的内部工具
- **受控工具访问**：仅允许特定 Agent 类别的已批准工具/API 目标
- **合规和审计**：记录和审查高风险工作流的网络意图和访问模式

## 采用指南

对于生产部署，务实的顺序是：

1. 从非生产 Agent 工作流开始，定义明确的工具/API 访问清单
2. 首先对 Agent 工作负载应用最小权限网络策略
3. 在广泛推出前添加策略验证和可观测性
4. 在策略漂移和误报得到控制后，扩展到多租户和监管工作负载

## 学习主题

- Agent 到工具流量的策略设计
- Kubernetes 中的多租户边界建模
- 与 AI 网关和协议层的治理集成
- Agent 网络访问的运维可观测性

## 参考资料

- [GitHub: kubernetes-sigs/kube-agentic-networking](https://github.com/kubernetes-sigs/kube-agentic-networking)
- [Kubernetes SIGs](https://github.com/kubernetes-sigs)
