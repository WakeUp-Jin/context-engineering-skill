# 具体工具实现参考

9 种已实现工具的设计要点和实现模式，按类型分组。

## 文件读取类

### ReadFile

读取单个文件内容，支持分页。

```typescript
{
  name: 'ReadFile',
  category: 'filesystem',
  isReadOnly: () => true,
  parameters: {
    file_path: { type: 'string', description: '文件路径' },
    offset: { type: 'number', description: '起始行号（1-indexed）' },
    limit: { type: 'number', description: '读取行数' },
  },
}
```

关键实现细节：
- 超过 100K 字符时自动截断，保留头尾
- 输出带行号前缀（`LINE_NUMBER|LINE_CONTENT`），便于 LLM 引用
- offset/limit 支持分页读取大文件

### ReadManyFiles

批量读取多个文件，支持 glob 模式。

```typescript
{
  name: 'ReadManyFiles',
  parameters: {
    paths: { type: 'array', items: { type: 'string' }, description: '文件路径数组或 glob 模式' },
    include: { type: 'string', description: '包含的文件模式' },
    exclude: { type: 'string', description: '排除的文件模式' },
  },
}
```

关键限制：
- 单文件最大 50K 字符
- 总输出最大 200K 字符
- 最多 50 个文件
- 使用 `glob` 包展开 glob 模式

### ListFiles

列举目录下的文件。

```typescript
{
  name: 'ListFiles',
  parameters: {
    path: { type: 'string', description: '目录路径' },
  },
}
```

## 搜索类

### Grep

全文搜索，采用多策略降级模式。

```typescript
{
  name: 'Grep',
  parameters: {
    pattern: { type: 'string', description: '搜索的正则表达式' },
    path: { type: 'string', description: '搜索目录' },
    include: { type: 'string', description: '文件类型过滤' },
  },
}
```

**多策略降级**（按优先级）：

```
ripgrep → git grep → system grep → JavaScript 实现
```

每种策略失败后自动降级到下一个。策略选择逻辑：

```typescript
async function selectStrategy(): Promise<GrepStrategy> {
  if (await isRipgrepAvailable()) return new RipgrepStrategy();
  if (await isGitGrepAvailable()) return new GitGrepStrategy();
  if (await isSystemGrepAvailable()) return new SystemGrepStrategy();
  return new JavaScriptStrategy();
}
```

结果限制：每文件最多 100 条匹配，按文件修改时间排序。

### Glob

文件名模式匹配搜索。

```typescript
{
  name: 'Glob',
  parameters: {
    pattern: { type: 'string', description: 'glob 模式（如 **/*.ts）' },
    path: { type: 'string', description: '搜索根目录' },
  },
}
```

**策略**：优先使用 ripgrep 的 `--files` 模式（更快），降级到 glob npm 包。

**超时控制**：

```typescript
const result = await withTimeout(
  globSearch(pattern, options),
  60_000,  // 60 秒超时
  new AbortController().signal
);
```

## 写入类

### WriteFile

文件写入，支持覆盖和追加模式。

```typescript
{
  name: 'WriteFile',
  isReadOnly: () => false,  // 非只读
  parameters: {
    file_path: { type: 'string' },
    content: { type: 'string' },
    mode: { type: 'string', enum: ['overwrite', 'append'] },
  },
}
```

**审核逻辑**（唯一需要确认的工具）：
- YOLO 模式：不确认
- AUTO_EDIT 模式：不确认
- DEFAULT 模式：显示文件路径和内容预览，等待用户确认

## 任务管理类

### TodoRead / TodoWrite

会话级 TODO 列表管理。

```typescript
// TodoItem 结构
interface TodoItem {
  id: string;
  content: string;
  status: 'pending' | 'in_progress' | 'completed' | 'cancelled';
}
```

**TodoWrite 的合并机制**：

```typescript
{
  parameters: {
    todos: { type: 'array', items: { /* TodoItem */ } },
    merge: { type: 'boolean', description: '是否与现有 todos 合并' },
  },
}
```

- `merge: true`：按 id 合并更新，保留未提及的 todo
- `merge: false`：全量替换

**校验规则**：
- 同一时间只能有一个 `in_progress` 状态
- id 和 content 为必填
- status 必须是有效枚举值

**存储**：`Map<sessionId, TodoItem[]>`，会话隔离。

## 子代理类

### Task

通过 AgentManager 创建子代理并执行任务。

```typescript
{
  name: 'Task',
  parameters: {
    description: { type: 'string', description: '任务简述' },
    prompt: { type: 'string', description: '详细任务描述' },
    // 参数根据 agentManager.listSubAgents() 动态生成
  },
}
```

关键实现：
- 创建子会话 `Session.getOrCreateSubSession`
- 子工具进度通过 `emitToolProgress` 回传给父 ExecutionStream
- 子代理自动排除 `Task` 工具（防止无限递归）
- 描述和可选的 agent type 参数根据注册的子代理动态生成

```typescript
async handler(args: TaskArgs, context?: InternalToolContext) {
  const agent = agentManager.createAgent(args.agentType || 'explore', {
    role: 'subagent',
  });
  const session = Session.getOrCreateSubSession(context.sessionId);
  const result = await agent.run(args.prompt, { session });
  return result;
}
```

## 工具辅助（utils）

### 进程管理

```typescript
// spawn.ts — 子进程管理
async function runCommand(
  cmd: string,
  args: string[],
  options?: { timeout?: number; signal?: AbortSignal }
): Promise<{ stdout: string; stderr: string; exitCode: number }>;
```

### 错误处理

```typescript
// error-utils.ts
function isAbortError(err: unknown): boolean;
function isTimeoutError(err: unknown): boolean;

async function withTimeout<T>(
  promise: Promise<T>,
  ms: number,
  signal?: AbortSignal
): Promise<T>;
```

### 工具检测

```typescript
// tool-detection.ts — 检测外部工具可用性
async function isRipgrepAvailable(): Promise<boolean>;
async function isGitGrepAvailable(): Promise<boolean>;
async function isSystemGrepAvailable(): Promise<boolean>;
```

### Ripgrep 管理

```typescript
// ripgrep.ts — 自动下载和管理 ripgrep 二进制
async function ensureRipgrep(): Promise<string>;  // 返回 rg 路径
async function files(dir: string, options?: RgFilesOptions): Promise<string[]>;
async function search(pattern: string, dir: string, options?: RgSearchOptions): Promise<Match[]>;
```

## 通用实现模式总结

1. **定义和执行分离**：`definitions.ts` 放元信息，`executors.ts` 放逻辑
2. **多策略降级**：搜索类工具优先用最快的外部工具，失败后降级
3. **输出限制**：所有工具都有输出大小限制，防止上下文膨胀
4. **超时控制**：IO 密集型操作设超时，配合 AbortSignal
5. **只读标记**：正确标记 isReadOnly 影响并行策略和审核判断
