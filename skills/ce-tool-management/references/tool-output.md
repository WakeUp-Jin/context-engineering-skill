# 工具输出格式化与压缩

工具输出需要经过两层处理后才能回写到上下文：格式化（让 LLM 理解）和压缩（避免上下文膨胀）。

## 第一层：renderResultForAssistant

每个工具可选实现 `renderResultForAssistant`，将原始结果转换为 LLM 容易理解的文本：

```typescript
// ReadFile 的输出格式化
function renderResultForAssistant(result: ReadFileResult): string {
  if (!result.success) {
    return `Error reading file: ${result.error}`;
  }

  const lines = result.data.content
    .split('\n')
    .map((line, i) => `${String(i + 1 + (result.data.offset || 0)).padStart(6)}|${line}`)
    .join('\n');

  let output = lines;

  if (result.data.totalLines > result.data.linesRead) {
    output += `\n\n[Showing lines ${result.data.offset + 1}-${result.data.offset + result.data.linesRead} of ${result.data.totalLines}]`;
  }

  return output;
}

// ListFiles 的输出格式化
function renderResultForAssistant(result: ListFilesResult): string {
  return result.data.files.join('\n');
}

// Grep 的输出格式化
function renderResultForAssistant(result: GrepResult): string {
  return result.data.matches
    .map(m => `${m.file}:${m.line}:${m.content}`)
    .join('\n');
}
```

## 第二层：ToolOutputSummarizer

当工具输出过长时，ToolOutputSummarizer 自动压缩，避免上下文膨胀：

```typescript
class ToolOutputSummarizer {
  private tokenEstimator: TokenEstimator;

  /** 触发阈值 */
  private readonly SUMMARIZE_THRESHOLD = 4000;   // tokens
  private readonly TRUNCATE_THRESHOLD = 100_000; // 字符

  async process(output: string): Promise<ToolOutputProcessResult> {
    const charCount = output.length;
    const tokenCount = this.tokenEstimator.estimate(output);

    if (charCount > this.TRUNCATE_THRESHOLD) {
      // 超长输出：先截断再摘要
      const truncated = this.truncate(output);
      return this.summarize(truncated);
    }

    if (tokenCount > this.SUMMARIZE_THRESHOLD) {
      // 普通长输出：直接用 LLM 摘要
      return this.summarize(output);
    }

    // 正常长度：原样返回
    return { content: output, wasSummarized: false };
  }
}
```

### 截断策略

保留头部和尾部，中间截断，确保不丢失关键信息：

```typescript
truncate(output: string): string {
  const lines = output.split('\n');
  const maxLines = 500;

  if (lines.length <= maxLines) {
    // 按字符截断
    const half = Math.floor(this.TRUNCATE_THRESHOLD / 2);
    return output.slice(0, half)
      + '\n\n... [truncated] ...\n\n'
      + output.slice(-half);
  }

  // 按行截断：保留头尾各一半
  const keepLines = Math.floor(maxLines / 2);
  return [
    ...lines.slice(0, keepLines),
    `\n... [${lines.length - maxLines} lines truncated] ...\n`,
    ...lines.slice(-keepLines),
  ].join('\n');
}
```

### LLM 摘要

使用 SECONDARY 层级模型（成本较低）生成摘要：

```typescript
async summarize(output: string): Promise<ToolOutputProcessResult> {
  try {
    const llm = await llmServiceRegistry.getService(ModelTier.SECONDARY);
    const summary = await llm.simpleChat(
      output,
      TOOL_OUTPUT_SUMMARY_PROMPT  // 专门的摘要提示词
    );
    return { content: summary, wasSummarized: true };
  } catch {
    // 摘要失败时 fallback 到截断
    return { content: this.truncate(output), wasSummarized: false };
  }
}
```

### 快速截断（不调用 LLM）

紧急情况下跳过 LLM 摘要，纯粹按 token 估算截断：

```typescript
quickTruncate(output: string, maxTokens: number): string {
  const estimated = this.tokenEstimator.estimate(output);
  if (estimated <= maxTokens) return output;

  const ratio = maxTokens / estimated;
  const keepChars = Math.floor(output.length * ratio);
  const half = Math.floor(keepChars / 2);

  return output.slice(0, half) + '\n...[truncated]...\n' + output.slice(-half);
}
```

## 第三层：generateSummary

ToolScheduler 在工具完成后生成一行简短摘要，用于 UI 展示和日志：

```typescript
function generateSummary(tool: InternalTool, args: any, result: any): string {
  // 根据工具类型生成简短摘要
  switch (tool.name) {
    case 'ListFiles':
      return `Found ${result.data?.files?.length || 0} files`;
    case 'Grep':
      return `Found ${result.data?.matches?.length || 0} matches in ${result.data?.fileCount || 0} files`;
    case 'ReadFile':
      return `Read ${result.data?.linesRead || 0} lines from ${args.file_path}`;
    default:
      return `Executed ${tool.name}`;
  }
}
```

## 输出处理流程

```
工具执行返回原始结果
        │
        ▼
renderResultForAssistant（格式化为 LLM 文本）
        │
        ▼
ToolOutputSummarizer.process（检查是否需要压缩）
        │
   ┌────┴────────────────┐
   │                     │
 正常长度              超长
   │                     │
 原样使用          截断 + LLM 摘要
   │                     │
   └────────┬────────────┘
            ▼
  generateSummary（生成一行 UI 摘要）
            │
            ▼
  回写到上下文 + 通知 ExecutionStream
```

## 设计要点

- **三层处理**：格式化 → 压缩 → 摘要，各司其职
- **成本控制**：摘要使用低成本 SECONDARY 模型
- **降级保障**：LLM 摘要失败时 fallback 到截断
- **不丢关键信息**：截断策略保留头尾
