---
name: ce-tool-management
description: |
  Agent 工具管理模块 - 覆盖工具的完整生命周期：定义、注册、审核、执行、输出处理。
  包含工具定义规范（InternalTool接口/三层描述/JSON Schema参数）、
  工具注册与发现（ToolManager/动态注册/按Agent过滤）、
  审核设计（ApprovalMode三级模式/Allowlist白名单/确认UI）、
  执行状态机（ToolScheduler/6状态流转/并行判断/deepParseArgs）、
  输出压缩（renderResultForAssistant/ToolOutputSummarizer/截断与LLM摘要）、
  具体工具实现参考（ReadFile/Grep/WriteFile/Task等9种工具）。
  当用户需要为 Agent 定义工具接口、注册管理工具、设计审核机制、
  管理工具执行状态、处理工具输出格式化与压缩、实现具体工具时使用此 Skill。
  即使用户没有明确提到"工具管理"，只要涉及 function calling、tool_calls 处理、
  MCP 工具设计、工具执行安全控制、工具输出太长需要压缩、或者需要实现具体的
  读取/搜索/写入等工具，也应主动使用此 Skill。
---

# 工具管理模块

工具是 LLM 关键的外部能力。MCP 和 Tool 本质一样——给模型提供"外部能力"。工具管理覆盖工具的完整生命周期：定义 → 注册 → 审核 → 执行 → 输出处理。

为什么需要这么完整的生命周期管理？因为实际生产环境中，工具不只是"调一个函数"那么简单——需要审核防止误操作、需要状态追踪方便调试、需要输出压缩避免上下文膨胀、需要并行控制保证数据一致性。省掉任何一环都会在规模扩大后遇到问题。

## 三个核心要素

1. **工具的定义**：定义一个工具对象需要哪些属性
2. **提供给 LLM 的工具参数**：可以是 tools 参数，也可以是提示词文本
3. **工具的执行函数**：大模型输出工具调用指令后，系统执行该函数

## 推荐目录结构

```
tool/
├── ReadFile/              # 每个工具一个目录
│   ├── definitions.ts     # 工具定义（名称、描述、参数 Schema）
│   └── executors.ts       # 工具执行函数 + 输出格式化
├── Grep/
│   ├── definitions.ts
│   ├── executors.ts
│   └── strategies/        # 多策略实现（如搜索工具）
├── ToolManager.ts         # 工具注册、查询、执行
├── ToolScheduler.ts       # 工具调度、审核、状态管理
├── Allowlist.ts           # 会话级审核白名单
├── types.ts               # 工具模块类型定义
├── utils/                 # 工具辅助（进程管理、错误处理等）
└── index.ts
```

## 工具生命周期流程

```
定义工具 → 注册到 ToolManager → Agent 请求执行
                                      │
                                      ▼
                              ToolScheduler 接管
                                      │
                            ┌─────────┴─────────┐
                            ▼                   ▼
                      需要审核？            不需要审核
                            │                   │
                     等待用户确认               直接执行
                            │                   │
                            └─────────┬─────────┘
                                      ▼
                              执行 handler 函数
                                      │
                                      ▼
                            输出处理（格式化/压缩）
                                      │
                                      ▼
                            结果回写到上下文
```

## 实现步骤指引

当帮用户实现工具系统时，按以下顺序推进：

1. **定义工具接口**：确定 InternalTool 接口和每个工具的参数 Schema（参阅 [references/tool-definition.md](references/tool-definition.md)）
2. **实现 ToolManager**：建立工具注册中心，支持按名称查询和 OpenAI 格式输出（参阅 [references/tool-registry.md](references/tool-registry.md)）
3. **添加审核层**：根据安全需求选择 ApprovalMode，实现 shouldConfirmExecute 和 Allowlist（参阅 [references/tool-approval.md](references/tool-approval.md)）
4. **实现 ToolScheduler**：管理执行状态机、并行判断、与 ExecutionStream 协作（参阅 [references/tool-execution.md](references/tool-execution.md)）
5. **处理输出**：实现 renderResultForAssistant 和 ToolOutputSummarizer（参阅 [references/tool-output.md](references/tool-output.md)）
6. **实现具体工具**：ReadFile、Grep、WriteFile 等，每个工具一个目录（参阅 [references/tool-implementations.md](references/tool-implementations.md)）

## 细分文档

按需阅读 `references/` 目录下的详细文档：

| 你的需求 | 阅读文档 |
|---------|---------|
| 如何定义一个工具的接口和参数 | [references/tool-definition.md](references/tool-definition.md) |
| 如何注册、管理和发现工具 | [references/tool-registry.md](references/tool-registry.md) |
| 如何设计工具执行的审核机制 | [references/tool-approval.md](references/tool-approval.md) |
| 如何管理工具执行状态和调度 | [references/tool-execution.md](references/tool-execution.md) |
| 如何处理工具输出（格式化、压缩） | [references/tool-output.md](references/tool-output.md) |
| 各类工具的具体实现参考 | [references/tool-implementations.md](references/tool-implementations.md) |
