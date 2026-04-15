# Claude Code Agent 专家学习指南：从 0 到 1

> 本文档基于 Claude Code 源码逆向分析，带你系统掌握工业级 AI Agent 的全部核心技术。
> 源码根目录：`extracted-source/src/`

---

## 目录

- [全局架构图](#全局架构图)
- [阶段 1：核心循环 — Agent 的心脏](#阶段-1核心循环--agent-的心脏)
- [阶段 2：工具系统 — Agent 的手和脚](#阶段-2工具系统--agent-的手和脚)
- [阶段 3：Prompt 工程 — Agent 的灵魂](#阶段-3prompt-工程--agent-的灵魂)
- [阶段 4：上下文管理 — Agent 的记忆](#阶段-4上下文管理--agent-的记忆)
- [阶段 5：多 Agent 编排 — 团队协作](#阶段-5多-agent-编排--团队协作)
- [阶段 6：安全与权限 — Agent 的刹车](#阶段-6安全与权限--agent-的刹车)
- [自己动手：从零实现一个 Agent 的推荐顺序](#自己动手从零实现一个-agent-的推荐顺序)
- [附录：完整文件索引](#附录完整文件索引)

---

## 全局架构图

```
用户输入
  │
  ▼
┌──────────────────────────────────────────────────┐
│  REPL.tsx (896KB)                                │
│  React/Ink 终端 UI，管理消息列表和用户交互         │
└──────────────┬───────────────────────────────────┘
               │ handlePromptSubmit → query()
               ▼
┌──────────────────────────────────────────────────┐
│  query.ts (1729行) — 核心 Agent Loop              │
│                                                   │
│  while (true) {                                   │
│    1. 消息压缩 (compact / snip / context collapse) │
│    2. 构建 System Prompt                          │
│    3. 流式调用 Anthropic API                       │
│    4. 解析响应中的 tool_use 块                     │
│    5. 执行工具 → 获取结果                          │
│    6. 追加工具结果到消息历史                        │
│    7. continue (回到步骤 1)                        │
│    8. 无 tool_use 时 break (end_turn)              │
│  }                                                │
└──────────────┬───────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────┐
│  Tool 系统 (43+ 工具)                             │
│                                                   │
│  Tool.ts (接口) → tools.ts (注册) → 分发执行       │
│  ├─ BashTool / FileReadTool / FileEditTool ...    │
│  ├─ AgentTool (子 Agent)                          │
│  └─ MCPTool (外部工具协议)                         │
│                                                   │
│  执行链路:                                         │
│  验证输入 → PreHook → 权限检查 → 执行 → PostHook  │
│  并发安全工具可并行，非安全工具串行                  │
└──────────────────────────────────────────────────┘
```

---

## 阶段 1：核心循环 — Agent 的心脏

> **目标**：理解 Agent 的本质就是 "LLM + 工具 + 循环"
> **预计精读时间**：2-3 天

### 必读文件

#### 1.1 `src/query.ts` — 核心 while(true) 循环（1729 行）

**这是整个项目最重要的文件，没有之一。**

**阅读方法**：

1. **第一遍**：只看函数签名和主循环结构
   - 搜索 `while (true)` 或 `queryLoop`，找到主循环入口
   - 忽略所有细节，只理解大框架：准备消息 → 调 API → 处理工具调用 → 循环/退出

2. **第二遍**：理解每一步做什么
   - 关注 `normalizeMessagesForAPI()` — 发给 API 前的消息规范化
   - 关注 `streamResponse` 相关逻辑 — 流式接收模型输出
   - 关注 `tool_use` 的处理分支 — 模型要求调用工具时怎么处理
   - 关注 `end_turn` 的 break 逻辑 — 模型什么时候说"我做完了"

3. **第三遍**：理解压缩机制
   - 搜索 `compact`、`snip`、`autocompact` — 这些是上下文窗口管理
   - 搜索 `maxOutputTokensRecovery` — 模型输出被截断时的恢复逻辑

**关键概念**：
```
stop_reason 的含义：
  - "end_turn"     → 模型完成了回答，break 退出循环
  - "tool_use"     → 模型要调用工具，执行后 continue
  - "max_tokens"   → 输出被截断，需要恢复后 continue
```

**要回答的问题**：
- 循环的 `State` 对象包含哪些字段？为什么需要这些字段？
- `ToolUseSummaryMessage` 是什么？为什么要生成工具使用摘要？
- 中断 (interrupt) 是怎么处理的？

---

#### 1.2 `src/QueryEngine.ts` — SDK/无头模式的入口（1295 行）

**阅读方法**：

1. 先看 `QueryEngine` 类的构造函数和公共方法
2. 重点看 `submitMessage()` — 这是 SDK 模式下每次用户输入的入口
3. 对比 `query.ts` 中的 `query()` 函数，理解它是如何封装底层循环的

**要回答的问题**：
- `QueryEngine` 和 REPL 模式的区别是什么？
- 它如何管理跨轮次的持久状态 (`mutableMessages`, `totalUsage`)？
- `AsyncGenerator<SDKMessage>` 的 yield 机制是怎么工作的？

---

#### 1.3 `src/types/message.ts` — 消息类型系统

**阅读方法**：

1. 从类型定义入手，理解每 种消息的用途
2. 画一个消息流转图：User → Assistant → ToolResult → Assistant → ...

**核心消息类型**：
```
UserMessage          → 用户输入
AssistantMessage     → 模型回复（包含 text + tool_use 块）
SystemMessage        → 系统指令
AttachmentMessage    → 附件（CLAUDE.md 内容、Hook 结果等）
ProgressMessage      → 工具执行进度
ToolUseSummaryMessage → 工具使用摘要（用于压缩后替代原始消息）
TombstoneMessage     → 被删除消息的占位符
```

**要回答的问题**：
- 为什么需要 `TombstoneMessage`？
- `AttachmentMessage` 和普通消息有什么区别？

---

## 阶段 2：工具系统 — Agent 的手和脚

> **目标**：理解如何设计一个可扩展的工具系统
> **预计精读时间**：3-5 天

### 必读文件

#### 2.1 `src/Tool.ts` — Tool 接口定义（792 行）

**这是工具系统的基石，所有工具都实现这个接口。**

**阅读方法**：

1. 先找到 `Tool<Input, Output, P>` 类型定义（泛型接口）
2. 逐个理解每个方法/属性的用途
3. 重点看 `buildTool()` 函数 — 它提供安全默认值

**核心接口成员**（按重要性排序）：

```typescript
interface Tool<Input, Output> {
  // 最重要的 3 个方法：
  name: string                           // 工具名称，模型通过名字调用
  inputSchema: ZodSchema                 // 输入参数的 Schema，自动转 JSON Schema 给模型
  call(args, context, ...): ToolResult   // 执行工具的核心逻辑

  // Prompt 相关：
  prompt(options): string                // 返回给模型的工具使用说明（在 system prompt 中）
  description(input): string             // 返回工具结果的描述文本

  // 安全相关：
  isConcurrencySafe(input): boolean      // 是否可并行执行（默认 false）
  isReadOnly(input): boolean             // 是否只读操作（默认 false）
  isDestructive(input): boolean          // 是否破坏性操作
  checkPermissions(input, context)       // 工具自身的权限检查

  // 生命周期：
  validateInput(input, context)          // 输入验证
  interruptBehavior(): 'cancel' | 'block' // 用户中断时的行为

  // UI 相关：
  renderToolUseMessage(...)              // 渲染工具调用中的 UI
  renderToolResultMessage(...)           // 渲染工具结果的 UI
}
```

**要回答的问题**：
- 为什么 `isConcurrencySafe()` 默认返回 `false`？（保守策略）
- `buildTool()` 的默认值设计体现了什么安全哲学？
- `ToolUseContext` 包含了哪些上下文信息？为什么要传递这么多？

---

#### 2.2 `src/tools.ts` — 工具注册表（389 行）

**阅读方法**：

1. 找到 `getAllBaseTools()` 函数，看所有工具是怎么注册的
2. 注意条件导入（`feature('X')` 和 `process.env.USER_TYPE === 'ant'`）— 这是特性门控
3. 找到 `assembleToolPool()` — 理解工具集合的组装逻辑（内置 + MCP + 过滤）

**要回答的问题**：
- 内置工具有哪些？为什么要分条件导入？
- `assembleToolPool` 如何处理内置工具和 MCP 工具的冲突？
- 懒加载（lazy require）是为了解决什么问题？

---

#### 2.3 `src/services/tools/toolExecution.ts` — 单个工具执行链路（1745 行）

**这是理解工具执行的完整生命周期最重要的文件。**

**阅读方法**：

1. 找到 `runToolUse()` 函数 — 这是单个工具执行的入口
2. 按顺序追踪执行步骤：

```
runToolUse() 的完整执行链路：
  ① 查找工具定义 (findToolByName)
  ② 解析和验证输入参数 (validateInput)
  ③ 运行 PreToolUse Hooks (runPreToolUseHooks)
  ④ 解析 Hook 权限决策 (resolveHookPermissionDecision)
  ⑤ 如需用户确认，调用 canUseTool()
  ⑥ 调用 tool.call() 执行工具
  ⑦ 处理工具结果（大结果持久化到磁盘）
  ⑧ 运行 PostToolUse Hooks
  ⑨ 生成工具结果消息
```

3. 特别关注权限被拒绝时的处理 — 工具不会直接失败，而是返回一个"被拒绝"的结果消息

**要回答的问题**：
- 大结果为什么要持久化到磁盘？
- Hook 返回的权限决策有哪些类型？如何与规则系统合并？
- 工具执行失败后，Agent 如何知道并恢复？

---

#### 2.4 `src/services/tools/toolOrchestration.ts` — 工具并发编排（188 行）

**阅读方法**：

1. 找到 `runTools()` 函数
2. 看 `partitionToolCalls()` — 如何把工具调用分为"并发安全"和"不安全"两组
3. 看 `runToolsConcurrently()` 和 `runToolsSerially()` 的实现

**核心逻辑**：
```
模型一次返回多个 tool_use 块时：
  ├─ 并发安全组（多个只读工具）→ Promise.all 并行执行
  └─ 非安全组（写操作）→ 串行执行，依次等待

最大并发数：CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY（默认 10）
```

**要回答的问题**：
- 为什么要限制最大并发数？
- 如果并发执行中某个工具失败，其他工具怎么处理？

---

#### 2.5 `src/services/tools/StreamingToolExecutor.ts` — 流式工具执行

**阅读方法**：

1. 理解核心思想：模型还在流式输出时，已经完整接收的工具调用就可以开始执行
2. 看 `addTool(block)` — 如何把新接收到的工具加入执行队列
3. 看 `canExecuteTool()` — 并发条件判断

**要回答的问题**：
- 流式执行 vs 等全部接收完再执行，性能差异有多大？
- 如何保证工具结果的顺序正确？

---

#### 2.6 精读一个具体工具：BashTool

**文件结构**：
```
src/tools/BashTool/
├── BashTool.tsx         (1143 行) — 核心实现
├── prompt.ts            (369 行)  — 给模型的指令文本
├── bashPermissions.ts   — 权限检查
├── bashSecurity.ts      — 安全相关
├── shouldUseSandbox.ts  — 沙箱判断
├── commandSemantics.ts  — 命令语义分析
├── destructiveCommandWarning.ts — 危险命令警告
├── pathValidation.ts    — 路径验证
├── readOnlyValidation.ts — 只读验证
├── modeValidation.ts    — 模式验证
├── sedEditParser.ts     — sed 命令解析
├── sedValidation.ts     — sed 命令验证
├── commentLabel.ts      — 注释标签
├── toolName.ts          — 工具名常量
├── utils.ts             — 工具函数
├── bashCommandHelpers.ts — 命令辅助
└── UI.tsx               — UI 组件
```

**阅读顺序**：
1. **`prompt.ts`** 先看这个！理解模型看到的工具说明
2. **`BashTool.tsx`** — 看 `call()` 方法，理解命令执行的核心逻辑
3. **`bashPermissions.ts`** — 理解如何判断一个命令是否需要用户确认
4. **`shouldUseSandbox.ts`** — 理解沙箱机制

**要回答的问题**：
- BashTool 如何判断一个命令是"安全"还是"危险"的？
- 沙箱模式的工作原理是什么？
- 为什么 sed/awk 等编辑命令需要特殊处理？

---

## 阶段 3：Prompt 工程 — Agent 的灵魂

> **目标**：理解如何通过 Prompt 让模型正确使用工具
> **预计精读时间**：2-3 天

### 必读文件

#### 3.1 `src/constants/prompts.ts` — 系统提示词主构建器（914 行）

**阅读方法**：

1. 找到 `getSystemPrompt()` 函数
2. 看它如何动态组装系统提示词：
   - 模型信息 + 环境信息（OS、工作目录、git 状态）
   - 每个工具的 `prompt()` 输出
   - MCP 服务器信息
   - 输出样式配置
3. 特别关注**约束指令**（告诉模型不要做什么的部分）— 这些是从实践中总结的经验

**要回答的问题**：
- 系统提示词的各个部分按什么顺序排列？为什么？
- 如何确保工具的 prompt 不会互相冲突？
- 环境信息为什么要动态注入？

---

#### 3.2 `src/utils/systemPrompt.ts` — 提示词优先级选择

**阅读方法**：

1. 找到 `buildEffectiveSystemPrompt()` 函数
2. 理解优先级链：

```
覆盖提示词 (loop mode) > 协调器提示词 > 代理提示词 > 自定义提示词 > 默认提示词
追加提示词 (appendSystemPrompt) 始终加在末尾
```

**要回答的问题**：
- 为什么需要这么多层级？
- 追加提示词为什么不参与优先级比较？

---

#### 3.3 各工具的 prompt.ts — 工具级提示词

**建议阅读**：
- `src/tools/BashTool/prompt.ts` — 最复杂的工具提示词之一
- `src/tools/FileEditTool/prompt.ts` — 文件编辑的精确指令
- `src/tools/AgentTool/prompt.ts` — 子 Agent 的调用指令
- `src/tools/GrepTool/prompt.ts` — 搜索工具的提示词设计

**阅读方法**：
1. 把自己当成模型，读 prompt，看你能理解多少
2. 注意"不要做什么"的指令 — 这些是血泪教训
3. 注意示例（few-shot）的使用

---

## 阶段 4：上下文管理 — Agent 的记忆

> **目标**：理解如何让 Agent 在有限的上下文窗口中工作
> **这是区分玩具 Agent 和工业级 Agent 的关键。**
> **预计精读时间**：3-4 天

### 必读文件

#### 4.1 `src/services/compact/compact.ts` — 消息压缩（1705 行）

**阅读方法**：

1. 先理解核心思想：当消息历史太长时，用 LLM 生成摘要替代旧消息
2. 找到压缩的触发条件和执行逻辑
3. 理解 `buildPostCompactMessages()` — 压缩后如何构建新的消息列表

**压缩层级**（由轻到重）：
```
Snip        → 截断过长的工具结果
MicroCompact → 轻量级压缩特定消息
ContextCollapse → 折叠旧消息，保留近期完整
AutoCompact  → 完整摘要压缩，用 LLM 生成摘要替代旧消息
```

**要回答的问题**：
- 压缩前后如何保证关键信息不丢失？
- 压缩摘要的 prompt 是什么样的？
- `CompactBoundaryMessage` 的作用是什么？

---

#### 4.2 `src/services/compact/autoCompact.ts` — 自动压缩触发（351 行）

**阅读方法**：

1. 找到触发条件的判断逻辑（token 阈值）
2. 理解 `calculateTokenWarningState()` — 何时警告、何时强制压缩

---

#### 4.3 `src/utils/messages.ts` — 消息处理工具库（5512 行，最大文件之一）

**阅读方法**：

1. 不要一次读完，按需查阅
2. 重点函数：
   - `normalizeMessagesForAPI()` — 发给 API 前的消息规范化
   - `createUserMessage()` / `createAssistantMessage()` — 消息创建
   - `filterUnresolvedToolUses()` — 过滤未配对的工具调用
   - `getMessagesAfterCompactBoundary()` — 获取压缩后的消息

---

#### 4.4 `src/utils/attachments.ts` — 附件系统（3997 行）

**阅读方法**：

1. 理解附件的作用：动态注入 CLAUDE.md、MCP 指令等上下文
2. 重点看 `getAttachmentMessages()` 和 `filterDuplicateMemoryAttachments()`

**要回答的问题**：
- CLAUDE.md 的内容是如何注入到对话中的？
- 如何避免重复注入相同的附件？

---

#### 4.5 `src/utils/messageQueueManager.ts` — 优先级命令队列（547 行）

**阅读方法**：

1. 理解三级优先级：`now` > `next` > `later`
2. 看队列的数据结构和操作

---

## 阶段 5：多 Agent 编排 — 团队协作

> **目标**：理解如何让多个 Agent 协同工作
> **预计精读时间**：3-4 天

### 必读文件

#### 5.1 `src/tools/AgentTool/AgentTool.tsx` — 子 Agent 工具（1397 行）

**阅读方法**：

1. 先看 `prompt.ts` — 理解模型看到的 Agent 工具说明
2. 看 `call()` 方法 — 理解子 Agent 的启动和等待
3. 注意不同类型的子 Agent：同步、异步、进程内

**要回答的问题**：
- 子 Agent 和主 Agent 共享什么状态？隔离什么状态？
- 子 Agent 的工具列表是如何过滤的？

---

#### 5.2 `src/tools/AgentTool/forkSubagent.ts` — 子 Agent fork 机制

**阅读方法**：

1. 理解 fork 的含义：创建一个独立的 Agent 实例
2. 看上下文隔离的实现

---

#### 5.3 `src/tools/AgentTool/agentToolUtils.ts` — 工具过滤

**阅读方法**：

1. 找到 `filterToolsForAgent()` 函数
2. 理解为什么子 Agent 不能使用某些工具（防止递归、防止权限越级）

**关键常量**（在 `src/constants/tools.ts`）：
```
ALL_AGENT_DISALLOWED_TOOLS — 所有子 Agent 禁用的工具（防止无限递归）
ASYNC_AGENT_ALLOWED_TOOLS  — 异步 Agent 允许的工具白名单
```

---

#### 5.4 `src/utils/agentContext.ts` — Agent 上下文隔离

**阅读方法**：

1. 理解 `AsyncLocalStorage` 的使用 — Node.js 中实现上下文隔离的标准方式
2. 看 `SubagentContext` 和 `TeammateAgentContext` 的区别

---

## 阶段 6：安全与权限 — Agent 的刹车

> **目标**：理解如何防止 Agent 做出危险操作
> **预计精读时间**：2 天

### 必读文件

#### 6.1 `src/types/permissions.ts` — 权限类型定义

**权限模式**：
```
default   → 敏感操作需要确认
plan      → 只读模式，只能看不能改
auto      → 自动允许（有规则限制）
bypass    → 跳过所有权限检查（危险）
```

---

#### 6.2 `src/services/tools/toolHooks.ts` — Hook 系统（650 行）

**阅读方法**：

1. 三个核心函数：
   - `runPreToolUseHooks()` — 工具执行前，可以 allow/deny/修改输入
   - `runPostToolUseHooks()` — 工具成功执行后
   - `runPostToolUseFailureHooks()` — 工具执行失败后

2. 看 `resolveHookPermissionDecision()` — 如何合并 Hook 决策和规则系统

**权限决策的三层防线**：
```
第一层：PreToolUse Hook（自动化检查，可由外部程序控制）
第二层：规则系统（alwaysAllow / alwaysDeny / alwaysAsk 规则）
第三层：用户确认（弹出对话框让用户选择）
```

---

#### 6.3 `src/hooks/useCanUseTool.ts` — 用户权限确认

**阅读方法**：

1. 理解弹出权限确认对话框的逻辑
2. 看用户选择后如何影响后续执行

---

## 自己动手：从零实现一个 Agent 的推荐顺序

按照以下顺序，每一步都是下一步的基础：

### Step 1：最小 ReAct 循环（1-2 天）

实现一个最简单的 Agent：
```typescript
// 伪代码
while (true) {
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    system: "你是一个助手，可以使用工具",
    messages: messageHistory,
    tools: [/* 工具定义 */],
  })

  messageHistory.push({ role: "assistant", content: response.content })

  if (response.stop_reason === "end_turn") break

  if (response.stop_reason === "tool_use") {
    for (const block of response.content) {
      if (block.type === "tool_use") {
        const result = await executeTool(block.name, block.input)
        messageHistory.push({
          role: "user",
          content: [{ type: "tool_result", tool_use_id: block.id, content: result }]
        })
      }
    }
  }
}
```

**对应源码**：`query.ts` 的主循环部分

---

### Step 2：工具系统（2-3 天）

定义 Tool 接口 + 实现 3 个基础工具：

```typescript
interface Tool<Input, Output> {
  name: string
  inputSchema: ZodSchema       // 用 Zod 定义参数
  call(input: Input): Promise<Output>
  prompt(): string             // 给模型的说明
}
```

实现：
- `ReadFileTool` — 读文件
- `WriteFileTool` — 写文件
- `BashTool` — 执行命令

**对应源码**：`Tool.ts` + `tools/Builtin/` 下的工具

---

### Step 3：权限系统（1-2 天）

在工具执行前加入权限检查：

```typescript
async function executeTool(name, input, context) {
  // 新增：权限检查
  const permission = await checkPermission(name, input, context)
  if (permission === "deny") return "Permission denied"

  const tool = findTool(name)
  return await tool.call(input)
}
```

**对应源码**：`toolExecution.ts` 的权限检查部分

---

### Step 4：上下文管理（2-3 天）

实现简单的消息压缩：
- 设置 token 上限
- 超限时，用 LLM 生成历史摘要
- 用摘要替代旧消息

**对应源码**：`compact.ts`

---

### Step 5：优化 Prompt（1-2 天）

- 动态注入环境信息
- 精心编写工具使用说明
- 加入约束和示例

**对应源码**：`prompts.ts` + 各工具的 `prompt.ts`

---

### Step 6：子 Agent（2-3 天）

- 实现 Agent 工具（Agent 内嵌 Agent）
- 工具过滤（子 Agent 不能用所有工具）
- 上下文隔离

**对应源码**：`AgentTool/` 目录

---

### Step 7：生产级增强（持续）

- 流式工具执行
- 并发调度
- Hook 系统
- 遥测和监控
- 会话恢复

---

## 附录：完整文件索引

### 核心循环

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/query.ts` | 1729 | 核心 Agent Loop，while(true) 循环 |
| `src/QueryEngine.ts` | 1295 | SDK/无头模式入口 |
| `src/types/message.ts` | - | 消息类型定义 |

### 工具系统

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/Tool.ts` | 792 | Tool 接口定义，buildTool() |
| `src/tools.ts` | 389 | 工具注册表 getAllBaseTools() |
| `src/services/tools/toolExecution.ts` | 1745 | 单个工具执行链路 |
| `src/services/tools/toolOrchestration.ts` | 188 | 工具并发编排 |
| `src/services/tools/StreamingToolExecutor.ts` | - | 流式工具执行 |
| `src/services/tools/toolHooks.ts` | 650 | Pre/Post Hook 系统 |
| `src/constants/tools.ts` | - | 工具限制和代理工具过滤 |

### 具体工具实现

| 目录 | 核心文件 | 说明 |
|------|----------|------|
| `src/tools/BashTool/` | BashTool.tsx (1143行) | Shell 命令执行 |
| `src/tools/FileReadTool/` | FileReadTool.ts (1183行) | 文件读取 |
| `src/tools/FileEditTool/` | FileEditTool.ts | 文件编辑 |
| `src/tools/FileWriteTool/` | FileWriteTool.ts | 文件写入 |
| `src/tools/GlobTool/` | GlobTool.ts | 文件模式匹配 |
| `src/tools/GrepTool/` | GrepTool.ts | 内容搜索 |
| `src/tools/AgentTool/` | AgentTool.tsx (1397行) | 子 Agent |
| `src/tools/LSPTool/` | LSPTool.ts | 语言服务器协议 |
| `src/tools/WebSearchTool/` | WebSearchTool.ts | 网络搜索 |
| `src/tools/WebFetchTool/` | WebFetchTool.ts | 网页抓取 |

### Prompt 系统

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/constants/prompts.ts` | 914 | 系统提示词主构建器 |
| `src/utils/systemPrompt.ts` | - | 提示词优先级选择 |
| `src/utils/queryContext.ts` | - | 上下文构建 |

### 上下文管理

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/services/compact/compact.ts` | 1705 | 消息压缩 |
| `src/services/compact/autoCompact.ts` | 351 | 自动压缩触发 |
| `src/utils/messages.ts` | 5512 | 消息处理工具库 |
| `src/utils/attachments.ts` | 3997 | 附件系统 |
| `src/utils/messageQueueManager.ts` | 547 | 优先级命令队列 |
| `src/utils/conversationRecovery.ts` | - | 会话恢复 |

### 多 Agent 编排

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/tools/AgentTool/AgentTool.tsx` | 1397 | 子 Agent 工具 |
| `src/tools/AgentTool/forkSubagent.ts` | - | 子 Agent fork |
| `src/tools/AgentTool/agentToolUtils.ts` | - | 工具过滤 |
| `src/utils/agentContext.ts` | - | Agent 上下文隔离 |

### 安全与权限

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/types/permissions.ts` | - | 权限类型定义 |
| `src/services/tools/toolHooks.ts` | 650 | Hook 系统 |
| `src/hooks/useCanUseTool.ts` | - | 用户权限确认 |
| `src/utils/permissions/` | - | 权限规则系统 |

### UI 层

| 文件 | 说明 |
|------|------|
| `src/screens/REPL.tsx` | 主交互界面（896KB，React/Ink） |
| `src/ink/` | 自定义 Ink 终端渲染框架 |
| `src/components/` | 144 个 React 终端 UI 组件 |
| `src/hooks/` | 85 个 React Hooks |
