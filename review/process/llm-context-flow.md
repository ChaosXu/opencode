# OpenCode TUI Session 历史消息与 LLM 上下文完整分析

> 分析时间: 2026-06-15
> 范围: TUI 的一个 session，每次对话时是否把之前的所有提问和回答内容都和当前问题一起组合起来发给 LLM
>
> 路径以 `packages/opencode/src/session/` 和 `packages/opencode/src/session/message-v2.ts` 为根。

---

## 一、短答案

**是的，每个 turn 都把所有历史消息 + 当前 user 消息一起发给 LLM**。但有几个重要的"裁剪"机制在管理这件事 — 不然 context 会无限膨胀：

1. **`filterCompactedEffect` 过滤手工压缩过的旧消息**
2. **自动 compaction 当 token 溢出时**
3. **`prune` 在 runLoop 结束后台清理过期 message 行**
4. **`truncateToolOutput` 截断单个 tool result 防止吃 context**

---

## 二、核心机制

### 2.1 runLoop 每轮都读全量历史

`packages/opencode/src/session/prompt.ts:1145-1147, 1327-1332`：

```ts
// 每轮 loop 开头
let msgs = yield* MessageV2.filterCompactedEffect(sessionID).pipe(
  Effect.provideService(Database.Service, database),
)
// ... 跑多轮（tool / subagent / compaction 处理）...
// 每轮构造 LLM 请求时：
const [skills, env, instructions, modelMsgs] = yield* Effect.all([
  sys.skills(agent),
  sys.environment(model),
  instruction.system().pipe(Effect.orDie),
  MessageV2.toModelMessagesEffect(msgs, model),    // ← 全量历史转 model messages
])
const system = [...env, ...instructions, ...(skills ? [skills] : [])]
const result = yield* handle.process({
  user: lastUser,
  agent, permission, sessionID, parentSessionID,
  system,                                          // system prompt
  messages: [...modelMsgs, ...(isLastStep ? [{role:"assistant", content: MAX_STEPS}] : [])],  // ← 全量历史
  tools, model, ...
})
```

→ **每个 LLM call** 都带：
- `system` (env + instructions + skills)
- `messages` = 全量 session 历史转成的 `ModelMessage[]` + lastAssistant 一些额外 part
- `tools` (tools 可用列表)

### 2.2 `toModelMessagesEffect` 转换逻辑

`packages/opencode/src/session/message-v2.ts:142-...`：遍历每条历史 `WithParts`，按 role 分类转换：

| Role | 转换 |
| ---- | ---- |
| `user` | text / file / compaction / subtask 各类 part → UI message parts |
| `assistant` | text / reasoning / step-start / **tool** part → 完整还原（text + tool-call + reasoning）|
| tool result | 从 `assistant.parts` 里 `state.status === "completed"` 的 tool part 转 tool result message |

→ **所有** user 提问、所有 assistant 文本 / 工具调用、所有 tool 结果都进 model messages。

---

## 三、3 个机制防止 context 无限增长

### 3.1 过滤已压缩的消息 — `filterCompactedEffect`（`message-v2.ts:532-589`）

```ts
export const filterCompactedEffect = Effect.fnUntraced(function* (sessionID) {
  return filterCompacted(yield* stream(sessionID))   // 从 DB stream 全部 message
})

function filterCompacted(msgs) {
  // 找最近的 "compaction" 标记
  // 截掉标记之前的旧消息
  return result
}
```

→ 当发生过**手工压缩**（`/summarize` / `/compact`），旧消息被打"compaction"标记后**永久剔除**。LLM 后续只会看到压缩后的摘要 + 之后的新消息。

### 3.2 自动压缩（`prompt.ts:1202-1221`）

```ts
if (task?.type === "compaction") {
  const result = yield* compaction.process({...})
  if (result === "stop") break
  continue
}

if (
  lastFinished && lastFinished.summary !== true &&
  (yield* compaction.isOverflow({ tokens: lastFinished.tokens, model }))
) {
  yield* compaction.create({ sessionID, agent: lastUser.agent, model: lastUser.model, auto: true })
  continue
}
```

→ 每轮 loop 检查 `isOverflow`（基于 `lastFinished.tokens` + model context limit）。**溢出** → 自动 `compaction.create` → 生成 summary 消息 → 下一轮用 summary 替代旧消息。

### 3.3 prune（`prompt.ts:1399`）

```ts
yield* compaction.prune({ sessionID }).pipe(Effect.ignore, Effect.forkIn(scope))
```

→ runLoop 结束（`break`）后**后台**调 `prune` 清理过期 message 行（已压缩、不再被 reference 的旧消息）

---

## 四、具体发给 LLM 的内容

| 阶段 | 模型 messages 包含 |
| ---- | ------------------- |
| 第 1 轮（用户第一次问） | `system` + 1 个 user message |
| 第 2 轮（用户第二次问） | `system` + 第 1 轮的 user + 第 1 轮的 assistant (text + 工具调用) + 第 1 轮的 tool results + 第 2 轮的 user |
| ... | 全部累积 |
| 超长时 | `system` + compaction 摘要（"What did we do so far?"）+ 摘要之后的新消息 |

---

## 五、几个值得注意的设计点

1. **每轮"全量"而不是"滑动窗口"** — 不是"最近 N 条"，是"所有不被 compaction 跳过的"
2. **tool result 跟着原始 assistant tool-call** — 保留完整的 tool-use ↔ tool-result 配对
3. **`differentModel` 检查**（`message-v2.ts:256`）— 如果历史里某条 assistant 是用别的 model 生成的，转换时**不带** providerMetadata（防止把别的 model 的标记塞给当前 model）
4. **tool result 输出截断**（`message-v2.ts:306` `truncateToolOutput`）— 单个 tool result 在重发时**截断**到 `toolOutputMaxChars`，防止单个工具输出吃掉 context
5. **media attachments 处理**（`message-v2.ts:158-170, 311-316`）— 不支持 media-in-tool-result 的 provider（如某些 OpenAI-compatible）会把图片/PDF 拆出成 user message
6. **compaction 触发"溢出"判定** — `isOverflow` 用 model context limit 比较 tokens；不同 model 阈值不同
7. **provider 切换不会丢消息** — 全部历史保留，只是 metadata 不交叉

---

## 六、关键文件索引

| 关注点 | 文件 |
| ------ | ---- |
| `filterCompactedEffect`（过滤已压缩消息） | `packages/opencode/src/session/message-v2.ts:532-589` |
| `toModelMessagesEffect`（全量转换） | `packages/opencode/src/session/message-v2.ts:142-...` |
| runLoop 调 filterCompacted | `packages/opencode/src/session/prompt.ts:1145-1147` |
| runLoop 调 toModelMessages | `packages/opencode/src/session/prompt.ts:1331` |
| Overflow 检测 | `packages/opencode/src/session/prompt.ts:1217-1221` |
| 自动压缩 | `packages/opencode/src/session/prompt.ts:1202-1212, 1382-1389` |
| runLoop 结束 prune | `packages/opencode/src/session/prompt.ts:1399` |
| Compaction 压缩实现 | `packages/opencode/src/session/compaction.ts` |
| 端到端 prompt flow | `review/process/tui-prompt-flow.md` 阶段 5-8 |
