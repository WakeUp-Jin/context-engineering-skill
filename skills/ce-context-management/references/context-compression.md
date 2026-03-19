# 上下文压缩机制

上下文压缩的目标：提炼关键信息，换新的上下文窗口，在尽量不降性能的前提下继续运行。

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

压缩结果加开篇语，作为新一轮对话的起点：

```
"Context has been compressed using structured 8-section algorithm.
Below is the preserved context from the previous conversation..."
```

#### Gemini 5 点压缩

Gemini 的方案更简洁，5 个字段：

```typescript
interface GeminiCompactSummary {
  overall_goal: string;        // 总体目标
  key_knowledge: string[];     // 关键事实和约束
  file_system_state: string[]; // 文件创建/修改/删除记录
  recent_actions: string[];    // 最近的重要操作
  current_plan: string;        // 当前计划与下一步
}
```

#### reason-code 实现

reason-code 的历史压缩流程：

```typescript
// 压缩提示词
const COMPRESSION_SYSTEM_PROMPT = `
你是一个对话历史压缩助手。请将以下对话历史压缩为简洁的摘要。

要求：
- 保留所有关键决策和结论
- 保留涉及的文件路径和代码变更
- 保留未完成的任务和计划
- 保留用户的偏好和约束条件
- 删除重复信息和不重要的对话
- 使用简洁的描述性语言
`;

// 在 HistoryContext.compress() 中调用
const llm = await llmServiceRegistry.getService(ModelTier.SECONDARY);
const summary = await llm.simpleChat(
  JSON.stringify(toCompress),
  COMPRESSION_SYSTEM_PROMPT
);
```

### 2. 工具消息裁剪

按 Anthropic 思路：在接近 token 上限时，自动移除过时的 tool_calls 和 tool_results，保留对话流。

核心逻辑：按轮次识别工具调用，只保留最近 N 轮。

```typescript
function trimToolMessages(
  messages: Message[],
  keepLastToolRounds: number = 3
): Message[] {
  // 1. 识别工具调用轮次
  const toolRounds = identifyToolRounds(messages);

  // 2. 只保留最近 N 轮
  const toKeep = toolRounds.slice(-keepLastToolRounds);
  const keepIndices = new Set(toKeep.flatMap(r => r.indices));

  // 3. 过滤：保留非工具消息 + 最近 N 轮工具消息
  return messages.filter((msg, i) => {
    if (msg.role !== 'tool' && !msg.tool_calls) return true;
    return keepIndices.has(i);
  });
}

function identifyToolRounds(messages: Message[]): ToolRound[] {
  const rounds: ToolRound[] = [];
  let currentRound: number[] = [];

  for (let i = 0; i < messages.length; i++) {
    const msg = messages[i];

    if (msg.role === 'assistant' && msg.tool_calls?.length) {
      // 新轮次开始
      if (currentRound.length > 0) {
        rounds.push({ indices: currentRound });
      }
      currentRound = [i];
    } else if (msg.role === 'tool') {
      currentRound.push(i);
    }
  }

  if (currentRound.length > 0) {
    rounds.push({ indices: currentRound });
  }

  return rounds;
}
```

### 3. 消息移除策略

不调用 LLM，直接移除部分消息。按消息优先级决定移除顺序（详见 [token-strategy.md](token-strategy.md)）。

## 压缩触发条件

```typescript
// ContextManager 中的配置
const DEFAULT_MODEL_LIMIT = 64000;       // 模型 Token 上限
const DEFAULT_COMPRESSION_THRESHOLD = 0.7; // 使用率 >= 70% 触发压缩
const DEFAULT_OVERFLOW_THRESHOLD = 0.95;   // 使用率 >= 95% 视为溢出

needsCompression(): boolean {
  return this.getTokenUsage().ratio >= this.compressionThreshold;
}

isOverflow(): boolean {
  return this.getTokenUsage().ratio >= this.overflowThreshold;
}
```

## 压缩后的处理

压缩完成后，摘要作为一条 system 消息放在历史开头：

```typescript
const summaryMessage: Message = {
  role: 'system',
  content: [
    '[对话历史摘要]',
    summary,
    '',
    '[历史中涉及的文件]:',
    ...retainedFiles.map(f => `- ${f}`),
  ].join('\n'),
};

// 新历史 = [摘要消息, ...保留的最近消息]
this.replace([summaryMessage, ...recentMessages]);
```

## 选择建议

| 手段 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| 大模型压缩 | 长对话、复杂任务 | 保留语义、质量高 | 有延迟、消耗 token |
| 工具消息裁剪 | 工具密集型对话 | 快速、无额外成本 | 可能丢失工具上下文 |
| 消息移除 | 紧急场景 | 最快 | 质量最差 |

实践中通常组合使用：先裁剪过时的工具消息，再对剩余历史做 LLM 摘要。
