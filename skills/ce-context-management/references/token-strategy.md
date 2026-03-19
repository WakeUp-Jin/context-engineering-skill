# Token 压缩策略

## TokenEstimator 实现

不依赖 tokenizer 库，通过字符特征快速估算：

```typescript
class TokenEstimator {
  /** 估算文本 token 数 */
  estimate(content: string | object): number {
    const text = typeof content === 'string' ? content : JSON.stringify(content);
    let tokens = 0;

    for (const char of text) {
      if (char.charCodeAt(0) > 0x4e00) {
        // 中文字符：约 1.5 字符 ≈ 1 token
        tokens += 1 / 1.5;
      } else {
        // 英文/ASCII：约 4 字符 ≈ 1 token
        tokens += 1 / 4;
      }
    }

    return Math.ceil(tokens);
  }

  /** 估算消息数组的 token 数 */
  estimateMessages(messages: Message[]): number {
    let total = 0;
    for (const msg of messages) {
      total += 4; // 角色等元数据开销
      if (msg.content) total += this.estimate(msg.content);
      if (msg.tool_calls) total += this.estimate(msg.tool_calls);
      if (msg.tool_call_id) total += this.estimate(msg.tool_call_id);
      if (msg.name) total += this.estimate(msg.name);
    }
    return total;
  }

  /** 格式化显示 */
  formatTokens(tokens: number): string {
    if (tokens >= 1_000_000) return `${(tokens / 1_000_000).toFixed(1)}M`;
    if (tokens >= 1_000) return `${(tokens / 1_000).toFixed(1)}K`;
    return String(tokens);
  }
}
```

## 三种压缩策略

### 1. 中间移除策略（middle-removal）

保留开头（系统指令和初始上下文）与结尾（最近的对话），删除中间部分。

```
[保留：开头 N 条] + [删除：中间消息] + [保留：结尾 M 条]
```

适用场景：流程性任务，开头的目标设定和最近的进展都重要。

### 2. 最旧移除策略（oldest-removal）

优先删除最旧的消息（系统消息除外），保留最近的对话。

```
[保留：系统消息] + [删除：旧消息] + [保留：最近 N 条]
```

适用场景：长期对话，越新的信息越重要。

### 3. 混合策略（hybrid）

结合前两者，根据对话特征自动选择：

```typescript
function selectStrategy(messages: Message[]): CompressionStrategy {
  const features = analyzeConversation(messages);

  if (features.hasStrongInitialContext) {
    return 'middle-removal';  // 保留开头的重要上下文
  }
  if (features.isLongRunning) {
    return 'oldest-removal';  // 长对话优先最新
  }
  return 'hybrid';
}
```

## 三层策略选择

### 第一层：按供应商/模型选择基础策略

```typescript
function getBaseStrategy(provider: string, model: string): string {
  if (provider === 'anthropic') return 'oldest-removal';
  if (provider === 'openai' && model.includes('gpt-4')) return 'hybrid';
  return 'middle-removal';
}
```

### 第二层：对话特征分析

当基础策略为 hybrid 时，进一步分析：

```typescript
function analyzeConversation(messages: Message[]) {
  return {
    hasStrongInitialContext: messages[0]?.content?.length > 500,
    isLongRunning: messages.length > 50,
    toolCallRatio: countToolCalls(messages) / messages.length,
    averageMessageLength: avgLength(messages),
  };
}
```

### 第三层：自适应选择

置信度低于 0.6 时，同时运行两种策略，选效率更高的：

```typescript
function adaptiveSelect(messages: Message[]): CompressionStrategy {
  const middleResult = applyMiddleRemoval(messages);
  const oldestResult = applyOldestRemoval(messages);

  const middleEfficiency = calcEfficiency(middleResult);
  const oldestEfficiency = calcEfficiency(oldestResult);

  return middleEfficiency > oldestEfficiency ? 'middle-removal' : 'oldest-removal';
}

// 效率公式
function calcEfficiency(result: CompressionResult): number {
  return result.tokenReductionRate * 0.6 + result.messageRetentionRate * 0.4;
}
```

## 消息优先级

压缩前为每条消息分配优先级，高优先级的消息不会被移除：

```typescript
enum MessagePriority {
  CRITICAL = 4,  // 系统消息 → 绝不移除
  HIGH = 3,      // 工具消息、首尾消息、长消息、含函数调用
  NORMAL = 2,    // 普通对话消息
  LOW = 1,       // 短消息（< 20 tokens）且无问号
}

function assignPriority(msg: Message, index: number, total: number): MessagePriority {
  if (msg.role === 'system') return MessagePriority.CRITICAL;

  if (msg.role === 'tool') return MessagePriority.HIGH;
  if (msg.tool_calls?.length) return MessagePriority.HIGH;
  if (index === 0 || index === total - 1) return MessagePriority.HIGH;

  const tokens = tokenEstimator.estimate(msg.content || '');
  if (tokens < 20 && !(msg.content || '').includes('?')) {
    return MessagePriority.LOW;
  }

  return MessagePriority.NORMAL;
}
```

## 压缩触发条件

```
currentTokenCount >= maxTokens × 0.9
```

maxTokens 按模型动态计算，不写死：

```typescript
function getMaxTokens(model: string): number {
  const modelLimits: Record<string, number> = {
    'gpt-4-turbo': 128000,
    'gpt-4': 8192,
    'claude-3-opus': 200000,
    'deepseek-chat': 64000,
  };
  return modelLimits[model] || 64000;
}
```

## 设计要点

- **不过早压缩**：70% 使用率才触发，给 LLM 足够的上下文
- **不丢关键信息**：消息优先级保护系统消息和工具调用
- **自适应选择**：不确定时同时评估两种策略
- **模型感知**：不同模型用不同策略和阈值
