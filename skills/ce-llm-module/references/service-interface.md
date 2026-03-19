# ILLMService 统一接口设计

## 核心接口

所有 LLM 服务实现统一接口，上层代码不关心具体供应商：

```typescript
interface ILLMService {
  /** 核心补全方法：接收消息和工具定义，返回 LLM 响应 */
  complete(
    messages: Message[],
    tools?: OpenAITool[],
    options?: LLMChatOptions
  ): Promise<LLMResponse>;

  /** 简单对话：无工具、单轮对话 */
  simpleChat(
    userInput: string,
    systemPrompt?: string
  ): Promise<string>;

  /** 可选：带工具循环的生成方法 */
  generate?(
    userInput: string,
    imageData?: ImageData,
    stream?: boolean
  ): Promise<string>;

  /** 获取服务配置 */
  getConfig(): LLMServiceConfig;
}
```

## 配置类型

```typescript
interface LLMConfig {
  provider: string;        // 'deepseek' | 'openrouter' | 'openai' | ...
  model: string;           // 模型名称
  apiKey: string;          // API Key
  baseURL?: string;        // 自定义 Base URL
  maxIterations?: number;  // 工具循环最大迭代次数
  temperature?: number;
  maxTokens?: number;
}
```

## 响应类型

```typescript
interface LLMResponse {
  content: string | null;
  tool_calls?: ToolCall[];
  finish_reason: 'stop' | 'tool_calls' | 'length' | 'content_filter';
  usage?: {
    prompt_tokens: number;
    completion_tokens: number;
    total_tokens: number;
    cache_hit_tokens?: number;      // DeepSeek 特有
    cache_miss_tokens?: number;     // DeepSeek 特有
    reasoning_tokens?: number;      // 推理模型特有
  };
}

interface ToolCall {
  id: string;
  type: 'function';
  function: {
    name: string;
    arguments: string;  // JSON 字符串
  };
}
```

## 聊天选项

```typescript
interface LLMChatOptions {
  /** 流式输出 */
  stream?: boolean;

  /** 流式 chunk 回调 */
  onChunk?: (chunk: string) => void;

  /** 中断信号 */
  abortSignal?: AbortSignal;

  /** ExecutionStream 引用（用于 UI 更新） */
  executionStream?: ExecutionStream;

  /** 温度 */
  temperature?: number;

  /** 最大输出 tokens */
  maxTokens?: number;
}
```

## complete 方法是核心

`complete` 是最重要的方法——上下文管理器组装好消息后，通过 complete 输入给 LLM：

```typescript
// ExecutionEngine 中的调用
const messages = await contextManager.getContext(true);
const tools = toolManager.getFormattedToolsFor(agentConfig);

const response = await llmService.complete(messages, tools, {
  stream: true,
  abortSignal: this.abortController.signal,
  executionStream: this.executionStream,
});

if (response.tool_calls?.length) {
  // 有工具调用 → 交给 ToolScheduler 处理
  await toolScheduler.scheduleBatchFromToolCalls(response.tool_calls);
} else {
  // 无工具调用 → 返回最终结果
  return response.content;
}
```

## simpleChat 的典型用途

不需要工具和复杂上下文时使用：

```typescript
// 历史压缩时使用 simpleChat
const summary = await secondaryLLM.simpleChat(
  JSON.stringify(messagesToCompress),
  COMPRESSION_SYSTEM_PROMPT
);

// 工具输出摘要时使用 simpleChat
const outputSummary = await secondaryLLM.simpleChat(
  longToolOutput,
  TOOL_OUTPUT_SUMMARY_PROMPT
);
```
