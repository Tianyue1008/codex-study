# Codex Prompt 组装与 Skill 注入 学习笔记

> **源码基线**：`/root/codex-src`，commit `95273b84fbff40f9df7d71d50f1ca52c6489b25d`（2026-10-14）
> 行号对照这份源码验证过。`codex-source-analysis` 第 07 章写作于更早的 commit（当时是单个 `core-skills` crate），**路径与签名已漂移，机制结论仍成立**，差异见 §9。
> 路径约定：以下未加前缀的行号都在 `codex-rs/` 下。

---

## 0. 一句话结论

**Prompt 不是一段固定的 system prompt，而是每个 turn 按"预算 + 命中"动态编排出来的复合结构；Skill 不是模板，而是一条从发现到依赖接线的运行时能力管线。**

三个必须同时成立的判断：

1. **多来源**：模型默认指令、配置覆盖、会话历史、运行时目录（技能/插件/环境）、本轮按需注入的正文，各有不同优先级与生命周期；
2. **多角色**：策略与能力清单走 `developer`，被选中的执行上下文走 `user`；
3. **多时序**：目录常驻、正文按轮、base instructions 进历史永久化。

---

## 1. 术语对齐

| 中文 | 代码里的名字 | 含义 |
|---|---|---|
| **总纲 / 人格** | base instructions（`instructions_template`） | 模型行为基座，三级优先选取，进历史后不再变 |
| **上下文片段** | `ContextualUserFragment` | 带 `role()` + `markers()` + `content_kind()` 的消息片段 |
| **目录** | `AvailableSkillsInstructions` | 技能清单，`role = "developer"` |
| **正文** | `SkillInstructions` | 命中技能的 `SKILL.md`，`role = "user"`，`<skill>` 包裹 |
| **技能来源** | `SkillSourceKind` | Host / Executor / Cloud / Custom，决定寻址方式 |
| **目录面** | `CatalogSurface` | TurnInput / HostWorldState / ExecutorWorldState / CloudWorldState |
| **显式注入** | explicit invocation | 命中 → 读全文 → 进 `user` 段，**这是唯一改变可见上下文的路径** |
| **隐式调用** | implicit invocation | 模型自己跑了技能脚本/读了 `SKILL.md` → 只记遥测，**不注入** |

> ⚠️ 最容易错的一点：**"技能被用过了"≠"技能正文进了上下文"**。显式和隐式走两条完全不同的路。

---

## 2. 四层装配结构

```
┌──────────────────────────────────────────────────────────────┐
│ L1  base instructions        developer                        │
│     session/mod.rs:830  配置覆盖 > 历史快照 > 模型默认           │
├──────────────────────────────────────────────────────────────┤
│ L2  developer 段（聚合为一条）  session/mod.rs:4470            │
│     ├─ 技能目录（AvailableSkillsInstructions）                  │
│     ├─ 插件推荐 / 环境信息 / 多 agent 角色                       │
│     └─ 模型切换指令（插到最前）                                  │
├──────────────────────────────────────────────────────────────┤
│ L3  user 段（上下文片段）                                       │
│     └─ 环境上下文、时间提醒、以及本轮命中的 <skill> 正文          │
├──────────────────────────────────────────────────────────────┤
│ L4  turn input 注入（本轮临时）  session/turn.rs:1099           │
│     └─ 显式技能正文 + 插件能力 + extension 贡献                  │
└──────────────────────────────────────────────────────────────┘
```

**时间尺度**：L1 会话级（一次定终身）→ L2 线程/回合级（稳定内容不进增量）→ L3 回合级 → L4 单轮。

**注意 L2 的合并语义**：developer 侧所有片段最终压成**一条**消息（`session/mod.rs:4681`）。这不是省 token 的优化，而是"同一 role 的连续片段合并"这条通用规则的结果；模型看到的 developer 段是一整块，不是七八条。

---

## 3. 逐层拆解

### 3.1 L1：base instructions 三级优先（`session/mod.rs:830`）

| 优先级 | 来源 | 何时生效 |
|---|---|---|
| 1 | 显式配置覆盖（`config.base_instructions`） | 用户/企业强制指定 |
| 2 | 会话历史里的快照（`session_meta.base_instructions`） | resume / fork 一个旧会话 |
| 3 | 当前模型的默认指令（`render_model_instructions`） | 全新会话 |

**为什么历史要压过模型默认？** 因为换模型不该悄悄换人设，resume 也不该让人格漂移。代价是：同一份代码、不同历史来源的用户，看到的 Codex 可能不是同一个 Codex。

**运行时真身在哪？** 模型默认指令内联在 `models-manager/models.json` 的 `model_messages.instructions_template` 里；`models-manager/prompt.md` 是兜底基线常量。仓库里 `core/*.md` 那几个快照文件，在当前树**已没有任何 `include_str!` 引用**，纯"可读资产"。

### 3.2 L2：目录与能力清单（developer）

由 extension 的 context contributor 产出，按 slot 分：

- `PromptSlot::DeveloperPolicy` / `DeveloperCapabilities` → 进 developer 聚合；
- `PromptSlot::ContextWindow` → 单独进 context window hints。

技能目录在这里只给 **名字 + 描述 + 位置**，不给正文。

### 3.3 L3 / L4：正文注入（user）

命中后的 `SKILL.md` 全文以 `<skill><name/><path/>正文</skill>` 形式进入 `user` 侧；插件能力、extension 贡献与之并列。

**这一层是唯一承载"这次任务我要你按什么流程做"的地方。** 审计只看 user 段，就能知道某轮真正激活了哪些能力 —— 这是双通道分 role 最实际的收益。

---

## 4. Skill 全链路：八步

### 第 1 步 发现（系统决策，有硬上限）

枚举技能根：仓库级、用户级、系统级、管理员级、插件级，外加执行环境与云端两类远程来源；扫描目录找 `SKILL.md`，按路径去重。

两道硬闸：`MAX_SCAN_DEPTH = 6`、`MAX_SKILLS_DIRS_PER_ROOT = 2000`（`ext/skills/src/loader/mod.rs:24`）。这是给 monorepo 装的性能保险丝。

**这一步只决定"能不能被发现"，不决定"会不会被注入"。**

### 第 2 步 解析（主严旁宽）

- 主文件 frontmatter：缺失 / 无法修复的非法 YAML → 排除；
- 但会先做一次**行级修复**（`skills/src/parser.rs` 的 `repair_frontmatter_scalar_fields`），救回 `description: Build for AWS: ECS` 这类第三方写法；
- 旁路可选元数据（依赖声明、图标）损坏 → 降级忽略，不阻断主文件（`skills/src/metadata.rs`）。

> 所以"前门严格"应更新为 **先修后判**。

### 第 3 步 建目录

通过校验的技能形成内存目录。此时模型还看不见。

### 第 4 步 预算与渲染（本章技术含量最高）

配额优先级（`ext/skills/src/render.rs:126`）：显式配置 token 上限（上限 10k）> 上下文窗口 2% > 兜底 8000 字符。

三级退化（`render.rs:331`）：

| 层级 | 条件 | 结果 |
|---|---|---|
| 一级 | `full_cost <= budget` | 全量描述 |
| 二级 | 最小形态放得下 | **逐字符轮转分配**（`render.rs:414`） |
| 三级 | 最小形态也放不下 | 按序保留能容纳者，其余 `Omitted` + 告警 |

二级的轮转逻辑（注释原文：*round-robin so no skill monopolizes the remaining budget*）：所有技能同时逐字符加描述额度，谁先加满谁退出，剩余额度自动流给长描述。

**结论：描述被压缩 ≠ 技能消失。** 只有三级退化才真的丢技能。

**当前版本还有一个文档没写的新机制**：目录不止一份，而是 Host / Executor / Cloud **三份共享同一笔预算**（`world_state_catalogs.rs`）。默认给云端留大头、本地文件预留 1/4（`FILESYSTEM_BUDGET_DIVISOR = 4`），且云配额在预算压力下**只能收缩不能扩张**（防止远端膨胀挤掉本地技能）。

### 第 5 步 判定命中（规则决策，确定性）

三类证据，全部精确匹配，**无语义、无模型参与**：

1. 结构化输入：UI 点选、带 `skill://` 路径的引用；
2. 文本标记：`$名字`、`[$名字](skill://…)`；
3. 路径特征：`skill://` 前缀或末段为 `SKILL.md`。

两条硬约束（`ext/skills/src/selection.rs`）：

- 结构化命中会 `blocked_plain_names` 屏蔽同名文本命中 —— 防"路径命中 A、`$名字` 命中 B"的双注入；
- 命中即全文，没有"先看摘要"的中间档。

**必须区分两套触发：**

| | 系统触发 | 模型触发 |
|---|---|---|
| 证据 | 用户显式指定 | 任务与描述语义匹配 |
| 实现 | 确定性规则 | 目录文案里的自然语言指令 |
| 后果 | 强制注入正文 | 只被观测为 implicit，不注入 |

### 第 6 步 注入正文（失败降级）

`ext/skills/src/host_prompt.rs:69` `load_skill_prompts`：

- 读失败 → 只产告警并跳过，**不中断本轮**；
- 正文超 `MAX_SKILL_PROMPT_BYTES = 8000` → 截断 + 明确告知模型；
- 名字/路径上限：`MAX_SKILL_NAME_BYTES = 256`、`MAX_SKILL_PATH_BYTES = 1024`；
- 已被 extension 注入过的 host 技能会被 `InjectedHostSkillPrompts` 过滤，避免重复。

### 第 7 步 依赖接线（系统 + 用户共同决策）

`core/src/mcp_skill_dependencies.rs:40`：选中技能声明了 MCP 依赖而本地未装 → 弹 `RequestUserInputQuestion`（`Install` / `Continue anyway`，`:264`）→ 确认后写配置、走 OAuth、刷新凭证。

把"技能的执行环境"纳入主流程治理，而不是丢给用户拼。

### 第 8 步 观测

- explicit：`codex.skill.injected` counter（status / skill / invoke_type / plugin_id / model_slug），`core/src/skills.rs:42`；
- implicit：`detect_implicit_skill_invocation`（`core/src/skills.rs:140`），识别"跑了技能目录里的脚本 / 读了 `SKILL.md`"，**不回注正文**。

---

## 5. 跨轮编排：两个 turn 时历史长什么样

三条规则先记住：

1. **历史累积**：turn 1 写入的项不会被 turn 2 重排，它们只是“排在前面”；
2. **每个请求都是「前置总纲 + 全部历史 + 本轮新增」**，不是只发增量；
3. **turn 边界只决定“新增什么”**：本轮用户消息、本轮命中正文、世界状态变化量。

### 5.1 一个 turn 内的写入顺序

| 顺序 | 内容 | 角色 | 时机 |
|---|---|---|---|
| 1 | 世界状态全量（首轮）或差量 | developer | turn 开始（`session/mod.rs:4470`） |
| 2 | 用户本轮消息 | user | 采样前记录（`session/turn.rs:379`） |
| 3 | 命中技能的 `<skill>` 正文 | user | 用户消息**之后**（`session/turn.rs:409`） |
| 4 | 工具调用与结果 | assistant / user | 每个 step 追加 |

> 注意第 2 与第 3 的先后：正文在用户消息之后，不是之前。正文是“针对这句话的附注”，不是需求的一部分。

### 5.2 两个 turn 的完整历史

Turn 1（用户说“帮我把 ppt 改一下”，未提技能）：

```
dev   总纲
dev   目录 + 插件 + 环境 + 权限      ← 首轮全量
user  帮我把 ppt 改一下
assistant  工具调用
user       工具结果
assistant  改好了
```

Turn 2（用户说 `$ppt-master 再改一版`）：

```
dev   总纲                          ← 每次请求都放，内容一字不差
dev   目录 + 环境 + 权限              ← 历史里已有，不重发
dev   世界状态差量                    ← 只有变化时才有
user  帮我把 ppt 改一下              ← turn 1 原样在前
assistant  工具调用
user       工具结果
assistant  改好了
user       $ppt-master 再改一版        ← 本轮消息
user       <skill> ppt-master 正文 </skill>   ← 本轮注入
```

### 5.3 各内容的重发规则

| 内容 | 首次出现 | 后续请求 | 跨 turn |
|---|---|---|---|
| base instructions | turn 1 请求 #1 | **每次请求都带**（内容与 id 不变） | 保留 |
| 技能目录 | 首轮全量 | 仅变化时追加差量 | 保留 |
| 环境 / 权限 | 首轮全量 | 仅变化时追加差量 | 保留 |
| 用户消息 | 每轮一次 | 不重复发 | 保留 |
| 技能正文 | 命中当轮 | 不重复发，但**已在历史中** | 保留（重新提及会再注入一份） |
| 工具调用与结果 | 当步 | 不重复发，但已在历史中 | 保留 |
| compaction 摘要 | 压缩时 | 替代早期历史 | 成为新基准 |

### 5.4 四个容易想错的地方

**① “会话级”≠“只发一次”。** base instructions 在每次请求里都是第 0 条；“会话级”指内容在会话内不变，不是只在开头发一遍。真正只发一次的是世界状态的全量，之后发差量。

**② 技能正文会永久留在历史里。** 目录文案里“本轮未提及的技能不要继续套用”是**行为约束**，不是上下文事实；物理上正文仍在，只有 compaction 能让它消失。

**③ 重新提及 = 再注入一份。** 命中只看本轮输入，不看历史里是否已有同技能正文；代码只有同轮内的防重复保护，没有跨轮去重。因此历史里会出现同一技能的多个副本。

**④ 差量是相对“历史已记录版本”算的。** 不是和上一轮比，而是和历史里的基准比；压缩后会保留一份最新完整状态作为新基准，所以比较不会错乱。

### 5.5 位置不变，但距离、副本、显著性在变

技能正文一旦写入，就锚定在“注入它的那一轮用户消息之后”，后续内容全部追加在它后面。系统从不搬运已记录的历史项，**唯一例外是上下文压缩**。

真正变化的是三件事：

| 变化 | 机制后果 | 体验后果 |
|---|---|---|
| **距离变远** | 离当前采样点的条目数增加，近因偏置导致遵循度下降 | “改到一半忘了流程” |
| **副本堆积** | 每轮重新提及就多一份全文，占预算 | 长会话上下文膨胀 |
| **压缩** | 早期历史被摘要吸收或替换；base instructions 单独保留（`compact.rs:286`） | 唯一真正“消失/移位”的场景 |

**实践推论**：模型“忘了流程”时，补一句 `$技能名` 比说“按刚才那个流程来”更有效——不是状态被重置，而是新注入的副本离采样点更近、权重更高。

---

## 6. 这只是"渐进式注入"吗？

是，但属于 **"两段强制 + 一段自觉"**：

| 层次 | 内容 | 决策方 |
|---|---|---|
| 第一层 | 目录常驻 developer | 系统强制 |
| 第二层 | 命中后 `SKILL.md` 全文进 user | 系统强制 |
| 第三层 | `references/`、`scripts/`、`assets/` | 模型自觉（靠目录文案约束） |

外加三个渐进维度：

- **预算渐进**：目录内部还可压缩描述、最终丢弃条目；
- **来源渐进**：Host 给短路径 + 根别名表；Executor / Cloud 只给包标识，正文需模型主动调 `skills.read`；
- **时间渐进**：命中只看本轮输入，未重新提及就不再注入；但已注入的正文**仍留在历史里**（见 §5.4②），行为上靠目录文案要求“不跨轮套用”，物理上靠 compaction 清理。

**不算渐进的两处**：隐式调用（只观测）、正文截断（预算失败兜底，不是分片加载）。

**三个真实缺陷：**

1. 第二层是二元的 —— 大技能一旦命中即全价，截断会让模型拿到不完整流程，**比不读更危险**；
2. 第三层是君子协定，上下文压力大时最先崩；
3. 渐进决策分散在"系统强制 / 模型自觉 / 预算退化"三处，**可解释性差**。

---

## 7. 五个关键设计决策

| 决策 | 理由 | 代价 |
|---|---|---|
| 目录走 developer、正文走 user | 区分"可用"与"本次要用"，审计只看 user 段 | 多一条消息 + 合并去重逻辑 |
| 目录常驻 + 正文按需 | 固定成本压到上下文窗口个位数百分比，技能库可上百 | 正文层无分片按需读取 |
| 触发用确定性规则 | 可复现、可测试、几乎零误报 | 用户越自然表达，越依赖模型自觉 |
| 预算分层退化 + 逐字符轮转 | 任何规模下都保证"目录可见"，损失均匀 | 算法复杂，排障需看渲染报告 |
| 依赖自动接线 | 技能从"文本包"升级为"可执行包" | 安全边界从"读文件"扩到"装服务器 + OAuth" |

---

## 8. 失败模式矩阵（排障主表）

| 症状 | 层面 | 根因 |
|---|---|---|
| 技能完全不在目录里 | 发现/解析 | 根未枚举、超扫描上限、frontmatter 非法、被禁用 |
| 目录里有，`$名字` 无反应 | 规则层 | 名字重复、结构化输入已占用该名、路径与真实路径不一致（symlink） |
| 注入成功但模型不按流程走 | 正文/指令层 | `SKILL.md` 质量差、被截断、与其他指令冲突 |
| 描述是半截的 | 预算层 | 二级退化，正常，未丢技能 |
| 少了几个技能 | 预算层 | 三级退化，真的丢技能 |
| 模型"用过"技能但无正文 | 观测层 | implicit，只有遥测 |
| 明明有技能却没用 | 模型侧 | 用户未显式点名，系统不命中，模型也未自觉 |

---

## 9. 与 `codex-source-analysis` 第 07 章的差异核验

| 文档结论 | 核验 |
|---|---|
| 双通道注入（developer 目录 / user 正文） | ✅ 成立 |
| 预算三层退化 + 字符轮转 | ✅ 成立 |
| 扫描深度与目录数硬上限 | ✅ 成立 |
| implicit 仅遥测、不回注 | ✅ 成立 |
| MCP 依赖闭环 | ✅ 成立 |
| `$name` 需"名字唯一 + 不与 connector slug 冲突" | ⚠️ 已过时：禁用改为条目标记，唯一性校验简化，connector 冲突校验移到 app/plugin 侧 |
| frontmatter 严格、旁路宽容 | ⚠️ 需修订：现在会先做行级修复 |
| 仓内 prompt 快照与运行时指令双源 | ✅ 更严重：快照文件已无编译引用 |
| 缺"本轮为什么没选某技能"的解释 | ✅ 仍成立（底层数据已不少，缺面向用户的一句话） |

**文档完全没覆盖的四个维度**：三份目录共享预算、别名寻址、world-state 缓存与去重、shadow selection 实验通道。

---

## 10. 源码锚点

| 关注点 | 位置 |
|---|---|
| base instructions 三级优先 | `core/src/session/mod.rs:830` |
| developer 段聚合 | `core/src/session/mod.rs:4438`、`:4470`、`:4681` |
| 显式技能注入主链路 | `core/src/session/turn.rs:1099` |
| 目录 fragment（developer） | `ext/skills/src/fragments.rs:40` |
| 正文 fragment（user / `<skill>`） | `ext/skills/src/fragments.rs:81` |
| 目录文案（三种 prompt kind） | `ext/skills/src/catalog_prompt.rs` |
| 预算常量与三级退化 | `ext/skills/src/render.rs:19`、`:126`、`:331`、`:414` |
| 三目录共享预算 | `ext/skills/src/world_state_catalogs.rs:40`、`:274` |
| 命中规则 | `ext/skills/src/selection.rs` |
| 正文读取与截断 | `ext/skills/src/host_prompt.rs:69` |
| 扫描硬上限 | `ext/skills/src/loader/mod.rs:24` |
| 写入顺序：用户消息 → 技能正文 | `core/src/session/turn.rs:379`、`:409` |
| 注入项写入历史 | `core/src/session/mod.rs:3613` |
| 压缩保留 base instructions | `core/src/compact.rs:286` |
| frontmatter 解析与修复 | `skills/src/parser.rs` |
| 遥测（explicit / implicit） | `core/src/skills.rs:42`、`:140` |
| MCP 依赖安装 | `core/src/mcp_skill_dependencies.rs:40`、`:264` |
| 运行时指令真身 | `models-manager/models.json` → `model_messages.instructions_template` |

---

## 11. 自测题

1. 为什么"历史里的 base instructions"要优先于"当前模型默认指令"？反过来会出什么问题？
2. 三级退化里哪一级会真的丢技能？丢了之后模型还能感知到什么？
3. 为什么触发用精确匹配而不是语义匹配？这个代价由谁承担？
4. implicit 为什么不回注正文？如果改成回注会遇到什么问题？
5. 三份目录共享预算时，"云配额只能收缩"在防什么？
6. 若要给"为什么这轮没用我的技能"加一句用户可见解释，你会输出哪几个字段？
7. 同一个技能在两轮都被提及，历史里会出现几份正文？代码在哪一层防了重复、哪一层没防？
8. 模型"忘了流程"时，为什么补一句 `$技能名` 比说"按刚才的流程来"更有效？
