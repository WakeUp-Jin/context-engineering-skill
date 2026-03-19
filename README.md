# Context Engineering Skills

基于上下文工程的 Agent 后端开发技能集合。

核心理念：**开发重心在上下文的获取和整理，应用的关键核心是 LLM**。保证核心是 LLM，后续模型能力提升时 Agent 效果自然变好；将开发重心放在上下文，可以发挥应用开发者的能力和创造力。

## Skills

| Skill | 说明 | 适用场景 |
|-------|------|----------|
| [ce-overview](skills/ce-overview/) | 架构总览 | 从零开始搭建 Agent 后端、理解四大模块的协作关系、选择项目结构 |
| [ce-tool-management](skills/ce-tool-management/) | 工具管理 | 定义工具接口、注册管理工具、设计审核机制、管理执行状态、处理工具输出 |
| [ce-context-management](skills/ce-context-management/) | 上下文管理 | 设计上下文架构、实现压缩策略、管理 Token 预算、解决上下文问题 |
| [ce-llm-module](skills/ce-llm-module/) | LLM 模块 | 设计统一 LLM 接口、实现工厂模式、适配多供应商 |
| [ce-agent-patterns](skills/ce-agent-patterns/) | Agent 形态 | 构建单/多智能体、实现执行引擎、设计 Agent 交互模式 |
| [ce-evaluation](skills/ce-evaluation/) | Agent 评估 | 设计评估方案、实现评估器、针对不同类型 Agent 制定评估策略 |

## 安装

### 方式一：通过 Skills CLI 安装（推荐）

```bash
# 安装全部 Skills
npx skills add xjk/context-engineering-skill

# 安装单个 Skill
npx skills add xjk/context-engineering-skill/ce-tool-management
```

### 方式二：手动安装

```bash
git clone https://github.com/xjk/context-engineering-skill.git
cp -r context-engineering-skill/skills/ce-* ~/.cursor/skills/
```

只安装单个 Skill：

```bash
cp -r context-engineering-skill/skills/ce-tool-management ~/.cursor/skills/
```

### 支持的安装目录

| 平台 | 路径 |
|------|------|
| Cursor | `~/.cursor/skills/` |
| 通用 | `~/.agents/skills/` |

安装后重启 IDE 即可生效。每个 Skill 可独立安装，无互相依赖关系。

## 参考项目

- **CLI 模板项目**：`npm create context-template`，一键安装基础版脚手架
- **实践项目 ReasonCode**：进阶版的完整实现，工具审核、执行调度、上下文压缩等设计成熟
- **上下文工程实践指南**：系统化的方法论文档，涵盖从设计到评估的完整流程
