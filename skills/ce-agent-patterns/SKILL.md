---
name: ce-agent-patterns
description: |
  Agent 形态模块 - 决定如何使用上下文和 LLM 来解决用户问题的执行器设计。
  包含单智能体设计（Agent类/生命周期/工具过滤/AgentConfig/四种预设）、
  多智能体设计（AgentManager/监督者/群体/层级模式/子代理规模控制）、
  执行引擎详解（ExecutionEngine主循环/工具调度/溢出控制/流式输出/中断处理）、
  协同与自主Agent交互形态（场域理论/双向信息流/意图上下文vs状态上下文）。
  当用户需要构建单智能体、设计多智能体协作、实现执行引擎、
  或理解协同Agent与自主Agent的区别时使用此 Skill。
  即使用户没有明确提到"Agent形态"，只要涉及 Agent 类设计、工具循环实现、
  主代理与子代理协作、Agent 预设配置、执行引擎的 while 循环、
  maxLoops 控制、AbortController 中断、或者 Agent 如何调度工具等话题，
  也应主动使用此 Skill。
---

# Agent 形态模块

Agent 形态是执行器——决定如何使用上下文和 LLM 来解决用户问题。可以是工作流、单智能体、多智能体等形态。

为什么单智能体优先？因为单 Agent 的上下文天然连贯——所有信息在一个上下文窗口里，不存在信息传递损耗。多 Agent 引入了上下文断裂、通信开销和协调复杂度。只有当单 Agent 确实不够用时（并行任务、超窗口信息量、工具超过 30 个），才考虑升级为多 Agent。

## 核心选择：单智能体 vs 多智能体

- **单智能体**：单 LLM + 工具 + 提示词，独立完成工作。上下文天然连贯。
- **多智能体**：多个子 Agent 协作。适合并行、大信息量、复杂工具场景，但上下文管理更复杂。

**渐进式构建建议**：小模块单智能体 → 多个单智能体 → 多智能体系统 → 再视情况升级。

## 推荐目录结构

```
agent/
├── Agent.ts                  # 统一 Agent 执行器
├── AgentManager.ts           # Agent 注册与管理
├── config/
│   ├── types.ts              # AgentConfig 类型
│   └── presets/              # 预设配置
│       ├── build.ts          # 通用编程代理
│       ├── explore.ts        # 代码库探索（子代理）
│       ├── steward.ts        # 管家模式
│       └── explanatory.ts    # 解释型代理
└── execution/
    └── ExecutionEngine.ts    # 执行引擎
```

## Agent 执行流程总览

```
用户输入
    │
    ▼
Agent.run(userInput)
    │
    ├── init()：获取 LLM 服务、设置系统提示
    │
    ▼
ExecutionEngine.execute()  ← 主循环
    │
    ├── 检查溢出 → 触发压缩
    ├── 获取上下文（ContextManager.getContext）
    ├── 调用 LLM（llmService.complete）
    │     │
    │     ├── 有 tool_calls → ToolScheduler 调度执行
    │     │                    → 结果回写上下文
    │     │                    → 继续下一轮循环
    │     │
    │     └── 无 tool_calls → 返回最终结果
    │
    └── 异常/中断处理
```

## 实现步骤指引

当帮用户实现 Agent 形态时，按以下顺序推进：

1. **设计 Agent 基类**：确定 AgentConfig 结构，实现 Agent 类的 init/run/reset 生命周期（参阅 [references/single-agent.md](references/single-agent.md)）
2. **实现 ExecutionEngine**：主循环 while + LLM 调用 + tool_calls 检测 + 工具调度（参阅 [references/execution-engine.md](references/execution-engine.md)）
3. **添加安全控制**：maxLoops 溢出保护、AbortController 中断、token 膨胀检测
4. **创建 Agent 预设**：根据场景定义 build/explore/steward 等预设配置
5. **（如需）扩展为多 Agent**：实现 AgentManager 注册中心、子 Agent 创建和上下文隔离（参阅 [references/multi-agent.md](references/multi-agent.md)）
6. **理解交互模式**：区分协同 Agent（场域理论、双向流）和自主 Agent 的设计差异（参阅 [references/agent-interaction.md](references/agent-interaction.md)）

## 细分文档

按需阅读 `references/` 目录下的详细文档：

| 你的需求 | 阅读文档 |
|---------|---------|
| 设计单智能体的 Agent 类 | [references/single-agent.md](references/single-agent.md) |
| 构建多智能体协作系统 | [references/multi-agent.md](references/multi-agent.md) |
| 实现执行引擎（工具循环） | [references/execution-engine.md](references/execution-engine.md) |
| 理解协同 Agent 和自主 Agent | [references/agent-interaction.md](references/agent-interaction.md) |
