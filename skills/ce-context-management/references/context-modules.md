# 各上下文类详解

## SystemPromptContext

存储系统提示词，支持多片段拼接：

```typescript
class SystemPromptContext extends BaseContext<string> {
  /** 设置系统提示词（替换所有） */
  setPrompt(prompt: string): void {
    this.clear();
    this.add(prompt);
  }

  /** 获取合并后的完整提示词 */
  getPrompt(): string {
    return this.getAll().join('\n\n');
  }

  /** 格式化为一条 system 消息 */
  format(): Message[] {
    const prompt = this.getPrompt();
    if (!prompt) return [];
    return [{ role: 'system', content: prompt }];
  }
}
```

系统提示词可以分片段添加，最终合并为一条 `role: 'system'` 消息。这种设计允许不同模块各自追加系统指令。

## HistoryContext（含压缩机制）

存储会话历史消息，是最复杂的上下文类——内含压缩逻辑：

```typescript
class HistoryContext extends BaseContext<Message> {
  private tokenEstimator: TokenEstimator;

  /** 格式化为消息数组（直接返回历史） */
  format(): Message[] {
    return this.getAll();
  }
}
```

### 压缩流程

当历史过长触发压缩时：

```typescript
async compress(): Promise<CompressionResult> {
  const messages = this.getAll();

  // 1. 历史太短不压缩
  if (messages.length < 4) {
    return { compressed: false, reason: 'too_few_messages' };
  }

  // 2. 找到分割点：保留最近约 30% 的历史
  const splitIndex = this.findSplitPoint(messages, 0.3);

  // 3. 调整分割点：不能在 tool 调用中间截断
  const adjustedIndex = this.adjustForToolCalls(messages, splitIndex);

  // 4. 分割为"要压缩的"和"要保留的"
  const toCompress = messages.slice(0, adjustedIndex);
  const toKeep = messages.slice(adjustedIndex);

  // 5. 使用 LLM 生成摘要
  const summary = await this.generateSummary(toCompress);

  // 6. 提取文件引用（在摘要中保留）
  const files = this.extractRetainedFiles(toCompress);

  // 7. 替换历史：[摘要消息, ...保留的历史]
  const summaryMessage: Message = {
    role: 'system',
    content: `[对话历史摘要]\n${summary}\n\n[历史中涉及的文件]: ${files.join(', ')}`,
  };

  this.replace([summaryMessage, ...toKeep]);

  return { compressed: true, removedCount: toCompress.length };
}
```

### 分割点计算

从后往前累加 token，直到达到保留比例：

```typescript
private findSplitPoint(messages: Message[], preserveRatio: number): number {
  const totalTokens = this.tokenEstimator.estimateMessages(messages);
  const preserveTokens = totalTokens * preserveRatio;

  let accumulated = 0;
  for (let i = messages.length - 1; i >= 0; i--) {
    accumulated += this.tokenEstimator.estimateMessages([messages[i]]);
    if (accumulated >= preserveTokens) {
      return i;
    }
  }
  return 0;
}
```

### 工具调用边界调整

不能在 tool_calls 和对应 tool 响应之间截断：

```typescript
private adjustForToolCalls(messages: Message[], index: number): number {
  // 如果 index 处的消息是 tool 响应，向前移动到对应的 assistant 消息之前
  while (index > 0 && messages[index].role === 'tool') {
    index--;
  }
  // 如果 index 处是带 tool_calls 的 assistant，也需要包含后续的 tool 响应
  if (messages[index].role === 'assistant' && messages[index].tool_calls) {
    // 找到所有对应 tool 响应的结束位置
    const toolCallIds = new Set(messages[index].tool_calls.map(tc => tc.id));
    let endIndex = index + 1;
    while (endIndex < messages.length
      && messages[endIndex].role === 'tool'
      && toolCallIds.has(messages[endIndex].tool_call_id)) {
      endIndex++;
    }
    index = endIndex;
  }
  return index;
}
```

## CurrentTurnContext

管理当前对话轮次的消息（工具调用和响应）：

```typescript
class CurrentTurnContext extends BaseContext<Message> {
  format(): Message[] {
    return this.getAll();
  }

  /** 将当前轮次归档到历史 */
  archiveTo(history: HistoryContext, userInput: string): void {
    // 先归档用户输入
    history.add({ role: 'user', content: userInput });
    // 再归档当前轮次的所有消息
    for (const msg of this.getAll()) {
      history.add(msg);
    }
    // 清空当前轮次
    this.clear();
  }

  /** 检查是否有 tool 调用 */
  hasToolCalls(): boolean {
    return this.getAll().some(
      msg => msg.role === 'assistant' && msg.tool_calls?.length
    );
  }

  /** 检查是否有未完成的 tool 调用 */
  hasPendingToolCalls(): boolean {
    const assistantMsgs = this.getAll().filter(
      msg => msg.role === 'assistant' && msg.tool_calls?.length
    );
    const toolResponses = this.getAll().filter(msg => msg.role === 'tool');
    const responseIds = new Set(toolResponses.map(m => m.tool_call_id));

    return assistantMsgs.some(msg =>
      msg.tool_calls?.some(tc => !responseIds.has(tc.id))
    );
  }

  /** 获取最后一条 assistant 消息 */
  getLastAssistantMessage(): Message | undefined {
    const all = this.getAll();
    for (let i = all.length - 1; i >= 0; i--) {
      if (all[i].role === 'assistant') return all[i];
    }
    return undefined;
  }

  /** 中断场景下的清理 */
  sanitize(): void {
    const cleaned = sanitizeCurrentTurn(this.getAll());
    this.replace(cleaned);
  }
}
```

## 基础版额外的上下文类型

CLI 模板项目提供了更多上下文类型，适合不同场景：

### ConversationContext

```typescript
class ConversationContext extends BaseContext<Message> {
  // 纯对话历史，不含压缩逻辑
}
```

### MemoryContext

```typescript
class MemoryContext extends BaseContext<MemoryEntry> {
  // 用户记忆（key/value），支持跨会话持久化
  set(key: string, value: string): void;
  get(key: string): string | undefined;
}
```

### StructuredOutputContext

```typescript
class StructuredOutputContext extends BaseContext<OutputSchema> {
  // 结构化输出约束，如 JSON Schema
  // 注入到 system prompt 中指导 LLM 输出格式
}
```

### RelevantContext

```typescript
class RelevantContext extends BaseContext<ContextChunk> {
  // 场景相关上下文（如 RAG 检索结果）
}
```

### ToolMessageSequenceContext

```typescript
class ToolMessageSequenceContext extends BaseContext<Message> {
  // 工具调用链：assistant → tool → assistant → tool...
  // 维护 tool_calls 和 tool 响应的配对关系
}
```

## 选择建议

- **入门项目**：使用基础版的 7 种上下文，结构清晰、职责明确
- **生产项目**：使用进阶版的 3 种上下文（System/History/CurrentTurn），更精简、性能更好
- 进阶版将 Memory、StructuredOutput 等融入 SystemPromptContext，将工具消息融入 CurrentTurnContext
