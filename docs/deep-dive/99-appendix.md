# 99 · 附录：文件索引、术语表、环境变量

> 速查手册。带着具体问题来这里"查字典"，不要通读。

---

## A. 关键文件索引（按行数 Top 20）

全仓 `packages/*/src` 共 **267** 个 `.ts` 文件。下面是最大的 20 个（含生成文件；判断手写复杂度时可跳过 `*.generated.ts`）：

| 行数 | 文件 | 对应章节 |
|------|------|---------|
| 17177 | `ai/src/models.generated.ts` | 02 |
| 5165 | `coding-agent/src/modes/interactive/interactive-mode.ts` | 09 |
| 2791 | `coding-agent/src/core/agent-session.ts` | 04 / 06 / 09 |
| 2296 | `coding-agent/src/core/package-manager.ts` | 07 / 13 |
| 1961 | `tui/src/components/editor.ts` | 08 |
| 1534 | `tui/src/tui.ts` | 08 |
| 1402 | `coding-agent/src/core/session-manager.ts` | 06 |
| 1391 | `coding-agent/src/core/extensions/types.ts` | 07 |
| 1348 | `ai/src/providers/openai-codex-responses.ts` | 02 |
| 1285 | `tui/src/keys.ts` | 08 |
| 1241 | `coding-agent/src/modes/interactive/components/tree-selector.ts` | 09 |
| 1155 | `ai/src/providers/openai-completions.ts` | 02 |
| 1151 | `ai/src/providers/anthropic.ts` | 02 |
| 1134 | `coding-agent/src/modes/interactive/theme/theme.ts` | 09 |
| 1053 | `tui/src/utils.ts` | 08 |
| 1023 | `coding-agent/src/core/extensions/runner.ts` | 07 |
| 1019 | `coding-agent/src/core/settings-manager.ts` | 11 |
| 998 | `agent/src/harness/agent-harness.ts` | 03 |
| 976 | `ai/src/providers/amazon-bedrock.ts` | 02 |
| 922 | `coding-agent/src/core/resource-loader.ts` | 07 / 12 |

### 各包入口/契约文件

| 包 | 入口 | 类型契约 |
|----|------|---------|
| pi-ai | `packages/ai/src/index.ts` | `packages/ai/src/types.ts` |
| pi-agent-core | `packages/agent/src/index.ts` → `base.ts` | `packages/agent/src/types.ts` |
| pi-coding-agent | `packages/coding-agent/src/main.ts`（CLI）/ `index.ts`（SDK） | `core/sdk.ts` |
| pi-tui | `packages/tui/src/index.ts` | `tui.ts`（`Component` 接口） |

---

## B. 术语表

| 术语 | 含义 | 出处 |
|------|------|------|
| **Provider** | pi-ai 里一个 LLM 后端适配器（anthropic/openai-completions/codex-responses/bedrock…），把统一的 `Message[]` 翻译成各家 API 的请求/SSE 流 | 02 |
| **streamSimple / streamFn** | pi-ai 暴露的统一流式接口；coding-agent 在 `sdk.ts:301` 包一层注入鉴权后传给 agent | 02 / 04 |
| **Agent** | pi-agent-core 的低层抽象：一次"调模型→产出工具调用→执行→回灌"的循环驱动器（`agent.ts` + `agent-loop.ts`） | 03 |
| **AgentHarness** | pi-agent-core 的高层封装（`harness/`），自带 session 管理；coding-agent **未**用它，而是基于低层 `Agent` 自建 | 03 |
| **agentLoop / runLoop** | agent 的核心循环（`agent-loop.ts:155`），发 `AgentEvent` 流 | 03 |
| **AgentEvent** | 循环过程中产生的事件（assistant 文本、工具开始/结束、错误…），`types.ts:408` | 03 |
| **AgentSession** | coding-agent 的中枢编排器（`agent-session.ts`, 2791 行）：把 Agent、SessionManager、compaction、工具、扩展粘在一起，对外发 `AgentSessionEvent` | 04 / 09 |
| **AgentTool** | 运行时工具对象（name + execute），由 `ToolDefinition` 经 `wrapToolDefinition` 产生 | 05 |
| **ToolDefinition** | 扩展声明工具的契约（`extensions/types.ts:435`），含 JSON schema 参数与 `execute` | 05 / 07 |
| **内置工具** | 7 个：read / bash / edit / write / grep / find / ls（`core/tools/index.ts`） | 05 |
| **SessionEntry** | 持久化会话树的节点联合类型（9 种），JSONL 落盘 | 06 |
| **SessionManager** | 会话树的增删查 + 迁移（v1→v2→v3，`CURRENT_SESSION_VERSION=3`） | 06 |
| **Compaction** | 上下文压缩：超阈值时摘要旧消息腾出 token（`DEFAULT_COMPACTION_SETTINGS`） | 06 |
| **estimateTokens** | token 估算，约 `chars/4` | 06 |
| **Extension** | 用户/第三方扩展，经 jiti 加载，用 `ExtensionAPI` 注册 provider/tool/command/hook | 07 |
| **ExtensionRunner** | 扩展事件分发器，`emit` 约 30 种事件 | 07 |
| **Hook** | 扩展挂载点（如 prompt 前/工具前后），通过事件触发 | 07 |
| **Skill** | `SKILL.md` 描述的可发现能力，按需注入系统提示 | 12 |
| **Slash command** | 交互模式的 `/` 命令，22 个内置 | 12 |
| **Component** | pi-tui 的渲染单元（`render()` 返回行），差分渲染 | 08 |
| **Mode** | 运行模式：interactive / print / rpc（`resolveAppMode`） | 10 |

---

## C. 环境变量速查

源码中 `packages/*/src` 静态出现 21 个 `PI_*` 字符串，其中包含 provider 兼容开关、OAuth 回调配置、生成/注释中的 API 字符串，以及动态拼出的目录变量。下面列的是当前代码实际读取或设置的 pi 自有运行时环境变量；目录类变量名由 `${APP_NAME.toUpperCase()}_CODING_AGENT_*` 动态拼出，APP_NAME 默认 `pi`，所以是 `PI_CODING_AGENT_DIR` / `PI_CODING_AGENT_SESSION_DIR`（`config.ts:495-496`）。

| 变量 | 作用 | 出处 |
|------|------|------|
| `PI_CODING_AGENT_DIR` | 覆盖 agent 配置目录（默认 `~/.pi/agent`） | `config.ts:495,516` |
| `PI_CODING_AGENT_SESSION_DIR` | 覆盖会话存储目录 | `config.ts:496` |
| `PI_PACKAGE_DIR` | 覆盖包资源目录定位 | `config.ts:369` |
| `PI_SHARE_VIEWER_URL` | 分享查看器 base URL（默认 `https://pi.dev/session/`） | `config.ts:502,506` |
| `PI_CODING_AGENT` | 标记运行于 pi（`cli.ts` 启动时设 `true`） | `cli.ts` |
| `PI_OFFLINE` | 离线模式，禁网络相关行为 | provider/version-check |
| `PI_SKIP_VERSION_CHECK` | 跳过新版本检查 | `utils/version-check.ts` |
| `PI_EXPERIMENTAL` | 开启实验特性 | `core/experimental.ts` |
| `PI_TELEMETRY` | 遥测开关 | `core/telemetry.ts` |
| `PI_TIMING` | 输出计时信息 | — |
| `PI_STARTUP_BENCHMARK` | 启动性能基准 | `cli/startup-ui.ts` |
| `PI_CACHE_RETENTION` | provider prompt cache 保留策略兼容开关（`long`/默认 `short`） | `ai/providers/*` |
| `PI_OAUTH_CALLBACK_HOST` | OAuth 本地回调监听 host | `ai/utils/oauth/*` |
| `PI_DEBUG_REDRAW` | TUI 重绘调试 | tui |
| `PI_TUI_DEBUG` | TUI 调试输出 | tui |
| `PI_TUI_WRITE_LOG` | TUI 写日志 | tui |
| `PI_HARDWARE_CURSOR` | 用硬件光标 | tui |
| `PI_CLEAR_ON_SHRINK` | 终端缩小时清屏 | tui |

> 其它读取的常见系统变量：`HOME`/`USERPROFILE`（家目录）、`EDITOR`/`VISUAL`（外部编辑器）、`HTTP_PROXY`/`HTTPS_PROXY`（代理）、`DISPLAY`/`WAYLAND_DISPLAY`（剪贴板/图像）、`WSL_*`/`TMUX`/`TERMUX_VERSION`（终端环境探测）。

---

## D. 配置目录与文件布局

`CONFIG_DIR_NAME = ".pi"`（`config.ts:491`，来自 `package.json` 的 `piConfig.configDir`）。

```
~/.pi/agent/                  # getAgentDir()，可被 PI_CODING_AGENT_DIR 覆盖
├── auth.json                 # OAuth/API key 凭证（AuthStorage）
├── models.json               # 自定义模型注册
├── settings.json             # 用户设置（SettingsManager）
├── themes/                   # 自定义主题
├── prompts/                  # 自定义提示模板
├── tools/                    # 自定义工具
├── bin/                      # 托管二进制（fd/rg 等）
├── sessions/                 # 会话 JSONL（可被 PI_CODING_AGENT_SESSION_DIR 覆盖）
└── extensions/               # 已安装扩展
```

项目级设置（`.pi/settings.json` 在项目根）会与全局 `deepMergeSettings` 合并，项目覆盖全局（第 11 章）。

---

## E. 关键常量速查

| 常量 | 值 | 文件 |
|------|-----|------|
| `CURRENT_SESSION_VERSION` | 3 | `session-manager.ts:30` |
| `DEFAULT_COMPACTION_SETTINGS` | `{enabled:true, reserveTokens:16384, keepRecentTokens:20000}` | `compaction.ts:122` |
| estimateTokens | `chars / 4` | `compaction.ts` |
| `DEFAULT_MAX_LINES`（工具输出截断） | 2000 | `truncate.ts:11` |
| `DEFAULT_MAX_BYTES` | 50KB | `truncate.ts:12` |
| `GREP_MAX_LINE_LENGTH` | 500 | `truncate.ts:13` |
| 默认激活工具 | `["read","bash","edit","write"]` | `sdk.ts:244` |
| 全部内置工具 | 7（read/bash/edit/write/grep/find/ls） | `core/tools/index.ts:84` |
| 内置 slash 命令 | 22 | `slash-commands.ts:18-41` |
| 扩展事件类型 | ~30 | `extensions/runner.ts` |
| `MIN_RENDER_INTERVAL_MS` | 16 | `tui.ts:309` |
| `CURSOR_MARKER` | `\x1b_pi:c\x07` | `tui.ts:120` |
| 主题热重载防抖 | 100ms | `theme.ts:866-900` |
| 内置 provider api | 9 | `register-builtins.ts` |
| 锁步版本 | 0.79.9 | 各 `package.json` |
| 支持平台二进制 | 6（darwin/linux/windows × arm64/x64） | `build-binaries.sh` |

---

## F. 验证方法备忘

本文档所有事实均以命令实测，未从函数名推断。复核示例：

```bash
find packages/*/src -name "*.ts" | wc -l                    # 267
wc -l packages/coding-agent/src/core/agent-session.ts        # 2791
grep -rhoE "PI_[A-Za-z0-9_]+" packages/*/src | sort -u      # 静态 PI_* 字符串
grep -n "CURRENT_SESSION_VERSION" packages/coding-agent/src/core/session-manager.ts
```

---

**返回**：[README](./README.md) · 上一章 [13 基础设施](./13-infrastructure.md)
