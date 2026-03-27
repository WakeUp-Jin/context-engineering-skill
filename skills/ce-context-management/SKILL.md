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

## 关键设计原则：内部状态与 LLM 消息格式分离

上下文模块的内部数据结构和发送给 LLM 的 message 数组是**两套不同的东西**，必须在概念和实现上分开。

**内部状态**：每个上下文模块有自己的数据结构（ContextItem、PromptSegment、纯文本等），用于存储、压缩、持久化。这是模块自己的"流转状态"。

**LLM 消息格式**：最终发给 LLM 的 `Message[]` 数组（role/content/tool_calls）。这是 API 协议，不是内部数据结构。

模块不应直接操作 message 格式，而是通过 `format()` 方法声明"我的内容投递到哪里"，由 ContextManager 统一转换。

## 投递目标：只有两种

概念上可以设计任意多种上下文模块，但最终拼接到 message 数组时，所有内容只有两个去向：

| 投递目标 | 含义 | 最终效果 |
|---------|------|---------|
| **system** | 合并进唯一的 system message | 多个模块的 system 内容合并为一条 `{"role": "system", "content": "..."}` |
| **messages** | 放入对话消息列表 | 作为独立的 user/assistant/tool 消息排入数组 |

各模块的 `format()` 返回 `ContextParts`，声明内容投递到 system 还是 messages：

```
ContextParts {
  system_parts: SystemPart[]   → 合并为 1 条 system message
  message_items: ContextItem[] → 逐个转为对话 message
}
```

## 上下文模块

| 模块 | 内部存储类型 | 投递到 system | 投递到 messages |
|------|------------|--------------|----------------|
| SystemPromptContext | PromptSegment（带优先级的文本片段） | 核心提示词 + 动态 segment | — |
| LongTermMemoryContext | string（记忆文本） | 用户记忆 | — |
| ShortTermMemoryContext | ContextItem（对话消息） | 压缩摘要 | 对话消息（user/assistant/tool） |

更多模块可以按需扩展（RAG 检索结果、结构化输出约束等），每个新模块只需决定内容投递到 system 还是 messages。

## 推荐目录结构

```
context/
├── base.py                     # BaseContext 抽象基类
├── types.py                    # ContextItem / ContextParts / SystemPart 等类型
├── manager.py                  # ContextManager 统一管理器
├── modules/
│   ├── system_prompt.py        # 系统提示词（分段式）
│   ├── short_term_memory.py    # 短期记忆（对话 + 压缩）
│   └── long_term_memory.py     # 长期记忆（持久化）
└── utils/
    ├── compressor.py           # 上下文压缩器
    ├── message_sanitizer.py    # 消息清理与验证
    └── token_estimator.py      # Token 估算
```

## 上下文组装流程

```
ContextManager.get_context()
        │
        ▼
  _collect_parts(): 从各模块收集 ContextParts
        │
  ┌─────┴─────────────────────────────────────┐
  │ SystemPromptContext.format() → ContextParts│
  │   system_parts: [SystemPart(...)]         │
  │   message_items: []                       │
  │                                           │
  │ LongTermMemoryContext.format() → ContextParts│
  │   system_parts: [SystemPart(...)]         │
  │   message_items: []                       │
  │                                           │
  │ ShortTermMemoryContext.format() → ContextParts│
  │   system_parts: [SystemPart(...)]  (摘要) │
  │   message_items: [item, item, ...]  (对话) │
  └─────┬─────────────────────────────────────┘
        │
        ▼
  合并所有 system_parts → 渲染为带 XML 标签的 1 条 system message
  转换所有 message_items → 逐个 to_message() 为对话 message
        │
        ▼
  sanitize_messages() 最终验证（tool_calls 配对检查）
        │
        ▼
  返回 [system_msg, user_msg, assistant_msg, ...] 给 LLM
```

## 实现步骤指引

当帮用户实现上下文管理时，按以下顺序推进：

1. **定义内部数据结构**：ContextItem（对话消息的内部表示）、SystemPart（带标签的 system 内容）、ContextParts（投递目标容器）（参阅 [references/context-architecture.md](references/context-architecture.md)）
2. **实现上下文基类**：BaseContext 泛型基类，format() 返回 ContextParts（参阅 [references/context-architecture.md](references/context-architecture.md)）
3. **实现各上下文模块**：各模块的 format() 声明内容投递到 system 还是 messages（参阅 [references/context-modules.md](references/context-modules.md)）
4. **构建 ContextManager**：统一管理器，_collect_parts() 收集 + get_context() 组装最终 message 数组（参阅 [references/context-architecture.md](references/context-architecture.md)）
5. **添加压缩机制**：压缩后的摘要投递到 system_parts，对话消息留在 message_items（参阅 [references/context-compression.md](references/context-compression.md) 和 [references/token-strategy.md](references/token-strategy.md)）
6. **实现消息清理**：sanitize_messages 保证最终 message 数组中 tool_calls 和 tool 响应配对（参阅 [references/message-sanitizer.md](references/message-sanitizer.md)）
7. **防御上下文问题**：上线后关注污染、干扰、冲突、混淆四类问题（参阅 [references/context-problems.md](references/context-problems.md)）

## Gotchas

- `format()` 返回的是 `ContextParts`（投递目标声明），不是 `Message[]`。模块不应该知道 LLM API 的 message 格式。
- 所有 system 内容最终合并为**一条** system message。多条 system message 会导致部分模型报错或忽略后续 system 消息。
- system_parts 中的 `SystemPart` 带 XML 标签和 description，用于标注每段内容的身份和用途。没有标注的内容在调试时难以辨认。
- 压缩摘要是 system 级别的背景信息，投递到 system_parts 而不是 message_items。它不是一条独立的对话消息。
- ContextItem 有 `to_message()` 转换方法（转为 LLM API 格式）和 `to_dict()` 持久化方法（保留所有元数据），这两个用途不同，不要混用。

## 细分文档

按需阅读 `references/` 目录下的详细文档：

| 你的需求 | 阅读文档 |
|---------|---------|
| 设计上下文的整体架构和数据结构 | [references/context-architecture.md](references/context-architecture.md) |
| 了解各上下文模块的实现细节 | [references/context-modules.md](references/context-modules.md) |
| 实现上下文压缩机制 | [references/context-compression.md](references/context-compression.md) |
| 设计 Token 压缩策略 | [references/token-strategy.md](references/token-strategy.md) |
| 解决上下文问题（污染/干扰/冲突/混淆） | [references/context-problems.md](references/context-problems.md) |
| 实现消息清理和验证 | [references/message-sanitizer.md](references/message-sanitizer.md) |
