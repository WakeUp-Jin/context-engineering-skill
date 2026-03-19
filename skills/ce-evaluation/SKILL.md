---
name: ce-evaluation
description: |
  Agent 评估模块 - 使提示词设计从"艺术"走向"工程"。
  包含评估方法论（代码评分/人工评分/模型评分/80-20数据集划分/pass@k与pass^k）、
  评估器实现（EventBus事件总线/TestCase结构/评估函数/Agent埋点/格式化输出）、
  不同类型Agent评估（编码Agent/对话Agent/研究Agent/计算机使用Agent的评估维度和策略）。
  当用户需要设计Agent评估方案、实现评估器、建立测试用例、
  或针对特定类型Agent制定评估策略时使用此 Skill。
  即使用户没有明确提到"评估"，只要涉及 Agent 质量度量、prompt 优化效果验证、
  Agent 稳定性测试、pass@k/pass^k 指标计算、AB 测试提示词、
  或者想知道如何科学地迭代改进 Agent 等话题，也应主动使用此 Skill。
---

# Agent 评估模块

评估用于提高 Agent 稳定性和发现模型边界，使提示词设计从"艺术"走向"工程"。没有评估，优化就是盲目的。

为什么 80/20 数据集划分？开发集用来迭代 prompt 和工具，留存集用来验证改进是否泛化。如果只在开发集上测试，prompt 会过拟合到这些 case 上，换个场景就不灵了。差距控制在 ~10% 以内说明没有过拟合。

## 评估组成

每个评估由四部分组成：
1. **示例输入**：测试用例
2. **标准答案**：期望的行为或输出
3. **模型输出**：Agent 实际的响应
4. **分数**：评估结果

## 推荐目录结构

```
evaluation/
├── types.ts       # 类型定义（TestCase、ExpectedBehavior 等）
├── EventBus.ts    # 事件总线（收集 Agent 和工具调用数据）
├── dataset.ts     # 测试数据集
├── evaluate.ts    # 评估逻辑
└── example.ts     # 使用示例
```

## 评估流程

```
准备测试用例（输入 + 期望行为）
        │
        ▼
  划分数据集
  ├── 80% 开发集（迭代优化用）
  └── 20% 留存集（验证泛化用）
        │
        ▼
  在开发集上迭代：
  ├── 运行 Agent
  ├── 收集执行数据（EventBus）
  ├── 评估结果
  └── 优化提示词/工具/上下文
        │
        ▼
  在留存集上验证
  （开发集与留存集准确率差距控制在约 10% 以内）
        │
        ▼
  发布
```

## 实现步骤指引

当帮用户实现评估系统时，按以下顺序推进：

1. **定义 TestCase 结构**：输入（userInput）+ 期望行为（expectedBehavior）+ 评估函数（参阅 [references/eval-methods.md](references/eval-methods.md)）
2. **实现 EventBus**：Agent 和工具调用的事件收集总线，用于记录执行过程（参阅 [references/eval-implementation.md](references/eval-implementation.md)）
3. **建立数据集**：准备 10-20 个测试用例，按 80/20 划分开发集和留存集
4. **实现评估逻辑**：根据 Agent 类型选择评分方式（代码评分、模型评分、人工评分）
5. **迭代优化**：在开发集上改进 prompt/工具/上下文，在留存集上验证泛化
6. **按 Agent 类型调整**：编码 Agent 看 pass@k，对话 Agent 看人工评分+模型评分（参阅 [references/eval-agent-types.md](references/eval-agent-types.md)）

## 细分文档

按需阅读 `references/` 目录下的详细文档：

| 你的需求 | 阅读文档 |
|---------|---------|
| 了解评估方法论和最佳实践 | [references/eval-methods.md](references/eval-methods.md) |
| 实现评估器（EventBus、测试用例、评估函数） | [references/eval-implementation.md](references/eval-implementation.md) |
| 针对不同类型 Agent 设计评估方案 | [references/eval-agent-types.md](references/eval-agent-types.md) |
