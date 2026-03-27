# 上下文压缩机制

上下文压缩的目标：提炼关键信息，换新的上下文窗口，在尽量不降性能的前提下继续运行。

## 压缩在哪一层操作

压缩操作在**内部数据结构层**（ContextItem），不是在 LLM message 格式层。

```
压缩前：ShortTermMemoryContext.items = [ContextItem, ContextItem, ContextItem, ...]
                                                     ↓ compress()
压缩后：summary = "摘要文本"
        items = [保留的最近 ContextItem, ...]

format() 时：
  summary → SystemPart(tag="conversation_summary") → system_parts
  items   → message_items
```

摘要是 system 级别的背景信息，通过 `format()` 投递到 `system_parts`，最终合并进那唯一的 system message。

## 三种压缩手段

### 1. 大模型压缩（推荐）

使用 LLM 自身对历史进行摘要，保留语义而非原文。

#### ClaudeCode `/compact` 9 段式

ClaudeCode 的压缩提示词使用 XML 结构化，要求 LLM 从历史中提取 9 个维度的信息：

```xml
<格式要求>
请将对话历史压缩为以下 9 个部分：

1. <primary_request>     主要请求与意图
2. <technical_concepts>  技术概念与架构决策
3. <files_and_code>      涉及的文件与代码变更
4. <errors_and_fixes>    遇到的错误与修复方案
5. <problem_solving>     问题解决的过程与思路
6. <user_messages>       重要的用户消息和偏好
7. <pending_tasks>       待处理的任务
8. <current_work>        当前正在进行的工作
9. <optional_next_steps> 可选的下一步计划
</格式要求>
```

#### Gemini 5 点压缩

Gemini 的方案更简洁：

```typescript
interface GeminiCompactSummary {
  overall_goal: string;        // 总体目标
  key_knowledge: string[];     // 关键事实和约束
  file_system_state: string[]; // 文件创建/修改/删除记录
  recent_actions: string[];    // 最近的重要操作
  current_plan: string;        // 当前计划与下一步
}
```

### 2. 工具消息裁剪

按 Anthropic 思路：在接近 token 上限时，自动移除过时的 tool_calls 和 tool_results，保留对话流。

核心逻辑：按轮次识别工具调用，只保留最近 N 轮。

```typescript
function trimToolMessages(
  messages: ContextItem[],     // 注意：操作 ContextItem，不是 Message
  keepLastToolRounds: number = 3
): ContextItem[] {
  const toolRounds = identifyToolRounds(messages);
  const toKeep = toolRounds.slice(-keepLastToolRounds);
  const keepIndices = new Set(toKeep.flatMap(r => r.indices));

  return messages.filter((msg, i) => {
    if (msg.role !== 'tool' && !msg.tool_calls?.length) return true;
    return keepIndices.has(i);
  });
}
```

### 3. 消息移除策略

不调用 LLM，直接移除部分消息。按消息优先级决定移除顺序（详见 [token-strategy.md](token-strategy.md)）。

## 压缩流程

压缩在 ShortTermMemoryContext 内部执行，只动 `turnStart` 之前的消息：

```typescript
async compress(summarizeFn, keepRatio = 0.3): CompressionResult {
  // 1. 分割：当前轮次受保护
  const compressible = this.items.slice(0, this.turnStart);
  const currentTurn = this.items.slice(this.turnStart);

  if (compressible.length === 0) return { compressed: false };

  // 2. 如果已有摘要，作为上下文一起压缩
  const itemsWithContext = [...compressible];
  if (this.summary) {
    itemsWithContext.unshift(new ContextItem({
      role: "system",
      content: `之前的摘要：\n${this.summary}`,
      source: "summary",
    }));
  }

  // 3. LLM 压缩
  const result = await compressor.compressWithLLM(itemsWithContext, keepRatio, summarizeFn);
  if (!result.compressed) return result;

  // 4. 重建状态
  const keptItems = compressible.slice(-result.keptCount);
  this.summary = result.summary;
  this.items = [...keptItems, ...currentTurn];
  this.turnStart = keptItems.length;

  // 5. 持久化：写入检查点 + 轮转存储段
  this.storage.saveCheckpoint(result.summary, checkpointLine);
  this.storage.rotate();

  return result;
}
```

### 压缩后摘要如何进入 system message

压缩本身不直接操作 message 格式。摘要存储在 `this.summary` 中，在 `format()` 时通过 `SystemPart` 投递到 `system_parts`：

```typescript
format(): ContextParts {
  const parts = new ContextParts();

  if (this.summary) {
    parts.system_parts.push(new SystemPart({
      tag: "conversation_summary",
      description: "以下是之前对话的压缩摘要",
      content: this.summary,
    }));
  }

  parts.message_items.push(...this.items);
  return parts;
}
```

ContextManager 的 `get_context()` 将这个 `SystemPart` 和其他模块的 system_parts 一起渲染为 XML 标签，合并为一条 system message。

## 压缩触发条件

```typescript
// ShortTermMemoryContext
needsCompression(maxTokens: number, threshold: number = 0.8): boolean {
  return this.estimateTokens() >= maxTokens * threshold;
}

// ContextManager 配置
interface CompressionConfig {
  contextWindow: number;            // 模型上下文窗口大小
  compressionThreshold: number;     // 触发压缩的使用率（默认 0.8）
  compressKeepRatio: number;        // 压缩时保留的比例（默认 0.3）
}
```

## 分割点计算

从后往前累加 token，直到达到保留比例。不能在 tool_calls 和对应 tool 响应之间截断：

```typescript
private findSplitPoint(items: ContextItem[], preserveRatio: number): number {
  const totalTokens = this.estimator.estimateItems(items);
  const preserveTokens = totalTokens * preserveRatio;

  let accumulated = 0;
  for (let i = items.length - 1; i >= 0; i--) {
    accumulated += this.estimator.estimateItem(items[i]);
    if (accumulated >= preserveTokens) return i;
  }
  return 0;
}

// 调整：不在 tool 调用链中间截断
private adjustForToolCalls(items: ContextItem[], index: number): number {
  while (index > 0 && items[index].role === 'tool') index--;
  if (items[index].role === 'assistant' && items[index].tool_calls?.length) {
    const callIds = new Set(items[index].tool_calls.map(tc => tc.id));
    let end = index + 1;
    while (end < items.length && items[end].role === 'tool' && callIds.has(items[end].tool_call_id)) {
      end++;
    }
    index = end;
  }
  return index;
}
```

## 选择建议

| 手段 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 大模型压缩 | 长对话、复杂任务 | 保留语义、质量高 | 有延迟、消耗 token |
| 工具消息裁剪 | 工具密集型对话 | 快速、无额外成本 | 可能丢失工具上下文 |
| 消息移除 | 紧急场景 | 最快 | 质量最差 |

实践中通常组合使用：先裁剪过时的工具消息，再对剩余历史做 LLM 摘要。
