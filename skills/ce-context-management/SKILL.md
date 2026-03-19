---
name: ce-context-management
description: |
  Agent 上下文管理模块 - Agent 的核心，负责编排上下文输入给 LLM。
  包含上下文架构（BaseContext基类/ContextManager/消息组装顺序）、
  各上下文类详解（SystemPromptContext/HistoryContext含LLM压缩/CurrentTurnContext）、
  上下文压缩机制（ClaudeCode 9段式/Gemini 5点/工具消息裁剪/消息移除）、
  Token压缩策略（TokenEstimator/三种策略/消息优先级/自适应选择）、
  上下文问题与解决（污染/干扰/冲突/混淆四大问题）、
  消息清理与验证（sanitizeMessages/validateMessages/tool_calls配对）。
  当用户需要设计上下文架构、实现压缩策略、管理Token预算、
  解决上下文问题、或实现消息清理验证时使用此 Skill。
  即使用户没有明确提到"上下文管理"，只要涉及消息序列组装、对话历史压缩、
  token 预算控制、LLM 输入编排、system prompt 设计、conversation history 管理、
  tool_calls 与 tool 响应的配对验证、或者 Agent 对话变长后响应质量下降等问题，
  也应主动使用此 Skill。更长的上下文不等于更好的响应，设计重点不在"获取"而在"整理"。
---

# 上下文管理模块

上下文管理是 Agent 的核心。更长的上下文不等于更好的响应——过度加载上下文会导致 Agent 以意识不到的方式失败。上下文可能被污染、分散模型注意力、令模型困惑或产生冲突，这对 Agent 来说是"致命的危险"。

开发重心在上下文的**获取和整理**。当模型可以获取到足够的上下文时，设计重点不再是"获取"，而是"整理"——通过修剪、压缩、删除等方式，使保留的上下文既够用又高效。

为什么精简为三类上下文？因为上下文类型越多，组装和维护的复杂度就越高。实践中发现 System/History/CurrentTurn 三类足以覆盖所有场景——Memory、StructuredOutput 等可以融入 SystemPrompt，工具消息融入 CurrentTurn。三类模型既够用又容易理解和调试。

## 三类上下文

| 类型 | 实现类 | 职责 |
|------|--------|------|
| 系统提示词 | SystemPromptContext | 系统指令、规则、角色设定 |
| 会话历史 | HistoryContext | 对话历史记录，含压缩机制 |
| 当前轮次 | CurrentTurnContext | 当前对话轮次的消息序列 |

基础版（CLI 模板）额外支持更多上下文类型：ConversationContext、MemoryContext、StructuredOutputContext、RelevantContext、ExecutionHistoryContext、ToolMessageSequenceContext。

## 推荐目录结构

```
context/
├── base/BaseContext.ts          # 上下文基类
├── modules/
│   ├── SystemPromptContext.ts   # 系统提示词
│   ├── HistoryContext.ts        # 会话历史（含压缩）
│   └── CurrentTurnContext.ts    # 当前轮次
├── utils/
│   ├── tokenEstimator.ts        # Token 估算
│   ├── messageSanitizer.ts      # 消息清理与验证
│   └── ToolOutputSummarizer.ts  # 工具输出摘要
├── ContextManager.ts            # 统一管理器
├── types.ts
└── index.ts
```

## 上下文组装流程

```
ContextManager.getContext()
        │
        ▼
  检查是否需要压缩（needsCompression）
        │
  ┌─────┴─────┐
  │           │
  是          否
  │           │
  ▼           │
history.compress()  │
  │           │
  └─────┬─────┘
        ▼
  buildMessages() 组装消息序列：
        │
  ┌─────┴─────────────────────────────┐
  │ 1. system: SystemPromptContext    │
  │ 2. history: HistoryContext        │
  │ 3. user: 当前用户输入              │
  │ 4. current: CurrentTurnContext    │
  └───────────────────────────────────┘
        │
        ▼
  sanitizeMessages() 最终验证
        │
        ▼
  返回 Message[] 给 LLM
```

## 实现步骤指引

当帮用户实现上下文管理时，按以下顺序推进：

1. **设计上下文基类**：实现 BaseContext 泛型基类，定义 add/get/format 等标准方法（参阅 [references/context-architecture.md](references/context-architecture.md)）
2. **实现三类上下文**：SystemPromptContext、HistoryContext、CurrentTurnContext 各自的 format() 和特有方法（参阅 [references/context-modules.md](references/context-modules.md)）
3. **构建 ContextManager**：统一管理器，实现 getContext() 消息组装和 needsCompression() 阈值检测（参阅 [references/context-architecture.md](references/context-architecture.md)）
4. **添加压缩机制**：在 HistoryContext 中实现 compress()，选择 LLM 摘要/工具消息裁剪/消息移除策略（参阅 [references/context-compression.md](references/context-compression.md) 和 [references/token-strategy.md](references/token-strategy.md)）
5. **实现消息清理**：sanitizeMessages 保证 tool_calls 和 tool 响应配对（参阅 [references/message-sanitizer.md](references/message-sanitizer.md)）
6. **防御上下文问题**：上线后关注污染、干扰、冲突、混淆四类问题（参阅 [references/context-problems.md](references/context-problems.md)）

## 细分文档

按需阅读 `references/` 目录下的详细文档：

| 你的需求 | 阅读文档 |
|---------|---------|
| 设计上下文的整体架构 | [references/context-architecture.md](references/context-architecture.md) |
| 了解各上下文类的实现细节 | [references/context-modules.md](references/context-modules.md) |
| 实现上下文压缩机制 | [references/context-compression.md](references/context-compression.md) |
| 设计 Token 压缩策略 | [references/token-strategy.md](references/token-strategy.md) |
| 解决上下文问题（污染/干扰/冲突/混淆） | [references/context-problems.md](references/context-problems.md) |
| 实现消息清理和验证 | [references/message-sanitizer.md](references/message-sanitizer.md) |
