# 执行引擎详解

ExecutionEngine 是 Agent 的核心循环——接收用户输入，反复调用 LLM 和工具，直到得到最终结果。

## 主循环

```typescript
class ExecutionEngine {
  private llmService: ILLMService;
  private contextManager: ContextManager;
  private toolScheduler: ToolScheduler;
  private maxLoops: number;
  private abortSignal?: AbortSignal;

  async execute(userInput: string): Promise<ExecutionResult> {
    let loopCount = 0;

    while (loopCount < this.maxLoops) {
      // 1. 检查中断
      if (this.abortSignal?.aborted) {
        return this.buildCancelledResult();
      }

      // 2. 检查上下文溢出
      if (this.contextManager.isOverflow()) {
        await this.contextManager.history.compress();
      }

      // 3. 获取上下文（含自动压缩）
      const messages = await this.contextManager.getContext(true);

      // 4. 获取工具定义
      const tools = this.toolScheduler.getFormattedTools();

      // 5. 调用 LLM
      const response = await this.llmService.complete(messages, tools, {
        stream: true,
        abortSignal: this.abortSignal,
        executionStream: this.executionStream,
      });

      // 6. 更新统计
      this.updateStats(response.usage);

      // 7. 判断是否有工具调用
      if (response.tool_calls?.length) {
        // 有工具调用 → 调度执行
        await this.handleToolCalls(response);
        loopCount++;
        continue;
      }

      // 8. 无工具调用 → 返回最终结果
      return this.buildSuccessResult(response);
    }

    // 超过最大循环次数
    return this.buildErrorResult('Max iterations exceeded');
  }
}
```

## 工具调用处理

```typescript
private async handleToolCalls(response: LLMResponse): Promise<void> {
  // 1. 将 assistant 消息（含 tool_calls）加入当前轮次
  this.contextManager.currentTurn.add({
    role: 'assistant',
    content: response.content,
    tool_calls: response.tool_calls,
  });

  // 2. 通过 ToolScheduler 批量调度
  await this.toolScheduler.scheduleBatchFromToolCalls(response.tool_calls);

  // 3. 将工具结果加入当前轮次
  for (const record of this.toolScheduler.getCompletedRecords()) {
    this.contextManager.currentTurn.add({
      role: 'tool',
      tool_call_id: record.request.callId,
      content: record.result,
    });
  }

  // 4. 发送事件（用于评估系统）
  for (const tc of response.tool_calls) {
    eventBus.emit('tool:call', {
      agentName: this.agentName,
      toolName: tc.function.name,
    });
  }
}
```

## 溢出控制

当上下文接近模型限制时，自动触发压缩：

```typescript
// 在主循环开头检查
if (this.contextManager.isOverflow()) {
  // 紧急压缩
  this.executionStream?.emit('compression:start');
  await this.contextManager.history.compress();
  this.executionStream?.emit('compression:complete');
}

// getContext(true) 也会在 70% 使用率时触发自动压缩
// 两层保障：70% 自动压缩 + 95% 溢出压缩
```

## 流式输出

通过 ExecutionStream 实时推送进度：

```typescript
// LLM 文本输出
executionStream.emit('content:delta', chunk);

// 工具调用开始
executionStream.emit('tool:start', { callId, toolName });

// 工具调用完成
executionStream.emit('tool:complete', { callId, result, summary });

// 压缩事件
executionStream.emit('compression:start');
executionStream.emit('compression:complete');
```

## 中断处理

```typescript
// Agent.abort() 触发
this.abortController.abort();

// ExecutionEngine 中的检查
if (this.abortSignal?.aborted) {
  // 1. 清理当前轮次不完整的消息
  this.contextManager.currentTurn.sanitize();

  // 2. 如果有正在执行的工具，等待其自然结束或超时
  // AbortSignal 会传递到工具执行层

  return this.buildCancelledResult();
}
```

## 轮次归档

一轮工具循环结束（LLM 返回无 tool_calls 的响应）后，将当前轮次归档到历史：

```typescript
// 在 buildSuccessResult 之前
this.contextManager.currentTurn.archiveTo(
  this.contextManager.history,
  userInput
);
```

这样下一次 `Agent.run()` 时，之前的对话会出现在历史中。

## 配置选项

```typescript
interface ExecutionConfig {
  maxLoops: number;            // 最大工具循环次数（默认 10）
  enableCompression: boolean;  // 是否启用自动压缩
  stream: boolean;             // 是否流式输出
}
```

## 设计要点

- **有限循环**：maxLoops 防止无限工具调用
- **双重压缩保障**：70% 自动 + 95% 溢出
- **中断安全**：AbortSignal 贯穿整个执行链
- **事件驱动**：ExecutionStream 和 EventBus 解耦 UI 和评估
- **状态归档**：每轮结束将当前轮次归档到历史
