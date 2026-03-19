# 工具定义规范

## 核心接口：InternalTool

每个工具都实现 `InternalTool` 接口，包含定义信息和执行逻辑：

```typescript
interface InternalTool<TArgs = any, TResult = any> {
  /** 工具名称（唯一标识） */
  name: string;

  /** 工具分类（如 filesystem、search、network） */
  category: string;

  /** 是否为内部工具 */
  internal: boolean;

  /** 工具描述（简短，详细描述放在 prompt 中） */
  description: string;

  /** 版本号 */
  version: string;

  /** 参数定义（JSON Schema 格式） */
  parameters: ToolParameterSchema;

  /** 工具执行函数（不传递给 LLM，仅系统内部调用） */
  handler: (args: TArgs, context?: InternalToolContext) => Promise<TResult>;

  /** 格式化工具结果给 AI（控制 LLM 看到什么） */
  renderResultForAssistant?: (result: TResult) => string;

  /** 是否只读（影响并发和审核策略） */
  isReadOnly?: () => boolean;

  /** 是否需要用户确认执行 */
  shouldConfirmExecute?: (
    args: TArgs,
    approvalMode: ApprovalMode,
    context?: InternalToolContext,
    allowlist?: Allowlist
  ) => Promise<ConfirmDetails | false>;
}
```

## 三层描述体系

工具的描述分为三个层次，帮助 LLM 准确理解何时使用工具：

1. **description**：简短描述，回答"这个工具做什么"
2. **purpose**：宏观功能定位，回答"这个工具为什么存在"、"它解决什么问题"
3. **useCase**：具体使用场景，回答"在什么情况下应该使用这个工具"

```typescript
// 示例：记忆搜索工具
{
  description: '搜索存储的记忆条目',
  purpose: 'Perform semantic search over stored memory entries to retrieve '
    + 'relevant knowledge and reasoning traces that can inform current decision-making.',
  useCase: 'Use when you need to find previously stored knowledge, code patterns, '
    + 'or technical information that may be relevant to answering current questions.',
}
```

清晰的描述直接影响 LLM 工具调用的成功率。工具数量超过 30 时，模型容易混淆，可用 RAG 做工具筛选。

## 参数定义（ToolParameterSchema）

参数使用 JSON Schema 格式，核心字段：

```typescript
interface ToolParameterDefinition {
  /** 参数数据类型 */
  type: 'string' | 'number' | 'boolean' | 'object' | 'array';

  /** 参数描述（帮助 LLM 理解参数用途） */
  description: string;

  /** 数值范围 */
  minimum?: number;
  maximum?: number;

  /** 数组长度限制 */
  minItems?: number;
  maxItems?: number;

  /** 字符串长度限制 */
  minLength?: number;
  maxLength?: number;

  /** 枚举值 */
  enum?: any[];

  /** 默认值 */
  default?: any;

  /** 数组元素类型 */
  items?: ToolParameterDefinition;
}
```

`type` 和 `description` 是必须的。参数命名本身也是描述的一部分，好的命名能减少 description 的负担。

## Zod Schema 方式（Kode/ClaudeCode 风格）

另一种工具定义风格使用 Zod Schema，支持运行时验证：

```typescript
interface Tool<TInput, TOutput> {
  name: string;
  description: () => Promise<string>;  // 异步描述（支持动态生成）
  inputSchema: z.ZodObject<any>;       // Zod 验证
  inputJSONSchema?: JSONSchema;        // 可选直接 JSON Schema
  call: (input: TInput) => AsyncGenerator<ToolOutput>;  // 流式输出
  isReadOnly: () => boolean;
  isConcurrencySafe: () => boolean;
  needsPermissions: (input: TInput) => boolean;
}
```

Zod `.describe()` 写清参数含义：

```typescript
const inputSchema = z.object({
  file_path: z.string().describe('要读取的文件的绝对路径或相对路径'),
  offset: z.number().optional().describe('从第几行开始读取，1-indexed'),
  limit: z.number().optional().describe('读取多少行'),
});
```

## 工具定义文件组织

每个工具一个目录，分离定义和执行：

```typescript
// ReadFile/definitions.ts — 工具定义
export const ReadFileTool: InternalTool = {
  name: 'ReadFile',
  category: 'filesystem',
  internal: true,
  description: '读取文件内容，支持分页读取',
  version: '1.0.0',
  parameters: {
    type: 'object',
    properties: {
      file_path: { type: 'string', description: '文件路径' },
      offset: { type: 'number', description: '起始行号' },
      limit: { type: 'number', description: '读取行数' },
    },
    required: ['file_path'],
  },
  handler: readFileExecutor,
  renderResultForAssistant: formatReadResult,
  isReadOnly: () => true,
  shouldConfirmExecute: async () => false,
};
```

```typescript
// ReadFile/executors.ts — 执行逻辑
export async function readFileExecutor(
  args: { file_path: string; offset?: number; limit?: number },
  context?: InternalToolContext
): Promise<ReadFileResult> {
  // 实际的文件读取逻辑
}

export function formatReadResult(result: ReadFileResult): string {
  // 格式化为 LLM 可理解的文本
}
```

## 统一结果接口

所有工具返回统一格式，便于上层处理：

```typescript
interface ToolResult<T> {
  success: boolean;
  error?: string;
  warning?: string;
  data: T | null;
}
```
