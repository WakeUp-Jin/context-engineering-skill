# 不同类型 Agent 的评估方法

不同 Agent 类型需要不同的评估维度和基准。

## 编码 Agent

### 结果评估

核心标准：**代码能跑、测试通过**。

```typescript
interface CodingAgentEval {
  // 代码是否可运行
  compiles: boolean;

  // 测试是否通过
  testsPass: boolean;

  // 基准测试
  benchmarks: 'SWE-bench' | 'Terminal-Bench' | 'custom';
}
```

### 过程评估

启发式规则检查代码质量：

```typescript
interface CodeQualityChecks {
  complexity: boolean;      // 圈复杂度是否合理
  duplication: boolean;     // 是否有重复代码
  naming: boolean;          // 命名是否规范
  security: boolean;        // 是否有安全隐患
  performance: boolean;     // 是否有性能问题
}
```

### 行为评估

Agent 是否合理使用工具：

```typescript
// 检查 Agent 是否使用了 Grep 来定位代码，而不是盲目搜索
const toolCallAssertion = {
  name: '使用搜索工具定位',
  check: (toolCalls: ToolCall[]) =>
    toolCalls.some(tc => tc.function.name === 'Grep'),
};
```

### 组合策略

```typescript
const codingEvalStrategy = {
  deterministic_tests: [...],  // 测试用例
  llm_rubric: [...],           // LLM 评审代码质量
  static_analysis: [...],      // 静态分析规则
  state_check: [...],          // 文件系统状态检查
  tool_calls: [...],           // 工具调用合理性
};
```

## 对话 Agent

### 状态验证

任务是否完成（如工单创建、退款处理）：

```typescript
interface ConversationalAgentEval {
  // 必须调用的工具
  requiredTools: string[];        // ['verify_identity', 'process_refund']

  // 最大对话轮次
  maxTurns: number;               // 10

  // 任务是否完成
  taskCompleted: boolean;

  // 交互质量
  interactionQuality: {
    empathy: number;              // 同理心 1-5
    accuracy: number;             // 信息准确性 1-5
    toolUsage: number;            // 工具使用合理性 1-5
  };
}
```

### 模拟用户

用 LLM 扮演用户进行多轮测试：

```typescript
async function simulateConversation(
  agent: Agent,
  userProfile: string,
  scenario: string,
  maxTurns: number
): Promise<ConversationLog> {
  const userLLM = await createLLMService({ provider: 'openai', model: 'gpt-4' });
  const log: Message[] = [];

  for (let turn = 0; turn < maxTurns; turn++) {
    // 模拟用户发言
    const userMsg = await userLLM.simpleChat(
      `你是 ${userProfile}。场景：${scenario}。请作为用户回复。`,
      JSON.stringify(log.slice(-4))
    );
    log.push({ role: 'user', content: userMsg });

    // Agent 回复
    const agentResult = await agent.run(userMsg);
    log.push({ role: 'assistant', content: agentResult.content });

    // 检查任务是否完成
    if (isTaskComplete(agentResult)) break;
  }

  return { log, turns: log.length / 2 };
}
```

参考基准：tau-Bench、tau2-Bench（多轮对话场景）。

## 研究 Agent

### 评估维度

```typescript
interface ResearchAgentEval {
  // 基础性：声明是否有来源支撑
  groundedness: number;       // 0-1

  // 覆盖性：相关来源是否被引用
  coverage: number;           // 0-1

  // 来源质量
  sourceQuality: number;      // 0-1

  // 综合准确性
  accuracy: number;           // 0-1
}
```

### 挑战

- 专家标准可能不一致
- 信息随时间变化
- 同一问题可能有多个合理答案

### 评估方法

参考 BrowseComp：答案易验证但问题难解，便于评估。

```typescript
// 问题设计原则
// 好问题：有明确答案，但需要多步搜索和推理才能找到
// 差问题：答案模糊或有争议
```

## 计算机使用 Agent

### 不仅看界面

UI 操作的评估不能只看截图，还需要验证**后端状态**：

```typescript
interface ComputerUseEval {
  // 界面状态
  uiState: {
    screenshot: string;       // 截图路径
    domSnapshot?: string;     // DOM 快照
  };

  // 后端状态验证（核心）
  backendState: {
    database: any;            // 数据库记录是否正确
    fileSystem: any;          // 文件是否创建/修改
    apiCalls: any;            // API 是否被正确调用
    orderStatus: any;         // 订单状态等业务数据
  };
}
```

### DOM vs 截图

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| DOM | 精确、省 token | 复杂页面 DOM 太大 | 文本密集页面 |
| 截图 | 直观、不受 DOM 复杂度影响 | 消耗视觉 token | 电商等视觉丰富页面 |

## 评估指标速查

| Agent 类型 | 核心指标 | 辅助指标 |
|-----------|---------|---------|
| 编码 Agent | 测试通过率、编译成功率 | 代码质量、工具使用合理性 |
| 对话 Agent | 任务完成率、轮次效率 | 同理心、信息准确性 |
| 研究 Agent | 基础性、覆盖性 | 来源质量、准确性 |
| 计算机使用 Agent | 后端状态正确率 | UI 操作效率 |

## 通用建议

1. **先定义成功标准**：在写评估代码之前，明确什么算"成功"
2. **从简单开始**：先用代码评分做基础验证，再加入 LLM 评分
3. **关注失败用例**：失败用例比成功用例更有价值
4. **区分能力和稳定性**：pass@k 看能力，pass^k 看稳定性
5. **持续评估**：模型更新、提示词修改后都要重新评估
