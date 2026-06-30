# 0003 - Skill Loading：按需加载的技能系统

> **核心思想**：_"用到时再加载，别全塞 prompt 里"_ — 技能内容通过 `newMessages`（新消息列表）注入对话，不作为 system prompt 的一部分。启动时只注入技能"目录"（名字+简介），运行时模型调用 `Skill` 工具时才展开完整内容。

---

## 前置知识

本文假设你已了解：

- Claude Code 的工具调用架构（`Tool` 基类、`ToolUseContext`、`buildTool` 模式）
- 消息类型（`user`、`assistant`、`tool_use`、`tool_result`）和对话流
- system-reminder / attachment（附件）的概念
- MCP (Model Context Protocol) 的基本概念

---

## 一、概述：要解决什么问题？

假设你的项目有一套 React 组件规范（2000 行）、一份 SQL 风格指南（1500 行）、一份 API 设计文档（3000 行）。如果一股脑全塞进 system prompt，Agent 每次调用 LLM 都带着这 6500 行，不管当前任务是改 CSS 颜色还是修 SQL 查询 — 99% 的内容和当前任务无关，白白消耗 token 和上下文窗口。

Claude Code 的技能系统通过**两级加载**设计解决了这个问题：

| 层                   | 位置                           | 时机                  | 代价                        |
| -------------------- | ------------------------------ | --------------------- | --------------------------- |
| **第一级：技能目录** | system-reminder 附件           | 启动/每轮注入         | ~100 tokens/skill，每轮都带 |
| **第二级：技能内容** | tool_result 返回的 newMessages | 模型调用 Skill 工具时 | ~2000 tokens/skill，按需    |

---

## 二、一个完整示例：从用户视角看技能加载

在深入源码之前，先看看一个完整的用户交互流程：

```
用户: "帮我 code review 最近的改动"

【第一级：模型已知的技能目录】
之前注入的 skill_listing 附件包含了:
  - code-review: 审查代码差异，发现bug — Use when asked to review a PR
  - commit: 自动生成提交信息 — Use when you've made changes

模型根据自己的理解，决定调用 code-review 技能。

【第二级：模型调用 Skill 工具】
> Skill({ skill: "code-review" })

CC 内部流程:
1. validateInput: ✓ 技能存在，类型为 prompt
2. checkPermissions: command.source = 'bundled' → safe-property check → ALLOW
3. call:
   - 找到 bundled command
   - 调用 getPromptForCommand() → 返回完整的 SKILL.md 内容
   - 内容约 3000 tokens
   - 通过 newMessages 注入为 user 消息
4. Tool result 显示: "Launching skill: code-review"

【后续轮次】
模型看到了完整的 code-review 指引，按步骤执行:
  > Bash("git diff")
  > Read("src/file1.ts")
  > Read("src/file2.ts")
  (分析差异...)
  (输出审查报告...)
```

---

## 三、流程总览：一张图看懂两级加载

```
                     启动阶段
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
    ▼                    ▼                    ▼
 initBundledSkills   getSkillDirCommands   getPluginSkills
 (注册内置技能)      (扫描磁盘目录)        (加载插件技能)
    │                    │                    │
    └────────────────────┼────────────────────┘
                         │
                         ▼
                   getCommands()
                   (聚合所有命令)
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
    getSkillToolCommands   getSlashCommandToolSkills
    (模型可见列表)         (用户调用技能列表)
              │                     │
              └──────────┬──────────┘
                         │
                         ▼
            formatCommandsWithinBudget()
            (预算控制 + 格式化)
                         │
                         ▼
              skill_listing 附件 ← 注入到 system-reminder
                         │
    ═════════════════════╪═════════════════════  第一级完成
                         │
                     运行时
                         │
                         ▼
              模型调用 Skill 工具
                         │
              ┌──────────┼──────────┐
              │                     │
              ▼                     ▼
         inline 模式            fork 模式
              │                     │
              ▼                     ▼
    processSlashCommand()   executeForkedSkill()
    getPromptForCommand()   子 Agent 运行
    registerSkillHooks()    提取结果文本
    addInvokedSkill()
              │                     │
              └──────────┬──────────┘
                         │
                         ▼
              newMessages 注入对话
              contextModifier 修改上下文
              (allowed-tools + model + effort)
                         │
    ═════════════════════╪═════════════════════  第二级完成
                         │
                     后续轮次
                         │
                         ▼
              模型读到技能内容
              按 SKILL.md 指引
              调用后续工具 (Read, Bash, Grep...)
                         │
                         ▼
              上下文压缩时
              addInvokedSkill() 的内容被保留
              (agentId 隔离，只保留活跃 Agent 的技能)
```

### 关键设计决策

1. **内容不作为 system prompt**：技能内容通过 `newMessages` 以 `user` 消息注入，进入当前 messages 列表。后续调用会随历史携带，直到上下文压缩或截断。这样不浪费每轮的 system prompt 配额。

2. **SKILL.md 是"指南针"而非"牢笼"**：SKILL.md 内容可以指引模型后续通过现有工具（Read、Bash、Grep 等）访问 `references/`、`scripts/`、`assets/` 等伴随资源。技能不是封闭的，而是通向更丰富资源的入口。

3. **与上下文压缩的天然衔接**：按需加载解决"不该提前带的不要带"，后续章节的 compact 机制解决"该丢的怎么丢"。

---

## 四、技能来源全景

Claude Code 实际从 **8 个来源** 加载技能，远比教学版的单一 `skills/` 目录丰富：

```typescript
// 来源类型 (loadSkillsDir.ts:67-73)
export type LoadedFrom =
  | 'commands_DEPRECATED' // 旧格式 .claude/commands/
  | 'skills' // 新格式 .claude/skills/
  | 'plugin' // 插件提供的技能
  | 'managed' // 策略管理 (IT/admin 推送)
  | 'bundled' // 内置技能 (编译进二进制)
  | 'mcp' // MCP 服务器远程技能
```

### 各来源详解

| 来源                 | 目录/注册方式                       | 加载时机              | 说明                        |
| -------------------- | ----------------------------------- | --------------------- | --------------------------- |
| **User skills**      | `~/.claude/skills/`                 | 启动时扫描            | 用户个人技能，跨项目可用    |
| **Project skills**   | `.claude/skills/` (向上遍历到 HOME) | 启动时扫描            | 项目级技能，团队共享        |
| **Managed skills**   | `managed/.claude/skills/`           | 启动时扫描            | IT/admin 策略推送的受管技能 |
| **--add-dir skills** | `--add-dir/.claude/skills/`         | 启动时扫描            | 额外目录的技能              |
| **Legacy commands**  | `.claude/commands/`                 | 启动时扫描            | 旧格式命令，兼容过渡        |
| **Bundled skills**   | 代码中 `registerBundledSkill()`     | `initBundledSkills()` | ~15个内置技能，编译进二进制 |
| **Plugin skills**    | 插件 manifest 中的 `skills` 字段    | 启动时加载            | 插件提供的技能              |
| **MCP skills**       | MCP 服务器的 prompt 类命令          | 动态获取              | 远程 MCP 服务器提供的技能   |

### 加载与聚合流程 (`commands.ts`)

```typescript
// 1. getSkills() 并行加载四个来源
async function getSkills(cwd: string) {
  const [skillDirCommands, pluginSkills] = await Promise.all([
    getSkillDirCommands(cwd),   // 扫描磁盘: user/project/managed/--add-dir + legacy commands
    getPluginSkills(),           // 加载插件技能
  ])
  const bundledSkills = getBundledSkills()           // 内置技能 (同步)
  const builtinPluginSkills = getBuiltinPluginSkillCommands() // 内置插件技能
  return { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills }
}

// 2. loadAllCommands() 聚合成完整命令列表
const loadAllCommands = memoize(async (cwd) => {
  const [{ skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
         pluginCommands, workflowCommands] = await Promise.all([...])
  return [
    ...bundledSkills,          // 优先: 内置技能
    ...builtinPluginSkills,    // 内置插件技能
    ...skillDirCommands,       // 磁盘技能
    ...workflowCommands,       // 工作流命令
    ...pluginCommands,         // 插件命令
    ...pluginSkills,           // 插件技能
    ...COMMANDS(),             // 内置命令 (/help, /clear 等)
  ]
})

// 3. getSkillToolCommands() 筛选模型可见的技能
export const getSkillToolCommands = memoize(async (cwd) => {
  const allCommands = await getCommands(cwd)
  return allCommands.filter(cmd =>
    cmd.type === 'prompt' &&
    !cmd.disableModelInvocation &&
    cmd.source !== 'builtin' &&
    // 必须有描述或 whenToUse（确保模型知道何时使用）
    (cmd.loadedFrom === 'bundled' ||
     cmd.loadedFrom === 'skills' ||
     cmd.loadedFrom === 'commands_DEPRECATED' ||
     cmd.hasUserSpecifiedDescription ||
     cmd.whenToUse)
  )
})
```

### 文件格式：只支持目录格式

在 `/skills/` 目录下，**只支持目录格式**，不支持单文件 `.md`：

```
.claude/skills/
  code-review/
    SKILL.md          ← 技能主文件 (必须有)
    references/       ← 可选的伴随资源
    scripts/          ← 可选的脚本
  sql-style/
    SKILL.md
  ...
```

Legacy `/commands/` 目录**同时支持**目录格式（`SKILL.md`）和单文件格式（`something.md`），但这是为了向后兼容，新技能应使用 `/skills/` 目录。

---

## 五、第一级：启动时注入"目录"

### 5.1 技能目录如何进入 system prompt

技能的"目录"不直接放在 system prompt 正文中，而是以 **`skill_listing` 附件** 的形式注入：

```typescript
// utils/attachments.ts:2661-2750
async function getSkillListingAttachments(toolUseContext) {
  // 1. 获取模型可调用的所有技能
  const localCommands = await getSkillToolCommands(cwd)
  const mcpSkills = getMcpSkillCommands(...)

  // 2. 去重 (按名称)
  let allCommands = uniqBy([...localCommands, ...mcpSkills], 'name')

  // 3. 智能过滤: 如果启用技能搜索实验，只展示 bundled + MCP
  //    (用户/项目/插件技能通过异步发现机制)
  if (feature('EXPERIMENTAL_SKILL_SEARCH') && isSkillSearchEnabled()) {
    allCommands = filterToBundledAndMcp(allCommands)
  }

  // 4. 追踪已发送: 去重，跨轮不重复发送
  const newSkills = allCommands.filter(cmd => !sent.has(cmd.name))
  for (const cmd of newSkills) sent.add(cmd.name)

  // 5. 预算控制: 最多占上下文窗口的 1% (上限 8000 字符)
  const contextWindowTokens = getContextWindowForModel(...)
  const content = formatCommandsWithinBudget(newSkills, contextWindowTokens)

  return [{ type: 'skill_listing', content, skillCount, isInitial }]
}
```

### 5.2 预算控制（`SkillTool/prompt.ts`）

```typescript
// 技能列表预算: 上下文窗口的 1%
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01
export const DEFAULT_CHAR_BUDGET = 8_000 // 回退: 200k × 4 × 1%

// 每条描述最长 250 字符
export const MAX_LISTING_DESC_CHARS = 250

// 超额时: 内置技能保留完整描述，其余技能截断
// 极端情况: 非内置技能仅显示名称
```

格式化示例：

```
- code-review: 审查代码差异，发现正确性bug和复用/简化/效率问题 — Use when asked to review a PR
- commit: 自动生成提交信息并提交 — Use when you've made changes and want to commit them
- update-config: 配置 Claude Code harness — Use this skill to configure hook settings
```

### 5.3 技能去重

- 通过 `realpath` 解析符号链接 → 用真实路径去重
- 优先级：先加载的保留（managed > user > project > additional > legacy）
- 动态技能（通过文件操作发现）不与已有技能重复添加

---

## 六、第二级：Skill 工具 — 调用时加载

### 6.1 工具定义

```typescript
// SkillTool.ts:331
export const SkillTool: Tool<InputSchema, Output, Progress> = buildTool({
  name: 'Skill', // 工具名称
  searchHint: 'invoke a slash-command skill', // 自动分类提示
  maxResultSizeChars: 100_000,

  // 输入 Schema: 技能名 + 可选参数
  inputSchema: z.object({
    skill: z
      .string()
      .describe('The skill name. E.g., "commit", "review-pr", or "pdf"'),
    args: z.string().optional().describe('Optional arguments for the skill'),
  }),
  // ...
})
```

### 6.2 完整的调用生命周期

```
用户: "用 code-review 技能检查我的改动"
  ↓
模型调用: Skill({ skill: "code-review" })

  ↓ ① validateInput()
  ├── 允许带前导 / (/code-review → code-review)
  ├── 查找命令是否存在于注册表
  ├── 检查 disableModelInvocation (禁止模型调用)
  └── 检查是否为 prompt 类型 (非 local/jsx)

  ↓ ② checkPermissions()
  ├── 检查 deny 规则 (精确匹配 + 前缀匹配: "review:*")
  ├── [远程 canonical 技能: ant-only 实验特性，自动放行]
  ├── 检查 allow 规则
  ├── 检查 safe-property auto-allow (仅使用安全属性 → 自动放行)
  └── 否则 → 弹出权限对话框

  ↓ ③ call()
  ├── 记录技能使用 (usage tracking)
  ├── command.context === 'fork'? → executeForkedSkill()
  │   ├── 创建子 Agent
  │   ├── 在隔离环境中运行技能
  │   └── 返回结果文本
  │
  └── command.context === 'inline'? → processPromptSlashCommand()
      ├── getPromptForCommand() 展开 SKILL.md
      │   ├── 拼接: "Base directory for this skill: <dir>\n\n" + content
      │   ├── 替换 ${CLAUDE_SKILL_DIR} 占位符
      │   ├── 替换 ${CLAUDE_SESSION_ID} 占位符
      │   ├── 执行内联 shell 命令 (!`...`)
      │   └── 替换参数变量 ($ARGUMENTS)
      │
      ├── registerSkillHooks() 注册技能 hooks
      ├── addInvokedSkill() 记录到全局状态 (用于 compact 保留)
      └── 返回 newMessages → 注入对话

  ↓ ④ mapToolResultToToolResultBlockParam()
  ├── Forked 模式: "Skill completed. Result: ..."
  └── Inline 模式: "Launching skill: code-review"
```

### 6.3 `getPromptForCommand` — 内容展开的核心

```typescript
// loadSkillsDir.ts:344-399 (在 createSkillCommand 中定义)
async getPromptForCommand(args, toolUseContext) {
  let finalContent = baseDir
    ? `Base directory for this skill: ${baseDir}\n\n${markdownContent}`
    : markdownContent

  // 1. 参数替换 ($ARGUMENTS → 实际参数)
  finalContent = substituteArguments(finalContent, args, true, argumentNames)

  // 2. 占位符替换 (仅在 baseDir 存在时替换 ${CLAUDE_SKILL_DIR})
  //    Windows 平台会额外做反斜杠归一化处理
  if (baseDir) {
    const skillDir = process.platform === 'win32'
      ? baseDir.replace(/\\/g, '/')
      : baseDir
    finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
  }
  // ${CLAUDE_SESSION_ID} 始终替换
  finalContent = finalContent.replace(/\$\{CLAUDE_SESSION_ID\}/g, getSessionId())

  // 3. 执行内联 shell 命令 (安全: MCP 技能跳过)
  if (loadedFrom !== 'mcp') {
    finalContent = await executeShellCommandsInPrompt(finalContent, ...)
  }

  return [{ type: 'text', text: finalContent }]
}
```

### 6.4 `newMessages` 注入 — 关键区别

**这是与初版实现最关键的不同**：技能内容不作为 system prompt 的一部分，而是作为 `newMessages` 通过 `tool_result` 返回：

```typescript
// SkillTool.ts:735-755
const newMessages = tagMessagesWithToolUseID(
  processedCommand.messages.filter((m) => {
    if (m.type === 'progress') return false // 过滤进度消息
    if (m.type === 'user' && content.includes('<command-message>')) return false
    return true
  }),
  toolUseID,
)

return {
  data: { success: true, commandName, allowedTools, model },
  newMessages, // ← 技能内容在这里，不是 system prompt!
  contextModifier(ctx) {
    // 动态修改执行上下文 (详见 5.5)
    return modifiedContext
  },
}
```

tool_result 的展示文本只是 `"Launching skill: code-review"`，真正的技能内容已被注入到后续的 messages 中，接下来轮次模型就能看到。

### 6.5 `contextModifier` — 技能对执行环境的动态修改

当技能通过 `newMessages` 注入后，`contextModifier` 函数会修改当前工具的上下文。它做三件事（`SkillTool.ts:775-840`）：

**① 注入 allowed-tools（自动允许工具）**

```typescript
if (allowedTools.length > 0) {
  // 将技能声明的 allowed-tools 合并到 alwaysAllowRules 中
  // 这意味着技能列出的工具无需权限弹窗，直接可用
  alwaysAllowRules: {
    ...prev.alwaysAllowRules,
    command: [...new Set([...existing, ...allowedTools])]
  }
}
```

**② 覆盖模型（保留 `[1m]` 窗口后缀）**

```typescript
if (model) {
  // resolveSkillModelOverride() 会保留父会话的 [1m] 后缀
  // 否则 model: opus 在 opus[1m] 会话中会掉回 200K 窗口
  mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel)
}
```

**③ 覆盖 effort（推理努力级别）**

```typescript
if (effort !== undefined) {
  appState.effortValue = effort
}
```

这三个修改使技能可以临时改变执行环境，且在技能结束后不影响主会话的默认设置。

### 6.6 Skill 工具为何设计为普通工具

Skill 被设计为一个普通的工具函数（`buildTool` 模式），而非特殊机制。这意味着模型可以像调用 `Read`、`Bash` 一样自然地调用它，且享受与普通工具完全相同的权限检查、输入验证、遥测记录等基础设施。这也可以防止模型同时调用多个技能（工具的并发限制）。

---

## 七、SKILL.md 格式与 Frontmatter

### 7.1 文件格式

```markdown
---
name: Code Review
description: 审查代码差异，发现bug和简化机会
when_to_use: 当用户要求 code review 或 PR review 时
allowed-tools: Read, Bash(gh:*), Grep
model: opus
context: fork
user-invocable: true
---

# Code Review Skill

你是一个代码审查专家。当你被调用时，请按照以下步骤操作：

1. 首先读取 git diff
2. 逐文件分析...
3. 输出结构化的审查报告...
```

### 7.2 完整的 Frontmatter 字段

由 `parseSkillFrontmatterFields()` 解析（`loadSkillsDir.ts:185-265`），注意：`paths` 字段由独立的 `parseSkillPaths()` 函数单独解析（见第十章条件技能）：

| 字段                       | 内部表示                                  | 用途                                     | 默认值                   |
| -------------------------- | ----------------------------------------- | ---------------------------------------- | ------------------------ |
| `name`                     | → `displayName`                           | 显示名称                                 | 目录名                   |
| `description`              | → `description`                           | 技能描述                                 | 自动从 markdown 首行提取 |
| `when_to_use`              | → `whenToUse`                             | 指导模型何时调用此技能                   | -                        |
| `allowed-tools`            | → `allowedTools`                          | 技能可用工具的自动允许列表               | `[]`                     |
| `model`                    | → `model`                                 | 模型覆盖 (haiku/sonnet/opus/inherit)     | 继承主会话               |
| `context`                  | → `executionContext: 'fork' \| undefined` | 在子 Agent 中运行                        | `undefined` (即 inline)  |
| `agent`                    | → `agent`                                 | 指定子 Agent 类型                        | -                        |
| `effort`                   | → `effort`                                | 推理努力级别 (low/medium/high/xhigh/max) | 继承                     |
| `disable-model-invocation` | → `disableModelInvocation`                | 禁止模型通过 Skill 工具调用              | `false`                  |
| `user-invocable`           | → `userInvocable`                         | 用户可以通过 `/name` 调用                | `true`                   |
| `hooks`                    | → `hooks`                                 | 技能级别的 hook 配置（详见进阶主题）     | -                        |
| `arguments`                | → `argumentNames`                         | 参数名列表                               | -                        |
| `argument-hint`            | → `argumentHint`                          | 参数提示文本                             | -                        |
| `version`                  | → `version`                               | 技能版本号                               | -                        |
| `shell`                    | → `shell`                                 | 内联 shell 命令的执行配置                | -                        |

> **注**：`paths`（条件激活的 glob 模式）由独立的 `parseSkillPaths()` 函数解析（`loadSkillsDir.ts:159-178`），不在 `parseSkillFrontmatterFields()` 返回值中。详见第十章条件技能。

### 7.3 描述提取优先级

```typescript
// 1. 优先用 frontmatter 中的 description
const validatedDescription = coerceDescriptionToString(frontmatter.description)

// 2. 若无，从 markdown 第一行提取 (去掉 # 前缀)
const description =
  validatedDescription ??
  extractDescriptionFromMarkdown(markdownContent, 'Skill')
```

### 7.4 技能命名规范

技能的最终名称遵循以下规则（`loadSkillsDir.ts:536-559`）：

**在 `/skills/` 目录下**：技能名 = 目录名

```
.claude/skills/code-review/SKILL.md  →  技能名: "code-review"
```

**在 legacy `/commands/` 目录下**：

- 嵌套子目录用 `:` 分隔，形成命名空间：
  ```
  .claude/commands/frontend/react/SKILL.md  →  技能名: "frontend:react"
  ```
- 非 SKILL.md 的单文件 `.md` 以文件名（去掉 .md）作为命令名：
  ```
  .claude/commands/frontend/patterns.md  →  命令名: "frontend:patterns"
  ```

**MCP 技能**：使用 `serverName:skillName` 格式，如 `"my-server:pdf"`。

**模型调用时**：`Skill` 工具接受可选的 leading `/`（`"/code-review"` 和 `"code-review"` 等价），会自动去除。

---

## 八、技能的执行模式：Inline vs Fork

### 8.1 Inline 模式（默认）

技能内容被展开为 `user` 消息注入当前对话，模型在当前上下文中处理技能指令。适用于轻量级技能。

```
┌────────────────────────────────────┐
│  当前对话                          │
│                                    │
│  User: "用 code-review 检查改动"   │
│  Assistant: [调用 Skill 工具]      │
│  Tool Result: "Launching skill..." │
│  [newMessages 包含 SKILL.md 内容]  │ ← 内联注入
│  Assistant: "我来分析 diff..."     │ ← 模型读到技能内容后回复
└────────────────────────────────────┘
```

### 8.2 Fork 模式（`context: fork`）

技能在独立的子 Agent 中运行，有自己的 token 预算和上下文隔离。适用于重操作或需要隔离的技能。

```typescript
// SkillTool.ts:122-289
async function executeForkedSkill(command, commandName, args, context, ...) {
  const agentId = createAgentId()
  const { modifiedGetAppState, baseAgent, promptMessages, skillContent } =
    await prepareForkedCommandContext(command, args || '', context)

  // 从 forked agent 收集消息
  const agentMessages: Message[] = []

  // 在子 Agent 中运行
  for await (const message of runAgent({
    agentDefinition: command.effort ? { ...baseAgent, effort: command.effort } : baseAgent,
    promptMessages,
    toolUseContext: { ...context, getAppState: modifiedGetAppState },
    // ...
  })) { /* 收集消息 */ }

  // 提取结果文本
  const resultText = extractResultText(agentMessages, 'Skill execution completed')

  // 提取结果后释放消息内存
  agentMessages.length = 0

  return { data: { success: true, commandName, status: 'forked', agentId, result: resultText } }
}
```

### 8.3 I/O 对比

| 维度           | Inline                            | Fork                                        |
| -------------- | --------------------------------- | ------------------------------------------- |
| **上下文**     | 与主对话共享                      | 隔离的子 Agent                              |
| **token 消耗** | 随技能长度增加                    | 分离预算                                    |
| **结果可见性** | tool_result: "Launching skill: X" | tool_result: "Skill completed. Result: ..." |
| **适用场景**   | 指令型、简单扩展                  | 重操作、需隔离                              |
| **进度反馈**   | 无                                | 可通过 `onProgress` 报告                    |

---

## 九、技能的权限系统

### 9.1 权限检查流程

```typescript
// SkillTool.ts:432-578
async checkPermissions({ skill, args }, context) {
  // 1. Deny 规则检查 (精确 + 前缀: "review:*")
  const denyRules = getRuleByContentsForTool(permissionContext, SkillTool, 'deny')
  for (const [ruleContent, rule] of denyRules) {
    if (ruleMatches(ruleContent)) return { behavior: 'deny', ... }
  }

  // 2. 远程 canonical 技能自动授权 (ant-only 实验)
  //    deny 之后、allow 之前，确保用户配置的 deny 规则优先

  // 3. Allow 规则检查
  const allowRules = getRuleByContentsForTool(permissionContext, SkillTool, 'allow')
  for (const [ruleContent, rule] of allowRules) {
    if (ruleMatches(ruleContent)) return { behavior: 'allow', ... }
  }

  // 4. Safe-property auto-allow
  //    如果技能只使用"安全"属性 → 自动允许，不弹对话框
  if (skillHasOnlySafeProperties(commandObj)) {
    return { behavior: 'allow', ... }
  }

  // 5. 否则 → 弹出权限对话框
  return { behavior: 'ask', message: `Execute skill: ${commandName}`, suggestions }
}
```

### 9.2 Safe-property auto-allow 机制

这是一个精巧的默认安全设计：

```typescript
// 只有在此白名单内的属性才被视为"安全"
const SAFE_SKILL_PROPERTIES = new Set([
  'type', 'name', 'description', 'hasUserSpecifiedDescription',
  'isEnabled', 'isHidden', 'aliases', 'argumentHint', 'whenToUse',
  'version', 'disableModelInvocation', 'userInvocable', 'loadedFrom',
  'model', 'effort', 'source', 'skillRoot', 'context', 'agent', ...
])

function skillHasOnlySafeProperties(command) {
  for (const key of Object.keys(command)) {
    if (SAFE_SKILL_PROPERTIES.has(key)) continue
    const value = command[key]
    if (value === undefined || value === null) continue
    if (Array.isArray(value) && value.length === 0) continue
    if (typeof value === 'object' && Object.keys(value).length === 0) continue
    return false  // 发现非安全属性 → 需要权限
  }
  return true  // 全是安全属性 → 自动放行
}
```

**设计意图**：新增加的属性默认不在白名单中 → 默认需要权限。这确保未来的扩展不会意外造成安全漏洞。

### 9.3 权限建议

对话框提供两种建议：

1. **精确匹配**：允许特定技能 `Skill(code-review)`
2. **前缀匹配**：允许技能族 `Skill(review:*)`（支持参数化技能）

---

## 十、技能与上下文压缩（Compact）

### 10.1 技能在 compact 中的保留

当发生上下文压缩时，已调用的技能内容需要被保留。Claude Code 通过全局状态追踪来实现：

```typescript
// bootstrap/state.ts:1502-1563

// 记录技能调用
function addInvokedSkill(skillName, skillPath, content, agentId) {
  const key = `${agentId ?? ''}:${skillName}`
  STATE.invokedSkills.set(key, {
    skillName, // 技能名
    skillPath, // 文件路径 (用于恢复时重新加载)
    content, // 展开后的完整内容
    invokedAt: Date.now(),
    agentId, // 所属 Agent (fork 模式时隔离)
  })
}

// compact 清理时: 删除非活跃 Agent 的技能，保留活跃的
function clearInvokedSkills(preservedAgentIds) {
  if (!preservedAgentIds) {
    STATE.invokedSkills.clear()
    return
  }
  for (const [key, skill] of STATE.invokedSkills) {
    if (skill.agentId === null || !preservedAgentIds.has(skill.agentId)) {
      STATE.invokedSkills.delete(key) // 非活跃Agent的技能 → 清理
    }
  }
}
```

### 10.2 设计要点

- 技能按 `agentId` 隔离：fork 执行子 Agent 的技能不与主对话混淆
- `skillPath` 用于恢复：如果 compact 后需要重新加载，可以从文件路径重新读取
- `content` 保存已展开的内容：避免 compact 后重复展开时的副作用

---

## 十一、动态技能发现

### 11.1 按需发现（`discoverSkillDirsForPaths`）

当模型操作文件时，CC 会自动发现文件路径附近的技能目录：

```typescript
// loadSkillsDir.ts:861-915
export async function discoverSkillDirsForPaths(filePaths, cwd) {
  for (const filePath of filePaths) {
    let currentDir = dirname(filePath)
    // 从文件目录向上遍历到 cwd（不包括 cwd 本身）
    while (currentDir.startsWith(cwd + pathSep)) {
      const skillDir = join(currentDir, '.claude', 'skills')
      if (await fs.stat(skillDir).catch(() => false)) {
        // 检查目录是否被 gitignored
        if (await isPathGitignored(currentDir, cwd)) continue
        newDirs.push(skillDir)
      }
      currentDir = dirname(currentDir)
    }
  }
  // 深度优先排序 (最近文件的技能优先)
  return newDirs.sort(
    (a, b) => b.split(pathSep).length - a.split(pathSep).length,
  )
}
```

**使用场景**：当你有一个 monorepo，某子目录有专属技能，模型操作该子目录文件时会自动发现这些技能。

### 11.2 条件技能激活（`activateConditionalSkillsForPaths`）

技能可以在 frontmatter 中声明 `paths` 字段，仅当模型操作匹配文件时才激活：

```markdown
---
name: React Reviewer
description: React-specific code review
paths:
  - '**/*.tsx'
  - '**/*.jsx'
  - '**/components/**'
---
```

```typescript
// loadSkillsDir.ts:997-1058
export function activateConditionalSkillsForPaths(filePaths, cwd) {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths) // 使用 gitignore 风格的匹配
    for (const filePath of filePaths) {
      const relativePath = relative(cwd, filePath)
      if (skillIgnore.ignores(relativePath)) {
        // 路径匹配 → 激活技能 → 加入 dynamicSkills
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activated.push(name)
        break
      }
    }
  }
  if (activated.length > 0) skillsLoaded.emit() // 通知缓存失效
}
```

**注意事项**：

- 条件技能在启动时不显示给模型（节省列表空间）
- 只有当模型操作匹配文件时才出现
- 使用 `ignore` 库（gitignore 风格）进行路径匹配
- 激活后发出 `skillsLoaded` 信号，触发命令缓存清除

---

## 十二、内置技能（Bundled Skills）

### 12.1 注册方式

内置技能不依赖磁盘文件，直接编译在 CLI 二进制中。通过 `registerBundledSkill()` 注册：

```typescript
// skills/bundled/index.ts
export function initBundledSkills() {
  registerUpdateConfigSkill()
  registerKeybindingsSkill()
  registerVerifySkill()
  registerDebugSkill()
  registerRememberSkill()
  registerSimplifySkill()
  registerBatchSkill()
  registerStuckSkill()
  // ... 更多条件注册的技能
}

// 单个技能注册示例 (verify.ts)
registerBundledSkill({
  name: 'verify',
  description: '验证代码变更是否正确工作',
  whenToUse: '当需要确认修复有效、手动测试变更时',
  // getPromptForCommand 返回 ContentBlockParam[]
  async getPromptForCommand(args, context) { ... }
})
```

### 12.2 伴随文件提取

内置技能编译在 CLI 二进制里，没有磁盘目录，但有些技能需要附带参考资料（示例代码、schema 文件、配置模板）供模型通过 `Read`/`Grep` 访问。`files` 字段解决了这个问题。

**具体示例**——以 `verify` 技能为例（`skills/bundled/verify.ts` 和 `verifyContent.ts`）：

```typescript
// verifyContent.ts —— 编译时，文件内容被 Bun text loader 内联为字符串
import cliMd from './verify/examples/cli.md'
import serverMd from './verify/examples/server.md'

export const SKILL_FILES: Record<string, string> = {
  'examples/cli.md':    cliMd,      // key = 相对路径, value = 文件内容
  'examples/server.md': serverMd,
}

// verify.ts —— 注册时传入 files
registerBundledSkill({
  name: 'verify',
  description: DESCRIPTION,
  files: SKILL_FILES,                   // ← 伴随文件声明
  async getPromptForCommand(args) { ... }
})
```

**惰性提取机制**（`bundledSkills.ts:59-72`）：首次调用时把 `files` 写入磁盘，后续调用复用：

```typescript
let extractionPromise: Promise<string | null> | undefined

getPromptForCommand = async (args, ctx) => {
  // ??= 确保只在首次调用时提取一次，并发调用共享同一个 Promise
  extractionPromise ??= extractBundledSkillFiles(definition.name, files)
  const extractedDir = await extractionPromise
  const blocks = await inner(args, ctx)
  // 提取成功 → prompt 前加 "Base directory for this skill: <dir>"
  if (extractedDir) return prependBaseDir(blocks, extractedDir)
  return blocks
}
```

提取到磁盘后的目录结构：

```
~/.claude/bundled-skills/<per-process-nonce>/verify/
  examples/
    cli.md       ← 编译时的字符串 → 磁盘文件
    server.md    ← 同上
```

模型收到的 prompt 前缀：

```
Base directory for this skill: ~/.claude/bundled-skills/<nonce>/verify

(原来的 SKILL.md 内容...)
```

于是模型可以 `Read("examples/cli.md")` 像操作普通文件一样访问伴随资源。

**设计要点**：

| 设计             | 说明                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **惰性（lazy）** | `??=` 确保只在首次调用时提取，同进程内后续调用不重复提取                                                                  |
| **按进程隔离**   | `getBundledSkillsRoot()` 内包含 per-process nonce，不同进程的文件互不冲突                                                 |
| **安全写入**     | `resolveSkillFilePath()` 禁止 `..` 穿越和绝对路径；`O_NOFOLLOW \| O_EXCL` 防符号链接攻击；`0o700` 目录 + `0o600` 文件权限 |
| **Promise 去重** | 多个并发调用共享同一个 `extractionPromise`，不会竞争写入同一个文件                                                        |

> **非内置技能可以定义 `files` 字段吗？不能。** `files` 是 `BundledSkillDefinition` 类型的专属字段（`bundledSkills.ts:36`），`loadSkillsDir.ts` 中的 `createSkillCommand()` 和 `parseSkillFrontmatterFields()` 都没有处理 `files` 的逻辑。磁盘技能本身就有一个磁盘目录（`skillRoot`），伴随文件（`references/`、`scripts/` 等）直接放在目录里即可，不需要"提取"。

---

## 十三、技能使用追踪与遥测

### 13.1 使用追踪（`skillUsageTracking.ts`）

Claude Code 追踪每个技能的使用频率和最近使用时间，用**指数衰减评分**（7 天半衰期）对技能排名，数据持久化到用户全局配置。这驱动两个行为：

- **自动补全排序**：`/` 输入时最近常用的技能排前面
- **智能推荐**：模型可根据历史行为推荐合适的技能

### 13.2 遥测事件（`skillLoadedEvent.ts`）

会话启动时，`logSkillsLoaded()` 遍历所有 `getSkillToolCommands()` 的结果，为每个可用技能发射 `tengu_skill_loaded` 事件，记录技能名、来源、加载途径和当前字符预算。这使团队能够分析哪些技能最常被配置、哪些来源贡献了最多的技能。

---

## 十四、进阶主题

> **关于 `repl_main_thread`**：本文多处提到某些行为"仅限 `repl_main_thread`"。这是 Claude Code 中的一个 `QuerySource` 标识，代表用户在终端直接交互的 **主对话线程**。与之相对的是子 Agent 调用（`agent:builtin:*`、`agent:custom`）、SDK 调用（`sdk`）、compact 进程等。主对话是"有状态的"（连续交互、session 持久化、token 预算累积），子 Agent 是"无状态的"（执行完销毁）。因此以下行为只在主线程触发：技能自我改善、Session Memory、Prompt 建议、工具结果持久化、微压缩（micro-compact）。这个门控确保副作用不会在子 Agent 中意外发生。

### 14.1 技能 Hooks（`registerSkillHooks.ts`）

技能可以通过 frontmatter 中的 `hooks` 字段声明会话级 hook。当技能被调用时，`registerSkillHooks()` 在 `processSlashCommand` 中被调用：

```typescript
// utils/hooks/registerSkillHooks.ts
// 遍历所有 HOOK_EVENTS (PreToolUse, PostToolUse, Stop, Notification, etc.)
// 读取技能的 hooks frontmatter 配置 (已解析为 HooksSettings)
// 通过 addSessionHook() 注册为会话级 hook
```

关键特性：

- **`skillRoot` 注入**：hook 执行时，技能目录被注入为 `CLAUDE_PLUGIN_ROOT` 环境变量，使 hook 可以引用技能目录中的脚本
- **`once: true` 支持**：一次性 hook 在首次执行后自动移除
- **会话生命周期**：hook 绑定到 sessionId，会话结束后自动清理
- **策略限制**：当 hooks 被策略锁定（`isRestrictedToPluginOnly('hooks')`）时，非可信来源的技能 hooks 不注册

实际应用：技能可以安装 PreToolUse hook 在每次文件编辑前自动格式化，或 PostToolUse hook 在每次命令执行后运行 linter。

### 14.2 技能自我改善系统（`skillImprovement.ts`）

这是最精密的技能子系统之一。Claude Code 能**检测用户反馈并自动改进技能**：

```
用户调用 skill → 对话进行中 → 用户纠正或给出偏好
                                    ↓
每 5 轮触发 PostSampling hook → 小模型分析对话
                                    ↓
检测到改进点 → 终端内弹出 survey UI
  ┌──────────────────────────────────────┐
  │  Suggested improvements for          │
  │  "code-review" skill:                │
  │                                      │
  │  1. [Apply] Add TypeScript checks    │
  │     Reason: User requested TS support│
  │  0. [Dismiss]                        │
  └──────────────────────────────────────┘
      ↓ Apply
  另一个小模型读取当前 SKILL.md
  → 保留 frontmatter 和格式
  → 在适当位置插入改进内容
  → 写回 .claude/skills/<name>/SKILL.md
```

关键限制：

- 仅针对 **project skills**（非 bundled/MCP），仅限 `repl_main_thread`
- 需要 `SKILL_IMPROVEMENT` feature flag + GrowthBook 开关同时启用
- 完整遥测：`tengu_skill_improvement_detected` 和 `tengu_skill_improvement_survey` 事件

### 14.3 技能文件变更监听（`skillChangeDetector.ts`）

CC 使用 **Chokidar**（Node.js 文件监听库）监控所有技能目录的变更：

```
chokidar.watch([
  ~/.claude/skills/, .claude/skills/ (所有层级),
  --add-dir/.claude/skills/, .claude/commands/ (legacy)
])
      ↓ 文件变更
  ① 采集所有变更路径
  ② 300ms 去抖（等文件写入稳定）
  ③ 等待写入完成（1s 稳定性阈值）
  ④ 发射 ConfigChange hook
  ⑤ 清空命令缓存（memoize cache）
  ⑥ 下次 getCommands() 调用重新加载
  ⑦ resetSentSkillNames() 使技能可重新注入
```

特殊考量：

- **Bun 兼容**：使用 `stat()` 轮询（2s 间隔）替代 `fs.watch()`，回避 Bun 的 PathWatcherManager 死锁
- **ConfigChange hook**：可以返回阻塞结果阻止热重载
- **动态技能回调**：`onDynamicSkillsLoaded` 监听器在动态技能发现时触发缓存清除

### 14.4 远程 Canonical 技能

基于 `EXPERIMENTAL_SKILL_SEARCH` feature flag 的实验性子系统（ant-only）。它的目标是让模型发现和调用**远程托管**的技能，这些技能不在本地磁盘上，而是从 AKI/GCS 等远程存储加载。

#### 14.4.1 核心工作流

```
① 技能列表注入时
   skill_listing 附件被 filterToBundledAndMcp() 过滤
   → 仅展示 bundled + MCP 技能
   → 用户/项目/插件技能不出现在静态列表中（走异步发现）

② 模型需要技能时
   调用 DiscoverSkills 工具（类似 search 语义）
   → skillSearch 服务返回匹配的远程技能 slug 列表
   → 存入 session state（getDiscoveredRemoteSkill）

③ 模型调用远程技能
   Skill({ skill: "_canonical_<slug>" })
   → SkillTool 识别 _canonical_ 前缀
   → 从远程 URL 加载 SKILL.md（含本地缓存）
   → 直接注入为 user 消息（不经 processSlashCommand）
```

#### 14.4.2 源码详解

**模块加载**——远程技能相关的模块通过条件 `require` 加载（`SkillTool.ts:108-115`）：

```typescript
// 仅在 EXPERIMENTAL_SKILL_SEARCH 开启时加载，避免 akiBackend 等重型模块污染外部构建
const remoteSkillModules = feature('EXPERIMENTAL_SKILL_SEARCH')
  ? {
      ...require('../../services/skillSearch/remoteSkillState.js'), // 远程技能状态管理
      ...require('../../services/skillSearch/remoteSkillLoader.js'), // 远程技能加载器
      ...require('../../services/skillSearch/telemetry.js'), // 专用遥测
      ...require('../../services/skillSearch/featureCheck.js'), // 功能开关检查
    }
  : null
```

**validateInput — 拦截 `_canonical_` 前缀**（`SkillTool.ts:374-396`）：

```typescript
// 在查找本地命令注册表之前先拦截远程技能
// 因为远程技能不在 getCommands() 的返回列表中
if (feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant') {
  const slug = remoteSkillModules.stripCanonicalPrefix(normalizedCommandName)
  // "_canonical_react-best-practices" → slug = "react-best-practices"
  if (slug !== null) {
    const meta = remoteSkillModules.getDiscoveredRemoteSkill(slug)
    if (!meta) {
      return {
        result: false,
        message: 'Remote skill not discovered. Use DiscoverSkills first.',
      }
    }
    return { result: true } // 有效，加载在 call() 中进行
  }
}
```

**checkPermissions — 自动放行**（`SkillTool.ts:488-504`）：

```typescript
// 放在 deny 检查之后、allow 检查之前
// 意味着用户配置的 deny 规则仍然可以阻止远程技能
// 自动放行的理由是：远程技能内容由 Anthropic 策划，非用户编写
if (feature('EXPERIMENTAL_SKILL_SEARCH') && process.env.USER_TYPE === 'ant') {
  const slug = remoteSkillModules.stripCanonicalPrefix(commandName)
  if (slug !== null) {
    return { behavior: 'allow', ... }  // 自动授权
  }
}
```

**call — `executeRemoteSkill()`**（`SkillTool.ts:969-1108`）：

```typescript
async function executeRemoteSkill(slug, commandName, parentMessage, context) {
  // ① 从 session state 获取远程技能元数据（含 URL）
  const meta = getDiscoveredRemoteSkill(slug)
  if (!meta) throw new Error("Remote skill not discovered")

  // ② 从远程 URL 加载 SKILL.md
  //    loadRemoteSkill 处理 AKI (gs://) / GCS / HTTPS / S3 等协议
  const loadResult = await loadRemoteSkill(slug, meta.url)
  //    返回: { cacheHit, latencyMs, skillPath, content, fileCount, totalBytes, fetchMethod }

  // ③ 遥测
  logRemoteSkillLoaded({ slug, cacheHit, latencyMs, urlScheme, fileCount, totalBytes, fetchMethod })
  logEvent('tengu_skill_tool_invocation', { ..., execution_context: 'remote', is_remote: true })

  // ④ 剥离 YAML frontmatter + 注入 Base directory
  const { content: bodyContent } = parseFrontmatter(content, skillPath)
  let finalContent = `Base directory for this skill: ${skillDir}\n\n${bodyContent}`
  finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)

  // ⑤ 注册到 invokedSkills（与本地技能一致，compact 时保留）
  addInvokedSkill(commandName, skillPath, finalContent, agentId)

  // ⑥ 直接注入对话 — 不走 processSlashCommand
  return {
    data: { success: true, commandName, status: 'inline' },
    newMessages: tagMessagesWithToolUseID(
      [createUserMessage({ content: finalContent, isMeta: true })],
      toolUseID,
    ),
  }
}
```

#### 14.4.3 与本地技能的关键差异

| 维度         | 本地技能                                          | 远程 Canonical 技能                                        |
| ------------ | ------------------------------------------------- | ---------------------------------------------------------- |
| **命名**     | 普通名称（`"code-review"`）                       | `_canonical_<slug>`（`"_canonical_react-best-practices"`） |
| **发现方式** | 静态 `skill_listing` 附件 + 动态目录扫描          | 独立的 `DiscoverSkills` 工具调用 + session state           |
| **存储位置** | 磁盘文件                                          | 远程 AKI/GCS/HTTPS URL（带本地缓存）                       |
| **加载方式** | `getPromptForCommand()` → `processSlashCommand()` | `loadRemoteSkill()` 从 URL 拉取，不经 slash command 展开   |
| **内容处理** | `!command` shell 执行、`$ARGUMENTS` 变量替换      | 声明式 markdown，**不做任何展开**                          |
| **权限检查** | deny → allow → safe-property → ask                | deny → **自动放行**（内容经 Anthropic 策划）               |
| **遥测标记** | `execution_context: 'inline'` 或 `'fork'`         | `execution_context: 'remote'` + `is_remote: true`          |
| **列表可见** | 出现在 `skill_listing`                            | 不出现，通过 `DiscoverSkills` 按需发现                     |

#### 14.4.4 对 `skill_listing` 的反向影响

当 `EXPERIMENTAL_SKILL_SEARCH` 启用时，`skill_listing` 附件的生成逻辑发生变化（`attachments.ts:2651-2658`）：

```typescript
function filterToBundledAndMcp(commands: Command[]): Command[] {
  const filtered = commands.filter(
    (cmd) => cmd.loadedFrom === 'bundled' || cmd.loadedFrom === 'mcp',
  )
  // 如果 bundled + mcp 超过 30 个，进一步缩小到 bundled-only
  if (filtered.length > 30) {
    return filtered.filter((cmd) => cmd.loadedFrom === 'bundled')
  }
  return filtered
}
```

**为什么这么做**：远程技能发现机制承担了"长尾发现"的角色，静态列表只需保留少量"意图明确"的技能（内置 + MCP），用户/项目/插件技能（可能 200+）通过 `DiscoverSkills` 异步发现。这样既节省了每轮的 token 消耗，又让模型不被长列表淹没。

#### 14.4.5 模型如何被引导使用

system prompt 中包含专门指引（`prompts.ts:337-338`）：

> "Relevant skills are automatically surfaced each turn as 'Skills relevant to your task:' reminders. If you're about to do something those don't cover — a mid-task pivot, an unusual workflow, a multi-step plan — call DiscoverSkills with a specific description of what you're doing."

即：每轮自动推送相关技能，模型也可以通过 `DiscoverSkills` 工具主动搜索。

### 14.5 内置插件技能（Built-in Plugin Skills）

与 bundled skill 不同，内置插件技能注册在 `plugins/builtinPlugins.ts`：

- **用户可开关**：在 `/plugin` UI 中显示为 "Built-in" 分类，支持启用/禁用
- **多组件**：一个内置插件可以同时提供 skills、hooks、MCP servers
- **标识**：使用 `{name}@builtin` 格式区分于市场插件（`{name}@{marketplace}`）
- **转换**：`skillDefinitionToCommand()` 将 `BundledSkillDefinition` 映射为 `Command` 对象，`source` 设为 `'bundled'`

### 14.6 `--bare` 模式下的技能加载

当使用 `--bare` 运行时，技能发现行为大幅变化（`loadSkillsDir.ts:658-675`）：

| 模式   | User/Project/Managed/Legacy | --add-dir     | Bundled     |
| ------ | --------------------------- | ------------- | ----------- |
| 正常   | ✅ 自动扫描                 | ✅ 额外扫描   | ✅ 始终加载 |
| --bare | ❌ 全部跳过                 | ✅ 仅扫描这些 | ✅ 始终加载 |

如果 `--bare` 模式下没传 `--add-dir`，**零个磁盘技能**被加载。策略锁定仍然生效。

### 14.7 `/init` 命令的技能创建

`/init` 命令的 8 阶段工作流中有 4 个阶段直接涉及技能创建：Phase 1 询问是否要设置 skills/hooks，Phase 3 构建"偏好队列"，Phase 6 LLM 创建 `.claude/skills/<name>/SKILL.md` 文件（含完整 frontmatter，永不覆盖已有技能），Phase 7 跟随 `update-config` 技能构造 hook。这是用户获取项目技能的**主要入口**。

### 14.8 `/skills` UI 命令

用户输入 `/skills` 时渲染一个 `SkillsMenu` Ink 对话框，按来源分组展示所有已加载技能（Project / User / Managed / Plugin / MCP），每条显示名称和 token 估算（仅 frontmatter 部分），方便审计。

---

## 十五、关键文件清单

| 文件                                                 | 职责                                         | 核心导出                                                                                                                                     |
| ---------------------------------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/skills/loadSkillsDir.ts`                        | 磁盘加载 + 动态发现 + 条件激活 + 命名规范    | `getSkillDirCommands`, `createSkillCommand`, `parseSkillFrontmatterFields`, `discoverSkillDirsForPaths`, `activateConditionalSkillsForPaths` |
| `src/tools/SkillTool/SkillTool.ts`                   | Skill 工具（验证/权限/fork/remote/inline）   | `SkillTool`                                                                                                                                  |
| `src/tools/SkillTool/prompt.ts`                      | 模型提示 + 预算管理                          | `getPrompt`, `formatCommandsWithinBudget`                                                                                                    |
| `src/tools/SkillTool/UI.tsx`                         | Skill 工具 UI 渲染                           | 渲染组件                                                                                                                                     |
| `src/commands.ts`                                    | 命令/技能聚合 + 筛选                         | `getCommands`, `getSkillToolCommands`, `getSlashCommandToolSkills`                                                                           |
| `src/skills/bundledSkills.ts`                        | 内置技能注册表 + 伴随文件安全提取            | `registerBundledSkill`, `getBundledSkills`                                                                                                   |
| `src/skills/bundled/index.ts`                        | 内置技能初始化入口                           | `initBundledSkills`                                                                                                                          |
| `src/bootstrap/state.ts`                             | 技能调用状态追踪（compact 保留）             | `addInvokedSkill`, `getInvokedSkills`, `clearInvokedSkills`                                                                                  |
| `src/utils/attachments.ts`                           | 技能列表附件注入                             | `getSkillListingAttachments`                                                                                                                 |
| `src/utils/processUserInput/processSlashCommand.tsx` | 斜杠命令处理（内联展开 + hook 注册）         | `processPromptSlashCommand`                                                                                                                  |
| `src/utils/hooks/registerSkillHooks.ts`              | 技能 hooks 注册（会话级 + once 支持）        | `registerSkillHooks`                                                                                                                         |
| `src/utils/suggestions/skillUsageTracking.ts`        | 技能使用追踪（指数衰减评分）                 | `recordSkillUsage`                                                                                                                           |
| `src/utils/skills/skillChangeDetector.ts`            | 技能文件变更监听 + 热重载                    | Chokidar 文件监听                                                                                                                            |
| `src/utils/hooks/skillImprovement.ts`                | 技能自我改善（LLM 检测反馈 → 重写 SKILL.md） | `initSkillImprovement`, `applySkillImprovement`                                                                                              |
| `src/utils/telemetry/skillLoadedEvent.ts`            | 技能遥测（会话启动时记录）                   | `logSkillsLoaded`                                                                                                                            |
| `src/skills/mcpSkillBuilders.ts`                     | MCP 技能的构建函数注册（依赖注入）           | 注册点                                                                                                                                       |
| `src/commands/skills/index.ts`                       | `/skills` 命令入口                           | 命令注册                                                                                                                                     |
| `src/commands/init.ts`                               | `/init` 命令（含技能创建阶段）               | 命令注册                                                                                                                                     |

---

## 十六、总结：与教学版(s07)的对比

教学版 `learn-claude-code/s07_skill_loading` 是一个优雅的简化，抓住了核心概念：

| 维度                 | 教学版 (s07)               | Claude Code 源码                                                      |
| -------------------- | -------------------------- | --------------------------------------------------------------------- |
| **技能来源**         | 1 个 `skills/` 目录        | 8 个来源 (user/project/managed/--add-dir/legacy/bundled/plugin/MCP)   |
| **Frontmatter 字段** | `name`, `description`      | 15+ 字段 (model, context, hooks, paths, effort, agent, shell...)      |
| **加载方式**         | `tool_result` 直接返回文本 | `newMessages` 注入 user 消息，tool_result 只显示 "Launching skill: X" |
| **执行模式**         | 仅有 inline                | inline + fork (子Agent隔离) + remote (远程 canonical)                 |
| **权限模型**         | 无                         | 五级检查 (deny → remote → allow → safe-property → ask)                |
| **预算管理**         | 无                         | 上下文窗口 1%，上限 8000 字符，bundled 技能不截断                     |
| **动态发现**         | 无                         | 文件操作触发目录扫描 + 条件技能路径匹配                               |
| **压缩保留**         | 无                         | agentId 隔离的全局状态追踪                                            |
| **技能改善**         | 无                         | LLM 驱动的自我改善（PostSampling hook + survey UI）                   |
| **热重载**           | 无                         | Chokidar 文件监听 + 300ms 去抖 + ConfigChange hook                    |
| **用户创建**         | 手动写文件                 | `/init` 命令的 8 阶段工作流自动创建                                   |

教学版的简化是刻意的，它让你在 100 行 Python 中理解两级加载的核心：**目录注入 system prompt → 按需加载完整内容**。而 Claude Code 的 11000+ 行 TypeScript 实现在此基础上建立了工业级的健壮性、安全性和灵活性。
