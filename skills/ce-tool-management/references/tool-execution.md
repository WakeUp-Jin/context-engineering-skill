# 工具执行状态机与调度

## ToolScheduler 职责

ToolScheduler 管理工具调用的完整生命周期：参数解析 → 确认检查 → 执行 → 输出处理 → UI 通知。

## 执行状态机

工具调用经历 6 个状态：

```
validating → scheduled → awaiting_approval → executing → success
                │              │                          │
                │              └── cancelled               └── error
                │
                └── 不需要审核时直接跳到 executing
```

```typescript
type SchedulerToolCallStatus =
  | 'validating'        // 正在验证参数
  | 'scheduled'         // 已排入调度队列
  | 'awaiting_approval' // 等待用户确认
  | 'executing'         // 正在执行
  | 'success'           // 执行成功
  | 'error'             // 执行失败
  | 'cancelled';        // 用户取消
```

## 工具调用记录

每次工具调用都有完整的记录结构：

```typescript
interface SchedulerToolCallRequest {
  callId: string;           // 工具调用 ID（来自 LLM 响应）
  toolName: string;         // 工具名称
  args: any;                // 解析后的参数
  rawArgs: string;          // 原始参数字符串
  toolCategory: string;     // 工具分类
  paramsSummary: string;    // 参数摘要（用于 UI 展示）
  thinkingContent?: string; // LLM 的思考过程
}

interface SchedulerToolCallRecord {
  request: SchedulerToolCallRequest;
  status: SchedulerToolCallStatus;
  result?: any;
  error?: string;
  summary?: string;         // 执行摘要
  startTime?: number;
  endTime?: number;
}
```

## 调度流程

```typescript
async scheduleToolCall(toolCall: ToolCallFromLLM): Promise<void> {
  // 1. 解析参数
  const args = deepParseArgs(JSON.parse(toolCall.rawArgs));

  // 2. 获取工具实例
  const tool = this.toolManager.getTool(toolCall.toolName);

  // 3. 检查是否需要确认
  const confirmDetails = await tool.shouldConfirmExecute(
    args, this.approvalMode, this.context, this.allowlist
  );

  if (confirmDetails) {
    // 4a. 需要确认 → 等待用户
    this.setStatus(callId, 'awaiting_approval');
    await this.waitForConfirmation(callId, confirmDetails);
  }

  // 4b. 执行工具
  this.setStatus(callId, 'executing');
  const result = await this.toolManager.execute(toolCall.toolName, args, this.context);

  // 5. 输出处理（可选摘要/压缩）
  const processed = await this.processOutput(result, tool);

  // 6. 生成执行摘要
  const summary = this.generateSummary(tool, args, processed);

  // 7. 通知 ExecutionStream 完成
  this.executionStream.completeToolCall(callId, processed, summary);
}
```

## 并行执行判断

批量工具调用时，ToolScheduler 根据只读性决定执行策略：

```typescript
canExecuteInParallel(toolCalls: ToolCallFromLLM[]): boolean {
  return toolCalls.every(call => {
    const tool = this.toolManager.getTool(call.toolName);
    return tool?.isReadOnly?.() ?? false;
  });
}

async scheduleBatchFromToolCalls(toolCalls: ToolCallFromLLM[]): Promise<void> {
  if (this.canExecuteInParallel(toolCalls)) {
    // 全部只读 → 并行执行
    await Promise.all(toolCalls.map(tc => this.scheduleToolCall(tc)));
  } else {
    // 存在写操作 → 串行执行
    for (const tc of toolCalls) {
      await this.scheduleToolCall(tc);
    }
  }
}
```

## 参数深度解析

LLM 返回的工具参数可能包含嵌套的 JSON 字符串，需要递归解析：

```typescript
function deepParseArgs(args: any): any {
  if (typeof args === 'string') {
    try { return deepParseArgs(JSON.parse(args)); }
    catch { return args; }
  }
  if (Array.isArray(args)) {
    return args.map(deepParseArgs);
  }
  if (args && typeof args === 'object') {
    return Object.fromEntries(
      Object.entries(args).map(([k, v]) => [k, deepParseArgs(v)])
    );
  }
  return args;
}
```

## 与 ExecutionStream 的协作

ToolScheduler 通过 ExecutionStream 向 UI 报告状态变化：

```
executionStream.startValidating(callId)     // 开始验证
executionStream.awaitingApproval(callId)    // 等待确认
executionStream.updateToExecuting(callId)   // 开始执行
executionStream.completeToolCall(callId, result, summary)  // 完成
executionStream.errorToolCall(callId, error)               // 错误
executionStream.cancelToolCall(callId)                     // 取消
```

## 设计要点

- **状态可追踪**：每个工具调用都有明确的状态和时间记录
- **安全默认**：写入操作串行执行，只读操作可并行
- **输出感知**：执行后自动判断是否需要摘要/压缩
- **UI 解耦**：通过 ExecutionStream 抽象，不直接操作 UI
