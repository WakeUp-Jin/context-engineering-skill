---
name: ce-llm-module
description: |
  Agent LLM 模块 - 通过统一接口和工厂模式对接多种模型供应商。
  包含 ILLMService 统一接口（complete/simpleChat/generate）、
  工厂模式与服务注册表（createLLMService/LLMServiceRegistry/ModelTier分层缓存）、
  多供应商适配（消息格式转换/工具格式转换/流式输出/重试与退避策略）。
  支持 DeepSeek、OpenRouter、OpenAI、Anthropic 等供应商的差异适配。
  当用户需要设计统一 LLM 接口、实现工厂模式、适配多供应商、
  处理流式输出、实现重试机制时使用此 Skill。
  即使用户没有明确提到"LLM模块"，只要涉及调用大模型 API、封装 OpenAI/Anthropic/DeepSeek 接口、
  处理 chat completions 返回值、实现流式输出、tool_calls 解析、多模型切换、
  或者模型调用失败重试等话题，也应主动使用此 Skill。
---

# LLM 模块

LLM 模块是 Agent 的"方舟反应堆"。通过统一接口和工厂模式，支持多种模型供应商的接入，处理不同供应商的消息格式、工具格式和错误重试的差异。

为什么要自己封装而不直接用 LangChain 等框架？因为 Agent 需要与上下文管理、工具系统深度集成——自定义封装可以添加重试、压缩、缓存等定制功能，更轻量且避免供应商锁定。当你只需要 complete() 和 simpleChat() 两个方法时，没必要引入一个几十个模块的框架。

## 核心设计思路

每种 LLM 供应商的**输入输出格式都不同**：
- 输出结果格式不同 → 需要统一为内部消息格式
- 要求的 message 和 tool 格式不同 → 需要按供应商转换

自定义封装的优势：
- 添加重试机制、上下文压缩等定制功能
- 与系统其他组件无缝集成
- 更轻量，只包含必要功能
- 避免供应商锁定

## 推荐目录结构

```
llm/
├── services/
│   ├── DeepSeekService.ts      # DeepSeek 服务实现
│   └── OpenRouterService.ts    # OpenRouter 服务实现
├── types/
│   └── index.ts                # 统一类型定义
├── utils/
│   └── helpers.ts              # 工具函数
├── factory.ts                  # LLM 工厂函数
├── LLMServiceRegistry.ts       # 服务注册表（按 ModelTier 缓存）
└── index.ts
```

## 使用示例

```typescript
// 1. 使用工厂函数创建 LLM 服务
const service = await createLLMService({
  provider: 'deepseek',
  model: 'deepseek-chat',
  apiKey: process.env.DEEPSEEK_API_KEY,
  maxIterations: 5,
});

// 2. 简单对话（无工具）
const answer = await service.simpleChat(
  '用一句话解释什么是大语言模型？',
  '你是一个 AI 助手'
);

// 3. 完整调用（带工具、带上下文）
const messages = contextManager.getContext();
const tools = toolManager.getFormattedTools();
const response = await service.complete(messages, tools);
```

## 实现步骤指引

当帮用户实现 LLM 模块时，按以下顺序推进：

1. **定义统一接口**：设计 ILLMService（complete/simpleChat）和相关类型（LLMConfig、LLMResponse、ToolCall）（参阅 [references/service-interface.md](references/service-interface.md)）
2. **实现第一个供应商**：选择主力供应商（如 DeepSeek），实现 complete 和 simpleChat，处理流式输出（参阅 [references/provider-adapters.md](references/provider-adapters.md)）
3. **添加工厂和注册表**：实现 createLLMService 工厂函数和 LLMServiceRegistry（按 ModelTier 分层缓存）（参阅 [references/factory-pattern.md](references/factory-pattern.md)）
4. **适配更多供应商**：添加 OpenRouter 等，处理消息/工具格式差异和特殊参数
5. **添加重试和降级**：实现指数退避重试和 FALLBACK tier 降级

## 细分文档

按需阅读 `references/` 目录下的详细文档：

| 你的需求 | 阅读文档 |
|---------|---------|
| 设计统一的 LLM 服务接口 | [references/service-interface.md](references/service-interface.md) |
| 实现工厂模式和服务管理 | [references/factory-pattern.md](references/factory-pattern.md) |
| 适配不同的 LLM 供应商 | [references/provider-adapters.md](references/provider-adapters.md) |
