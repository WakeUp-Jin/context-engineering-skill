# 消息清理与验证

消息清理保证发送给 LLM API 的消息符合规范——特别是 tool_calls 和 tool 响应的配对关系。不完整的工具调用链会导致 API 报错。

## 核心规则

1. 每个带 `tool_calls` 的 assistant 消息，必须有对应**所有** `tool_call_id` 的 tool 消息
2. 每条 tool 消息必须有合法的 `tool_call_id`，且该 id 在前面的 assistant 消息中出现过

## sanitizeMessages

清理完整消息序列，用于 `ContextManager.buildMessages()` 的最后一步：

```typescript
interface SanitizeResult {
  messages: Message[];
  removed: number;
  issues: string[];
}

function sanitizeMessages(messages: Message[]): Message[] {
  const result = [...messages];
  const removedIndices = new Set<number>();

  // 收集所有 tool_call_id
  const allToolCallIds = new Map<string, number>(); // id → assistant 消息的 index
  for (let i = 0; i < result.length; i++) {
    const msg = result[i];
    if (msg.role === 'assistant' && msg.tool_calls) {
      for (const tc of msg.tool_calls) {
        allToolCallIds.set(tc.id, i);
      }
    }
  }

  // 收集所有 tool 响应的 id
  const toolResponseIds = new Set<string>();
  for (const msg of result) {
    if (msg.role === 'tool' && msg.tool_call_id) {
      toolResponseIds.add(msg.tool_call_id);
    }
  }

  // 规则 1：移除不完整的 assistant 消息
  // （有 tool_calls 但缺少部分 tool 响应）
  for (let i = 0; i < result.length; i++) {
    const msg = result[i];
    if (msg.role === 'assistant' && msg.tool_calls) {
      const allResponded = msg.tool_calls.every(
        tc => toolResponseIds.has(tc.id)
      );
      if (!allResponded) {
        removedIndices.add(i);
        // 同时移除已有的部分 tool 响应
        for (const tc of msg.tool_calls) {
          for (let j = 0; j < result.length; j++) {
            if (result[j].role === 'tool' && result[j].tool_call_id === tc.id) {
              removedIndices.add(j);
            }
          }
        }
      }
    }
  }

  // 规则 2：移除孤立的 tool 消息
  // （tool_call_id 不在任何 assistant 消息中）
  for (let i = 0; i < result.length; i++) {
    const msg = result[i];
    if (msg.role === 'tool' && msg.tool_call_id) {
      if (!allToolCallIds.has(msg.tool_call_id)) {
        removedIndices.add(i);
      }
    }
  }

  return result.filter((_, i) => !removedIndices.has(i));
}
```

## validateMessages

只校验不修改，用于检测问题：

```typescript
interface ValidationResult {
  valid: boolean;
  issues: ValidationIssue[];
}

function validateMessages(messages: Message[]): ValidationResult {
  const issues: ValidationIssue[] = [];

  // 检查 tool_calls 完整性
  for (const msg of messages) {
    if (msg.role === 'assistant' && msg.tool_calls) {
      for (const tc of msg.tool_calls) {
        const hasResponse = messages.some(
          m => m.role === 'tool' && m.tool_call_id === tc.id
        );
        if (!hasResponse) {
          issues.push({
            type: 'missing_tool_response',
            detail: `tool_call ${tc.id} (${tc.function.name}) 缺少对应的 tool 响应`,
          });
        }
      }
    }
  }

  // 检查孤立 tool 消息
  const validCallIds = new Set(
    messages
      .filter(m => m.role === 'assistant' && m.tool_calls)
      .flatMap(m => m.tool_calls!.map(tc => tc.id))
  );

  for (const msg of messages) {
    if (msg.role === 'tool' && msg.tool_call_id) {
      if (!validCallIds.has(msg.tool_call_id)) {
        issues.push({
          type: 'orphaned_tool_message',
          detail: `tool 消息的 tool_call_id ${msg.tool_call_id} 无对应的 assistant tool_call`,
        });
      }
    }
  }

  return { valid: issues.length === 0, issues };
}
```

## sanitizeCurrentTurn

专门用于中断场景（如用户按 ESC 取消）下的当前轮次清理：

```typescript
function sanitizeCurrentTurn(messages: Message[]): Message[] {
  // 与 sanitizeMessages 逻辑相同
  // 但作用范围仅限当前轮次的消息
  // 中断时可能存在未完成的 tool_calls
  return sanitizeMessages(messages);
}
```

使用场景：

```typescript
// CurrentTurnContext 中断处理
class CurrentTurnContext {
  sanitize(): void {
    const cleaned = sanitizeCurrentTurn(this.getAll());
    this.replace(cleaned);
  }
}
```

## 何时触发清理

| 时机 | 调用方法 | 说明 |
|------|---------|------|
| 组装消息时 | `sanitizeMessages()` | `buildMessages()` 的最后一步 |
| 用户中断时 | `sanitizeCurrentTurn()` | ESC 取消导致 tool 调用不完整 |
| 历史压缩前 | `validateMessages()` | 压缩前先检查，避免压缩不完整数据 |
| 调试时 | `validateMessages()` | 只检查不修改，定位问题 |

## 设计要点

- **不丢有效数据**：只移除确实不完整的消息，完整的工具调用链不受影响
- **防御性编程**：LLM 可能返回不规范的 tool_calls，清理器兜底
- **两种模式**：sanitize 自动修复，validate 只报告——根据场景选择
