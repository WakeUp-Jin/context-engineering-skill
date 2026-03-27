# 各上下文模块详解

每个模块继承 `BaseContext<T>`，存储自己的内部状态，通过 `format()` 返回 `ContextParts` 声明内容投递到 system 还是 messages。

## SystemPromptContext

存储系统提示词，支持**分段注册**——不同模块可以各自追加系统指令片段（如 skill 目录、工具描述等），最终按优先级合并。

**内部存储类型**：`PromptSegment`（带 id、priority、enabled 的文本片段）

```typescript
interface PromptSegment {
  id: string;         // 片段标识，用于替换和管理
  content: string;    // 片段内容
  priority: number;   // 优先级（高 = 排在前面），默认 5
  enabled: boolean;   // 是否启用
}
```

**核心方法**：

```typescript
class SystemPromptContext extends BaseContext<PromptSegment> {

  constructor(corePrompt?: string) {
    // 初始化时注册核心提示词片段（priority 最高）
    this.add({ id: "core", content: corePrompt || DEFAULT_PROMPT, priority: 100, enabled: true });
  }

  /** 注册/替换一个片段 */
  registerSegment(segment: PromptSegment): void {
    this.items = this.items.filter(s => s.id !== segment.id);
    this.add(segment);
  }

  /** 按优先级降序合并所有启用的片段为一段文本 */
  getPrompt(): string {
    const enabled = this.items.filter(s => s.enabled);
    enabled.sort((a, b) => b.priority - a.priority);
    return enabled.map(s => s.content).filter(c => c.trim()).join("\n\n");
  }

  /** 投递到 system_parts */
  format(): ContextParts {
    const prompt = this.getPrompt();
    if (!prompt) return new ContextParts();
    return new ContextParts({
      system_parts: [
        new SystemPart({ tag: "system_prompt", description: "", content: prompt })
      ]
    });
  }
}
```

**设计要点**：
- 核心提示词 `system_prompt` 的 description 为空——它本身就是指令，不需要额外解释
- Skill 目录、工具描述等通过 `registerSegment()` 动态注入，优先级低于核心提示词
- 所有片段最终合并为一个 SystemPart，不是多个

## ShortTermMemoryContext（对话记忆 + 压缩）

管理对话消息流，是最复杂的上下文模块——内含压缩逻辑和持久化。

**内部存储类型**：`ContextItem`（不是 `Message`！ContextItem 携带 source、priority、thinking、usage 等元数据）

**关键区分**：format() 同时返回两种投递目标：
- 压缩摘要 → `system_parts`（属于背景指令级别）
- 对话消息 → `message_items`（属于对话流）

```typescript
class ShortTermMemoryContext extends BaseContext<ContextItem> {
  private summary: string = "";       // 压缩摘要
  private turnStart: number = 0;      // 当前轮次起始位置

  /** 追加消息到内存 + 持久化 */
  appendMessage(item: ContextItem): void {
    this.add(item);
    this.storage.append(item.toDict());  // 持久化用 toDict()，保留所有元数据
  }

  /** 标记当前轮次起始位置（Agent 每轮开始时调用） */
  markTurnStart(): void {
    this.turnStart = this.items.length;
  }

  /** 投递：摘要 → system，对话 → messages */
  format(): ContextParts {
    const parts = new ContextParts();

    if (this.summary) {
      parts.system_parts.push(
        new SystemPart({
          tag: "conversation_summary",
          description: "以下是之前对话的压缩摘要",
          content: this.summary,
        })
      );
    }

    parts.message_items.push(...this.items);
    return parts;
  }
}
```

**为什么摘要投递到 system 而不是 messages？**

摘要是对过去对话的浓缩背景，属于"Agent 需要知道的背景信息"，和系统提示词、用户记忆是同一层级。如果放到 messages 里作为一条独立消息，它会：
- 产生额外的 system message（多条 system 消息问题）
- 或者作为 user/assistant 消息，角色语义不对

### 轮次跟踪

`turnStart` 标记当前轮次在 items 列表中的起始位置：

```
items: [旧消息...] [当前轮次消息...]
                    ↑ turnStart

压缩只动 turnStart 之前的消息，当前轮次始终受保护。
```

### 启动时的加载与清理

从持久化存储恢复时，使用 `ContextItem.fromDict()` 保留所有元数据，然后做 sanitize：

```typescript
private loadMemory(): void {
  const checkpoint = this.storage.loadCheckpoint();

  if (checkpoint) {
    this.summary = checkpoint.summary;
    const raw = this.storage.loadFromLine(checkpoint.keptFromLine);
    this.items = raw.map(d => ContextItem.fromDict(d));  // 从持久化格式恢复
  } else {
    this.items = this.storage.loadAll().map(d => ContextItem.fromDict(d));
  }

  this.sanitizeOnLoad();  // 清理不完整的 tool 调用链
  this.turnStart = this.items.length;
}
```

## LongTermMemoryContext

持久化用户记忆，跨会话保存。内容全部投递到 system。

**内部存储类型**：`string`（记忆文本）

```typescript
class LongTermMemoryContext extends BaseContext<string> {
  private store: MemoryStore;  // 文件系统持久化

  /** 从持久化存储刷新到内存 */
  refresh(): void {
    this.clear();
    const text = this.store.getMemoryText();
    if (text?.trim()) this.add(text);
  }

  /** 全部投递到 system_parts */
  format(): ContextParts {
    const allText = this.getAll();
    let text: string;

    if (allText.length === 0) {
      const stored = this.store.getMemoryText();
      if (!stored?.trim()) return new ContextParts();
      text = stored;
    } else {
      text = allText.join("\n");
    }

    return new ContextParts({
      system_parts: [
        new SystemPart({
          tag: "long_term_memory",
          description: "以下是你对用户的长期记忆，请基于这些信息个性化回复",
          content: text,
        })
      ]
    });
  }
}
```

**设计要点**：
- description 承担了"行为指引"的作用——告诉模型这段内容是什么、怎么用
- 写入记忆不通过 format()，而是通过 MemoryStore / 记忆工具直接写入

## 扩展模块示例

概念上可以设计更多模块，每个模块只需决定投递到 system 还是 messages：

### RAG 检索结果

```typescript
class RAGContext extends BaseContext<RetrievedChunk> {
  format(): ContextParts {
    if (this.isEmpty()) return new ContextParts();
    const content = this.items.map(c => c.text).join("\n\n---\n\n");
    return new ContextParts({
      system_parts: [
        new SystemPart({
          tag: "retrieved_context",
          description: "以下是从知识库检索到的相关内容，请基于这些信息回答",
          content,
        })
      ]
    });
  }
}
```

### 结构化输出约束

```typescript
class StructuredOutputContext extends BaseContext<OutputSchema> {
  format(): ContextParts {
    if (this.isEmpty()) return new ContextParts();
    const schemas = this.items.map(s => JSON.stringify(s, null, 2)).join("\n");
    return new ContextParts({
      system_parts: [
        new SystemPart({
          tag: "output_format",
          description: "请严格按照以下格式输出",
          content: schemas,
        })
      ]
    });
  }
}
```

## 选择建议

- **最小可行版**：SystemPromptContext + ShortTermMemoryContext 两个模块即可运行
- **标准版**：加上 LongTermMemoryContext，支持跨会话记忆
- **进阶版**：按需扩展 RAG、结构化输出等模块——每个新模块只需实现 `format() → ContextParts`
