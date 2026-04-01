# Claude Code 源码深度分析 — 04 Agent 循环与安全模型

---

## Agent 循环的状态机设计

`queryLoop()` 的核心是一个**显式状态机**，不用递归而用 `while(true)` + `state` 对象：

```typescript
// query.ts:268
let state: State = {
  messages: params.messages,
  toolUseContext: params.toolUseContext,
  autoCompactTracking: undefined,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  maxOutputTokensOverride: undefined,
  pendingToolUseSummary: undefined,
  stopHookActive: undefined,
  turnCount: 1,
  transition: undefined,  // 记录上一次循环的"原因"（用于测试断言）
}

while (true) {
  // 每轮开始解构 state（不直接修改）
  let { toolUseContext } = state
  const { messages, autoCompactTracking, ... } = state
  
  // ... 执行一轮 ...
  
  // 在 continue 点重新赋值整个 state（7 个 continue 站点）
  state = {
    messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
    toolUseContext: toolUseContextWithQueryTracking,
    autoCompactTracking: tracking,
    turnCount: nextTurnCount,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    pendingToolUseSummary: nextPendingToolUseSummary,
    maxOutputTokensOverride: undefined,
    stopHookActive,
    transition: { reason: 'next_turn' },  // 记录原因
  }
}
```

**为什么用状态机而非递归？**
1. 避免深度递归导致的栈溢出（长任务可能运行数百轮）
2. 状态对象可以完整捕获"上次循环发生了什么"（`transition.reason`）
3. 更易于调试：日志可以直接看 `state.transition`

### 循环退出条件（Terminal）

```typescript
// 正常完成
return { reason: 'completed' }

// 各种中断
return { reason: 'aborted_streaming' }      // 用户 Ctrl+C（流式期间）
return { reason: 'aborted_tools' }          // 用户 Ctrl+C（工具执行期间）
return { reason: 'hook_stopped' }           // hook 阻止继续
return { reason: 'stop_hook_prevented' }    // stop hook 阻止
return { reason: 'max_turns', turnCount }   // 达到最大轮数
return { reason: 'blocking_limit' }         // token 硬限制
return { reason: 'prompt_too_long' }        // 无法压缩的 413
return { reason: 'image_error' }            // 图像处理错误
return { reason: 'model_error', error }     // 模型异常
```

---

## 循环继续的 7 个触发点（Continue Sites）

每个 `continue` 都携带 `transition.reason`，便于测试断言：

| continue 原因 | 触发条件 |
|---|---|
| `next_turn` | 正常 tool_use 后续 |
| `max_output_tokens_escalate` | 升级到 64k 输出 token 重试 |
| `max_output_tokens_recovery` | 注入"继续输出"的 meta 消息（最多 3 次）|
| `reactive_compact_retry` | 被动压缩成功后重试 |
| `collapse_drain_retry` | context collapse drain 后重试 |
| `stop_hook_blocking` | stop hook 返回阻断消息，继续对话 |
| `token_budget_continuation` | token budget 触发"继续"nudge |

---

## max_output_tokens 恢复机制（精妙！）

当 LLM 输出截断时，Claude Code 注入一个 meta 消息并继续：

```typescript
// query.ts:1224
const recoveryMessage = createUserMessage({
  content: `Output token limit hit. Resume directly — no apology, no recap ` +
           `of what you were doing. Pick up mid-thought if that is where the ` +
           `cut happened. Break remaining work into smaller pieces.`,
  isMeta: true,  // 不展示给用户，不发给 API 的前端
})
```

**3 次重试上限**（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`），防止死循环。

**升级策略**（先尝试扩容再截断）：
1. 如果用的是默认 8k 输出限制，先尝试升级到 64k 重试
2. 如果 64k 还截断，再用 meta 消息继续
3. 3 次 meta 消息后放弃，surface 错误

---

## SubAgent（子 Agent）架构

```
主 Agent (queryLoop)
    │
    │  tool_use: AgentTool
    ▼
AgentTool.call()
    │
    ├── createSubagentContext()  ← 克隆 toolUseContext（不同 agentId）
    ├── 构建子 Agent 的 messages（含任务描述）
    └── query(subAgentParams)   ← 递归调用同一个 query()！
            │
            ├── 子 Agent 有自己的 Agent Loop
            ├── 使用 agentId 隔离权限和状态
            ├── setAppState 是 no-op（不影响父 Agent 状态）
            └── 完成后返回结果给父 AgentTool
```

**关键隔离点**：
- `agentId?: AgentId` — 子 Agent 有唯一 ID，hooks 可区分
- `setAppState` 在子 Agent 中为 no-op（状态隔离）
- `setAppStateForTasks` 用于需要跨 Agent 注册的基础设施（如 session hooks）
- `localDenialTracking` — 子 Agent 有本地拒绝计数器（不影响父 Agent）

---

## 任务规划：EnterPlanMode

```typescript
// EnterPlanModeTool 触发
toolUseContext.setAppState(state => ({
  ...state,
  toolPermissionContext: {
    ...state.toolPermissionContext,
    mode: 'plan',        // 切换为只读模式
    prePlanMode: state.toolPermissionContext.mode,  // 记录之前的模式
  }
}))
```

**plan 模式的效果**：
1. 所有写操作被拒绝（isReadOnly 检查）
2. 模型可能使用更大的 context window（200k 时切换）
3. 用户看到不同的 UI 提示

**退出计划模式**：`ExitPlanModeTool` 恢复 `prePlanMode`。

---

## Worktree 沙箱机制

```typescript
// EnterWorktreeTool
// 1. 创建临时 git worktree
git worktree add .claude/worktrees/<name> -b <branch>

// 2. 切换工作目录到 worktree
process.chdir(worktreePath)

// 3. CLAUDE.md 等配置继承自原始仓库
// 4. 退出时自动清理（若无修改）或保留（若有修改）
```

这实现了真正的代码隔离：在 worktree 中的所有操作不影响主工作区。

---

## Stop Hooks 系统

Stop Hooks 在模型**决定停止**（无 tool_use）后执行，可以**阻止停止**：

```typescript
// query/stopHooks.ts
const stopHookResult = yield* handleStopHooks(
  messagesForQuery,
  assistantMessages,
  systemPrompt, userContext, systemContext,
  toolUseContext, querySource, stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  // 注入错误消息，继续循环
  state = { 
    messages: [..., ...stopHookResult.blockingErrors],
    stopHookActive: true,
    transition: { reason: 'stop_hook_blocking' }
  }
  continue  // 继续循环！Claude 需要处理 hook 返回的错误
}
```

**Stop Hooks 的典型用例**：
- 代码风格检查：运行 lint，若有错误注入到对话，让 Claude 修复
- 测试验证：运行测试，若失败让 Claude 继续修复
- 安全扫描：停止前验证无漏洞

**死循环防护**：
- `stopHookActive` 标志：hook 触发后设为 true，防止嵌套触发
- 不重置 `hasAttemptedReactiveCompact`：防止 compact→too-long→error→stop-hook→compact 死循环

---

## Hooks 系统完整架构

```
用户配置 (~/.claude/settings.json):
{
  "hooks": {
    "PreToolUse": [{ "matcher": "Bash", "hooks": [{"type": "command", "command": "..."}] }],
    "PostToolUse": [...],
    "Stop": [...],
    "PostSampling": [...]
  }
}

执行点:
  PreToolUse  → toolExecution.ts → executePreToolHooks()
  PostToolUse → toolHooks.ts     → executePostToolHooks()
  Stop        → stopHooks.ts     → handleStopHooks()
  PostSampling→ query.ts:1000    → executePostSamplingHooks()（fire & forget）

Hook 类型:
  command: 执行 shell 命令（可读取/修改 tool input/output）
  ...（可扩展）

Hook 输入（通过环境变量或 stdin JSON 传递）:
  TOOL_NAME, TOOL_INPUT (JSON), TOOL_OUTPUT (JSON)
  
Hook 输出:
  exit 0        → 继续
  exit 2        → 阻断（block）
  stdout JSON   → 修改 tool output / 注入消息
```

---

## 安全分类器（Auto-mode）

在 `bypassPermissions` 模式（`--dangerously-skip-permissions`）下，仍有最后一道防线：

```typescript
// BashTool
toAutoClassifierInput(input) {
  return `${input.command}`  // 返回实际命令
}

// yoloClassifier.ts（auto 模式）
// 用 Claude 本身分析命令是否危险
// 参考 CLAUDE.md 内容和历史操作
```

这是"用 AI 守护 AI"的递归安全设计。

---

## 关键设计决策 #3：权限是"运行时状态"而非"编译期配置"

权限模式可以在会话中**动态切换**：
- 用户可以 `/acceptEdits` 临时提升权限
- 模型可以 `EnterPlanMode` 降低到只读
- SubAgent 可以有不同的权限模式

这与传统工具"启动时配置"的方式截然不同，更贴近真实工作场景的动态性。
