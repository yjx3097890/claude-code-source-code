# 现代 CLI 工具开发最佳实践

> 基于 Claude Code v2.1.88 反编译源码，提炼其在 CLI 工具开发中的工程实践。

---

## 一、入口与启动架构

### 1.1 分层入口

Claude Code 有三个入口点，服务不同的使用场景：

```
src/entrypoints/
├── cli.tsx      # CLI 主入口：版本检查、参数解析、daemon 模式
├── mcp.ts       # MCP 服务器入口：作为 MCP 工具提供者
└── sdk/         # SDK 入口：程序化调用（无头模式）
```

`cli.tsx` 的 `main()` 函数是整个应用的起点：

```typescript
async function main(): Promise<void> {
  // 1. 极早期初始化（在任何 import 之前）
  //    - 设置 process.env
  //    - 注册 uncaughtException handler

  // 2. 动态导入主模块（延迟加载，加速启动）
  const { run } = await import('../main.js')

  // 3. 执行主逻辑
  await run()
}
```

### 1.2 Commander.js 命令注册

`main.tsx` 中的 `run()` 函数使用 Commander.js 注册了完整的 CLI 命令树：

```typescript
async function run(): Promise<CommanderCommand> {
  const program = new Command()
    .name('claude')
    .version(MACRO.VERSION)
    .option('-p, --print <prompt>', '非交互模式')
    .option('--model <model>', '指定模型')
    .option('--system-prompt <prompt>', '自定义系统提示词')
    .option('--allowedTools <tools...>', '允许的工具')
    .option('--max-turns <n>', '最大轮次')
    // ... 50+ 选项
}
```

### 1.3 启动优化

```typescript
function initializeEntrypoint(isNonInteractive: boolean): void {
  // 预取系统上下文（git 状态、shell 信息等）
  prefetchSystemContextIfSafe()

  // 延迟预取（不阻塞启动）
  startDeferredPrefetches()

  // 运行设置迁移
  runMigrations()

  // 预连接 API（TCP 握手 + TLS）
  apiPreconnect()
}
```

关键优化：
- `prefetchSystemContextIfSafe()`：在用户输入之前就开始收集环境信息
- `apiPreconnect()`：预建立 TCP/TLS 连接，减少首次 API 调用延迟
- 延迟加载：大量模块使用动态 `import()` 或 `require()`，只在需要时加载

---

## 二、React/Ink 终端 UI 框架

### 2.1 为什么用 React 做终端 UI

Claude Code 使用 React + Ink 构建终端 UI，这不是常见的选择。优势在于：

- **组件化**：100+ UI 组件可复用
- **状态管理**：React 的状态模型天然适合复杂的交互式 UI
- **声明式渲染**：描述"UI 应该是什么样"而非"如何更新 UI"
- **虚拟 DOM**：只更新变化的部分，减少终端闪烁

### 2.2 自定义 Ink 实现

Claude Code 没有直接使用 Ink 库，而是维护了一个深度定制的版本（`src/ink/`，40+ 文件）：

```
src/ink/
├── ink.tsx              # 核心渲染引擎（1,664 行）
├── reconciler.ts        # React reconciler
├── dom.ts               # 虚拟 DOM
├── renderer.ts          # 渲染器
├── output.ts            # 输出管理
├── render-node-to-output.ts  # 节点渲染
├── render-border.ts     # 边框渲染
├── render-to-screen.ts  # 屏幕渲染
├── selection.ts         # 文本选择
├── searchHighlight.ts   # 搜索高亮
├── hit-test.ts          # 点击测试
├── focus.ts             # 焦点管理
├── screen.ts            # 屏幕管理
├── terminal.ts          # 终端控制
├── parse-keypress.ts    # 按键解析
└── ...
```

定制的功能包括：
- **文本选择**：终端中的文本选择和复制
- **搜索高亮**：在终端输出中搜索和高亮
- **点击测试**：鼠标点击事件处理
- **超链接**：终端超链接支持
- **备用屏幕**：alt screen 管理
- **虚拟滚动**：大量消息的高效渲染

### 2.3 Ink 核心类

```typescript
class Ink {
  constructor(options: Options) {
    // 初始化 React reconciler
    // 设置终端模式（raw mode, mouse tracking）
    // 注册 resize handler
  }

  onRender() {
    // React 触发重渲染时调用
    // 1. 布局计算（yoga-layout）
    // 2. 渲染节点到输出缓冲区
    // 3. 差异更新到终端
  }

  // 备用屏幕管理
  enterAlternateScreen(): void { ... }
  exitAlternateScreen(): void { ... }

  // 文本选择
  copySelection(): string { ... }
  clearTextSelection(): void { ... }

  // 搜索
  setSearchHighlight(query: string): void { ... }

  // 鼠标事件
  dispatchClick(col, row): boolean { ... }
  dispatchHover(col, row): void { ... }
}
```

### 2.4 虚拟滚动

```typescript
// src/components/VirtualMessageList.tsx
function VirtualMessageList({ messages, ... }) {
  // 只渲染可见区域的消息
  // 滚动时动态加载/卸载消息
  // 支持 sticky prompt（固定在顶部的用户输入）
  // 支持搜索定位
}
```

---

## 三、命令系统设计

### 3.1 80+ 斜杠命令

```
src/commands/
├── agents/          # /agents — 代理管理
├── compact/         # /compact — 手动压缩
├── config/          # /config — 设置管理
├── help/            # /help — 帮助
├── login/           # /login — 认证
├── mcp/             # /mcp — MCP 服务器管理
├── memory/          # /memory — 记忆系统
├── plan/            # /plan — 规划模式
├── resume/          # /resume — 恢复会话
├── review/          # /review — 代码审查
├── vim/             # /vim — Vim 模式
├── voice/           # /voice — 语音模式
├── stickers/        # /stickers — 贴纸（彩蛋）
├── teleport/        # /teleport — 跨设备传送
├── worktree/        # /worktree — Git worktree
└── ...              # 70+ 更多命令
```

### 3.2 命令接口

```typescript
// src/types/command.ts
interface Command {
  name: string
  description: string
  aliases?: string[]
  isEnabled?: () => boolean
  isAvailable?: () => boolean | Promise<boolean>
  execute: (args: string[], context: CommandContext) => Promise<void>
  // 可选：自动补全
  getSuggestions?: (partial: string) => string[]
}
```

### 3.3 命令发现与过滤

```typescript
async function getCommands(cwd: string): Promise<Command[]> {
  const commands = [
    ...builtinCommands,
    ...pluginCommands,
    ...skillCommands,
    ...mcpSkillCommands,
  ]

  // 过滤不可用的命令
  return commands.filter(cmd => meetsAvailabilityRequirement(cmd))
}

// Bridge 模式下过滤不安全的命令
function filterCommandsForRemoteMode(commands: Command[]): Command[] {
  return commands.filter(cmd => isBridgeSafeCommand(cmd))
}
```

---

## 四、快捷键系统

### 4.1 可配置的快捷键

```typescript
// src/keybindings/
├── defaultBindings.ts     # 默认快捷键映射
├── loadUserBindings.ts    # 加载用户自定义快捷键
├── parser.ts              # 快捷键字符串解析（"ctrl+shift+p"）
├── match.ts               # 快捷键匹配
├── resolver.ts            # 冲突解决
├── validate.ts            # 验证
├── reservedShortcuts.ts   # 保留快捷键（不可覆盖）
├── schema.ts              # 配置 schema
└── template.ts            # 配置模板生成
```

### 4.2 Vim 模式

```typescript
// src/vim/
├── motions.ts       # 移动命令（h/j/k/l, w/b, 0/$, gg/G）
├── operators.ts     # 操作符（d/c/y）
├── textObjects.ts   # 文本对象（iw/aw, i"/a"）
├── transitions.ts   # 模式转换（normal → insert → visual）
└── types.ts         # 类型定义
```

---

## 五、多模式输出

### 5.1 交互模式 vs 非交互模式

```typescript
// 交互模式：React/Ink 终端 UI
if (isInteractive) {
  const { render } = await import('./screens/REPL.js')
  render(<REPL ... />)
}

// 非交互模式：纯文本流式输出
if (options.print) {
  await runHeadless(prompt, options)
}

// 结构化输出模式：JSON 流
if (options.outputFormat === 'stream-json') {
  await runHeadlessStreaming(prompt, options)
}
```

### 5.2 StructuredIO：SDK 通信协议

```typescript
class StructuredIO {
  // stdin: NDJSON 控制消息
  // stdout: NDJSON 事件流

  async *read() {
    // 从 stdin 读取控制消息
    // 解析为 SDKControlResponse
  }

  async write(message: StdoutMessage): Promise<void> {
    // 序列化为 NDJSON 写入 stdout
  }

  // 权限请求/响应
  createCanUseTool(): CanUseToolFn {
    // 通过 stdin/stdout 与宿主进程协商权限
  }
}
```

---

## 六、插件与扩展系统

### 6.1 插件架构

```typescript
// src/plugins/
├── builtinPlugins.ts    # 内置插件注册
└── bundled/             # 捆绑的插件

// src/services/plugins/
├── loader.ts            # 插件加载器
└── ...
```

### 6.2 技能系统

```typescript
// src/skills/
├── bundledSkills.ts     # 内置技能
├── loadSkillsDir.ts     # 从目录加载技能
└── mcpSkillBuilders.ts  # MCP 技能构建器
```

技能是比插件更轻量的扩展——本质上是预定义的提示词模板，可以通过 `/skill-name` 调用。

### 6.3 MCP 集成

```typescript
// src/services/mcp/ — 23 个文件
// 完整的 MCP 协议实现，包括：
// - 服务器发现和连接
// - OAuth 认证
// - 工具调用代理
// - 资源读取
// - 频道权限
// - 配置管理
```

---

## 七、会话持久化

### 7.1 Transcript 记录

每条消息都被持久化到磁盘（JSONL 格式），支持：
- `/resume` 恢复之前的会话
- 跨会话搜索
- 调试和审计

### 7.2 会话恢复

```typescript
// src/commands/resume/
// 列出历史会话，选择恢复
// 重建消息历史和工具上下文
// 处理文件状态变化
```

### 7.3 文件历史快照

```typescript
// src/utils/fileHistory.ts
// 在每次文件编辑前保存快照
// 支持 /rewind 回退到之前的状态
```

---

## 八、错误处理与诊断

### 8.1 Doctor 命令

```typescript
// src/screens/Doctor.tsx
// /doctor 命令执行全面的环境诊断：
// - Node.js 版本检查
// - API 连接测试
// - 认证状态
// - MCP 服务器状态
// - 权限配置
// - 已知问题检测
```

### 8.2 分层错误处理

```typescript
// API 错误分类
function startsWithApiErrorPrefix(text: string): boolean { ... }
function isPromptTooLongMessage(msg: AssistantMessage): boolean { ... }
function isMediaSizeError(raw: string): boolean { ... }

// 连接错误诊断
function extractConnectionErrorDetails(error): string { ... }
function getSSLErrorHint(error): string | null { ... }

// 用户友好的错误消息
function sanitizeAPIError(apiError: APIError): string { ... }
```

### 8.3 遥测与监控

```typescript
// 双管道遥测
// 1. 第一方 → Anthropic（事件日志、使用统计）
// 2. Datadog（性能指标、错误追踪）

// GrowthBook feature flags
// 远程功能开关，支持 A/B 测试和渐进发布
```

---

## 九、跨平台支持

### 9.1 Shell 适配

```typescript
function getShellInfoLine(): string {
  if (env.platform === 'win32') {
    return `Shell: ${shellName} (use Unix shell syntax, not Windows —
            e.g., /dev/null not NUL, forward slashes in paths)`
  }
  return `Shell: ${shellName}`
}
```

### 9.2 PowerShell 工具

```typescript
// Windows 上提供 PowerShellTool 作为 BashTool 的替代
function getPowerShellTool() {
  if (env.platform === 'win32') {
    return PowerShellTool
  }
  return null
}
```

### 9.3 路径处理

```typescript
// src/utils/windowsPaths.ts
// Windows 路径规范化
// 正斜杠/反斜杠转换
// UNC 路径支持
```

---

## 十、构建与分发

### 10.1 Bun 编译时特性

```typescript
// feature() — 编译时条件编译
import { feature } from 'bun:bundle'
if (feature('DAEMON')) {
  // 内部构建：包含
  // 外部构建：死代码消除
}

// MACRO — 编译时常量注入
const version = MACRO.VERSION  // → '2.1.88'
```

### 10.2 单文件打包

最终产物是一个约 12MB 的 `cli.js`，包含：
- 所有 TypeScript 源码（编译后）
- 内联的依赖
- 但不包含 node_modules（`--packages=external`）

### 10.3 设置迁移

```typescript
// src/migrations/
├── migrateFennecToOpus.ts              # 模型名 Fennec → Opus
├── migrateSonnet45ToSonnet46.ts        # Sonnet 4.5 → 4.6
├── migrateOpusToOpus1m.ts              # Opus → Opus [1m]
├── resetAutoModeOptInForDefaultOffer.ts # 重置自动模式选择
└── ...
```

每次版本更新时自动运行迁移，确保用户设置兼容。

---

## 十一、关键工程实践总结

| 实践 | 实现 | 收益 |
|------|------|------|
| 分层入口 | cli.tsx / mcp.ts / sdk/ | 同一代码库服务多种使用场景 |
| React 终端 UI | 自定义 Ink 框架 | 组件化、声明式、高效更新 |
| 虚拟滚动 | VirtualMessageList | 长会话不卡顿 |
| 延迟加载 | 动态 import() | 启动速度优化 |
| API 预连接 | apiPreconnect() | 减少首次调用延迟 |
| 可配置快捷键 | keybindings 系统 | 用户自定义 + Vim 模式 |
| 多模式输出 | 交互/非交互/结构化 | 适配不同使用场景 |
| 会话持久化 | JSONL transcript | 恢复、搜索、审计 |
| 设置迁移 | migrations/ | 版本升级无感 |
| 编译时条件 | feature() + MACRO | 内外用户差异化 + 死代码消除 |
| 双管道遥测 | 1P + Datadog | 产品分析 + 运维监控 |
| 插件/技能/MCP | 三层扩展体系 | 从轻量到重量级的扩展能力 |
