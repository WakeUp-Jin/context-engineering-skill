# 多供应商适配

不同 LLM 供应商的 API 差异主要体现在：消息格式、工具定义格式、流式输出处理、特殊参数。

## 服务实现基本结构

每个供应商一个服务类，实现 ILLMService 接口：

```typescript
class DeepSeekService implements ILLMService {
  private client: OpenAI;  // 使用 OpenAI SDK 兼容接口

  constructor(config: LLMConfig) {
    this.client = new OpenAI({
      apiKey: config.apiKey,
      baseURL: config.baseURL || 'https://api.deepseek.com',
    });
  }

  async complete(
    messages: Message[],
    tools?: OpenAITool[],
    options?: LLMChatOptions
  ): Promise<LLMResponse> {
    const params: any = {
      model: this.config.model,
      messages,
    };

    // 推理模型不设置 temperature
    if (!this.isReasoningModel()) {
      params.temperature = options?.temperature ?? 0.7;
    }

    if (tools?.length) {
      params.tools = tools;
    }

    if (options?.stream) {
      return this.completeStream(params, options);
    }

    const response = await this.client.chat.completions.create(params);
    return this.normalizeResponse(response);
  }
}
```

## 供应商差异对照

### 消息格式差异

```typescript
// OpenAI / DeepSeek：标准格式
{ role: 'system', content: '...' }
{ role: 'user', content: '...' }
{ role: 'assistant', content: '...', tool_calls: [...] }
{ role: 'tool', tool_call_id: '...', content: '...' }

// Anthropic：system 单独字段，content 为 blocks
{
  system: '系统提示词',         // 独立字段
  messages: [
    { role: 'user', content: [{ type: 'text', text: '...' }] },
    { role: 'assistant', content: [{ type: 'tool_use', ... }] },
    { role: 'user', content: [{ type: 'tool_result', ... }] },
  ]
}

// Gemini：提示词注入工具
// 解析 tool_code 块，工具调用为 functionCall
```

### 工具格式差异

```typescript
// OpenAI 标准格式（大多数兼容）
{
  type: 'function',
  function: {
    name: 'ReadFile',
    description: '...',
    parameters: { /* JSON Schema */ }
  }
}

// Anthropic 格式
{
  name: 'ReadFile',
  description: '...',
  input_schema: { /* JSON Schema */ }
}
```

### 流式输出处理

```typescript
async function consumeStream(
  stream: AsyncIterable<ChatCompletionChunk>,
  onChunk?: (chunk: string) => void,
  executionStream?: ExecutionStream
): Promise<LLMResponse> {
  let content = '';
  let toolCalls: ToolCall[] = [];
  let usage: any = null;

  for await (const chunk of stream) {
    const delta = chunk.choices[0]?.delta;

    if (delta?.content) {
      content += delta.content;
      onChunk?.(delta.content);
      executionStream?.emit('content:delta', delta.content);
    }

    if (delta?.tool_calls) {
      // 增量拼接 tool_calls 参数
      for (const tc of delta.tool_calls) {
        if (!toolCalls[tc.index]) {
          toolCalls[tc.index] = { id: tc.id, type: 'function', function: { name: '', arguments: '' } };
        }
        if (tc.function?.name) toolCalls[tc.index].function.name += tc.function.name;
        if (tc.function?.arguments) toolCalls[tc.index].function.arguments += tc.function.arguments;
      }
    }

    if (chunk.usage) {
      usage = chunk.usage;
    }
  }

  return { content: content || null, tool_calls: toolCalls.length ? toolCalls : undefined, usage };
}
```

### tool_calls 规范化

不同供应商返回的 tool_calls 格式可能不完全一致：

```typescript
function normalizeToolCalls(raw: any[]): ToolCall[] {
  return raw.map(tc => ({
    id: tc.id || generateId(),
    type: 'function',
    function: {
      name: tc.function?.name || tc.name || '',
      arguments: typeof tc.function?.arguments === 'string'
        ? tc.function.arguments
        : JSON.stringify(tc.function?.arguments || tc.arguments || {}),
    },
  }));
}
```

## 重试与退避策略

```typescript
async completeWithRetry(
  params: any,
  maxRetries: number = 3
): Promise<LLMResponse> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await this.client.chat.completions.create(params);
    } catch (error) {
      if (!this.isRetryableError(error) || attempt === maxRetries) {
        throw error;
      }
      // 指数退避 + 随机抖动
      const delay = Math.min(1000 * Math.pow(2, attempt) + Math.random() * 1000, 30000);
      await sleep(delay);
    }
  }
  throw new Error('Max retries exceeded');
}

isRetryableError(error: any): boolean {
  const status = error?.status;
  // 429（限流）和 5xx（服务端错误）可重试
  if (status === 429 || (status >= 500 && status < 600)) return true;
  // 400（请求错误）和 401（认证错误）不重试
  if (status === 400 || status === 401) return false;
  // 网络错误可重试
  return error?.code === 'ECONNRESET' || error?.code === 'ETIMEDOUT';
}
```

## 特殊供应商处理

### OpenRouter：Claude cache_control

```typescript
// 对大文本启用 Anthropic 缓存
addCacheControlForClaude(messages: Message[]): Message[] {
  return messages.map(msg => {
    if (msg.content && msg.content.length > 2000) {
      return { ...msg, cache_control: { type: 'ephemeral' } };
    }
    return msg;
  });
}
```

### DeepSeek：推理模型特殊处理

```typescript
// deepseek-reasoner 等推理模型
if (this.isReasoningModel()) {
  // 不设置 temperature（推理模型有固定策略）
  delete params.temperature;
  // 使用 reasoning_content 字段
}

// usage 中包含额外的缓存信息
usage: {
  cache_hit_tokens: number;
  cache_miss_tokens: number;
  reasoning_tokens: number;
}
```

## 设计要点

- **OpenAI SDK 兼容**：大部分供应商兼容 OpenAI SDK，可复用同一套 client
- **格式转换集中**：在服务类内部完成格式差异的适配
- **降级保护**：首次调用提供工具，重试时可禁用工具（避免工具误用导致循环失败）
- **上下文感知**：接近 token 上限（如 80%）时触发压缩，而非等到报错
