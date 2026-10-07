---
title: API Footguns 与 Non-goals
type: design
status: published
updated: 2026-03-22
---

# API Footguns 与 Non-goals

对应 `docs/spec.md` §12–13。这些是**合同**，不是「建议以后加防护」的愿望清单。深度学习时把症状映射回条款。

## Footguns（按主题分组）

### 事务与 Tx

| 条款 | 要点 |
|---|---|
| Detached draft work | 回调 settle 后，draft 读写与所有 `Tx` 操作拒绝。settle 前的 fire-and-forget 仍可能改活跃事务——不支持 |
| Read after write | 第一次 table write 前读完所需表行。之后 document draft 仍可用，table read 不行 |
| Long transactions | async commit 占 mutation line。禁止在里面 await 模型/工具/进程/网络/人类、嵌套 Session commit、或 Session waiter。只用当前 `Tx` 的方法 |
| Explicit creation | 只有 typed `tx.doc()` 创建缺席 document。snapshot/state/watch 返回 `undefined` |
| Family initialization | 首次获取缺席 family 地址选用 seed；已存在与后续调用忽略 seed |

### Conversation / Run / Inbox

| 条款 | 要点 |
|---|---|
| Writing to a busy conversation | 只有 Harness 往有活跃 run 的 conversation 追加。自定义 task / `Harness.commit` 在 generation 准备请求时乱 append，可能错放 system prompt entries；用 write submission |
| Queued items after a failed run | 失败与 task abort **不动** inbox。queued follow-ups 的 `wait()` 不 settle，直到下次 submission 的 boundary 放置它们，或宿主撤回。手动/后台 compaction 的 summary 也可能充当这种 submission |
| Raw head writes into the past | 直接 `appendEntry` 把 head 指到活跃范围之前：模型上下文变了，但 mounted view 仍只持有已有 entries，直到重建 mount。用 write submission（带 stale 检查） |
| Heads/edits during compaction | summary 反映选 range 时的上下文。summarize 期间的应用层 edit/head 可能被更远的 cut 弄丢或撤销。这类 write 先做完再 compact |
| Summary timestamps | 排队 summary 的 user message 时间是 compaction **完成**时，不是放置时。用 message 时间戳估 usage 会误判；Harness 用 entry 顺序 |

### Ownership / Subagent / Hooks

| 条款 | 要点 |
|---|---|
| Double-counted subagent spend | 拥有子 conversation 的 tool **不要**在 result 里再报子 usage；子 `pi.usage` 已计 |
| Owned work holds its owner | 有 live ordinary owned work 的 task 停在 `completing`。在即将完成的 subagent 上继续审问会延长 hold；不该 hold owner 的工作用 conversation-owned + `background` |
| Work created by hooks | hooks 在 asking task 的 invocation 内；它们创建的 owned work 会 hold 该 task。要延迟 finish 的工作必须在 task 提交 outcome **之前**创建。发布后的 listener 无法延长。run 释放后，其 owned work 不得再写 transcript |
| Compensation in abort handlers | abort handler 不能创建 owned children。在做副作用的那一层补偿，或 inline，或 background conversation-owned + `waitForTask()` |
| Hook memo names | hooks 与 task 共享 memo 命名空间；加前缀 |

### Prompt / Agent selection

| 条款 | 要点 |
|---|---|
| Moving default selections | 挂在默认 selection 上的 conversation，会随 settings/installed extensions 变；每次变更追加 `pi.system`，打 miss prompt cache |
| Unstable prompt text | section 输出无真实内容变化却变（如嵌入时间）→ 追加 system deltas，破坏 cache |
| Guards left out of a selection | 数组 selection 漏掉 permissions 之类的 guard extension = 不跑它。没有 registry 全局 hook |
| Selection edits and copies | `{ add, remove }` 总是编辑 **host 默认** selection。写到已从 owner 拷贝过 selection 的 child 上时，会变成「host 默认 + edit」，不是「owner selection + edit」。应从 resolved agent 算 child 的数组 |
| Copies are one-time | 新 task-owned conversation 创建时拷贝 owner 的 `pi.agent`；之后 owner 变更不传播。fork 保留其 asOf 拷贝 |

### Tools / Env / Reload

| 条款 | 要点 |
|---|---|
| Environment on recovery | 恢复后重跑工具会按 conversation **当前** `cwd` 再构建 env，可能与首次不同 |
| Early resource disposal | uninstall/replace extension 只停新使用；立刻释放资源可能弄死仍在跑的 calls |
| Compaction spend | 每次手动/后台 compaction 每次 attempt 都付一次 summarization；summary 最终 stale 也付钱 |

### Watch / State / Schema

| 条款 | 要点 |
|---|---|
| Trusted immutable revisions | snapshot/state/watch 值可能共享 tracker 容器；禁止 mutate，要可变先拷贝。无运行时 freezing |
| Watch activation | `start()` 前 `watch.value` 是获取时的 immutable revision；先用它初始化。start 后属性在每次 callback 前推进到已投递 revision；旧引用保持稳定 |
| Buffered watches | 慢/未 start 最多 100 exact frames；溢出用整份最新值替换未投递后缀 → 中间状态可能丢。要审计每个 transition：写入 immutable entry/journal 再扫 |
| Watch stop | `stop()` 防未来 callback，不 abort/join 已在跑的 |
| Durable progress cadence | 客户端只见已提交进度；崩溃可丢当前未提交 throttle 窗口 |
| Checkpoint starvation | `checkpointWhen` 永不 true → replay 存储可无界增长 |
| Wrong fork setting | `current`/`initial`/`asOf` 是产品语义 |
| Schema stability | kind 与可见 mount path 是持久/公开协议；migration 不能改名 kind |
| Large terminal results | terminal record 可查询；大结果放 entry/长寿命 doc，outcome 只留 ID；勿引用同 outcome 退休的 task doc |
| Raw transcript | view entries ≠ model context；要 reducer |
| Reserved `pi.` names | 约定留给 built-ins；不强制；撞名会撞 Harness 行为 |

### Lifecycle / Storage

| 条款 | 要点 |
|---|---|
| Non-cooperative code at close | close join 每个 invocation；忽略 signal 的 handler/tool/hook 会卡住 close 与 Storage；取消 close wait 结束不了它 |
| Services outliving the Harness | 先撤回 Chord services、detach clients，再 close Harness |
| Fatal storage errors | 不确定 storage 失败后 Session 中毒；不要 catch 后继续用 |
| JSONL durability | 默认 JSONL 处理普通进程崩溃；无 durable 模式不保证掉电后 acknowledged commit 仍在 |

## Non-goals（初期明确不做）

Pico5 初期没有：

- 整 Session 的 DOM
- 可见但不可持久的发布
- Session-kernel 语义事件 journal / 独立维护的事件状态
- session-scoped rewindable documents
- 自动 checkpoint 启发式
- 自动第三方 view mounting
- CRDT / 离线多写者 merge
- 任意 Chord ops 的 SQL 翻译
- JSONL 全局 compaction 或自动损坏修复
- 已移除 Pico 原型的兼容层
- 单进程内强制终止不合作的 extension 代码

## 怎么用这份清单学习

1. 读完 builtin-tasks 后，把每个 footgun 想成一个失败测试标题
2. 在 `test/examples` 里找最接近的正面示例，再想「若违反会怎样」
3. 实现自定义 task/extension 前，至少重读：Long transactions、Owned work holds owner、Compensation in abort handlers、Busy conversation writes、Fatal storage errors
