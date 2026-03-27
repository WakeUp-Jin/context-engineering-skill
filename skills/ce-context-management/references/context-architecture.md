# 上下文架构设计

## 核心概念：内部状态 vs LLM 消息格式

上下文管理系统有两层数据结构，它们之间有明确的转换边界：

```
内部状态层                              LLM API 层
┌─────────────────┐                  ┌──────────────────┐
│ ContextItem     │  to_message()    │ Message dict     │
│ (role, content, │ ──────────────→  │ {"role": "...",  │
│  source, priority,│                │  "content": "..."}│
│  thinking, usage,│                 │                  │
│  tool_calls, ...)│                 │                  │
└─────────────────┘                  └──────────────────┘
│ to_dict()                            ↑
▼                                      │ 只在最终组装时转换
┌─────────────────┐                    │
│ JSONL 持久化     │                    │
│ (全量元数据)     │                    │
└─────────────────┘
```

**ContextItem** 是内部流转的数据结构，比 message dict 更丰富——它携带 source（来源标记）、priority（压缩优先级）、thinking（模型推理链）、usage（token 用量和成本）等元数据。

**Message dict** 只是 LLM API 认识的格式（role/content/tool_calls），是最终输出，不是内部结构。

**ContextItem 提供两种序列化**：
- `to_message()`：转为 LLM API 格式（丢弃内部元数据）
- `to_dict()`：转为持久化格式（保留所有元数据）

## 内部数据结构

### ContextItem：对话消息的内部表示

```typescript
interface ContextItem {
  // LLM API 也需要的字段
  role: string;                    // "user" | "assistant" | "tool"
  content: string | null;
  tool_calls: ToolCall[];          // assistant 发起的工具调用
  tool_call_id: string | null;     // tool 响应对应的调用 ID
  name: string | null;

  // 内部元数据（不发给 LLM）
  source: string;                  // 来源标记："user" | "llm" | "tool" | "summary"
  priority: MessagePriority;       // 压缩优先级
  created_at: number;              // 时间戳
  metadata: Record<string, any>;

  // 模型推理链（部分 API 需要回传）
  thinking: string | null;
  thinking_token_estimate: number;

  // token 用量（仅 assistant 消息有值）
  usage: ItemUsage;
}

// 转换方法
to_message(): MessageDict     // → LLM API 格式
to_dict(): Record             // → 持久化格式（JSONL）
from_message(msg): ContextItem // ← 从 message dict 创建
from_dict(d): ContextItem      // ← 从持久化数据恢复
```

### SystemPart：带身份标识的 system 内容

每个需要注入 system message 的模块，产出 `SystemPart` 而不是直接写 `{"role": "system", "content": "..."}`：

```typescript
interface SystemPart {
  tag: string;          // XML 标签名，如 "long_term_memory"
  description: string;  // 描述文字，如 "以下是你对用户的长期记忆"
  content: string;      // 实际内容
}
```

`SystemPart.render()` 渲染为带 XML 标签的文本：

```typescript
render(): string {
  if (this.description) {
    return `<${this.tag} description="${this.description}">\n${this.content}\n</${this.tag}>`;
  }
  return `<${this.tag}>\n${this.content}\n</${this.tag}>`;
}
```

渲染效果：

```xml
<system_prompt>
你是一个 AI 助手...
</system_prompt>

<long_term_memory description="以下是你对用户的长期记忆，请基于这些信息个性化回复">
用户偏好：中文回复...
</long_term_memory>

<conversation_summary description="以下是之前对话的压缩摘要">
之前讨论了文档创建...
</conversation_summary>
```

XML 标签的作用：
1. LLM 能清晰识别每段内容的边界和含义
2. description 属性提供行为指引（"请基于这些信息个性化回复"）
3. 调试时一眼就能看出哪段是什么来源

### ContextParts：投递目标容器

每个上下文模块的 `format()` 返回 `ContextParts`，声明内容投递到哪里：

```typescript
interface ContextParts {
  system_parts: SystemPart[];   // 合并进唯一的 system message
  message_items: ContextItem[]; // 作为独立 message 放入对话列表
}
```

这是概念和实现分离的关键——模块不需要知道 LLM API 的 message 格式，只需要说"我的内容放到 system 里"或"我的内容放到 messages 列表里"。

## BaseContext 抽象基类

所有上下文模块继承自 `BaseContext<T>`，泛型 `T` 表示单条存储项的类型：

```typescript
abstract class BaseContext<T> {
  protected items: T[] = [];

  add(item: T): void;
  get(index: number): T | undefined;
  getAll(): T[];
  clear(): void;
  removeLast(): T | undefined;
  replace(items: T[]): void;
  slice(start: number, end?: number): T[];
  getCount(): number;
  isEmpty(): boolean;

  /** 子类返回 ContextParts，声明内容投递到 system 还是 messages */
  abstract format(): ContextParts;
}
```

子类对应关系：

| 子类 | T 类型 | 用途 |
|------|--------|------|
| SystemPromptContext | `PromptSegment` | 系统提示词（支持分段注册） |
| ShortTermMemoryContext | `ContextItem` | 对话消息流 + 压缩摘要 |
| LongTermMemoryContext | `string` | 持久化用户记忆 |

注意：`format()` 的返回值是 `ContextParts`，不是 `Message[]`。模块不直接产出 LLM message。

## ContextManager

统一管理器，协调各模块并负责最终的 message 数组组装。

### 核心方法：get_context()

```typescript
get_context(): MessageDict[] {
  // 1. 从各模块收集 ContextParts
  const [systemParts, messageItems] = this.collectParts();

  // 2. 合并所有 system_parts 为一条 system message
  const messages: MessageDict[] = [];
  if (systemParts.length > 0) {
    const rendered = systemParts
      .filter(p => p.content.trim())
      .map(p => p.render())
      .join("\n\n");
    messages.push({ role: "system", content: rendered });
  }

  // 3. 将 message_items 逐个转为 message dict
  for (const item of messageItems) {
    messages.push(item.toMessage());
  }

  // 4. 最终清理（tool_calls 配对验证）
  return sanitizeMessages(messages);
}
```

### 收集各模块的 ContextParts

```typescript
private collectParts(): [SystemPart[], ContextItem[]] {
  const allSystem: SystemPart[] = [];
  const allMessages: ContextItem[] = [];

  // 收集顺序决定了 system message 内部的段落顺序
  for (const parts of [
    this.systemPrompt.format(),       // 1. 核心指令（最前面）
    this.longTermMemory?.format(),     // 2. 用户记忆
    this.shortTermMemory.format(),     // 3. 压缩摘要 + 对话消息
  ]) {
    if (parts) {
      allSystem.push(...parts.system_parts);
      allMessages.push(...parts.message_items);
    }
  }

  return [allSystem, allMessages];
}
```

### 为什么不直接让每个模块返回 Message[]？

如果每个模块直接返回 `Message[]`，就会出现多个 `role: "system"` 消息。这有三个问题：
1. 部分模型不支持多条 system 消息，会报错或忽略
2. 多条 system 消息的内容在语义上是割裂的
3. prefix caching 要求 system 消息内容稳定且连续，多条会降低缓存命中率

通过 ContextParts 投递目标机制，所有 system 级内容在最后一步才合并为一条消息。

## Token 使用率计算

用于判断是否需要触发压缩（在发送给 LLM 之前预判）：

```typescript
estimateTokens(): number {
  const items = this.buildContextItems();
  return this.tokenEstimator.estimateItems(items);
}

needsCompression(): boolean {
  return this.shortTermMemory.estimateTokens()
    >= this.config.contextWindow * this.config.compressionThreshold;
}
```

注意：这是粗略估算，不是计费。真正的 token 消耗从 LLM API 响应的 `usage` 字段获取。估算的唯一用途是"在调用 LLM 之前预判上下文是否快撑爆了"。

## 消息组装顺序

最终 message 数组的结构：

```
[
  {"role": "system",    "content": "<system_prompt>...</system_prompt>\n\n<long_term_memory>...</long_term_memory>\n\n<conversation_summary>...</conversation_summary>"},
  {"role": "user",      "content": "..."},
  {"role": "assistant", "content": "...", "tool_calls": [...]},
  {"role": "tool",      "tool_call_id": "...", "content": "..."},
  {"role": "assistant", "content": "..."},
  {"role": "user",      "content": "..."},
  ...
]
```

1. **system**（1 条）：所有 system_parts 合并，核心指令在最前面，LLM 视为最高优先级
2. **对话消息**（N 条）：按时间顺序排列的 user/assistant/tool 消息

这个顺序确保 LLM 先理解规则和背景，再处理对话历史和当前任务。
