# 评估器实现

基于事件总线收集 Agent 执行数据，对比期望行为，输出评估结果。

## 测试用例结构

```typescript
interface TestCase {
  id: string;
  description: string;
  input: string;                      // 用户输入
  expected: ExpectedBehavior;         // 期望行为
}

interface ExpectedBehavior {
  /** 期望使用的 Agent 名称列表 */
  agents: string[];

  /** 期望的工具调用：{ agentName: [toolNames] } */
  tools: Record<string, string[]>;
}
```

示例：

```typescript
const testCase: TestCase = {
  id: 'S1',
  description: '单工具调用：读取文件',
  input: '请读取 package.json 并告诉我项目名称',
  expected: {
    agents: ['build'],
    tools: {
      build: ['ReadFile'],
    },
  },
};
```

## EventBus 事件总线

单例事件总线，在 Agent 执行过程中收集数据：

```typescript
class EventBus {
  private static instance: EventBus;
  private listeners: Map<string, Function[]> = new Map();

  // 收集的数据
  private data: CollectedData = {
    agents: [],
    tools: {},
    editResult: null,
  };

  static getInstance(): EventBus {
    if (!EventBus.instance) {
      EventBus.instance = new EventBus();
    }
    return EventBus.instance;
  }

  emit(event: string, payload: any): void {
    switch (event) {
      case 'agent:call':
        this.data.agents.push(payload.agentName);
        break;
      case 'tool:call':
        if (!this.data.tools[payload.agentName]) {
          this.data.tools[payload.agentName] = [];
        }
        this.data.tools[payload.agentName].push(payload.toolName);
        break;
      case 'edit:complete':
        this.data.editResult = payload;
        break;
    }

    // 通知监听器
    const handlers = this.listeners.get(event) || [];
    handlers.forEach(fn => fn(payload));
  }

  getData(): CollectedData {
    return { ...this.data };
  }

  reset(): void {
    this.data = { agents: [], tools: {}, editResult: null };
  }
}
```

## Agent 侧埋点

在 Agent 和工具执行时发送事件：

```typescript
// Agent 执行时
class ExecutionEngine {
  async execute(userInput: string) {
    eventBus.emit('agent:call', { agentName: this.agentName });

    // ... 执行循环 ...

    // 工具调用时
    for (const tc of response.tool_calls) {
      eventBus.emit('tool:call', {
        agentName: this.agentName,
        toolName: tc.function.name,
      });
    }
  }
}
```

## 评估函数

对比期望行为和实际执行数据：

```typescript
interface EvaluateResult {
  passed: boolean;
  agentMatch: {
    passed: boolean;
    missing: string[];    // 期望但未调用的 Agent
    extra: string[];      // 未期望但调用的 Agent
  };
  toolMatch: {
    passed: boolean;
    details: Record<string, {
      missing: string[];  // 期望但未调用的工具
      extra: string[];    // 未期望但调用的工具
    }>;
  };
}

function evaluate(testCase: TestCase, actual: CollectedData): EvaluateResult {
  // Agent 匹配
  const expectedAgents = new Set(testCase.expected.agents);
  const actualAgents = new Set(actual.agents);
  const missingAgents = [...expectedAgents].filter(a => !actualAgents.has(a));
  const extraAgents = [...actualAgents].filter(a => !expectedAgents.has(a));

  // 工具匹配
  const toolDetails: Record<string, any> = {};
  for (const [agent, expectedTools] of Object.entries(testCase.expected.tools)) {
    const actualTools = actual.tools[agent] || [];
    const expectedSet = new Set(expectedTools);
    const actualSet = new Set(actualTools);

    toolDetails[agent] = {
      missing: [...expectedSet].filter(t => !actualSet.has(t)),
      extra: [...actualSet].filter(t => !expectedSet.has(t)),
    };
  }

  const agentPassed = missingAgents.length === 0;
  const toolPassed = Object.values(toolDetails)
    .every((d: any) => d.missing.length === 0);

  return {
    passed: agentPassed && toolPassed,
    agentMatch: { passed: agentPassed, missing: missingAgents, extra: extraAgents },
    toolMatch: { passed: toolPassed, details: toolDetails },
  };
}
```

## 格式化输出

```typescript
function formatResult(result: EvaluateResult): string {
  const lines: string[] = [];
  lines.push(result.passed ? '✅ PASSED' : '❌ FAILED');

  if (!result.agentMatch.passed) {
    lines.push(`  Agent 不匹配：`);
    if (result.agentMatch.missing.length) {
      lines.push(`    缺少: ${result.agentMatch.missing.join(', ')}`);
    }
  }

  if (!result.toolMatch.passed) {
    for (const [agent, detail] of Object.entries(result.toolMatch.details)) {
      if (detail.missing.length) {
        lines.push(`  工具不匹配 [${agent}]：缺少 ${detail.missing.join(', ')}`);
      }
    }
  }

  return lines.join('\n');
}
```

## 运行评估

```typescript
async function runTest(testCase: TestCase): Promise<EvaluateResult> {
  // 重置事件总线
  eventBus.reset();

  // 创建 Agent 并运行
  const agent = agentManager.createAgent('build');
  await agent.run(testCase.input);

  // 收集数据并评估
  const actual = eventBus.getData();
  return evaluate(testCase, actual);
}

// 批量运行
async function runAllTests(): Promise<void> {
  const tests = getSimpleAgentTests();
  for (const test of tests) {
    const result = await runTest(test);
    console.log(`${test.id}: ${formatResult(result)}`);
  }
}
```

## 设计建议

- **单 Agent 评估**：主要验证工具调用是否正确
- **多 Agent 评估**：额外验证主 Agent 的协调行为和子 Agent 选择
- **复杂 Agent**：使用事件总线收集执行数据
- **简单 Agent**：可以直接从返回结果中评估，不需要事件总线
