# Codex Agent Loop 学习笔记

> **源码基线**：`/root/codex-src`，commit `95273b84fbff40f9df7d71d50f1ca52c6489b25d`（2026-10-14）
> 所有行号均对照这份源码验证过。`codex-source-analysis` 文档写作于更早的 commit，行号可能漂移约 2 倍，**以源码为准**。
> 路径约定：以下未加前缀的行号都在 `codex-rs/` 下。

---

## 0. 一句话结论

**Codex 不是"一个会写代码的 CLI"，而是一个线程化、可持久化、可治理的代理运行时。**

它用**四层结构**把一个用户请求拆成"会话 → 任务 → 回合 → 步"，每一层都有独立的存活时间和独立的退出条件；最内层用**一套明确写在代码里的规则**（9 条布尔或）决定"要不要再来一次模型调用"，而不是把决定权交给模型。

---

## 1. 术语对齐（最容易出错的地方）

| 中文 | 代码里的名字 | 含义 | 数量级 |
|---|---|---|---|
| **线程** | `Thread`（`ThreadManager` 管的） | 一份独立的对话历史 + 一个独立的会话循环 | 0~N |
| **会话** | `Session` / `submission_loop` | 收消息、分发 Op 的前台循环 | 1 个 / 线程 |
| **任务** | `Turn`（`turn_id`）= 一个 `task` | 你说一句话 → 它彻底干完 | 1 个 / 你说一句话 |
| **回合** | `run_turn` | 驱动 N 次模型调用的函数（内部是个 `loop`） | 1 次（插话时可能 2 次） |
| **步** | `step`（`StepContext`） | 一次模型调用 + 它要求的工具执行 | 1~N |
| **模型调用** | sampling request | 一次 HTTP 请求（含若干次重试） | 1~N |

> ⚠️ **关键**：`turn` 指**整个任务**，不是里面的一圈。里面的那一圈叫 **step**。
> 证据：`core/src/session/step_context.rs:25` 的 `StepContext` 里装着 `turn: Arc<TurnContext>` —— **step 在 turn 里面**。

---

## 2. 四层结构图

```
┌───────────────────────────────────────────────────────────────┐
│ 线程层  ThreadManager                                          │
│ threads: HashMap<ThreadId, Arc<CodexThread>>  (thread_manager.rs:421) │
│ ┌───────────────────────────────────────────────────────────┐ │
│ │ 会话层  submission_loop  (session/handlers.rs:440)         │ │
│ │ ┌───────────────────────────────────────────────────────┐ │ │
│ │ │ 任务层  一个 turn = 一个 task (tasks/regular.rs:103)    │ │ │
│ │ │ ┌───────────────────────────────────────────────────┐ │ │ │
│ │ │ │ 回合层  run_turn 的 loop (session/turn.rs:454)     │ │ │ │
│ │ │ │  ┌─────────────────────────────────────────────┐  │ │ │ │
│ │ │ │  │ step 1 : 模型调用 + 工具                     │  │ │ │ │
│ │ │ │  │ step 2 : 模型调用 + 工具                     │  │ │ │ │
│ │ │ │  │ step 3 : 模型调用（无工具）                   │  │ │ │ │
│ │ │ │  └─────────────────────────────────────────────┘  │ │ │ │
│ │ │ └───────────────────────────────────────────────────┘ │ │ │
│ │ └───────────────────────────────────────────────────────┘ │ │
│ └───────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

**时间尺度**：一次 HTTP 请求 ≈ 毫秒级重试 / 秒级采样 / 十秒级回合 / 分钟级任务 / 小时级会话。

---

## 3. 逐层的存活与退出

### 3.4 线程层 `ThreadManager`

**不是循环，是注册表。**

| 退出路径 | 代码 | 说明 |
|---|---|---|
| 进程退出 | `thread_manager.rs:1435` `shutdown_all_threads_bounded(timeout)` | 并发关闭所有 thread |
| 删除某个会话 | `thread_manager.rs:1390` `remove_thread()` | |
| 关闭单个 | `codex_thread.rs:319` `shutdown_and_wait()` | 内部发 `Op::Shutdown` |

**关闭是"先通知、再等待、超时也算结果"**：

```rust
// thread_manager.rs:1456
match timeout(timeout, thread.shutdown_and_wait()).await {
    Ok(Ok(())) => Complete,        // 从 map 移除
    Ok(Err(_)) => SubmitFailed,    // 保留
    Err(_)     => TimedOut,        // 保留
}
```

**thread 结束 ≠ 数据消失**：对话历史是 rollout 文件，可 `--resume` 回来。

---

### 3.3 会话层 `submission_loop`（`session/handlers.rs:440`）

`tokio::select!` 同时盯三件事，**4 条退出路径**：

| # | 退出条件 | 行号 | 性质 |
|---|---|---|---|
| 1 | `Op::Shutdown` | `:716` → `:735` | 唯一正常关门 |
| 2 | `Op::SuspendTurnAndShutdown`（先落盘再关） | `:592` | 优雅退出 |
| 3 | `rx_sub.recv()` 返回 `Err`（发送端全 drop） | `:454` | 异常关闭 |
| 4 | `local_agent_runtime.shutdown` 被取消 | `:449` | 上层下令关停 |

**兜底 teardown**（`:738`）：

```rust
if !shutdown_received {
    shutdown_session_runtime(&sess).await;   // 关 MCP、子进程、存盘
    emit_thread_stop_lifecycle(...).await;
}
debug!("Agent loop exited");
```

> 它由 `session/mod.rs:1024` 处 `tokio::spawn` 启动 —— **和任务并发运行，不阻塞**。

**关键性质：任务跑的时候它一直在 `select!` 里等消息。这是 Esc 能生效的唯一原因。**

---

### 3.2 任务层 `tasks`

**一个任务 = 一个 turn。** 两条路：正常跑完 / 被作废。

**turn 的结束只有 3 条路**（全部在任务层）：

| 路径 | 条件 | 位置 | 结局 |
|---|---|---|---|
| ① **正常结束** | `run_turn` 返回 **且** 队列无插话 | `:119` → `:120` | `Ok(last_message)` |
| ② **出错结束** | `run_turn` 返回 `Err`（`.await?` 冒泡） | `:113` | `Err(...)` |
| ③ **被作废** | `abort_all_tasks` → 取消令牌 → 回合层 `Err(TurnAborted)`（`turn.rs:861`） | 走路径② | 作废 |

```rust
// tasks/regular.rs:103
loop {
    let last_agent_message = run_turn(...)          // :104  内部的 step 循环已因
        .await?;                                    // :113  ★② Err 直接冒泡★
    if ctx.terminal_error...is_some() {             // :116  致命错误，不重跑
        return Ok(last_agent_message);
    }
    if !sess.input_queue.has_pending_input(...).await {   // :119 ★① 正常结束★
        return Ok(last_agent_message);                    // :120
    }
    next_input = Vec::new();                        // :122  还有插话 → 再跑一轮
}
```

> **注意：任务层看不见 step。** 它只问两件事：`run_turn` 回来了吗？我这儿还有话要传吗？
> 「还有没有 step」这个信息只存在于回合层内部，任务层完全不知道。

**关于「怎么判断没有 step 了」**：架构上不存在这个判断。step 不是清单里的一项，而是循环的转数——`needs_follow_up == false` 时循环 `break`，**不再产生**下一个 step，这就是「没有 step 了」。

**任务体收尾**（`tasks/mod.rs:376`）：

```rust
let task_result = task.run(...).await;
sess.flush_rollout().await;                    // 落盘（失败只 warn）
if !token.is_cancelled() {
    sess.on_task_finished(ctx, task_result).await;   // 清 active_turn
}
```

**被作废**（`abort_all_tasks`，`tasks/mod.rs:564`）：

| `TurnAbortReason`（`protocol.rs:4386`） | 触发 |
|---|---|
| `Interrupted` | 你按 Esc |
| `Replaced` | 来了新任务（`spawn_task` 第一行就 abort 旧的） |
| `ReviewEnded` | review 结束 |
| `BudgetLimited` | 预算/额度耗尽 |

作废流程：`take_active_turn()` → 取消令牌置位 → `handle_task_abort()`（发 `TurnAborted`）→ `clear_pending()`（清 5 张 pending 表）。

⚠️ **被取消时不调 `on_task_finished`**（`if !token.is_cancelled()` 挡住），因为收尾已在 abort 路径做过 —— 两条路，只收尾一次。

**清 `active_turn` 的三条件**（`tasks/mod.rs:646`）：tasks 为空 + `Arc::ptr_eq` 确认是同一个 turn_state（防 ABA）。

---

### 3.1 回合层 `run_turn`（`session/turn.rs:166`，`loop` 在 `:454`）

**它是个循环：每次转一圈 = 一个 step。**

**正常退出（最常见）**：

```rust
// :604
let needs_follow_up = model_needs_follow_up || has_pending_input;
// :718
if !needs_follow_up {
    let stop = run_turn_stop_hooks(...).await;
    if stop.should_block { ...; continue; }   // :754 hook 否决"结束"
    if stop.should_stop  { break; }           // :766
    break;                                    // :825 → Ok(last_agent_message)
}
// :827
continue;
```

**13 条退出路径**分四组：

| 组 | 行为 | 位置 | 返回 |
|---|---|---|---|
| A 正常收工 | `break` | `:825`（无后续）、`:766`（hook 说停） | `Ok(Some(msg))` |
| B 静默收工 | `break`/`Ok(None)` | `:476` `:334` `:388` `:330` `:690` `:713` `:776` | `Ok(None)` |
| C 出错但会话不死 | `break` | `:884`（图片非法）、`:901`（其它错误） | `Ok(last_message)` |
| D 被打断 | `return Err` | `:861` `TurnAborted`、`:704` `:733` `:810` | `Err` |

**3 个额外 `continue`（不走 `needs_follow_up`）**：

| 情况 | 行号 |
|---|---|
| 压缩完成，换新上下文 | `:699` |
| Stop hook 要求继续（塞提示） | `:754` |
| 预算触发的压缩 | `:858` |

---

### 3.0 step 内部

一次 step = `run_sampling_request`（`turn.rs:1707`）+ 工具执行。

- **重试循环**（`:1743`）：`ContextWindowExceeded` / `UsageLimitReached` → 立刻 `Err`；其它 → 在 `stream_max_retries` 内重试，之后永久切 HTTPS。
- **事件消费**：`try_run_sampling_request`（`turn.rs:2618`）
- **工具并行**：`drain_in_flight`（`:2565`）用 `FuturesOrdered`（声明 `:2685`）—— **并行执行，按提交顺序回填结果**。

---

## 4. 核心：`needs_follow_up` 的 9 条规则

**它是"默认 false + 逐条置 true"**，全项目没有一处"置 false"：

```rust
// turn.rs:2686
let mut needs_follow_up = false;     // ← 出生就是 false
```

| # | 规则（成立即 true） | 代码位置 |
|---|---|---|
| 1 | 初始值 `false` | `turn.rs:2686` |
| 2 | **派发了工具调用** | `stream_events_utils.rs:356` |
| 3 | 工具参数错/失败，错误被塞回历史 | `stream_events_utils.rs:424` |
| 4 | 流中断但可恢复（拿到 interrupt） | `turn.rs:2747` |
| 5 | 流出错，无法恢复（丢弃流） | `turn.rs:2753` |
| 6 | **mailbox 抢占**（子代理来信等） | `turn.rs:2929` |
| 7 | 请求被 steer 抢占 | `turn.rs:1828` |
| 8 | **`end_turn == Some(false)`** | `turn.rs:3113` |
| 9 | **用户还有未消费的插话** | `turn.rs:604` |

等价的伪代码：

```rust
let mut 继续 = false;
if 派发了工具调用   { 继续 = true; }
if 工具参数错/失败  { 继续 = true; }
if 流中断可恢复     { 继续 = true; }
if 流坏掉了         { 继续 = true; }
if mailbox 抢占     { 继续 = true; }
if steer 抢占       { 继续 = true; }
if end_turn == false { 继续 = true; }
继续 = 继续 || 用户有插话;

if 继续 { continue; }
else { if StopHook 说"不许走" { continue; } break; }
```

### 三条性质

1. **全是 `||`**：没有优先级、没有权重、没有多数投票。任何一条成立就继续。
2. **全是确定性代码**：没有一条会再调一次 LLM 做判断。判据都是结构化事实。
3. **8 条靠"亲眼看到"，1 条靠"别人转述"**（`end_turn`）。

### 常见的因果关系（容易说错）

- **有工具调用 → 一定继续**（充分条件）
- **没有工具调用 → 不一定结束**（还得看 end_turn / 插话 / 抢占 / 流错）
- `needs_follow_up = false` 不是"判定出来的结论"，而是"**没有任何规则举手反对结束**"

**为什么默认 false？** 失效方向的选择：漏判时宁可**提前停**（用户再催一句），也不要**永远停不下来**（烧钱、无法中断）。

---

## 5. `end_turn` 的真相

```
模型      → 只产出文本 + 工具调用。它的输出里没有 end_turn 这个东西。
服务端    → 在 response.completed 事件里附带一个字段：{"response":{"id":"resp1","end_turn":false}}
Codex     → 只负责读，读不到就自己按规则算
```

源码证据：

```rust
// codex-api/src/common.rs:100-107
/// Did the model affirmatively end its turn? Some providers do not set this,
/// so we rely on fallback logic when this is `None`.
end_turn: Option<bool>,                 // ← 可以是"没有"

// codex-api/src/sse/responses.rs:467
end_turn: if interrupted { Some(false) } else { resp.end_turn }
// ↑ 流被中断时，Codex 亲手改写 —— 说明它不把 end_turn 当权威

// core/src/session/turn.rs:3113
if let Some(false) = end_turn { needs_follow_up = true; }
// ↑ 全项目唯一读取 end_turn 的地方。Some(true)/None 都什么也不做
```

**四种情况**：

| 服务端给的 | Codex 的反应 | 结果 |
|---|---|---|
| `Some(true)` | 不管它 | 看有没有工具调用 / 插话 |
| `Some(false)` | 记一笔"没完" | 继续 |
| `None`（没填） | 不管它 | 看有没有工具调用 / 插话 |
| 流被中断 | 强制当 `false` | 继续 |

> **一句话**：`end_turn` 是服务器填的备注（快递单备注栏），不是模型的输出，也不是 Codex 的判断。
> **忘了它，不影响理解 Codex 的循环。**

---

## 6. 插话（steer）

**定义**：模型还在干活时，你又输入了一句话。

**它不新开任务**：同一个 `Op::TurnInput`，带 `mode: StartOrSteer`，机器人现场判断

```rust
// turn_input.rs:329 start_or_steer() → :803 steer_input()
// :816  没有 active_turn → NoActiveTurn → 走 spawn_task（新任务、新 turn_id）
// :830  只有 TaskKind::Regular 可 steer（Review/Compact 会 ActiveTurnNotSteerable）
// :880  输入被 push 进 pending_input 队列
```

**两个后果**：

**① 正在飞的请求会被主动打断**

```rust
// input_queue.rs:293 watch_user_input()
loop {
    if 队列里有用户输入 { interrupt.cancel(); return; }   // 置位顶抢令牌
    ...
}
// turn.rs:2733  stream.next().or_cancel(&preempt)        // 请求被取消
// turn.rs:1825  if preempt.is_cancelled() { return needs_follow_up: true }  // 规则 7
```

**② 你的话被拼进下一次请求**，成为下一个 step 的 prompt 的一部分。

**这就是任务层那个 `loop` 存在的唯一理由**：模型说完了、`needs_follow_up=false` 时，如果队列里还有你的话，任务层接住 → 再跑一次 `run_turn`（同一个 turn_id、同一份历史）。**你的话一次都不能丢。**

**插话 vs Esc**：

| | 插话 | Esc |
|---|---|---|
| 任务继续吗 | 会，同一个任务 | 不会，任务作废 |
| turn_id | 不变 | 作废 |
| 已花 token | 继续用 | 浪费 |
| 路径 | `steer_input` → `pending_input` | `interrupt_task` → `abort_all_tasks(Interrupted)` |

---

## 7. 完整实例（四层状态 + 两个变量）

**任务**：`把 README 里的 npm 改成 pnpm，再跑一下 lint`

图例：`▣` 登记着 ｜ `◉` 等消息 ｜ `▶` 处理中 ｜ `●` 在跑 ｜ `◐` 挂起 ｜ `✔` 完成 ｜ `✕` 结束 ｜ `—` 不存在

| 时刻 | 线程层 | 会话层 | 任务层 | 回合层 | `end_turn` | `needs_follow_up` |
|---|---|---|---|---|---|---|
| T0 敲回车 | ▣ | ▶ 收 `Op::TurnInput`，spawn | ● 诞生 | — | — | — |
| T1 | ▣ | ◉ | ● | step1 发请求 | — | — |
| T2 响应① | ▣ | ◉ | ● | 派发 2 个 `exec_command` | `Some(false)` | **true**（工具 + 模型都说没完） |
| T3 | ▣ | ◉ | ● | step1 收尾 → step2 | — | true 生效 |
| T4 响应② | ▣ | ◉ | ● | `apply_patch` 改 README | `Some(false)` | **true** |
| T5 | ▣ | ◉ | ● | → step3 | — | true |
| T6 响应③ | ▣ | ◉ | ● | 要跑 `pnpm lint`（需审批） | `Some(false)` | **true** |
| T7 等审批 | ▣ | ◉ **照常醒着** | **◐ 挂起** | **◐ 等工具** | — | 已定 |
| T8 点允许 | ▣ | ▶ `Op::ExecApproval` | ● | 执行命令 | — | — |
| T9 lint 失败 | ▣ | ◉ | ● | → step4 | — | true |
| T10 响应④ | ▣ | ◉ | ● | `apply_patch` 修复 | `Some(false)` | **true** |
| T11 响应⑤ | ▣ | ◉ | ● | **不调工具**，说"改好了" | **`Some(true)`** | **false** ★ |
| T12 | ▣ | ◉ | ● 查插话队列 | **✕ break** → `Ok(Some(...))` | 弃 | 弃 |
| T13 | ▣ | ◉ | **✔ 返回**，落盘 → 清 active_turn | ✕ | — | — |
| T14 | ▣ | ◉ **等下一句** | — | — | — | — |
| T15 Ctrl+C | **✕ 移除** | ▶ `Op::Shutdown` → **✕** → teardown | — | — | — | — |

**读数**：

- 线程层整场只动 1 次（T15）
- 会话层整场只处理 3 个 Op（T0 / T8 / T15），**包括 T7 那种任务卡死时它仍醒着**
- 回合层转 5 圈 → `needs_follow_up` 被算 5 次
- `end_turn` 来了 5 个值，只有 T11 那个 `Some(true)` 起了作用（因为只有那一步没有工具调用）

**变体**：

| 变体 | `end_turn` | 其他信号 | 结果 |
|---|---|---|---|
| T11 后你插一句话 | `Some(true)` | 有插话 | 任务不结束，再跑一轮 |
| 服务端不填 | `None` | 无工具无插话 | 正常结束（靠兜底） |
| 服务端乱说完了但还在调工具 | `Some(true)` | 有工具 | **照样继续**，不信它 |
| T7 时按 Esc | 被强制 `false` | 取消令牌 | 走 `Err(TurnAborted)`，`needs_follow_up` 用不上 |

---

## 8. 三条完整的退出链

### ① 正常干完（自下而上，只退到第二层）

```
回合层 needs_follow_up=false → break → Ok(Some("改好了"))
  ↓
任务层 查 has_pending_input → 没有 → return
  ↓
任务体 flush_rollout → on_task_finished → active_turn = None
  ↓
会话层 继续 select!（★它从来没停过★）
  ↓
线程层 无变化
```

### ② 按 Esc（自上而下）

```
你按 Esc
  ↓
会话层 收 Op::Interrupt（唯一醒着、能发起中断的层）
  ↓
任务层 interrupt_task → abort_all_tasks(Interrupted) → 令牌置位 + clear_pending
  ↓
回合层 下一个 await 点 → Err(TurnAborted)（连带内层重试一起结束）
  ↓
任务体 token.is_cancelled() → 跳过 on_task_finished
  ↓
会话层 继续 select!
```

### ③ 退出 Codex（自上而下，全部关闭）

```
线程层 shutdown_all_threads_bounded(timeout)
  ↓ 并发
会话层 Op::Shutdown → break → teardown
  ↓
线程层 收到终止 → 从 HashMap 移除（超时的保留并记 TimedOut）
```

---

## 9. 用户可见事件 → 对应层

| 事件 | 发出层 | 含义 |
|---|---|---|
| `EventMsg::TurnComplete`（`protocol.rs:1527`） | 任务层 | 这一轮正常干完 |
| `EventMsg::TurnAborted`（`protocol.rs:1640`） | 任务层（abort 路径） | 这一轮被作废 |
| `EventMsg::ShutdownComplete`（`protocol.rs:1643`） | 会话层 | 会话关了 |
| `EventMsg::ExecApprovalRequest`（`protocol.rs:1595`） | 工具执行器 | 等你点按钮 |

**审批挂起时的四层状态**（T7 放大）：任务层整单挂起、回合层卡在 `drain_in_flight`、**会话层照常醒着**、线程层无感 —— 所以 Esc 仍然有效。

审批机制：执行器造 `oneshot` → `tx` 存进 `TurnState.pending_approvals[call_id]` → 发 `ExecApprovalRequest` → `await rx`（挂起，不占 CPU）→ 你点允许 → `Op::ExecApproval` → 会话层取出 `tx` → `send(Approved)` → 执行器继续。

---

## 10. 常见误解对照表

| 误解 | 实际 |
|---|---|
| `end_turn` 是模型输出的 | 是**服务端**在 `response.completed` 里附的字段，`Option<bool>`，可为空 |
| `end_turn` 决定这一轮结不结束 | 只在没有其它信号时起作用；有工具调用时它说什么都没用 |
| `needs_follow_up` 是"判定后写 false" | 出生就是 `false`，只有规则能把它置 `true` |
| 没有工具调用 → 一定会结束 | 不一定：`end_turn=false`、插话、抢占、流错都能继续 |
| `turn` = 里面的一圈 | `turn` = 整个任务；一圈叫 **step** |
| step 是预先排好的一串 | 循环现转出来的，转几圈事先不知道 |
| 回合层结束时 turn 就结束了 | 任务层还要查 `has_pending_input` |
| 会话层在任务跑的时候也闲着 | 它一直在 `select!` 里等，所以能收 Esc / 审批 |
| 正常干完会关掉会话 | 只退到任务层，会话继续等下一句 |
| 关软件会丢历史 | 关停是"通知 → 等待 → 超时也算结果"，历史已落盘 |

---

## 11. 验证命令

```bash
cd /root/codex-src/codex-rs

# —— 四层 ——
sed -n '1024,1027p' core/src/session/mod.rs        # submission_loop 被 spawn
sed -n '430,432p'   core/src/tasks/mod.rs          # tokio::spawn(task_future)
sed -n '100,116p'   core/src/tasks/regular.rs      # 任务层的 loop + 结束条件
sed -n '450,460p'   core/src/session/turn.rs       # 回合层的 loop

# —— 会话层退出 ——
sed -n '716,740p'   core/src/session/handlers.rs   # Shutdown → break
sed -n '738,757p'   core/src/session/handlers.rs   # 兜底 teardown

# —— 回合层退出（13 条）——
awk 'NR>=454 && NR<=906 {printf "%d: %s\n", NR, $0}' core/src/session/turn.rs \
  | grep -nE "break;|return |continue;"

# —— needs_follow_up 的全部来源 ——
grep -n "needs_follow_up" core/src/session/turn.rs
sed -n '350,360p' core/src/stream_events_utils.rs  # 工具调用 → true
sed -n '418,426p' core/src/stream_events_utils.rs  # 工具错误 → true

# —— end_turn 的真相 ——
sed -n '95,110p'  codex-api/src/common.rs          # Option + "some providers do not set this"
sed -n '460,472p' codex-api/src/sse/responses.rs   # interrupted 时强制改写
sed -n '3108,3118p' core/src/session/turn.rs       # 唯一读取点

# —— 插话 ——
sed -n '803,830p'  core/src/session/turn_input.rs  # steer_input
sed -n '293,313p'  core/src/session/input_queue.rs # watch_user_input → 抢占
sed -n '1820,1832p' core/src/session/turn.rs       # preempt → needs_follow_up

# —— 计数 ——
rg -c "^\s*Op::" core/src/session/handlers.rs      # → 34
```

**日志 span 名称**：`session_loop` → `turn` → `session_task.run` → `run_turn` → `stream_request` / `handle_responses`

---

## 12. 与 `codex-source-analysis` 文档的 4 处不一致

| 文档写的 | 实际源码 |
|---|---|
| `Op::UserInput` | `Op::TurnInput`（`protocol.rs:706`） |
| `ActiveTurn.tasks: IndexMap` | `ActiveTurn.task: Option<RunningTask>` + `finishing`（`state/turn.rs:30`） |
| 中途压缩在回合层 | `async_compaction::maybe_start()` |
| `run_turn` 在 131 行、`session/mod.rs` 3339 行 | `:166`；`session/mod.rs` 已五千余行 |

---

## 13. 待续

- 工具调用被派发后，结果**具体怎么塞回 prompt**（`drain_in_flight` → `record_conversation_items` 完整链路）
- Stop hook 到底是什么、为什么它能否决模型（第 16 章）
- 审批 → 沙箱 → 继续的完整路径（`SafetyCheck` 在 `core/src/safety.rs:19`）
- 压缩（compaction）触发时机与它对 turn 的影响（`turn.rs:699`、`:858`）

---

*基于对 `/root/codex-src` 的实际阅读整理；所有行号已逐条核对。*
