# Claude Code Prompt Engineering 深度分析

> 基于 Claude Code v2.1.88 反编译源码，解析其系统提示词的工程化设计。

---

## 一、系统提示词的整体架构

Claude Code 的系统提示词不是一个静态字符串，而是一个由多个"段落"(Section) 动态组装的数组。核心组装逻辑在 `src/constants/prompts.ts` 的 `getSystemPrompt()` 函数中：

```typescript
// 简化后的组装流程
return [
  // --- 静态内容 (可跨组织缓存) ---
  getSimpleIntroSection(),          // 身份定义 + 安全指令
  getSimpleSystemSection(),         // 系统行为规则
  getSimpleDoingTasksSection(),     // 任务执行准则
  getActionsSection(),              // 操作风险评估
  getUsingYourToolsSection(),       // 工具使用指导
  getSimpleToneAndStyleSection(),   // 语气风格
  getOutputEfficiencySection(),     // 输出效率

  // === 动态边界标记 ===
  SYSTEM_PROMPT_DYNAMIC_BOUNDARY,

  // --- 动态内容 (每次会话/每轮变化) ---
  sessionSpecificGuidance,          // 会话特定指导
  memoryPrompt,                     // CLAUDE.md 记忆
  envInfo,                          // 环境信息
  languageSection,                  // 语言偏好
  outputStyleSection,               // 输出风格
  mcpInstructionsSection,           // MCP 服务器指令
  scratchpadInstructions,           // 临时目录
  functionResultClearing,           // 工具结果清理
  ...
]
```

这个设计有一个关键的工程决策：**静态/动态分界线**。

### 1.1 Prompt Cache 分界线

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

分界线之前的内容是跨用户、跨会话可缓存的（`cacheScope: 'global'`），之后的内容包含用户/会话特定信息，不能全局缓存。这直接影响 API 成本——缓存命中时 input token 费用降低 90%。

源码注释明确警告：

> WARNING: Do not remove or reorder this marker without updating cache logic in:
> - src/utils/api.ts (splitSysPromptPrefix)
> - src/services/api/claude.ts (buildSystemPromptBlocks)

### 1.2 Section 缓存机制

动态段落本身也有缓存策略，通过 `systemPromptSection()` 和 `DANGEROUS_uncachedSystemPromptSection()` 区分：

```typescript
// 计算一次，缓存到 /clear 或 /compact
systemPromptSection('memory', () => loadMemoryPrompt())

// 每轮重新计算，会破坏 prompt cache
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns'  // 必须说明原因
)
```

命名中的 `DANGEROUS_` 前缀是刻意的——它提醒开发者：每个 uncached section 都会在值变化时破坏 prompt cache，增加 API 成本。

---

## 二、提示词的分层设计

### 2.1 身份层：简洁的角色定义

```typescript
function getSimpleIntroSection(): string {
  return `
You are an interactive agent that helps users with software engineering tasks.
Use the instructions below and the tools available to you to assist the user.

${CYBER_RISK_INSTRUCTION}
IMPORTANT: You must NEVER generate or guess URLs for the user unless you are
confident that the URLs are for helping the user with programming.`
}
```

身份定义极其简短——只有两句话。没有冗长的角色扮演描述，没有"你是一个专业的..."之类的套话。这是一个值得注意的设计选择：让模型的行为由后续的具体指令塑造，而非靠身份描述。

### 2.2 安全层：CYBER_RISK_INSTRUCTION

安全指令被单独提取为常量，由 Safeguards 团队维护：

```typescript
// 文件头部有明确的所有权声明
// IMPORTANT: DO NOT MODIFY THIS INSTRUCTION WITHOUT SAFEGUARDS TEAM REVIEW

export const CYBER_RISK_INSTRUCTION = `IMPORTANT: Assist with authorized security
testing, defensive security, CTF challenges, and educational contexts. Refuse
requests for destructive techniques, DoS attacks, mass targeting, supply chain
compromise, or detection evasion for malicious purposes...`
```

这个指令的设计特点：
- 不是简单的"不要做坏事"，而是明确列出了允许和禁止的边界
- 允许：授权安全测试、防御安全、CTF、教育
- 禁止：破坏性技术、DoS、大规模攻击、供应链攻击、恶意逃避检测
- 双用途工具（C2 框架、凭证测试）需要明确的授权上下文

### 2.3 行为层：精细的编码准则

`getSimpleDoingTasksSection()` 是最长的段落，包含了极其具体的编码行为指导。这些不是泛泛的"写好代码"，而是从实际使用中提炼的反模式修正：

```
不要添加超出要求的功能、重构代码或做"改进"。
Bug 修复不需要清理周围的代码。
简单功能不需要额外的可配置性。
不要给你没改过的代码添加 docstring、注释或类型注解。
```

```
不要为不可能发生的场景添加错误处理、回退或验证。
信任内部代码和框架保证。只在系统边界验证（用户输入、外部 API）。
```

```
不要为一次性操作创建 helper、utility 或抽象。
不要为假设的未来需求设计。三行相似的代码比过早的抽象更好。
```

这些指令直接对抗 LLM 的常见倾向：过度工程化、过度注释、过度抽象。

### 2.4 内部/外部用户差异化

源码中大量使用 `process.env.USER_TYPE === 'ant'` 来区分 Anthropic 内部用户和外部用户：

```typescript
// 内部用户获得更详细的指令
...(process.env.USER_TYPE === 'ant'
  ? [
      `Default to writing no comments. Only add one when the WHY is
       non-obvious: a hidden constraint, a subtle invariant, a workaround
       for a specific bug...`,
      `Report outcomes faithfully: if tests fail, say so with the relevant
       output... Never claim "all tests pass" when output shows failures...`,
    ]
  : []),
```

内部用户的额外指令包括：
- 更严格的注释规范（默认不写注释）
- 诚实报告结果（不虚报测试通过）
- 完成前验证（运行测试、检查输出）
- 发现用户误解时主动指出

这些指令标注了 `@[MODEL LAUNCH]` 注释，表示它们是针对特定模型版本（Capybara v8）的行为校正，待验证后可能推广到外部用户。

---

## 三、工具提示词的工程化

### 3.1 工具使用优先级

```typescript
function getUsingYourToolsSection(enabledTools: Set<string>): string {
  const providedToolSubitems = [
    `To read files use ${FILE_READ_TOOL_NAME} instead of cat, head, tail, or sed`,
    `To edit files use ${FILE_EDIT_TOOL_NAME} instead of sed or awk`,
    `To create files use ${FILE_WRITE_TOOL_NAME} instead of cat with heredoc`,
    `To search for files use ${GLOB_TOOL_NAME} instead of find or ls`,
    `To search content use ${GREP_TOOL_NAME} instead of grep or rg`,
    `Reserve ${BASH_TOOL_NAME} exclusively for system commands and terminal operations`,
  ]
}
```

这解决了一个实际问题：模型倾向于用 Bash 做所有事情（因为 Bash 是万能的），但专用工具提供更好的用户体验（权限控制、进度显示、结果格式化）。

### 3.2 并行工具调用指导

```
You can call multiple tools in a single response. If you intend to call
multiple tools and there are no dependencies between them, make all
independent tool calls in parallel. However, if some tool calls depend
on previous calls, do NOT call these tools in parallel.
```

这个指令直接影响性能——并行工具调用可以显著减少等待时间。

### 3.3 子代理使用策略

```typescript
function getAgentToolSection(): string {
  return isForkSubagentEnabled()
    ? `Calling ${AGENT_TOOL_NAME} without a subagent_type creates a fork,
       which runs in the background and keeps its tool output out of your
       context. Reach for it when research or multi-step implementation
       work would otherwise fill your context with raw output you won't
       need again. **If you ARE the fork** — execute directly; do not
       re-delegate.`
    : `Use the ${AGENT_TOOL_NAME} tool with specialized agents when the
       task matches the agent's description. Subagents are valuable for
       parallelizing independent queries or for protecting the main context
       window from excessive results...`
}
```

关键设计：
- Fork 模式：后台运行，工具输出不进入主上下文——这是上下文管理的核心策略
- 防止递归委托："如果你就是 fork，直接执行，不要再委托"
- 明确使用场景：研究或多步实现工作

---

## 四、操作风险评估框架

`getActionsSection()` 定义了一个完整的风险评估框架：

```
Carefully consider the reversibility and blast radius of actions.

Examples of risky actions that warrant user confirmation:
- Destructive operations: deleting files/branches, dropping database tables,
  rm -rf, overwriting uncommitted changes
- Hard-to-reverse operations: force-pushing, git reset --hard, amending
  published commits
- Actions visible to others: pushing code, creating/closing PRs or issues,
  sending messages, posting to external services
- Uploading content to third-party web tools publishes it — consider whether
  it could be sensitive
```

这个框架的核心原则：
1. **可逆性**：本地、可逆的操作可以自由执行；不可逆的操作需要确认
2. **影响范围**：只影响本地 vs 影响共享系统
3. **一次授权不等于永久授权**："A user approving an action once does NOT mean they approve it in all contexts"
4. **不要用破坏性操作绕过障碍**："try to identify root causes and fix underlying issues rather than bypassing safety checks"

---

## 五、输出控制策略

### 5.1 内部用户：面向人类的写作

内部用户获得了详细的写作指导：

```
When sending user-facing text, you're writing for a person, not logging
to a console. Assume users can't see most tool calls or thinking — only
your text output.

When making updates, assume the person has stepped away and lost the thread.
Write so they can pick back up cold: use complete, grammatically correct
sentences without unexplained jargon.

Avoid semantic backtracking: structure each sentence so a person can read
it linearly, building up meaning without having to re-parse what came before.
```

### 5.2 外部用户：极简输出

```
IMPORTANT: Go straight to the point. Try the simplest approach first
without going in circles. Do not overdo it. Be extra concise.

Keep your text output brief and direct. Lead with the answer or action,
not the reasoning. Skip filler words, preamble, and unnecessary transitions.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan
```

### 5.3 数值化长度锚点

内部用户还有实验性的数值化长度限制：

```typescript
systemPromptSection('numeric_length_anchors',
  () => 'Length limits: keep text between tool calls to ≤25 words. ' +
        'Keep final responses to ≤100 words unless the task requires more detail.',
)
```

源码注释说明这带来了约 1.2% 的输出 token 减少，相比定性的"be concise"更有效。

---

## 六、环境感知提示词

### 6.1 动态环境信息

```typescript
async function computeSimpleEnvInfo(modelId, additionalWorkingDirectories) {
  const envItems = [
    `Primary working directory: ${cwd}`,
    `Is a git repository: ${isGit}`,
    `Platform: ${env.platform}`,
    `Shell: ${shellName}`,
    `OS Version: ${unameSR}`,
    modelDescription,           // 当前模型名称和 ID
    knowledgeCutoffMessage,     // 知识截止日期
    // 最新模型家族信息，帮助模型推荐正确的 API
    `The most recent Claude model family is Claude 4.5/4.6...`,
    `Claude Code is available as a CLI, desktop app, web app, and IDE extensions.`,
  ]
}
```

### 6.2 卧底模式下的信息抑制

```typescript
// Undercover: keep ALL model names/IDs out of the system prompt
if (process.env.USER_TYPE === 'ant' && isUndercover()) {
  // suppress — 不告诉模型它是什么模型
}
```

在卧底模式下，系统提示词中所有模型名称、ID、版本号都被移除，防止在公开仓库的 commit 中泄露内部信息。

---

## 七、记忆系统与提示词的集成

### 7.1 CLAUDE.md 记忆加载

```typescript
systemPromptSection('memory', () => loadMemoryPrompt())
```

`loadMemoryPrompt()` 从多个层级加载记忆文件：
- 项目级：`CLAUDE.md`（项目根目录）
- 用户级：`~/.claude/CLAUDE.md`
- 目录级：子目录中的 `CLAUDE.md`

这些记忆文件的内容被注入到系统提示词的动态部分，让模型了解项目特定的约定、规范和上下文。

### 7.2 MCP 服务器指令

```typescript
function getMcpInstructions(mcpClients: MCPServerConnection[]): string | null {
  // 只包含已连接且有指令的服务器
  const clientsWithInstructions = connectedClients.filter(
    client => client.instructions,
  )
  // 每个服务器的指令作为独立段落
  return `# MCP Server Instructions
The following MCP servers have provided instructions for how to use their
tools and resources:
${instructionBlocks}`
}
```

MCP 指令被标记为 `DANGEROUS_uncached`，因为服务器可能在轮次之间连接/断开。

---

## 八、提示词的多模式适配

### 8.1 自主模式 (Proactive/KAIROS)

当进入自主模式时，系统提示词完全切换：

```typescript
if (proactiveModule?.isProactiveActive()) {
  return [
    `You are an autonomous agent. Use the available tools to do useful work.`,
    // ... 精简的自主模式指令
    getProactiveSection(),  // 包含 tick 心跳、sleep 控制、终端焦点感知
  ]
}
```

自主模式的提示词包含独特的指令：
- **Tick 心跳**：`<tick>` 标签作为"你醒了，现在做什么？"的信号
- **Sleep 控制**：用 SleepTool 控制唤醒间隔，平衡 API 成本和响应速度
- **终端焦点感知**：根据用户是否在看终端调整自主程度
- **首次唤醒**：问候用户并等待指示，不要自行开始探索

### 8.2 协调器模式

```typescript
if (feature('COORDINATOR_MODE') && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) {
  return asSystemPrompt([
    getCoordinatorSystemPrompt(),
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}
```

### 8.3 自定义代理

```typescript
// 代理提示词可以替换或追加到默认提示词
if (agentSystemPrompt && isProactiveActive()) {
  // 自主模式：追加（代理指令叠加在自主代理提示词之上）
  return [...defaultSystemPrompt, `\n# Custom Agent Instructions\n${agentSystemPrompt}`]
} else if (agentSystemPrompt) {
  // 普通模式：替换
  return [agentSystemPrompt]
}
```

---

## 九、提示词优先级体系

`buildEffectiveSystemPrompt()` 定义了清晰的优先级：

```
0. overrideSystemPrompt  — 最高优先级，完全替换（如 loop 模式）
1. coordinatorSystemPrompt — 协调器模式
2. agentSystemPrompt — 自定义代理
   - 自主模式：追加到默认提示词
   - 普通模式：替换默认提示词
3. customSystemPrompt — --system-prompt 参数
4. defaultSystemPrompt — 标准 Claude Code 提示词

+ appendSystemPrompt 始终追加到末尾（除 override 外）
```

---

## 十、关键设计模式总结

| 模式 | 实现 | 目的 |
|------|------|------|
| 静态/动态分界 | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` | 最大化 prompt cache 命中率 |
| Section 缓存 | `systemPromptSection()` | 避免不必要的重计算 |
| DANGEROUS 命名 | `DANGEROUS_uncachedSystemPromptSection()` | 提醒开发者 cache 破坏成本 |
| 条件编译 | `feature()` + `process.env.USER_TYPE` | 内外用户差异化 + 死代码消除 |
| 反模式修正 | 具体的"不要做X"指令 | 对抗 LLM 的过度工程化倾向 |
| 数值锚点 | "≤25 words" / "≤100 words" | 比定性描述更有效的输出控制 |
| 风险框架 | 可逆性 × 影响范围矩阵 | 结构化的操作安全评估 |
| 信息抑制 | 卧底模式下移除模型信息 | 防止内部信息泄露 |
| 分析-摘要模式 | `<analysis>` + `<summary>` | 提高压缩摘要质量（分析后丢弃） |
