# Codex 源码学习

读 [openai/codex](https://github.com/openai/codex) 的个人源码笔记：**按调用链纵向切**，每篇只回答一个具体问题，行号逐条对照源码核验。

> **基线**：`openai/codex` @ [`95273b8`](https://github.com/openai/codex/tree/95273b84fbff40f9df7d71d50f1ca52c6489b25d)（2026-10-14）
> 工作区约 127 个 crate，主体在 `codex-rs/`。笔记先于/后于基线的阅读都以此为锚，行号漂移时以源码为准。

```bash
git clone https://github.com/openai/codex.git
cd codex && git checkout 95273b84fbff40f9df7d71d50f1ca52c6489b25d
```

## 笔记进度

| # | 笔记 | 回答的问题 | 主要源码 | 状态 |
|---|------|-----------|----------|------|
| 01 | [Agent 循环](docs/01-agent-loop.md) | 一次模型调用是怎么被"再来一次"和"停下来"驱动的 | `core/src/session/turn.rs`、`core/src/session/step_context.rs`、`core/src/thread_manager.rs` | ✅ |
| 02 | 工具系统 | 模型看到哪些工具，schema 从哪来，调用怎么路由到实现 | `tools/`、`core/src/tools/` | ⬜ |
| 03 | 命令执行 `unified_exec` | 一条 shell 命令从 tool call 到进程退出，超时/流式/PTY 怎么处理 | `core/src/exec.rs`、`exec/`、`shell-command/` | ⬜ |
| 04 | `apply_patch` | 为什么不用整文件重写，patch 语法怎么校验与容错 | `apply-patch/` | ⬜ |
| 05 | 沙箱 | macOS Seatbelt / Linux bwrap / Windows WFP 三者如何统一到一个抽象 | `linux-sandbox/`、`bwrap/`、`windows-sandbox-rs/` | ⬜ |
| 06 | 执行策略 | Starlark `execpolicy` 如何把"允许/拒绝/需审批"变成可配置判定 | `execpolicy/` | ⬜ |
| 07 | 审批与网络 | `ask`/`on-failure`/`never` 与出网代理如何共同构成权限边界 | `network-proxy/`、`core/src/safety.rs` | ⬜ |
| 08 | 上下文压缩 | 长会话如何触发 compact、保留什么、丢什么 | `core/src/compact*.rs`、`core/src/context/` | ⬜ |
| 09 | [Prompt 组装与 Skill 注入](docs/09-prompt-assembly.md) | system prompt / skills 如何拼成最终请求，预算与命中如何决定注入 | `core/src/session/mod.rs`、`core/src/session/turn.rs`、`ext/skills/`、`skills/`、`models-manager/` | ✅ |
| 10 | Provider 与流式事件 | 一次 sampling request 的 SSE 事件如何变成内部事件 | `model-provider/`、`codex-api/` | ⬜ |
| 11 | 会话持久化 | rollout / thread-store 如何保证中断后可恢复 | `rollout/`、`thread-store/`、`state/` | ⬜ |
| 12 | App-Server 协议 | 外部应用如何驱动 Codex（JSON-RPC 双向协议） | `app-server/`、`protocol/` | ⬜ |
| 13 | MCP 双向集成 | 消费 MCP server，以及自己作为 MCP server 暴露 | `codex-mcp/`、`rmcp-client/` | ⬜ |
| 14 | Hook 与生命周期 | 哪些时点可插入外部逻辑，失败如何传导 | `hooks/` | ⬜ |
| 15 | TUI 渲染 | 事件流如何变成可交互终端界面 | `tui/` | ⬜ |
| 16 | Code Mode | V8 里跑工具编排，与传统 tool call 的取舍 | `code-mode/`、`code-mode-runtime/` | ⬜ |

## 阅读顺序建议

- **只想搞懂「一轮对话怎么跑起来」**：01 → 02 → 03
- **关心安全边界**：01 → 05 → 06 → 07
- **关心长会话稳定性**：01 → 08 → 11
- **要接自己的应用**：01 → 10 → 12 → 13

## 每篇的写法

约定统一，便于横向对照：

1. **先给结论**：一句话说清这层在解决什么，避免读完仍不知道重点
2. **术语对齐**：中文词 ↔ 代码里的真实命名 ↔ 数量级（最容易出错的地方）
3. **结构与退出条件**：状态机/调用链，重点是"什么时候结束"，不是"有哪些字段"
4. **行号可复现**：正文给出的验证命令能直接在源码仓库里跑出对应结果
5. **常见误解对照表**：写明"看起来是 A，实际是 B"及证据
6. **未解之处**：留 `待续` 而非猜测

## 姊妹研究（他人作品）

按子系统横向拆解的公开 Codex 研究，与本笔记的纵向视角互补：

- [xiaonancs/codex-source-analysis](https://github.com/xiaonancs/codex-source-analysis) — 2 总纲 + 25 章 + 3 附录，覆盖 `codex-rs` / `codex-cli` / SDK

> 该系列写作于更早的 commit，行号常有漂移。与本笔记冲突之处，以源码为准，并在各篇「与既有文档的不一致」一节记录。

## 许可

笔记内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)。所引用源码版权归 OpenAI 所有。
