---
title: 不变量与核心记录
type: design
status: published
updated: 2026-03-22
---

# 不变量与核心记录

对应 `docs/spec.md` §1–2。这是整个库的「物理定律」。

## 术语（再压一遍）

| 词 | 含义 |
|---|---|
| Session | 拥有一条 mutation line，以及 conversations / entries / tasks / submissions / documents |
| conversation | transcript 作用域；可 fork |
| entry | 不可变 transcript 记录 |
| task | 挂在某个 conversation 上的 durable 状态机 |
| document | 可变 JSON；用 Chord ops + 偶发完整 base 表示 |
| definition | 描述一种 document（或 family）的 typed token |
| turn / run | turn = 一次 assistant 回复 + 其 tool calls；run = 从 admitted input 到最终答案 |

## 八条 Required Invariants

1. **一次 Session commit 跨所有 record 与 document 写入原子**
2. **document 更新只在 storage commit 成功后发布**
3. **所有可见进度都是 durable；没有易失发布路径**
4. **外部副作用不在 Session mutation 事务内跑**（模型、进程、网络、人类）
5. **Entries 与 IDs 提交后不可变、不可复用**
6. **Document draft 在事务回调 settle 时全部撤销**：Session 同步 prepare 或 abort 每个打开的变更；写入 draft 的值按值拷贝，且必须是严格 JSON
7. **mutation line 一直持有到 storage settlement 与 committed-state 采纳**。commit/close observer 只在线上同步跑，且只捕获不可变状态；document-state / watch 用户回调稍后、离线跑
8. **不确定的 storage 失败对打开的 Session 是致命的**：不发布任何东西，必须 reopen。preparation / checkpoint 失败发生在 storage admission 之前，正常回滚

深度学习时反复问自己：某个现象违反了哪一条？

## ID 与记录骨架

ID 是带 brand 的 number（编译期）；运行时 / JSON / SQLite 里仍是普通数字。

关键类型：

- `ConversationId` / `EntryId` / `TaskId<R>` / `SubmissionId` / `DocumentId`
- `Seq`：commit 间严格递增，允许有 gap；禁止把实体 ID 当 document commit point 用
- `ROOT_CONVERSATION_ID = 1`

### ConversationOwnership vs parent/owner

```ts
type ConversationOwnership =
  | { kind: "ownerless" }
  | { kind: "task"; taskId: TaskId };
```

持久化后的 `ConversationRecord` 里：

- **`parent`**：历史继承（entries / historical documents）
- **`owner`**：任务归属（subtree abort、idle 遍历）
- 两者分离。创建时总是显式给 `ConversationOwnership`；Session 从 named task 推导持久化 owner 的 conversation，调用方从不手搓 owner 对
- owner 在 owning task 终端后仍保留记录

### Entry

```text
EntryRecord:
  id, conversationId, kind
  model?: Message[]      // 给模型看的消息
  data?: JsonValue       // 应用数据（如 tool diagnostics）
  head?: EntryId         // 改变后续上下文起点；不删旧 entry
  edits?: ContextEdit[]  // omit / replace 某条目标 entry 的模型视图
  byTaskId?: TaskId
```

Pico 写入的 system message 恒为 `content: ""`；真实 prompt 全靠 named sections（见 §7.4）。

### Submission

- **input**：`queued` → `placed` → `done` | `unanswered`
- **write**：`queued` → `done` | `unanswered`
- `requestId` 用于幂等重提

## 活跃 transcript 与模型上下文推导（§2.1）

这是「UI 看到的 entries」和「发给模型的 messages」分叉的地方。

活跃 transcript：从**最新适用的 `head`** 到 tail 的 raw entry 范围。`head` 改后续上下文，不从 storage 删除旧 entry。fork 遍历：child entries，再经每个 `parent.at` 上限接上 parent。

模型上下文推导（压缩版）：

1. 找 cutoff 处或之前最新带 `head` 的可见 entry `H`
2. `from = H.head`，没有 `H` 则从 transcript 起点
3. 扫描 `from`→cutoff 的可见 entries
4. 每个 target 取范围内最新 edit；`omit` 不贡献；`replace` 用其 messages
5. 有 `H` 时：上下文 = `H` + 范围内非 head entries；否则 = 该范围
6. 保留每个位置化 system message 及其 tool/section 变更
7. 把每个 assistant 的 tool results（下一 assistant 之前）按 tool-call 顺序紧挨移到它后面（provider 协议要求）
8. 没有结果的 call 合成 error result（例如 fork 切断、interrupted tail）；丢掉没有前置 call 的 results
9. 无 model 的 entry，以及 stop reason 为 `aborted` / `error` / `deferred` 的 assistant，不进入后续 provider 请求
10. 若只有 user messages 在 system 之前，把 system 挪到最前——否则 provider 不把后续 tool 变更当「初始 prompt/tool set」，会整表重写 tools 并打爆 prompt cache

**Views 携带 raw active entries。** UI reduction 与 model-context reduction 是不同消费者。

## 为什么这很重要

- 崩溃恢复后「模型接着看什么」完全由这段推导决定，不是由内存里的 UI 状态决定
- compaction / reset 通过写带 `head` 的 entry 改变上下文，而不是删历史
- 你若自己 `appendEntry` 乱写 head/edit，很容易让 view 与 model context 分叉——这是 footgun 区的高频来源
