# 单智能体设计

## Agent 类

统一 Agent 执行器，支持主代理和子代理两种角色：

```typescript
class Agent {
  private config: AgentConfig;
  private runtime: SharedRuntime;  // ContextManager、ToolManager、StatsManager 等

  private llmService: ILLMService;
  private executionEngine: ExecutionEngine;
  private abortController: AbortController;

  constructor(config: AgentConfig, runtime: SharedRuntime) {
    this.config = config;
    this.runtime = runtime;
    this.abortController = new AbortController();
  }
}
```

## 生命周期

### init：初始化

```typescript
async init(options?: AgentInitOptions): Promise<void> {
  // 1. 获取 LLM 服务
  this.llmService = await llmServiceRegistry.getService(
    this.config.modelTier || ModelTier.PRIMARY
  );

  // 2. 设置系统提示词（三种来源，按优先级）
  const systemPrompt = options?.systemPrompt          // 传入的优先
    || this.config.systemPrompt                       // 配置的静态提示词
    || this.config.systemPromptBuilder?.();            // 配置的构建器

  this.runtime.contextManager.systemPrompt.setPrompt(systemPrompt);

  // 3. 创建执行引擎
  this.executionEngine = new ExecutionEngine({
    llmService: this.llmService,
    contextManager: this.runtime.contextManager,
    toolManager: this.runtime.toolManager,
    abortSignal: this.abortController.signal,
  });
}
```

### run：执行

```typescript
async run(userInput: string, options?: AgentRunOptions): Promise<AgentResult> {
  if (!this.llmService) await this.init(options);

  try {
    const result = await this.executionEngine.execute(userInput);

    // 统计记录
    this.runtime.statsManager.recordRun(result.usage);

    return {
      content: result.content,
      toolCalls: result.toolCallRecords,
      usage: result.usage,
    };
  } catch (error) {
    if (this.abortController.signal.aborted) {
      return { content: null, cancelled: true };
    }
    throw error;
  }
}
```

### abort：中断

```typescript
abort(): void {
  this.abortController.abort();
  // 清理当前轮次的不完整消息
  this.runtime.contextManager.currentTurn.sanitize();
}
```

### compress：手动压缩

```typescript
async compress(): Promise<void> {
  await this.runtime.contextManager.history.compress();
}
```

## 工具过滤

不同 Agent 看到不同的工具集：

```typescript
interface AgentConfig {
  tools?: {
    include?: string[];    // 白名单
    exclude?: string[];    // 黑名单
  };
}

// 子代理自动排除 Task 工具（防止递归）
const tools = this.runtime.toolManager.getFilteredTools(
  this.config,
  this.config.role === 'subagent'  // excludeTask
);
```

## AgentConfig 类型

```typescript
interface AgentConfig {
  name: AgentType;
  role: AgentRole;
  description: string;
  systemPrompt?: string;
  systemPromptBuilder?: () => string;
  tools?: { include?: string[]; exclude?: string[] };
  modelTier?: ModelTier;
  execution?: {
    maxLoops?: number;          // 最大工具循环次数
    enableCompression?: boolean; // 是否启用自动压缩
  };
  hidden?: boolean;  // 是否在 UI 中隐藏
}

type AgentRole = 'primary' | 'subagent' | 'all';
type AgentType = 'build' | 'steward' | 'explore' | 'explanatory';
```

## 预设配置

四种内置预设，覆盖常见 Agent 角色：

```typescript
// build：通用编程代理（主代理）
const buildAgent: AgentConfig = {
  name: 'build',
  role: 'primary',
  description: '通用编程代理，可以读写文件、搜索代码、执行任务',
  systemPromptBuilder: buildSystemPrompt,
  modelTier: ModelTier.PRIMARY,
};

// explore：代码库探索（子代理）
const exploreAgent: AgentConfig = {
  name: 'explore',
  role: 'subagent',
  description: '快速探索代码库，只使用只读工具',
  tools: { include: ['Glob', 'Grep', 'ReadFile', 'ListFiles', 'ReadManyFiles'] },
  modelTier: ModelTier.SECONDARY,
};

// steward：管家模式（主代理，最小工具集）
const stewardAgent: AgentConfig = {
  name: 'steward',
  role: 'primary',
  description: '管家模式，最小工具集，专注对话和阅读',
  tools: { include: ['ReadFile', 'ListFiles', 'ReadManyFiles'] },
};

// explanatory：解释型（主代理）
const explanatoryAgent: AgentConfig = {
  name: 'explanatory',
  role: 'primary',
  description: '解释型模式，穿插教育性说明',
  systemPromptBuilder: buildExplanatorySystemPrompt,
};
```

## 设计要点

- **统一 Agent 类**：主代理和子代理共用一个类，通过 config.role 区分行为
- **系统提示词优先级**：传入 > 静态配置 > 构建器
- **中断安全**：abort 后自动清理不完整的上下文
- **可扩展**：新增 Agent 类型只需添加预设配置
