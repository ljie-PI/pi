# 第七章：交互模式

## 一句话概括

交互模式是 Pi 的核心用户体验层，通过 TUI 组件渲染消息、工具输出和编辑器，支持按键绑定、命令处理和主题切换。

## 架构图

```mermaid
%%{init: {'theme': 'neutral'}}%%
graph TB
    subgraph Core
        IM[InteractiveMode]
        SM[SessionManager]
        RT[Runtime]
    end

    subgraph TUI
        TUI[TUI 类]
        ED[Editor]
        MSG[MessageArea]
        FTR[Footer]
        OVL[Overlays]
    end

    subgraph Components
        UMC[UserMessage]
        AMC[AssistantMessage]
        TEC[ToolExecution]
        BXC[BashExecution]
        SUC[SessionSelector]
        MDC[ModelSelector]
    end

    IM --> SM
    IM --> RT
    IM --> TUI

    TUI --> ED
    TUI --> MSG
    TUI --> FTR
    TUI --> OVL

    MSG --> UMC
    MSG --> AMC
    MSG --> TEC
    MSG --> BXC

    OVL --> SUC
    OVL --> MDC
```

## InteractiveMode 核心

### 初始化

[modes/interactive/interactive-mode.ts](file:///workspace/packages/coding-agent/src/modes/interactive/interactive-mode.ts)：

```typescript
export class InteractiveMode {
    constructor(
        private runtime: AgentSessionRuntime,
        private options: InteractiveModeOptions,
    ) {}

    async initialize(): Promise<void> {
        // 1. 初始化 TUI
        this.tui = new TUI({
            terminal: this.terminal,
            onInput: (input) => this.handleInput(input),
        });

        // 2. 设置组件
        this.setupComponents();

        // 3. 订阅会话事件
        this.subscribeToSessionEvents();

        // 4. 初始化主题
        this.themeController = new InteractiveThemeController(this.tui);

        // 5. 启动 TUI
        await this.tui.start();
    }
}
```

### 主循环

```typescript
async run(): Promise<void> {
    await this.initialize();

    // 主循环
    while (!this.isTerminated()) {
        await this.handleNextInput();
    }

    await this.shutdown();
}

private async handleNextInput(): Promise<void> {
    const input = await this.tui.readInput();

    switch (input.type) {
        case "text":
            // 文本输入
            break;
        case "command":
            // 命令输入（/ 开头的）
            await this.handleCommand(input.text);
            break;
        case "key":
            // 按键输入
            await this.handleKey(input.key);
            break;
    }
}
```

## TUI 组件系统

### TUI 类

[tui.ts](file:///workspace/packages/tui/src/tui.ts)：

```typescript
export class TUI {
    private children: Component[] = [];
    private overlay: Component | null = null;
    private dirty = true;

    constructor(options: TUIOptions) {
        this.terminal = options.terminal;
        this.onInput = options.onInput;
    }

    start(): void {
        // 启动终端输入监听
        this.terminal.onData((data) => this.handleInput(data));

        // 启动渲染循环
        this.renderLoop();
    }

    requestRender(): void {
        this.dirty = true;
    }
}
```

### 差分渲染

```typescript
private async renderLoop(): Promise<void> {
    while (this.running) {
        if (this.dirty) {
            const lines = this.renderAll();
            await this.terminal.render(lines);
            this.dirty = false;
        }
        await sleep(RENDER_INTERVAL);
    }
}

private renderAll(): string[] {
    const lines: string[] = [];

    // 渲染子组件
    for (const child of this.children) {
        lines.push(...child.render(this.width));
    }

    // 渲染覆盖层
    if (this.overlay) {
        lines.push(...this.overlay.render(this.width));
    }

    return lines;
}
```

## 组件模型

### Component 接口

```typescript
export interface Component {
    render(width: number): string[];
    handleInput?(key: KeyEvent): boolean | "consumed";
    invalidate?(): void;
}
```

### 内置组件

| 组件 | 文件 | 用途 |
|------|------|------|
| `Editor` | [components/editor.ts](file:///workspace/packages/tui/src/components/editor.ts) | 多行文本输入 |
| `Input` | [components/input.ts](file:///workspace/packages/tui/src/components/input.ts) | 单行输入 |
| `Text` | [components/text.ts](file:///workspace/packages/tui/src/components/text.ts) | 文本显示 |
| `Box` | [components/box.ts](file:///workspace/packages/tui/src/components/box.ts) | 容器/背景 |
| `Markdown` | [components/markdown.ts](file:///workspace/packages/tui/src/components/markdown.ts) | Markdown 渲染 |
| `Image` | [components/image.ts](file:///workspace/packages/tui/src/components/image.ts) | 图片显示 |
| `SelectList` | [components/select-list.ts](file:///packages/tui/src/components/select-list.ts) | 选择列表 |
| `Loader` | [components/loader.ts](file:///workspace/packages/tui/src/components/loader.ts) | 加载指示器 |
| `SettingsList` | [components/settings-list.ts](file:///workspace/packages/tui/src/components/settings-list.ts) | 设置列表 |

## 消息渲染

### AssistantMessageComponent

[modes/interactive/components/assistant-message.ts](file:///workspace/packages/coding-agent/src/modes/interactive/components/assistant-message.ts)：

```typescript
export class AssistantMessageComponent implements Component {
    render(width: number): string[] {
        const lines: string[] = [];

        // 渲染思考块（如果展开）
        if (this.showThinking && this.message.thinking) {
            lines.push(...this.renderThinking(width));
        }

        // 渲染文本内容
        for (const content of this.message.content) {
            if (content.type === "text") {
                lines.push(...this.renderText(content.text, width));
            }
            if (content.type === "toolCall") {
                lines.push(...this.renderToolCall(content, width));
            }
        }

        return lines;
    }
}
```

### ToolExecutionComponent

[modes/interactive/components/tool-execution.ts](file:///workspace/packages/coding-agent/src/modes/interactive/components/tool-execution.ts)：

```typescript
export class ToolExecutionComponent implements Component {
    render(width: number): string[] {
        const lines: string[] = [];

        // 工具名称
        lines.push(`${this.toolName}...`);

        // 如果有部分结果，流式更新
        if (this.partialResult) {
            lines.push(...this.renderPartialResult(this.partialResult, width));
        }

        // 如果完成，渲染最终结果
        if (this.isComplete) {
            lines.push(...this.renderResult(this.result, width));
        }

        return lines;
    }
}
```

## Footer

### FooterDataProvider

[modes/interactive/components/footer.ts](file:///workspace/packages/coding-agent/src/modes/interactive/components/footer.ts)：

```typescript
export class FooterComponent implements Component {
    render(width: number): string[] {
        const parts: string[] = [];

        // 工作目录
        parts.push(this.cwd);

        // 会话名称
        if (this.sessionName) {
            parts.push(this.sessionName);
        }

        // Token 使用
        const tokens = this.tokenUsage;
        parts.push(`↑${tokens.input} ↓${tokens.output}`);
        if (tokens.cacheRead > 0) {
            parts.push(`R${tokens.cacheRead}`);
        }
        if (tokens.cacheWrite > 0) {
            parts.push(`W${tokens.cacheWrite}`);
        }

        // 成本
        if (this.cost) {
            parts.push(`$${this.cost.toFixed(4)}`);
        }

        // 当前模型
        parts.push(this.model);

        return [parts.join(" | ")];
    }
}
```

## 主题系统

### Theme 结构

[modes/interactive/theme/theme.ts](file:///workspace/packages/coding-agent/src/modes/interactive/theme/theme.ts)：

```typescript
export interface Theme {
    name: string;
    colors: {
        background: string;
        foreground: string;
        // ... 其他颜色
    };
    styles: {
        userMessage: string;
        assistantMessage: string;
        toolCall: string;
        toolResult: string;
        error: string;
        // ...
    };
}
```

### 内置主题

- `dark` - 深色主题
- `light` - 浅色主题

### 主题热重载

```typescript
export class InteractiveThemeController {
    startWatching(): void {
        // 监听主题文件变化
        watchFile(this.themePath, () => {
            this.reloadTheme();
        });
    }

    private reloadTheme(): void {
        const theme = loadTheme(this.themePath);
        this.tui.applyTheme(theme);
        this.tui.requestRender();
    }
}
```

## 命令处理

### Slash Commands

```typescript
async handleCommand(text: string): Promise<void> {
    const parts = text.slice(1).split(/\s+/);
    const commandName = parts[0];
    const args = parts.slice(1);

    switch (commandName) {
        case "model":
            await this.showModelSelector();
            break;
        case "settings":
            await this.showSettings();
            break;
        case "compact":
            await this.session.compact(args.join(" "));
            break;
        case "new":
            await this.startNewSession();
            break;
        case "tree":
            await this.showTreeView();
            break;
        case "trust":
            await this.handleTrustCommand(args);
            break;
        // ...
    }
}
```

## 事件订阅

```typescript
private subscribeToSessionEvents(): void {
    // 会话变更时重新渲染
    this.session.onSessionChange(() => {
        this.requestRender();
    });

    // Token 使用变更时更新 footer
    this.session.onTokenUsageChange((usage) => {
        this.footer.update(usage);
    });

    // 工具执行更新
    this.session.onToolExecutionUpdate((event) => {
        this.updateToolExecution(event);
    });

    // 消息更新
    this.session.onMessageUpdate((event) => {
        this.updateMessage(event);
    });
}
```

## 关键文件表

| 文件 | 行数 | 职责 |
|------|------|------|
| [modes/interactive/interactive-mode.ts](file:///workspace/packages/coding-agent/src/modes/interactive/interactive-mode.ts) | 5731 | 交互模式主类 |
| [packages/tui/src/tui.ts](file:///workspace/packages/tui/src/tui.ts) | ~400 | TUI 核心类 |
| [packages/tui/src/components/editor.ts](file:///workspace/packages/tui/src/components/editor.ts) | ~300 | 编辑器组件 |
| [modes/interactive/components/assistant-message.ts](file:///workspace/packages/coding-agent/src/modes/interactive/components/assistant-message.ts) | ~500 | 助手消息渲染 |
| [modes/interactive/components/footer.ts](file:///workspace/packages/coding-agent/src/modes/interactive/components/footer.ts) | ~300 | Footer 组件 |
| [modes/interactive/theme/theme.ts](file:///workspace/packages/coding-agent/src/modes/interactive/theme/theme.ts) | 1261 | 主题系统 |
