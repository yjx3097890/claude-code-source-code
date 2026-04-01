# AI Agent 架构设计与工具调用深度分析

> 基于 Claude Code v2.1.88 反编译源码，解析其 Agent 循环、工具系统和多代理架构。

---

## 一、核心 Agent 循环

Claude Code 的 Agent 循环实现在 `src/query.ts` 的 `queryLoop()` 函数中，这是整个项目最大的文件（785KB）。循环的本质是一个 `while(true)` 无限循环，通过 `AsyncGenerator` 向外 yield 事件流。

### 1.1 循环的基本结构

```typescript
async function* queryLoop(params, consumedCommandUuids) {
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    turnCount: 1,
    // ... 其他可变状态
  }

  while (true) {
    // 1. 解构当前状态
    let { toolUseContext } = state
    const { messages, turnCount, ... } = state

    // 2. 预处理：snip → microcompact → context collapse → autocompact
    messagesForQuery = await applyToolResultBudget(messagesForQuery, ...)
    messagesForQuery = snipModule.snipCompactIfNeeded(messagesForQuery)
    messagesForQuery = (await deps.microcompact(messagesForQuery, ...)).messages
    const { compactionResult } = await deps.autocompact(messagesForQuery, ...)

    // 3. 调用 Claude API（流式）
    for await (const message of deps.callModel({ messages, systemPrompt, tools, ... })) {
      yield message  // 流式输出给消费者
      if (message has tool_use blocks) {
        needsFollowUp = true
        streamingToolExecutor.addTool(toolBlock, message)
      }
    }

    // 4. 如果没有工具调用，处理停止逻辑
    if (!needsFollowUp) {
      // 处理 prompt-too-long 恢复、max_output_tokens 恢复、stop hooks
      return { reason: 'completed' }
    }

    // 5. 执行工具调用
    for await (const update of toolUpdates) {
      yield update.message
      toolResults.push(...)
    }

    // 6. 获取附件（文件变更通知、记忆预取、技能发现等）
    for await (const attachment of getAttachmentMessages(...)) {
      yield attachment
    }

    // 7. 更新状态，继续循环
    state = {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      turnCount: nextTurnCount,
      // ...
    }
  }
}
```

### 1.2 状态管理：不可变参数 + 可变状态

循环的状态管理采用了一个精巧的模式：

```typescript
// 不可变参数 — 循环期间永不改变
const { systemPrompt, userContext, canUseTool, maxTurns, ... } = params

// 可变状态 — 每次迭代通过 state = { ... } 整体替换
let state: State = { messages, toolUseContext, turnCount, ... }
```

每个 `continue` 站点（循环内有 7 个不同的 continue 路径）都通过构造新的 `State` 对象来更新状态，而不是逐个修改字段。这避免了遗漏更新的 bug。

### 1.3 循环的退出路径

循环有多个退出路径，每个都返回一个带有 `reason` 的终端状态：

| 退出原因 | 触发条件 |
|----------|----------|
| `completed` | 模型返回文本（无工具调用） |
| `aborted_streaming` | 用户在流式输出期间中断 |
| `aborted_tools` | 用户在工具执行期间中断 |
| `blocking_limit` | 上下文达到硬限制 |
| `prompt_too_long` | API 返回 prompt-too-long 且恢复失败 |
| `image_error` | 图片大小/格式错误 |
| `model_error` | API 调用异常 |
| `hook_stopped` | Hook 阻止继续 |
| `stop_hook_prevented` | Stop hook 阻止继续 |
| `max_turns` | 达到最大轮次限制 |

---

## 二、工具系统架构

### 2.1 Tool 接口

每个工具通过 `buildTool()` 工厂函数创建，实现统一的 `Tool` 接口：

```typescript
interface Tool<Input, Output, Progress> {
  name: string
  inputSchema: ZodSchema<Input>

  // 生命周期
  validateInput?(input): ValidationResult
  checkPermissions(input, context): PermissionResult
  call(input, context): Promise<Output>

  // 能力声明
  isEnabled(): boolean
  isConcurrencySafe(input): boolean    // 能否并行执行？
  isReadOnly(input): boolean           // 是否只读？
  isDestructive?(input): boolean       // 是否不可逆？
  interruptBehavior?(): 'cancel' | 'block'

  // AI 接口
  prompt(options): string              // 给 LLM 的工具描述
  description(): string

  // 渲染
  renderToolUseMessage(input): ReactNode
  renderToolResultMessage(output): ReactNode
  renderToolUseProgressMessage?(progress): ReactNode
}
```

关键设计点：
- `isConcurrencySafe` 决定工具能否并行执行——这直接影响 `StreamingToolExecutor` 的调度策略
- `isReadOnly` 和 `isDestructive` 用于权限系统的风险评估
- `prompt()` 是动态的，可以根据上下文生成不同的工具描述

### 2.2 工具注册与过滤

```typescript
function getAllBaseTools(): Tools {
  return [
    BashTool, FileReadTool, FileEditTool, FileWriteTool,
    GlobTool, GrepTool, AgentTool, WebFetchTool, WebSearchTool,
    MCPTool, SkillTool, AskUserQuestionTool, ...
    // 条件工具
    ...(getPowerShellTool() ? [getPowerShellTool()] : []),
    ...(feature('KAIROS') ? [SleepTool] : []),
  ]
}

// 运行时过滤
function getTools(permissionContext): Tools {
  let tools = getAllBaseTools()
  tools = filterToolsByDenyRules(tools, permissionContext.denyRules)
  // 根据权限模式进一步过滤
  if (permissionContext.mode === 'plan') {
    tools = tools.filter(t => t.isReadOnly || t.name === 'EnterPlanModeTool')
  }
  return tools
}
```

### 2.3 工具预设 (Presets)

```typescript
function parseToolPreset(preset: string): ToolPreset | null {
  // 'default' — 标准工具集
  // 'minimal' — 最小工具集（只有文件读写和搜索）
  // 'custom:tool1,tool2' — 自定义工具集
}
```

---

## 三、工具执行引擎

### 3.1 并行/串行分区

`toolOrchestration.ts` 中的 `partitionToolCalls()` 是工具执行的核心调度逻辑：

```typescript
function partitionToolCalls(toolUseMessages, toolUseContext): Batch[] {
  return toolUseMessages.reduce((acc, toolUse) => {
    const tool = findToolByName(tools, toolUse.name)
    const isConcurrencySafe = tool?.isConcurrencySafe(parsedInput)

    if (isConcurrencySafe && acc[acc.length - 1]?.isConcurrencySafe) {
      // 连续的并发安全工具合并到同一批次
      acc[acc.length - 1].blocks.push(toolUse)
    } else {
      // 新批次
      acc.push({ isConcurrencySafe, blocks: [toolUse] })
    }
    return acc
  }, [])
}
```

分区策略：
- 连续的只读工具 → 合并为一个并行批次
- 非只读工具 → 单独一个串行批次
- 结果：`[并行批次, 串行, 并行批次, 串行, ...]`

```
模型返回: [GrepTool, GlobTool, FileReadTool, FileEditTool, GrepTool, GrepTool]
分区结果: [
  { concurrent: true,  blocks: [GrepTool, GlobTool, FileReadTool] },
  { concurrent: false, blocks: [FileEditTool] },
  { concurrent: true,  blocks: [GrepTool, GrepTool] },
]
```

### 3.2 StreamingToolExecutor：流式并行执行

`StreamingToolExecutor` 是一个更高级的执行器，它在模型还在流式输出时就开始执行工具：

```typescript
class StreamingToolExecutor {
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    // 模型流式输出一个 tool_use block 时立即调用
    const tool = findToolByName(this.tools, block.name)
    const isConcurrencySafe = tool?.isConcurrencySafe(block.input)

    this.trackedTools.push({
      block, tool, isConcurrencySafe,
      status: 'queued',
    })

    // 如果可以执行，立即开始
    if (this.canExecuteTool(isConcurrencySafe)) {
      this.processQueue()
    }
  }

  private canExecuteTool(isConcurrencySafe: boolean): boolean {
    if (this.hasExecutingNonConcurrentTool) return false
    if (!isConcurrencySafe && this.hasExecutingTools()) return false
    return this.executingCount < this.maxConcurrency
  }
}
```

关键特性：
- **流式启动**：不等模型输出完毕，每收到一个 tool_use block 就尝试执行
- **并发控制**：最大并发数默认 10（`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`）
- **中断处理**：支持 cancel 和 block 两种中断行为
- **合成错误**：被中断的工具生成合成的 tool_result 错误消息

### 3.3 工具执行的完整流程

```
tool_use block 到达
    │
    ▼
StreamingToolExecutor.addTool()
    │
    ├─ 解析输入 (inputSchema.safeParse)
    ├─ 判断并发安全性 (isConcurrencySafe)
    ├─ 加入队列
    │
    ▼
processQueue()
    │
    ├─ canExecuteTool()? → 否 → 等待
    │
    ▼ 是
executeTool()
    │
    ├─ runPreToolUseHooks()     ← 用户定义的前置钩子
    │   ├─ 批准 → 继续
    │   ├─ 拒绝 → 生成错误 tool_result
    │   └─ 修改输入 → 用修改后的输入继续
    │
    ├─ canUseTool()             ← 权限检查
    │   ├─ 允许 → 继续
    │   ├─ 拒绝 → 生成拒绝 tool_result
    │   └─ 询问用户 → 等待用户决定
    │
    ├─ tool.call(input, context) ← 实际执行
    │
    ├─ runPostToolUseHooks()    ← 用户定义的后置钩子
    │
    └─ yield { message: toolResultMessage }
```

---

## 四、权限系统

### 4.1 四层权限防护

```
Layer 1: validateInput()
  └─ 工具自身的输入校验（格式、范围等）

Layer 2: PreToolUse Hooks
  └─ 用户定义的 shell 命令，可以：
     - 批准（跳过后续检查）
     - 拒绝（直接返回错误）
     - 修改输入（替换参数后继续）

Layer 3: Permission Rules
  └─ alwaysAllowRules: 匹配 → 自动允许
     alwaysDenyRules:  匹配 → 自动拒绝
     alwaysAskRules:   匹配 → 强制询问
     来源：settings, CLI args, session decisions

Layer 4: checkPermissions()
  └─ 工具特定逻辑（如路径沙箱验证）
```

### 4.2 BashTool 的安全分层

BashTool 是权限系统最复杂的工具，有 18 个源文件：

```
BashTool/
├── BashTool.tsx              # 主实现
├── bashPermissions.ts        # 权限判断
├── bashSecurity.ts           # 安全验证（注入检测）
├── bashCommandHelpers.ts     # 命令分段权限检查
├── modeValidation.ts         # 模式验证（plan mode 限制）
├── pathValidation.ts         # 路径验证（危险删除检测）
├── destructiveCommandWarning.ts  # 破坏性命令警告
├── commandSemantics.ts       # 命令语义理解（退出码解释）
└── ...
```

安全检查包括：
- 命令注入检测（引号提取、未转义字符检查）
- 不完整命令检测（悬挂的管道、未闭合的引号）
- 危险路径检测（`rm -rf /`、删除 home 目录等）
- 文件系统命令的路径沙箱验证
- Plan 模式下只允许只读命令

### 4.3 自动模式 (Auto Mode)

```typescript
function isAutoModeAllowlistedTool(toolName: string): boolean {
  // 自动模式下自动允许的工具白名单
}
```

自动模式有一个"熔断器"机制：

```typescript
function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  // 连续拒绝次数过多时，回退到交互式提示
}
```

---

## 五、多代理架构

### 5.1 任务类型体系

```typescript
type TaskType =
  | 'local_bash'           // 本地 Shell 命令
  | 'local_agent'          // 本地子代理
  | 'remote_agent'         // 远程代理（通过 Bridge）
  | 'in_process_teammate'  // 进程内队友
  | 'local_workflow'       // 本地工作流
  | 'monitor_mcp'          // MCP 监控
  | 'dream'                // 后台思考
```

### 5.2 AgentTool：子代理生成

AgentTool 支持多种内置代理类型：

```
AgentTool/
├── built-in/
│   ├── exploreAgent.ts         # 代码库探索代理
│   ├── planAgent.ts            # 规划代理
│   ├── generalPurposeAgent.ts  # 通用代理
│   └── claudeCodeGuideAgent.ts # Claude Code 指南代理
├── agentMemory.ts              # 代理记忆系统
├── agentToolUtils.ts           # 工具过滤和解析
└── forkSubagent.ts             # Fork 子代理模式
```

子代理的工具集是主代理的子集：

```typescript
function filterToolsForAgent({ tools, agentDefinition }): Tools {
  // 移除不适合子代理的工具
  // 例如：子代理不能再生成子代理（防止无限递归）
  // 子代理不能使用 AskUserQuestionTool（不能直接与用户交互）
}
```

### 5.3 子代理的系统提示词

```typescript
export const DEFAULT_AGENT_PROMPT = `You are an agent for Claude Code,
Anthropic's official CLI for Claude. Given the user's message, you should
use the tools available to complete the task. Complete the task fully —
don't gold-plate, but don't leave it half-done. When you complete the task,
respond with a concise report covering what was done and any key findings —
the caller will relay this to the user, so it only needs the essentials.`
```

子代理的提示词被增强了额外的约束：

```typescript
async function enhanceSystemPromptWithEnvDetails(existingPrompt, model) {
  const notes = `Notes:
- Agent threads always have their cwd reset between bash calls,
  so please only use absolute file paths.
- In your final response, share file paths (always absolute, never relative)
  that are relevant to the task.
- Do not use emojis.
- Do not use a colon before tool calls.`
  return [...existingPrompt, notes, envInfo]
}
```

### 5.4 Swarm 模式

```
src/utils/swarm/                    # 多代理 swarm
├── backends/
│   └── teammateModeSnapshot.ts     # 队友模式快照
├── teammatePromptAddendum.ts       # 队友提示词附加
└── ...
```

Swarm 模式允许多个代理作为"队友"协作，每个队友有自己的上下文窗口和工具集。

---

## 六、错误恢复机制

### 6.1 Prompt-Too-Long 恢复

当 API 返回 prompt-too-long 错误时，循环有三级恢复策略：

```
1. Context Collapse Drain
   └─ 释放已暂存的上下文折叠，减少 token 数
   └─ 如果成功 → continue（重试 API 调用）

2. Reactive Compact
   └─ 紧急压缩对话历史
   └─ 如果成功 → continue（用压缩后的消息重试）

3. 放弃
   └─ 向用户显示错误
   └─ return { reason: 'prompt_too_long' }
```

### 6.2 Max Output Tokens 恢复

```typescript
// 第一级：升级 token 限制
if (capEnabled && maxOutputTokensOverride === undefined) {
  // 从默认 8k 升级到 64k，重试同一请求
  state = { ...state, maxOutputTokensOverride: ESCALATED_MAX_TOKENS }
  continue
}

// 第二级：多轮恢复（最多 3 次）
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  const recoveryMessage = createUserMessage({
    content: `Output token limit hit. Resume directly — no apology,
              no recap of what you were doing. Pick up mid-thought if
              that is where the cut happened. Break remaining work into
              smaller pieces.`,
    isMeta: true,
  })
  state = { ...state, messages: [...messages, ...assistantMessages, recoveryMessage] }
  continue
}
```

### 6.3 模型降级

```typescript
catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    // 清理已有的 assistant messages
    // 剥离 thinking 签名（不同模型不兼容）
    yield createSystemMessage(
      `Switched to ${fallbackModel} due to high demand for ${originalModel}`
    )
    continue  // 用降级模型重试
  }
}
```

---

## 七、附件系统

每轮工具执行后，循环会注入多种附件：

```typescript
for await (const attachment of getAttachmentMessages(
  null, updatedToolUseContext, null,
  queuedCommandsSnapshot,
  [...messagesForQuery, ...assistantMessages, ...toolResults],
  querySource,
)) {
  yield attachment
  toolResults.push(attachment)
}
```

附件类型包括：
- **文件变更通知**：其他进程修改了文件
- **记忆预取**：相关的 CLAUDE.md 记忆
- **技能发现**：与当前任务相关的技能
- **队列命令**：用户在工具执行期间提交的新消息
- **任务通知**：后台任务完成通知

---

## 八、关键设计模式总结

| 模式 | 实现 | 解决的问题 |
|------|------|-----------|
| AsyncGenerator 循环 | `while(true) { yield* }` | 流式输出 + 状态管理 |
| 不可变状态替换 | `state = { ...newState }` | 避免 7 个 continue 站点的状态遗漏 |
| 流式工具执行 | `StreamingToolExecutor` | 模型输出和工具执行重叠 |
| 并发安全分区 | `partitionToolCalls()` | 只读并行、写入串行 |
| 三级错误恢复 | collapse → compact → 放弃 | 最大化恢复成功率 |
| Token 升级重试 | 8k → 64k → 多轮恢复 | 减少 slot 浪费 |
| 熔断器 | `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3` | 防止无限重试 |
| 附件注入 | 工具执行后注入上下文 | 保持模型对环境变化的感知 |
