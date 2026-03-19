# 多智能体设计

## 何时使用多智能体

单智能体优先的原则下，以下场景考虑多智能体：

- **大量并行任务**：多个独立子任务可同时执行
- **超单窗口信息量**：信息总量超过单模型上下文窗口
- **复杂工具交互**：工具数量 > 30 或需要特化的工具集
- **以"读取/研究"为主**：探索型任务天然适合分治

单智能体优先的场景：
- 强共享上下文、互相依赖
- 以"写入/编辑"为主（如代码编辑）
- 需要连续的上下文

## AgentManager

管理 Agent 的注册、创建和生命周期：

```typescript
class AgentManager {
  private configs: Map<string, AgentConfig> = new Map();
  private toolManager: ToolManager | null = null;

  constructor() {
    // 注册内置预设
    this.registerConfig(buildAgent);
    this.registerConfig(exploreAgent);
    this.registerConfig(stewardAgent);
    this.registerConfig(explanatoryAgent);
  }

  /** 延迟初始化 ToolManager（避免循环依赖） */
  initToolManager(toolManager: ToolManager): void {
    this.toolManager = toolManager;
  }

  /** 创建 Agent（可合并覆盖配置） */
  createAgent(name: string, overrides?: Partial<AgentConfig>): Agent {
    const baseConfig = this.configs.get(name);
    if (!baseConfig) throw new Error(`Unknown agent: ${name}`);

    const config = overrides ? { ...baseConfig, ...overrides } : baseConfig;
    const runtime = this.createRuntime(config);

    return new Agent(config, runtime);
  }

  /** 列出可用的子代理（供 Task 工具用） */
  listSubAgents(): AgentConfig[] {
    return Array.from(this.configs.values())
      .filter(c => c.role === 'subagent' || c.role === 'all');
  }

  /** 列出主代理（供 UI 切换用） */
  listPrimaryAgents(): AgentConfig[] {
    return Array.from(this.configs.values())
      .filter(c => (c.role === 'primary' || c.role === 'all') && !c.hidden);
  }
}
```

## 常见多智能体架构

### 1. 监督者模式（Supervisor）

中控协调者分配任务给子 Agent：

```
MainAgent（协调者）
    ├── SubAgent-A（代码探索）
    ├── SubAgent-B（文件编辑）
    └── SubAgent-C（测试执行）
```

协调者的关键能力：
- 理解用户意图，拆分为子任务
- 为每个子任务选择合适的子 Agent
- 汇总子 Agent 结果

### 2. 群体模式（Swarm）

子 Agent 按能力动态交接控制权：

```
Agent-A（识别到需要搜索）→ 交接给 Agent-B（搜索专家）
Agent-B（搜索完成）→ 交接给 Agent-C（分析专家）
```

### 3. 层级模式（Hierarchical）

多级管理结构：

```
总控 Agent
├── 前端 Lead Agent
│   ├── UI Agent
│   └── 样式 Agent
└── 后端 Lead Agent
    ├── API Agent
    └── 数据库 Agent
```

## 子代理任务分派

协调者分派任务时，描述要清晰：

```typescript
// Task 工具调用示例
{
  description: "探索用户认证模块",
  prompt: `
    目标：了解用户认证模块的代码结构
    输出格式：列出关键文件和它们的职责
    可用工具：Glob、Grep、ReadFile
    搜索起点：src/auth/
  `,
  agentType: 'explore',
}
```

## 子代理规模控制

按任务复杂度控制：
- **简单任务**：1 个子代理
- **中等任务**：2-4 个子代理
- **复杂任务**：10+ 个子代理

## 多智能体的上下文挑战

### 上下文断裂

主代理和子代理、子代理和子代理之间的上下文不连续。

解决方案：
- 任务描述中包含完整的背景信息
- 子代理结果通过 Task 工具回传给主代理
- 子会话独立管理（`Session.getOrCreateSubSession`）

### 工具混淆

工具数 > 30 时模型容易混淆。

解决方案：
- 每个子代理只暴露需要的工具子集
- 可用 RAG 做工具筛选

## 设计要点

- **预设驱动**：通过预设配置定义 Agent 类型，不需要写新的 Agent 类
- **延迟初始化**：ToolManager 延迟注入，避免循环依赖
- **子代理安全**：自动排除 Task 工具，防止无限递归
- **渐进复杂度**：从单 Agent 开始，按需演进为多 Agent
