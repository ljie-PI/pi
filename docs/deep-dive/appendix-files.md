# 附录：关键文件索引

## 按包组织

### packages/coding-agent

#### 核心（src/core/）

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| [src/core/agent-session.ts](file:///workspace/packages/coding-agent/src/core/agent-session.ts) | 3148 | AgentSession 核心类 |
| [src/core/agent-session-runtime.ts](file:///workspace/packages/coding-agent/src/core/agent-session-runtime.ts) | ~1500 | AgentSession 运行时 |
| [src/core/agent-session-services.ts](file:///workspace/packages/coding-agent/src/core/agent-session-services.ts) | ~800 | 服务构建 |
| [src/core/sdk.ts](file:///workspace/packages/coding-agent/src/core/sdk.ts) | ~500 | SDK 导出 |
| [src/core/session-manager.ts](file:///workspace/packages/coding-agent/src/core/session-manager.ts) | 1575 | 会话管理器 |
| [src/core/settings-manager.ts](file:///workspace/packages/coding-agent/src/core/settings-manager.ts) | 1195 | 设置管理器 |
| [src/core/auth-storage.ts](file:///workspace/packages/coding-agent/src/core/auth-storage.ts) | ~400 | 认证存储 |
| [src/core/model-registry.ts](file:///workspace/packages/coding-agent/src/core/model-registry.ts) | 992 | 模型注册表 |
| [src/core/model-resolver.ts](file:///workspace/packages/coding-agent/src/core/model-resolver.ts) | ~500 | 模型解析 |
| [src/core/trust-manager.ts](file:///workspace/packages/coding-agent/src/core/trust-manager.ts) | ~300 | 信任管理 |
| [src/core/compaction/compaction.ts](file:///workspace/packages/coding-agent/src/core/compaction/compaction.ts) | 888 | 压缩逻辑 |
| [src/core/extensions/types.ts](file:///workspace/packages/coding-agent/src/core/extensions/types.ts) | 1606 | 扩展 API 类型 |
| [src/core/extensions/loader.ts](file:///workspace/packages/coding-agent/src/core/extensions/loader.ts) | ~400 | 扩展加载器 |
| [src/core/extensions/runner.ts](file:///workspace/packages/coding-agent/src/core/extensions/runner.ts) | 1135 | 扩展运行器 |
| [src/core/tools/index.ts](file:///workspace/packages/coding-agent/src/core/tools/index.ts) | ~150 | 工具注册表 |
| [src/core/tools/bash.ts](file:///workspace/packages/coding-agent/src/core/tools/bash.ts) | ~4000 | bash 工具 |
| [src/core/tools/read.ts](file:///workspace/packages/coding-agent/src/core/tools/read.ts) | ~1000 | read 工具 |
| [src/core/tools/write.ts](file:///workspace/packages/coding-agent/src/core/tools/write.ts) | ~700 | write 工具 |
| [src/core/tools/edit.ts](file:///workspace/packages/coding-agent/src/core/tools/edit.ts) | ~1200 | edit 工具 |
| [src/core/tools/find.ts](file:///workspace/packages/coding-agent/src/core/tools/find.ts) | ~1500 | find 工具 |
| [src/core/tools/grep.ts](file:///workspace/packages/coding-agent/src/core/tools/grep.ts) | ~1500 | grep 工具 |
| [src/core/tools/ls.ts](file:///workspace/packages/coding-agent/src/core/tools/ls.ts) | ~1000 | ls 工具 |
| [src/core/tools/file-mutation-queue.ts](file:///workspace/packages/coding-agent/src/core/tools/file-mutation-queue.ts) | ~150 | 文件变更队列 |

#### 入口和模式（src/）

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| [src/cli.ts](file:///workspace/packages/coding-agent/src/cli.ts) | ~50 | CLI 入口 |
| [src/main.ts](file:///workspace/packages/coding-agent/src/main.ts) | 837 | 主逻辑 |
| [src/config.ts](file:///workspace/packages/coding-agent/src/config.ts) | ~200 | 配置常量 |
| [src/modes/interactive/interactive-mode.ts](file:///workspace/packages/coding-agent/src/modes/interactive/interactive-mode.ts) | 5731 | 交互模式 |
| [src/modes/print-mode.ts](file:///workspace/packages/coding-agent/src/modes/print-mode.ts) | ~300 | 打印模式 |
| [src/modes/rpc/rpc-mode.ts](file:///workspace/packages/coding-agent/src/modes/rpc/rpc-mode.ts) | 774 | RPC 模式 |

#### CLI 组件（src/cli/）

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| [src/cli/args.ts](file:///workspace/packages/coding-agent/src/cli/args.ts) | ~600 | 参数解析 |
| [src/cli/startup-ui.ts](file:///workspace/packages/coding-agent/src/cli/startup-ui.ts) | ~400 | 启动 UI |
| [src/cli/session-picker.ts](file:///workspace/packages/coding-agent/src/cli/session-picker.ts) | ~300 | 会话选择器 |
| [src/cli/file-processor.ts](file:///workspace/packages/coding-agent/src/cli/file-processor.ts) | ~200 | 文件处理 |

### packages/agent

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| [src/agent-loop.ts](file:///workspace/packages/agent/src/agent-loop.ts) | 748 | Agent 循环 |
| [src/agent.ts](file:///workspace/packages/agent/src/agent.ts) | 557 | Agent 类 |
| [src/types.ts](file:///workspace/packages/agent/src/types.ts) | ~450 | 类型定义 |
| [src/base.ts](file:///workspace/packages/agent/src/base.ts) | ~50 | 基础导出 |
| [src/harness/agent-harness.ts](file:///workspace/packages/agent/src/harness/agent-harness.ts) | 1064 | 测试 harness |
| [src/harness/types.ts](file:///workspace/packages/agent/src/harness/types.ts) | 833 | Harness 类型 |
| [src/harness/session/session.ts](file:///workspace/packages/agent/src/harness/session/session.ts) | ~400 | 会话类 |
| [src/harness/compaction/compaction.ts](file:///workspace/packages/agent/src/harness/compaction/compaction.ts) | 762 | 压缩逻辑 |

### packages/ai

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| [src/models.generated.ts](file:///workspace/packages/ai/src/models.generated.ts) | 17213 | 模型元数据 |
| [src/base.ts](file:///workspace/packages/ai/src/base.ts) | ~100 | 基础 API |
| [src/stream.ts](file:///workspace/packages/ai/src/stream.ts) | ~200 | 流式处理 |
| [src/types.ts](file:///workspace/packages/ai/src/types.ts) | ~400 | 类型定义 |
| [src/providers/anthropic.ts](file:///workspace/packages/ai/src/providers/anthropic.ts) | 1251 | Anthropic Provider |
| [src/providers/openai-responses.ts](file:///workspace/packages/ai/src/providers/openai-responses.ts) | ~1500 | OpenAI Provider |
| [src/providers/openai-completions.ts](file:///workspace/packages/ai/src/providers/openai-completions.ts) | 1262 | OpenAI Completions |
| [src/providers/google.ts](file:///workspace/packages/ai/src/providers/google.ts) | ~1000 | Google Provider |
| [src/providers/amazon-bedrock.ts](file:///workspace/packages/ai/src/providers/amazon-bedrock.ts) | 1070 | AWS Bedrock |
| [src/providers/register-builtins.ts](file:///workspace/packages/ai/src/providers/register-builtins.ts) | ~100 | Provider 注册 |
| [src/utils/overflow.ts](file:///workspace/packages/ai/src/utils/overflow.ts) | ~100 | 溢出处理 |
| [src/utils/event-stream.ts](file:///workspace/packages/ai/src/utils/event-stream.ts) | ~100 | 事件流 |
| [src/utils/oauth/index.ts](file:///workspace/packages/ai/src/utils/oauth/index.ts) | ~200 | OAuth 支持 |

### packages/tui

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| [src/tui.ts](file:///workspace/packages/tui/src/tui.ts) | ~400 | TUI 核心 |
| [src/components/editor.ts](file:///workspace/packages/tui/src/components/editor.ts) | ~300 | 编辑器组件 |
| [src/components/text.ts](file:///workspace/packages/tui/src/components/text.ts) | ~100 | 文本组件 |
| [src/components/markdown.ts](file:///workspace/packages/tui/src/components/markdown.ts) | ~100 | Markdown |
| [src/components/image.ts](file:///workspace/packages/tui/src/components/image.ts) | ~100 | 图片组件 |
| [src/components/select-list.ts](file:///workspace/packages/tui/src/components/select-list.ts) | ~80 | 选择列表 |
| [src/components/input.ts](file:///workspace/packages/tui/src/components/input.ts) | ~80 | 输入组件 |
| [src/components/box.ts](file:///workspace/packages/tui/src/components/box.ts) | ~50 | 容器组件 |
| [src/terminal.ts](file:///workspace/packages/tui/src/terminal.ts) | ~50 | 终端接口 |
| [src/keys.ts](file:///workspace/packages/tui/src/keys.ts) | 1400 | 按键处理 |
| [src/keybindings.ts](file:///workspace/packages/tui/src/keybindings.ts) | ~30 | 按键绑定 |

## 统计信息

### 按行数 Top 30

| 排名 | 文件 | 行数 |
|------|------|------|
| 1 | packages/ai/src/models.generated.ts | 17213 |
| 2 | packages/coding-agent/src/modes/interactive/interactive-mode.ts | 5731 |
| 3 | packages/tui/test/editor.test.ts | 4051 |
| 4 | packages/coding-agent/src/core/agent-session.ts | 3148 |
| 5 | packages/coding-agent/src/core/package-manager.ts | 2588 |
| 6 | packages/tui/src/components/editor.ts | 2307 |
| 7 | packages/ai/scripts/generate-models.ts | 2166 |
| 8 | packages/coding-agent/src/modes/interactive/components/tree-selector.ts | 1386 |
| 9 | packages/tui/src/keys.ts | 1400 |
| 10 | packages/agent/test/agent-loop.test.ts | 1351 |

### 按类型分布

| 类型 | 文件数 | 总行数 |
|------|--------|--------|
| TypeScript (.ts) | ~400 | ~200000 |
| 测试 (.test.ts) | ~100 | ~50000 |
| 配置文件 (.json) | ~20 | ~5000 |

## 依赖关系图

```
packages/coding-agent
├── packages/agent
├── packages/ai
│   └── (无外部 AI 依赖)
└── packages/tui
    └── (无外部依赖)

packages/agent
├── (无 workspace 依赖)
└── (纯运行时)

packages/ai
└── (无 workspace 依赖)

packages/tui
└── (无 workspace 依赖)
```
