# 工具注册与发现

## ToolManager 职责

ToolManager 是工具系统的注册中心，负责：
- 工具注册与存储
- 按名称查询工具
- 执行工具调用
- 输出 OpenAI 格式的工具定义
- 按 Agent 配置过滤可用工具

## 核心实现

```typescript
class ToolManager {
  private tools: Map<string, InternalTool> = new Map();

  constructor() {
    this.registerAllTools();
  }

  /** 注册单个工具 */
  register(tool: InternalTool): void {
    this.tools.set(tool.name, tool);
  }

  /** 批量注册所有内置工具 */
  private registerAllTools(): void {
    const toolsList = [
      ListFilesTool, ReadFileTool, ReadManyFilesTool,
      WriteFileTool, TodoReadTool, TodoWriteTool,
      GlobTool, GrepTool, TaskTool,
    ];
    for (const tool of toolsList) {
      this.register(tool);
    }
  }

  /** 执行工具 */
  async execute(
    name: string,
    args: any,
    context?: InternalToolContext
  ): Promise<any> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Tool not found: ${name}`);
    return tool.handler(args, context);
  }

  /** 查询工具 */
  getTool(name: string): InternalTool | undefined {
    return this.tools.get(name);
  }

  getToolNames(): string[] {
    return Array.from(this.tools.keys());
  }

  hasTool(name: string): boolean {
    return this.tools.has(name);
  }

  /** 清空工具（用于测试） */
  clear(): void {
    this.tools.clear();
  }
}
```

## OpenAI Function 格式输出

将内部工具定义转换为 LLM API 期望的格式：

```typescript
getFormattedTools(): OpenAITool[] {
  return Array.from(this.tools.values()).map(tool => ({
    type: 'function' as const,
    function: {
      name: tool.name,
      description: tool.description,
      parameters: tool.parameters,
    },
  }));
}
```

## 按 Agent 配置过滤工具

不同 Agent 需要不同的工具集。通过 AgentConfig 的 `tools` 字段控制：

```typescript
interface AgentConfig {
  tools?: {
    include?: string[];   // 白名单：只允许这些工具
    exclude?: string[];   // 黑名单：排除这些工具
    [toolName: string]: boolean | string[];  // 细粒度控制
  };
}

getFilteredTools(agentConfig: AgentConfig, excludeTask?: boolean): InternalTool[] {
  let filtered = Array.from(this.tools.values());

  if (agentConfig.tools?.include) {
    const allowed = new Set(agentConfig.tools.include);
    filtered = filtered.filter(t => allowed.has(t.name));
  }

  if (agentConfig.tools?.exclude) {
    const excluded = new Set(agentConfig.tools.exclude);
    filtered = filtered.filter(t => !excluded.has(t.name));
  }

  if (excludeTask) {
    filtered = filtered.filter(t => t.name !== 'Task');
  }

  return filtered;
}
```

预设配置示例：

```typescript
// explore 预设：只允许只读工具
const exploreAgent: AgentConfig = {
  name: 'explore',
  role: 'subagent',
  tools: {
    include: ['Glob', 'Grep', 'ReadFile', 'ListFiles', 'ReadManyFiles'],
  },
};

// steward 预设：最小工具集
const stewardAgent: AgentConfig = {
  name: 'steward',
  role: 'primary',
  tools: {
    include: ['ReadFile', 'ListFiles', 'ReadManyFiles'],
  },
};
```

## 动态注册

除了构造时批量注册，ToolManager 也支持运行时动态添加工具：

```typescript
// MCP 工具动态注册
const mcpTools = await mcpClient.listTools();
for (const mcpTool of mcpTools) {
  toolManager.register(convertMCPToInternal(mcpTool));
}
```

## 设计要点

- **集中注册**：所有工具在 ToolManager 中统一管理，避免散落在各处
- **格式统一**：无论内部如何定义，对外输出统一的 OpenAI function 格式
- **按需过滤**：不同 Agent 角色看到不同的工具集，减少工具混淆
- **关注点分离**：ToolManager 只负责注册和查找，审核和调度交给 ToolScheduler
