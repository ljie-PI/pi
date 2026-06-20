# 第三章：Agent Loop 机制

## 一句话概括

Agent Loop 是 Pi 的核心引擎，负责管理消息循环、LLM 调用、工具执行和事件分发，将 `AgentMessage[]` 转换为流式事件序列。

## 架构图

```mermaid
%%{init: {'theme': 'neutral'}}%%
graph LR
    subgraph Input
        PM[prompts: AgentMessage[]]
        CT[context: AgentContext]
        CF[config: AgentLoopConfig]
    end

    subgraph Core
        RL[runLoop]
        SR[streamAssistantResponse]
        ET[executeToolCalls]
    end

    subgraph Events
        AS[agent_start]
        TS[turn_start]
        MS[message_start/msg_update/msg_end]
        TE[tool_execution_start/update/end]
        AE[agent_end]
    end

    PM --> RL
    CT --> RL
    CF --> RL
    RL --> AS
    RL --> TS
    RL --> MS
    RL --> ET
    RL --> AE
    SR --> MS
    ET --> TE
```

## 核心类型

### AgentLoopConfig

[agent/src/types.ts](file:///workspace/packages/agent/src/types.ts) 中定义：

```typescript
export interface AgentLoopConfig {
    model: Model<any>;
    apiKey?: string;
    getApiKey?: (provider: string) => Promise<string | undefined>;

    // Context 转换
    transformContext?: (messages: AgentMessage[], signal: AbortSignal) => Promise<AgentMessage[]>;
    convertToLlm: (messages: AgentMessage[]) => Promise<Message[]>;

    // 工具相关
    tools?: AgentTool[];
    toolExecution?: "parallel" | "sequential";
    beforeToolCall?: BeforeToolCallHook;
    afterToolCall?: AfterToolCallHook;

    // 生命周期
    prepareNextTurn?: (context: NextTurnContext) => Promise<NextTurnSnapshot | undefined>;
    shouldStopAfterTurn?: (context: StopContext) => Promise<boolean>;

    // 消息队列
    getSteeringMessages?: () => Promise<AgentMessage[] | undefined>;
    getFollowUpMessages?: () => Promise<AgentMessage[] | undefined>;

    // 其他
    extraHeaders?: Record<string, string>;
    signal?: AbortSignal;
}
```

### AgentEvent

[agent/src/types.ts](file:///workspace/packages/agent/src/types.ts) 中定义：

```typescript
export type AgentEvent =
    | { type: "agent_start" }
    | { type: "turn_start" }
    | { type: "turn_end"; message: AssistantMessage; toolResults: ToolResultMessage[] }
    | { type: "message_start"; message: AgentMessage }
    | { type: "message_end"; message: AgentMessage }
    | { type: "message_update"; assistantMessageEvent: StreamingEvent; message: AssistantMessage }
    | { type: "tool_execution_start"; toolCallId: string; toolName: string; args: Record<string, unknown> }
    | { type: "tool_execution_update"; toolCallId: string; toolName: string; partialResult: unknown }
    | { type: "tool_execution_end"; toolCallId: string; toolName: string; result: AgentToolResult; isError: boolean }
    | { type: "agent_end"; messages: AgentMessage[] };
```

## Agent Loop 生命周期

### 1. agentLoop()

[agent-loop.ts:31-54](file:///workspace/packages/agent/src/agent-loop.ts#L31-L54)：

```typescript
export function agentLoop(
    prompts: AgentMessage[],
    context: AgentContext,
    config: AgentLoopConfig,
    signal?: AbortSignal,
    streamFn?: StreamFn,
): EventStream<AgentEvent, AgentMessage[]> {
    const stream = createAgentStream();

    void runAgentLoop(
        prompts,
        context,
        config,
        async (event) => { stream.push(event); },
        signal,
        streamFn,
    ).then((messages) => { stream.end(messages); });

    return stream;
}
```

### 2. runAgentLoop()

[agent-loop.ts:95-118](file:///workspace/packages/agent/src/agent-loop.ts#L95-L118)：

```typescript
export async function runAgentLoop(
    prompts: AgentMessage[],
    context: AgentContext,
    config: AgentLoopConfig,
    emit: AgentEventSink,
    signal?: AbortSignal,
    streamFn?: StreamFn,
): Promise<AgentMessage[]> {
    const newMessages: AgentMessage[] = [...prompts];
    const currentContext: AgentContext = {
        ...context,
        messages: [...context.messages, ...prompts],
    };

    await emit({ type: "agent_start" });
    await emit({ type: "turn_start" });

    // 为每个 prompt 发送事件
    for (const prompt of prompts) {
        await emit({ type: "message_start", message: prompt });
        await emit({ type: "message_end", message: prompt });
    }

    await runLoop(currentContext, newMessages, config, signal, emit, streamFn);
    return newMessages;
}
```

### 3. runLoop() - 双层循环

[agent-loop.ts:155-269](file:///workspace/packages/agent/src/agent-loop.ts#L155-L269)：

```typescript
async function runLoop(
    initialContext: AgentContext,
    newMessages: AgentMessage[],
    initialConfig: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink,
    streamFn?: StreamFn,
): Promise<void> {
    let currentContext = initialContext;
    let config = initialConfig;
    let firstTurn = true;
    let pendingMessages: AgentMessage[] = [];

    // 外层循环：处理消息队列
    while (true) {
        let hasMoreToolCalls = true;

        // 内层循环：处理工具调用
        while (hasMoreToolCalls || pendingMessages.length > 0) {
            if (!firstTurn) {
                await emit({ type: "turn_start" });
            } else {
                firstTurn = false;
            }

            // 处理待处理消息
            if (pendingMessages.length > 0) {
                for (const message of pendingMessages) {
                    await emit({ type: "message_start", message });
                    await emit({ type: "message_end", message });
                    currentContext.messages.push(message);
                    newMessages.push(message);
                }
                pendingMessages = [];
            }

            // 流式获取 LLM 响应
            const message = await streamAssistantResponse(
                currentContext, config, signal, emit, streamFn
            );
            newMessages.push(message);

            // 错误处理
            if (message.stopReason === "error" || message.stopReason === "aborted") {
                await emit({ type: "turn_end", message, toolResults: [] });
                await emit({ type: "agent_end", messages: newMessages });
                return;
            }

            // 处理工具调用
            const toolCalls = message.content.filter(c => c.type === "toolCall");
            const toolResults: ToolResultMessage[] = [];
            hasMoreToolCalls = false;

            if (toolCalls.length > 0) {
                const executedToolBatch = await executeToolCalls(
                    currentContext, message, config, signal, emit
                );
                toolResults.push(...executedToolBatch.messages);
                hasMoreToolCalls = !executedToolBatch.terminate;

                for (const result of toolResults) {
                    currentContext.messages.push(result);
                    newMessages.push(result);
                }
            }

            await emit({ type: "turn_end", message, toolResults });

            // 准备下一轮
            const nextTurnSnapshot = await config.prepareNextTurn?.({
                message, toolResults, context: currentContext, newMessages
            });
            if (nextTurnSnapshot) {
                currentContext = nextTurnSnapshot.context ?? currentContext;
                config = {
                    ...config,
                    model: nextTurnSnapshot.model ?? config.model,
                    reasoning: nextTurnSnapshot.thinkingLevel === undefined
                        ? config.reasoning
                        : nextTurnSnapshot.thinkingLevel === "off"
                            ? undefined
                            : nextTurnSnapshot.thinkingLevel,
                };
            }

            // 检查停止条件
            if (await config.shouldStopAfterTurn?.({
                message, toolResults, context: currentContext, newMessages
            })) {
                await emit({ type: "agent_end", messages: newMessages });
                return;
            }

            pendingMessages = await config.getSteeringMessages?.() || [];
        }

        // 检查 follow-up 消息（外层循环继续条件）
        const followUpMessages = await config.getFollowUpMessages?.() || [];
        if (followUpMessages.length > 0) {
            pendingMessages = followUpMessages;
            continue;
        }

        break;
    }

    await emit({ type: "agent_end", messages: newMessages });
}
```

## 工具调用机制

### 执行模式

工具调用支持两种模式：

1. **parallel（默认）**：所有工具并行执行
2. **sequential**：工具按顺序执行（用于有依赖的工具调用）

### 串行执行

[agent-loop.ts:395-449](file:///workspace/packages/agent/src/agent-loop.ts#L395-L449)：

```typescript
async function executeToolCallsSequential(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    toolCalls: AgentToolCall[],
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
    const messages: ToolResultMessage[] = [];

    for (const toolCall of toolCalls) {
        // 发送开始事件
        await emit({
            type: "tool_execution_start",
            toolCallId: toolCall.id,
            toolName: toolCall.name,
            args: toolCall.arguments,
        });

        // 准备工具调用
        const preparation = await prepareToolCall(...);

        let finalized: FinalizedToolCallOutcome;
        if (preparation.kind === "immediate") {
            finalized = { toolCall, result: preparation.result, isError: preparation.isError };
        } else {
            const executed = await executePreparedToolCall(preparation, signal, emit);
            finalized = await finalizeExecutedToolCall(preparation, executed, ...);
        }

        // 发送结束事件
        await emitToolExecutionEnd(finalized, emit);
        const toolResultMessage = createToolResultMessage(finalized);
        await emitToolResultMessage(toolResultMessage, emit);
        messages.push(toolResultMessage);

        if (signal?.aborted) break;
    }

    return { messages, terminate: shouldTerminateToolBatch(messages) };
}
```

### 并行执行

[agent-loop.ts:451-516](file:///workspace/packages/agent/src/agent-loop.ts#L451-L516)：

```typescript
async function executeToolCallsParallel(
    currentContext: AgentContext,
    assistantMessage: AssistantMessage,
    toolCalls: AgentToolCall[],
    config: AgentLoopConfig,
    signal: AbortSignal | undefined,
    emit: AgentEventSink,
): Promise<ExecutedToolCallBatch> {
    const finalizedCalls: FinalizedToolCallEntry[] = [];

    for (const toolCall of toolCalls) {
        await emit({ type: "tool_execution_start", ... });

        const preparation = await prepareToolCall(...);

        if (preparation.kind === "immediate") {
            // 立即返回结果
            const finalized = { toolCall, result: preparation.result, isError: preparation.isError };
            await emitToolExecutionEnd(finalized, emit);
            finalizedCalls.push(finalized);
        } else {
            // 延迟执行
            finalizedCalls.push(async () => {
                const executed = await executePreparedToolCall(preparation, signal, emit);
                const finalized = await finalizeExecutedToolCall(preparation, executed, ...);
                await emitToolExecutionEnd(finalized, emit);
                return finalized;
            });
        }
    }

    // 等待所有工具完成
    const orderedFinalizedCalls = await Promise.all(
        finalizedCalls.map(entry => typeof entry === "function" ? entry() : Promise.resolve(entry))
    );

    const messages: ToolResultMessage[] = [];
    for (const finalized of orderedFinalizedCalls) {
        const toolResultMessage = createToolResultMessage(finalized);
        await emitToolResultMessage(toolResultMessage, emit);
        messages.push(toolResultMessage);
    }

    return { messages, terminate: shouldTerminateToolBatch(orderedFinalizedCalls) };
}
```

## 消息类型转换

### AgentMessage → Message

转换发生在 [agent-loop.ts:289](file:///workspace/packages/agent/src/agent-loop.ts#L289)：

```typescript
const llmMessages = await config.convertToLlm(messages);
```

`convertToLlm` 函数将 `AgentMessage[]` 转换为 LLM 兼容的 `Message[]`：

| AgentMessage.role | Message.role | 转换说明 |
|---|---|---|
| `user` | `user` | 直接转换 |
| `assistant` | `assistant` | 保留 content，可能包含 tool_calls |
| `toolResult` | (inline in next user msg) | 转换为下一条 user 消息的 content |

## 事件流

### EventStream 实现

[ai/src/utils/event-stream.ts](file:///workspace/packages/ai/src/utils/event-stream.ts)：

```typescript
export class EventStream<TEvent, TEnd> {
    private events: TEvent[] = [];
    private ended = false;
    private endValue?: TEnd;
    private listeners: Array<(event: TEvent) => void> = [];

    push(event: TEvent): void {
        this.events.push(event);
        for (const listener of this.listeners) {
            listener(event);
        }
    }

    end(value: TEnd): void {
        this.ended = true;
        this.endValue = value;
    }

    on(handler: (event: TEvent) => void): () => void {
        this.listeners.push(handler);
        // 立即发送所有已缓存的事件
        for (const event of this.events) {
            handler(event);
        }
        return () => {
            this.listeners = this.listeners.filter(l => l !== handler);
        };
    }
}
```

## 关键文件表

| 文件 | 行数 | 职责 |
|------|------|------|
| [packages/agent/src/agent-loop.ts](file:///workspace/packages/agent/src/agent-loop.ts) | 748 | Agent 循环核心逻辑 |
| [packages/agent/src/types.ts](file:///workspace/packages/agent/src/types.ts) | ~450 | 类型定义 |
| [packages/agent/src/agent.ts](file:///workspace/packages/agent/src/agent.ts) | 557 | Agent 类 |
| [packages/ai/src/utils/event-stream.ts](file:///workspace/packages/ai/src/utils/event-stream.ts) | ~100 | 事件流实现 |
