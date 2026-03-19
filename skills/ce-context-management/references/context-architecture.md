# 上下文架构设计

## BaseContext 抽象基类

所有上下文模块继承自 `BaseContext<T>`，泛型 `T` 表示单条存储项的类型：

```typescript
abstract class BaseContext<T> {
  protected items: T[] = [];

  add(item: T): void {
    this.items.push(item);
  }

  get(index: number): T | undefined {
    return this.items[index];
  }

  getAll(): T[] {
    return [...this.items];
  }

  clear(): void {
    this.items = [];
  }

  removeLast(): T | undefined {
    return this.items.pop();
  }

  replace(items: T[]): void {
    this.items = [...items];
  }

  slice(start: number, end?: number): T[] {
    return this.items.slice(start, end);
  }

  getCount(): number {
    return this.items.length;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  /** 子类必须实现：将存储项格式化为 Message[] */
  abstract format(): Message[];
}
```

子类对应关系：

| 子类 | T 类型 | 用途 |
|------|--------|------|
| SystemPromptContext | `string` | 系统提示文本片段 |
| HistoryContext | `Message` | 历史消息 |
| CurrentTurnContext | `Message` | 当前轮次消息 |

## ContextManager

统一管理器，协调三类上下文并负责压缩控制：

```typescript
class ContextManager {
  private systemPrompt: SystemPromptContext;
  private history: HistoryContext;
  private currentTurn: CurrentTurnContext;

  /** 压缩配置 */
  private modelLimit: number = 64000;
  private compressionThreshold: number = 0.7;  // 70% 触发压缩
  private overflowThreshold: number = 0.95;    // 95% 视为溢出

  /** 配置压缩参数 */
  configureCompression(config: CompressionConfig): void {
    this.modelLimit = config.modelLimit ?? this.modelLimit;
    this.compressionThreshold = config.compressionThreshold ?? this.compressionThreshold;
    this.overflowThreshold = config.overflowThreshold ?? this.overflowThreshold;
  }

  /** 是否需要压缩 */
  needsCompression(): boolean {
    const usage = this.getTokenUsage();
    return usage.ratio >= this.compressionThreshold;
  }

  /** 是否已溢出 */
  isOverflow(): boolean {
    const usage = this.getTokenUsage();
    return usage.ratio >= this.overflowThreshold;
  }
}
```

### 核心方法：getContext

组装供 LLM 使用的完整消息序列：

```typescript
async getContext(autoCompress: boolean = true): Promise<Message[]> {
  // 自动压缩检查
  if (autoCompress && this.needsCompression()) {
    await this.history.compress();
    // 发送压缩事件通知 UI
    this.executionStream?.emitCompression({ type: 'complete' });
  }

  return this.buildMessages();
}

private buildMessages(): Message[] {
  const messages: Message[] = [];

  // 1. 系统提示词
  messages.push(...this.systemPrompt.format());

  // 2. 会话历史
  messages.push(...this.history.format());

  // 3. 当前用户输入（如果有）
  if (this.currentUserInput) {
    messages.push({ role: 'user', content: this.currentUserInput });
  }

  // 4. 当前轮次
  messages.push(...this.currentTurn.format());

  // 5. 消息清理：保证 tool_calls 和 tool 响应配对
  return sanitizeMessages(messages);
}
```

### Token 使用率计算

```typescript
getTokenUsage(): TokenUsage {
  const allMessages = this.buildMessages();
  const totalTokens = this.tokenEstimator.estimateMessages(allMessages);
  return {
    used: totalTokens,
    limit: this.modelLimit,
    ratio: totalTokens / this.modelLimit,
  };
}
```

### 文件引用提取

从历史中提取工具涉及的文件路径，在压缩摘要中保留：

```typescript
extractRetainedFiles(): string[] {
  const files = new Set<string>();
  for (const msg of this.history.getAll()) {
    if (msg.role === 'tool' && msg.name) {
      // 从 Read/Write/Edit 等工具的参数和结果中提取文件路径
      const paths = extractFilePaths(msg.content);
      paths.forEach(p => files.add(p));
    }
  }
  return Array.from(files);
}
```

## 消息组装顺序的重要性

消息序列的顺序直接影响 LLM 的理解：

1. **system**（最前）：规则和角色设定，LLM 视为最高优先级指令
2. **history**（中间）：过去的对话记录，提供上下文连续性
3. **user**（靠后）：当前用户输入，LLM 的注意力重点
4. **currentTurn**（最后）：工具调用结果等，最新的信息

这个顺序确保 LLM 先理解规则，再回顾历史，最后关注当前任务。

## 基础版 vs 进阶版对比

| 特性 | 基础版（CLI 模板） | 进阶版（reason-code） |
|------|-------------------|---------------------|
| 上下文类型 | 7 种 | 3 种（精简） |
| 压缩机制 | 无 | 自动压缩 + 阈值控制 |
| 消息清理 | 无 | sanitizeMessages |
| Token 估算 | 无 | TokenEstimator |
| 工具输出处理 | 无 | ToolOutputSummarizer |
| 溢出检测 | 无 | isOverflow() |
