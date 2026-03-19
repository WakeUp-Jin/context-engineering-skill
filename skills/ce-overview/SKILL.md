---
name: ce-overview
description: |
  上下文工程架构总览 - 基于上下文工程构建 Agent 后端的整体设计方法论。
  涵盖四大核心模块（工具管理、上下文管理、LLM模块、Agent形态）的协作关系、
  执行循环、推荐目录结构与渐进式增长路径。
  当用户需要从零开始搭建 Agent 后端、理解 Agent 各模块如何协作、
  选择项目结构、或进行整体架构设计时使用此 Skill。
  即使用户没有明确提到"上下文工程"或"架构总览"，只要涉及 Agent 后端开发、
  AI 应用架构设计、LLM 应用整体规划、搭建 Agent 框架等话题，也应主动使用此 Skill。
  注意：此 Skill 提供全局视角，具体模块的实现细节请参阅对应的专项 Skill。
---

# 架构总览：基于上下文工程的 Agent 后端设计

## 核心理念

整个后端架构按上下文工程来设计，**上下文管理是核心**——它是一个完整的大输入，以此输入作为前提来解决用户的问题。

**开发重心在上下文的获取和整理，应用的关键核心是 LLM**。保证核心是 LLM，后续模型能力提升时 Agent 效果自然变好；开发重心在上下文，可以发挥应用开发者的能力和创造力。

为什么这样拆分？因为模型能力在快速迭代，如果把大量业务逻辑写在 prompt 里或者复杂的 chain 里，模型升级时这些都要重写。而上下文工程把"获取和整理信息"作为开发重心，模型换代时只需换 LLM 模块，其他三个模块几乎不变。

## 四大核心模块

1. **工具模块和管理**：定义 Agent 的工具（MCP 或 Tool），管理工具输出裁剪、执行审核流程
2. **上下文管理**：统一编排注入给 LLM 的上下文，处理压缩、裁剪、污染、干扰、冲突等问题
3. **LLM 模块**：Agent 的"方舟反应堆"，通过统一接口支持多种模型供应商
4. **Agent 形态（执行器）**：工作流、单智能体、多智能体等执行形态

## 模块协作关系

```
用户输入
    │
    ▼
┌─────────────────────────────────────┐
│         Agent 形态（执行器）           │
│                                     │
│   ┌───────────┐    ┌──────────┐    │
│   │ 上下文管理  │───▶│ LLM 模块  │    │
│   └───────────┘    └──────────┘    │
│         │                │          │
│         │          tool_calls       │
│         ▼                │          │
│   ┌───────────┐          │          │
│   │ 工具模块   │◀─────────┘          │
│   └───────────┘                     │
│         │                           │
│    工具结果回写到上下文                 │
└─────────────────────────────────────┘
    │
    ▼
响应输出
```

## 执行循环

1. Agent 执行器接收用户输入
2. 上下文管理器组装完整的消息序列（system + history + user + tools）
3. 将组装好的上下文输入 LLM
4. LLM 返回文本响应或 tool_calls
5. 若有 tool_calls → 工具模块执行 → 结果回写到上下文 → 回到步骤 2
6. 若无 tool_calls → 返回最终响应

## 推荐目录结构

核心骨架始终是 agent / context / llm / tool 四个目录。从最小可运行版本开始，随需求增长逐步补充能力。标注 `[进阶]` 的文件在 MVP 阶段可以先不实现。

```
src/core/
├── agent/
│   ├── Agent.ts                     # Agent 执行器（核心入口）
│   ├── AgentManager.ts              # [进阶] 多 Agent 注册与管理
│   ├── config/presets/              # [进阶] 预设配置（build/explore 等）
│   └── execution/ExecutionEngine.ts # [进阶] 独立执行引擎（从 Agent 中抽离）
├── context/
│   ├── base/BaseContext.ts          # 上下文基类
│   ├── modules/
│   │   ├── SystemPromptContext.ts   # 系统提示词
│   │   ├── HistoryContext.ts        # 会话历史（含压缩）
│   │   └── CurrentTurnContext.ts    # 当前轮次（含归档）
│   ├── utils/
│   │   ├── tokenEstimator.ts        # [进阶] Token 估算
│   │   ├── messageSanitizer.ts      # [进阶] 消息验证与清理
│   │   └── ToolOutputSummarizer.ts  # [进阶] 工具输出摘要
│   ├── ContextManager.ts           # 统一管理器（含自动压缩）
│   └── types.ts
├── llm/
│   ├── services/                    # 供应商实现（至少一个）
│   │   └── DeepSeekService.ts
│   ├── types/index.ts
│   ├── factory.ts                   # LLM 工厂函数
│   └── LLMServiceRegistry.ts       # [进阶] 按 ModelTier 分层缓存
├── tool/
│   ├── [具体工具]/                   # 每个工具一个目录
│   │   ├── definitions.ts           #   工具定义
│   │   └── executors.ts             #   执行函数 + 输出格式化
│   ├── ToolManager.ts              # 工具注册与查询
│   ├── ToolScheduler.ts            # [进阶] 工具调度（含审核、状态机）
│   ├── Allowlist.ts                # [进阶] 审核白名单
│   └── types.ts
├── promptManager/                   # 提示词管理
├── session/                         # [进阶] 会话管理
└── stats/                           # [进阶] 统计与计费
```

**增长路径**：Agent.ts 跑通工具循环 → 加 ContextManager 管理消息 → 加 ToolScheduler 控制审核 → 加 tokenEstimator 和压缩 → 加 AgentManager 支持多 Agent → 加 session/stats 做生产化。

## 实现步骤指引

当帮用户搭建 Agent 后端时，按以下顺序推进：

1. **先搭 LLM 模块**：这是最底层的依赖。先确保能调通一个模型供应商（参阅 ce-llm-module）
2. **再搭上下文管理**：实现 BaseContext 和 ContextManager，组装消息序列（参阅 ce-context-management）
3. **然后建工具系统**：定义工具、注册到 ToolManager、实现执行循环（参阅 ce-tool-management）
4. **实现 Agent 执行器**：将上面三个模块串起来，实现 Agent 类跑通工具循环（参阅 ce-agent-patterns）
5. **按需补充进阶能力**：审核（ToolScheduler）、压缩（tokenEstimator）、多 Agent（AgentManager）等标注 `[进阶]` 的部分，遇到需求时再加
6. **持续用评估驱动优化**：建立评估体系，用数据而非直觉来改进（参阅 ce-evaluation）

## 相关 Skills

| 模块 | Skill | 说明 |
|------|-------|------|
| 工具管理 | ce-tool-management | 工具的完整生命周期：定义、注册、审核、执行、输出处理 |
| 上下文管理 | ce-context-management | 上下文架构、压缩策略、Token 管理、消息清理 |
| LLM 模块 | ce-llm-module | 统一接口、工厂模式、多供应商适配 |
| Agent 形态 | ce-agent-patterns | 单/多智能体、执行引擎、交互形态 |
| Agent 评估 | ce-evaluation | 评估方法论、评估器实现、多类型 Agent 评估 |

## 参考项目

- **CLI 模板项目**：`npm create context-template`，一键生成最小可运行的脚手架
- **实践项目 ReasonCode**：完整的生产级实现，包含工具审核、执行调度、上下文压缩等全部进阶能力
