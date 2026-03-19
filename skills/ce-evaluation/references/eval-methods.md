# 评估方法论

## 三种评估方法

### 1. 代码评分

通过代码逻辑自动判断，速度快、可扩展：

```typescript
function codeGrade(output: string, expected: string): number {
  // 精确匹配
  if (output.trim() === expected.trim()) return 1.0;

  // 关键词匹配
  const keywords = extractKeywords(expected);
  const matched = keywords.filter(k => output.includes(k));
  return matched.length / keywords.length;
}
```

适用场景：
- 情感分析（正面/负面/中性）
- 分类任务
- 提取任务（特定字段是否存在）
- 格式检查（JSON 是否合法）

**示例：情感分析评估**

```typescript
const testCases = [
  { input: '这个产品太棒了！', expected: 'positive' },
  { input: '质量差得不行', expected: 'negative' },
  { input: '还行吧，一般般', expected: 'neutral' },
  { input: '"太好了"（讽刺语气）', expected: 'negative' },  // 反讽
  { input: '虽然有缺点，但总体不错', expected: 'positive' },  // 转折
];
```

通过失败用例优化提示词：V1 准确率 83.33% → 加入反讽和转折规则后 V2 达到 100%。

### 2. 人工评分

专家评审，处理主观质量判断：

适用场景：
- 文本生成质量
- UI 设计评审
- 对话自然度
- 创造性输出

成本高、耗时长，通常结合代码评分使用。

### 3. 模型评分

使用 LLM 作为评估者，平衡客观与主观：

```typescript
const EVAL_PROMPT = `
你是一个客观的评估者。请评估以下 Agent 输出的质量。

原始问题：{question}
Agent 输出：{output}
评估标准：{criteria}

请给出 1-5 分的评分，并说明理由。
要求：保持客观、中立、不偏向。
`;
```

评估模型的提示词要素：
- 原始提示/问题
- 待评估的输出
- 评估标准或指南
- 打分说明

## 评估流程最佳实践

### 数据集划分

```
总数据集（建议 ≥ 100 组）
├── 开发集（80%）：用于迭代优化
└── 留存集（20%）：用于验证泛化

验收标准：开发集与留存集准确率差距 ≤ 10%
```

### 迭代优化

1. 在开发集上运行 Agent
2. 分析失败用例
3. 优化提示词/工具配置
4. 重新运行，比较改进
5. 在留存集上验证
6. 差距过大 → 说明过拟合开发集，需要调整

## pass@k 与 pass^k

两个关键指标衡量 Agent 的不同维度：

### pass@k（潜力指标）

k 次尝试中至少成功一次的概率。衡量 Agent 的**上限**。

```
pass@k = 1 - C(n-c, k) / C(n, k)
其中 n = 总尝试次数，c = 成功次数
```

适用于：工具类产品（用户可以重试）

### pass^k（稳定性指标）

k 次尝试全部成功的概率。衡量 Agent 的**一致性**。

```
pass^k = (c/n)^k
```

适用于：面向最终用户的 Agent（每次都要成功）

### 选择建议

- 开发阶段用 pass@k：评估 Agent 是否"能"解决问题
- 上线前用 pass^k：评估 Agent 是否"稳定"解决问题
- 两者差距大 → Agent 能力够但稳定性不足 → 需要优化提示词和上下文

## 组合评估策略

单一方法不够全面，实践中组合使用：

```typescript
interface EvalStrategy {
  // 确定性测试（代码评分）
  deterministic_tests: TestCase[];

  // LLM 评分（模型评分）
  llm_rubric: RubricItem[];

  // 静态分析（代码质量）
  static_analysis: AnalysisRule[];

  // 状态检查（任务完成度）
  state_check: StateAssertion[];

  // 工具调用检查
  tool_calls: ToolCallAssertion[];
}
```
