# 第五章：会话管理

## 一句话概括

Pi 使用 JSONL 文件存储会话，每个会话是一个树状结构，通过 `parentId` 支持原地分支和会话复用。

## 架构图

```mermaid
%%{init: {'theme': 'neutral'}}%%
graph TB
    subgraph Storage
        JSONL[(session.jsonl)]
        HEADER[SessionHeader]
        MSG[Message Entry]
        COMPACT[Compaction Entry]
        BRANCH[Branch Summary]
        CUSTOM[Custom Entry]
    end

    subgraph InMemory
        SM[SessionManager]
        ENTRIES[SessionEntry[]]
    end

    JSONL --> SM
    SM --> ENTRIES

    subgraph Operations
        CREATE[create]
        CONTINUE[continueRecent]
        FORK[forkFrom]
        COMPACT[compact]
        NAVIGATE[navigate to turn]
    end

    SM --> CREATE
    SM --> CONTINUE
    SM --> FORK
    SM --> COMPACT
    SM --> NAVIGATE
```

## Session 文件格式

### JSONL 结构

每个会话保存为 `.jsonl` 文件（[session-manager.ts](file:///workspace/packages/coding-agent/src/core/session-manager.ts)），每行是一个 JSON 对象：

```jsonl
{"type":"session","id":"abc123","timestamp":"2024-01-15T10:00:00Z","cwd":"/project"}
{"type":"message","id":"msg1","parentId":"abc123","role":"user","content":"Hello"}
{"type":"message","id":"msg2","parentId":"msg1","role":"assistant","content":"Hi!"}
{"type":"compaction","id":"comp1","parentId":"msg2","summary":"...","tokensBefore":5000}
```

### SessionHeader

[session-manager.ts:32-39](file:///workspace/packages/coding-agent/src/core/session-manager.ts#L32-L39)：

```typescript
export interface SessionHeader {
    type: "session";
    version?: number;  // v1 会话没有此字段
    id: string;
    timestamp: string;
    cwd: string;
    parentSession?: string;  // fork 来源
}
```

### SessionEntry 类型

[session-manager.ts:140-149](file:///workspace/packages/coding-agent/src/core/session-manager.ts#L140-L149)：

```typescript
export type SessionEntry =
    | SessionMessageEntry       // 消息条目
    | ThinkingLevelChangeEntry // 思考级别变更
    | ModelChangeEntry          // 模型变更
    | CompactionEntry           // 压缩条目
    | BranchSummaryEntry        // 分支摘要
    | CustomEntry               // 扩展自定义数据（不参与 LLM）
    | CustomMessageEntry        // 扩展自定义消息（参与 LLM）
    | LabelEntry                // 标签/书签
    | SessionInfoEntry;         // 会话元信息
```

## SessionManager

### 核心操作

[session-manager.ts](file:///workspace/packages/coding-agent/src/core/session-manager.ts)：

```typescript
export class SessionManager {
    // 创建新会话
    static create(cwd: string, sessionDir?: string, options?: NewSessionOptions): SessionManager;

    // 打开现有会话
    static open(path: string, sessionDir?: string, cwd?: string): SessionManager;

    // 从现有会话 fork
    static forkFrom(sourcePath: string, cwd: string, sessionDir?: string, options?: NewSessionOptions): SessionManager;

    // 继续最近会话
    static continueRecent(cwd: string, sessionDir?: string): SessionManager;

    // 列出所有会话
    static list(cwd: string, sessionDir?: string): Promise<SessionInfo[]>;
    static listAll(sessionDir?: string): Promise<SessionInfo[]>;

    // 内存模式
    static inMemory(cwd: string): SessionManager;
}
```

### 读取和写入

```typescript
export class SessionManager {
    // 追加条目到会话
    appendEntry(entry: SessionEntry): void;

    // 读取所有条目
    getEntries(): SessionEntry[];

    // 读取特定条目的消息
    getMessages(): AgentMessage[];

    // 获取当前分支的叶子条目
    getActivePath(): SessionEntry[];

    // 跳转到历史中的某一点
    navigateTo(entryId: string): SessionManager;
}
```

## 会话树结构

### 树状存储

Pi 使用 `parentId` 字段实现树状结构：

```
session
├── msg1 (user)
│   ├── msg2 (assistant)
│   │   └── msg3 (assistant) ← 当前活跃分支
│   └── msg4 (assistant)     ← 被放弃的分支
└── msg5 (user)
```

### 获取活跃路径

```typescript
getActivePath(): SessionEntry[] {
    const leaf = this.getLeafEntry();
    const path: SessionEntry[] = [];

    let current: SessionEntry | undefined = leaf;
    while (current) {
        path.unshift(current);
        current = this.findById(current.parentId);
    }

    return path;
}
```

### navigateTo

跳转到历史中的任意点，继续从该点开始：

```typescript
navigateTo(entryId: string): SessionManager {
    const entry = this.findById(entryId);
    if (!entry) throw new Error("Entry not found");

    // 创建新的 SessionManager，parentId 指向前一个叶子
    return SessionManager.forkFrom(this.path, this.cwd, this.sessionDir, {
        id: this.id,
        parentSession: this.entries[this.entries.length - 1].id,
    });
}
```

## Compaction（会话压缩）

### 触发条件

[compaction/compaction.ts](file:///workspace/packages/coding-agent/src/core/compaction/compaction.ts)：

```typescript
export function shouldCompact(
    entries: SessionEntry[],
    maxTokens: number,
    model: Model,
): boolean {
    const tokens = estimateContextTokens(entries);
    return tokens > maxTokens * 0.9;  // 接近上限时触发
}
```

### 压缩流程

1. **findCutPoint**：找到要压缩的起始位置
2. **collectEntriesForBranchSummary**：收集分支摘要信息
3. **serializeConversation**：序列化要压缩的对话
4. **generateSummary**：调用 LLM 生成摘要
5. **compact**：用压缩条目替换原条目

### 压缩条目格式

```typescript
export interface CompactionEntry<T = unknown> extends SessionEntryBase {
    type: "compaction";
    summary: string;
    firstKeptEntryId: string;  // 压缩后保留的第一个条目 ID
    tokensBefore: number;      // 压缩前的 token 数
    details?: T;               // 扩展数据
    fromHook?: boolean;        // 是否由扩展生成
}
```

### 文件操作追踪

[compaction/utils.ts](file:///workspace/packages/coding-agent/src/core/compaction/utils.ts)：

```typescript
export function computeFileLists(messages: AgentMessage[]): FileOperations {
    const fileOps = createFileOps();

    for (const msg of messages) {
        // 从工具调用中提取文件操作
        extractFileOpsFromMessage(msg, fileOps);
    }

    return fileOps;
}
```

## Branch Summary（分支摘要）

用于在分支导航时总结被放弃的分支：

```typescript
export interface BranchSummaryEntry<T = unknown> extends SessionEntryBase {
    type: "branch_summary";
    fromId: string;       // 分支起点 ID
    summary: string;      // 分支摘要
    details?: T;
    fromHook?: boolean;   // 是否由扩展生成
}
```

## 自定义条目

### CustomEntry（不参与 LLM）

[session-manager.ts:100-104](file:///workspace/packages/coding-agent/src/core/session-manager.ts#L100-L104)：

```typescript
export interface CustomEntry<T = unknown> extends SessionEntryBase {
    type: "custom";
    customType: string;   // 扩展标识
    data?: T;             // 扩展数据
}
```

**用途**：扩展存储内部状态，重载时可恢复。

### CustomMessageEntry（参与 LLM）

[session-manager.ts:131-137](file:///workspace/packages/coding-agent/src/core/session-manager.ts#L131-L137)：

```typescript
export interface CustomMessageEntry<T = unknown> extends SessionEntryBase {
    type: "custom_message";
    customType: string;
    content: string | (TextContent | ImageContent)[];
    details?: T;
    display: boolean;  // 是否在 TUI 中显示
}
```

**用途**：扩展注入自定义消息到 LLM 上下文。

## 会话持久化

### 自动保存

[agent-session.ts](file:///workspace/packages/coding-agent/src/core/agent-session.ts)：

```typescript
private persistMessage(message: AgentMessage): void {
    this.sessionManager.appendEntry({
        type: "message",
        id: message.id,
        parentId: this.getParentId(),
        timestamp: new Date().toISOString(),
        message,
    });
}
```

### 手动保存

```typescript
session.save(): void {
    this.sessionManager.flush();
}
```

## 关键文件表

| 文件 | 行数 | 职责 |
|------|------|------|
| [packages/coding-agent/src/core/session-manager.ts](file:///workspace/packages/coding-agent/src/core/session-manager.ts) | 1575 | 会话管理器核心 |
| [packages/coding-agent/src/core/compaction/compaction.ts](file:///workspace/packages/coding-agent/src/core/compaction/compaction.ts) | 888 | 压缩逻辑 |
| [packages/coding-agent/src/core/messages.ts](file:///workspace/packages/coding-agent/src/core/messages.ts) | ~300 | 消息创建工具 |
| [packages/agent/src/harness/session/jsonl-repo.ts](file:///workspace/packages/agent/src/harness/session/jsonl-repo.ts) | ~300 | JSONL 存储实现 |
