# 工具审核设计

工具审核是 Agent 安全的关键——不是所有工具调用都应该自动执行。写入文件、执行命令等操作需要用户确认。

## 审核模式（ApprovalMode）

三级审核策略，从严格到宽松：

```typescript
enum ApprovalMode {
  /** 默认模式：写入类操作需要确认 */
  DEFAULT = 'default',

  /** 自动编辑模式：编辑类操作自动批准 */
  AUTO_EDIT = 'auto-edit',

  /** YOLO 模式：全部自动批准（危险） */
  YOLO = 'yolo',
}
```

## 工具级审核：shouldConfirmExecute

每个工具通过 `shouldConfirmExecute` 方法决定是否需要用户确认：

```typescript
// WriteFile 的审核逻辑
shouldConfirmExecute: async (
  args: WriteFileArgs,
  approvalMode: ApprovalMode,
  context?: InternalToolContext,
  allowlist?: Allowlist
): Promise<ConfirmDetails | false> => {
  // YOLO 模式：不确认
  if (approvalMode === ApprovalMode.YOLO) return false;

  // AUTO_EDIT 模式：编辑类不确认
  if (approvalMode === ApprovalMode.AUTO_EDIT) return false;

  // DEFAULT 模式：返回确认详情，供 UI 展示
  return {
    type: 'file-write',
    panelTitle: '确认文件写入',
    filePath: args.file_path,
    fileName: path.basename(args.file_path),
    contentPreview: args.content.slice(0, 500),
    onConfirm: undefined,  // 由 ToolScheduler 注入
  };
}

// ReadFile 的审核逻辑（只读工具不需要确认）
shouldConfirmExecute: async () => false
```

## 确认详情（ConfirmDetails）

返回给 UI 层的确认信息结构：

```typescript
interface ConfirmDetails {
  /** 确认类型 */
  type: string;

  /** UI 面板标题 */
  panelTitle: string;

  /** 涉及的文件路径 */
  filePath?: string;

  /** 文件名 */
  fileName?: string;

  /** 内容预览 */
  contentPreview?: string;

  /** 要执行的命令（Shell 工具） */
  command?: string;

  /** 额外消息 */
  message?: string;

  /** 确认回调（由 ToolScheduler 注入） */
  onConfirm?: (outcome: ConfirmOutcome) => void;
}
```

## 确认结果（ConfirmOutcome）

用户可以选择三种结果：

```typescript
type ConfirmOutcome = 'once' | 'always' | 'cancel';
```

- **once**：仅本次批准
- **always**：本次批准并加入白名单（后续同类操作不再确认）
- **cancel**：取消执行

## 会话级白名单（Allowlist）

Allowlist 缓存已批准的操作标识，避免重复确认：

```typescript
class Allowlist {
  private approved: Set<string> = new Set();

  has(key: string): boolean {
    return this.approved.has(key);
  }

  add(key: string): void {
    this.approved.add(key);
  }

  remove(key: string): void {
    this.approved.delete(key);
  }

  clear(): void {
    this.approved.clear();
  }

  get size(): number {
    return this.approved.size;
  }
}
```

典型使用场景：Shell 工具批准某个命令根名后，同类命令不再确认。

## 审核流程在 ToolScheduler 中的完整实现

```
收到 tool_call
    │
    ▼
解析参数（deepParseArgs）
    │
    ▼
获取工具实例（ToolManager.getTool）
    │
    ▼
检查是否需要确认（tool.shouldConfirmExecute）
    │
    ├── 返回 false → 直接进入执行
    │
    └── 返回 ConfirmDetails
         │
         ▼
    状态 → awaiting_approval
         │
         ▼
    通过 onConfirmRequired 回调通知 UI
         │
         ├── 用户选 'once' → 执行，不记录白名单
         ├── 用户选 'always' → 执行，记录到 Allowlist
         └── 用户选 'cancel' → 状态 → cancelled
```

## 设计原则

- **默认安全**：写入类操作默认需要确认，只读操作默认不确认
- **渐进信任**：用户可以选择 `always` 建立信任，减少后续打扰
- **灵活策略**：通过 ApprovalMode 全局调整审核严格度
- **UI 解耦**：ToolScheduler 不关心 UI 如何展示确认框，只通过回调通信
