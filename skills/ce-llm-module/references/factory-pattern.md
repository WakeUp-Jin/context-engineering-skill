# 工厂模式与服务注册表

## createLLMService 工厂函数

根据配置创建对应的 LLM 服务实例：

```typescript
async function createLLMService(
  config: LLMConfig,
  toolManager?: ToolManager,
  eventManager?: EventManager
): Promise<ILLMService> {
  const apiKey = extractApiKey(config);
  const baseURL = getBaseURL(config);

  let service: ILLMService;

  switch (config.provider) {
    case 'deepseek':
      service = new DeepSeekService({ ...config, apiKey, baseURL });
      break;
    case 'openrouter':
      service = new OpenRouterService({ ...config, apiKey, baseURL });
      break;
    default:
      throw new Error(`Unsupported provider: ${config.provider}`);
  }

  if (toolManager) {
    service.setToolManager(toolManager);
  }
  if (eventManager) {
    service.setEventManager(eventManager);
  }

  return service;
}
```

## LLMServiceRegistry 服务注册表

按 ModelTier 管理和缓存 LLM 服务实例，避免重复创建：

```typescript
enum ModelTier {
  PRIMARY = 'primary',       // 主力模型（高质量）
  SECONDARY = 'secondary',   // 辅助模型（低成本，用于压缩/摘要）
  FALLBACK = 'fallback',     // 降级模型
}

class LLMServiceRegistry {
  private cache: Map<ModelTier, ILLMService> = new Map();
  private configHashes: Map<ModelTier, string> = new Map();

  /** 获取指定层级的服务（带缓存） */
  async getService(tier: ModelTier): Promise<ILLMService> {
    // 检查配置是否变更
    const currentHash = this.computeConfigHash(tier);
    if (this.configHashes.get(tier) !== currentHash) {
      this.invalidate(tier);
    }

    // 缓存命中
    if (this.cache.has(tier)) {
      return this.cache.get(tier)!;
    }

    // 创建新实例
    const config = ConfigService.getModelConfig(tier);
    const service = await createLLMService(config);
    this.cache.set(tier, service);
    this.configHashes.set(tier, currentHash);

    return service;
  }

  /** 配置变更时清除缓存 */
  invalidate(tier?: ModelTier): void {
    if (tier) {
      this.cache.delete(tier);
      this.configHashes.delete(tier);
    } else {
      this.cache.clear();
      this.configHashes.clear();
    }
  }

  /** 计算配置哈希（检测变更） */
  private computeConfigHash(tier: ModelTier): string {
    const config = ConfigService.getModelConfig(tier);
    return JSON.stringify({ provider: config.provider, model: config.model });
  }
}
```

## 使用场景

```typescript
const registry = new LLMServiceRegistry();

// Agent 主循环使用 PRIMARY 模型
const primaryLLM = await registry.getService(ModelTier.PRIMARY);
const response = await primaryLLM.complete(messages, tools);

// 历史压缩使用 SECONDARY 模型（低成本）
const secondaryLLM = await registry.getService(ModelTier.SECONDARY);
const summary = await secondaryLLM.simpleChat(history, COMPRESSION_PROMPT);

// 主模型失败时降级到 FALLBACK
try {
  return await primaryLLM.complete(messages, tools);
} catch {
  const fallbackLLM = await registry.getService(ModelTier.FALLBACK);
  return await fallbackLLM.complete(messages, tools);
}
```

## ModelTier 的意义

不同场景使用不同层级的模型，平衡质量和成本：

| 层级 | 用途 | 选择原则 |
|------|------|---------|
| PRIMARY | Agent 主循环、核心推理 | 质量优先 |
| SECONDARY | 历史压缩、工具输出摘要 | 成本优先 |
| FALLBACK | 主模型不可用时降级 | 可用性优先 |

## 工具辅助函数（utils/helpers.ts）

工厂和服务实现中频繁使用的辅助函数，集中在 `utils/helpers.ts` 中统一导出：

- **`extractApiKey(config)`** — 从配置或环境变量（`${PROVIDER}_API_KEY`）中提取 API Key，缺失时抛出明确错误
- **`getBaseURL(config)`** — 根据供应商返回默认 Base URL（如 DeepSeek → `api.deepseek.com`），支持自定义覆盖
- **`generateId(prefix?)`** — 为 tool_calls 生成唯一 ID（部分供应商返回的 tool_calls 缺少 id 字段）
- **`sleep(ms)`** — Promise 化的延迟等待，重试退避策略的基础设施
- **`isReasoningModel(model)`** — 通过正则匹配判断是否为推理模型（o1/o3/reasoner/deepseek-r 等），推理模型不能设置 temperature
- **`estimateTokens(text)`** — 不依赖 tokenizer 的 token 粗估（英文 ~4字符/token，中文 ~1.5字符/token）
- **`normalizeResponse(raw)`** — 将供应商原始响应统一为 LLMResponse 格式（content、tool_calls、usage）
- **`normalizeToolCalls(raw)`** — 标准化不同供应商的 tool_calls 结构（统一 id、function.name、function.arguments）

设计原则：纯函数为主、防御性编码、集中管理避免重复、不绑定特定供应商逻辑。

## 设计要点

- **延迟创建**：首次 getService 时才创建实例
- **配置感知**：配置变更自动重建实例
- **单例缓存**：同一 tier 共享一个服务实例
- **职责分离**：工厂负责创建，注册表负责管理
