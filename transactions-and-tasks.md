---
title: 事务线与 Task 内核
type: design
status: published
updated: 2026-03-22
---

# 事务线与 Task 内核

对应 `docs/spec.md` §4–5。这是「崩溃后为何能续跑」的机械原理。

## Session mutation line（§4）

一次 `session.commit(async tx => { ... })`：

1. 占用 mutation line（含回调执行、preparation、storage settlement、baseline adoption、publication enqueue）
2. **commit observers** 在线上同步跑
3. **document-state / watch 用户回调** 稍后离线跑
4. 模型 / 进程 / 工具 / 网络 / 人类效果在线外跑

表（conversations / entries / tasks / submissions）规则：

- 第一次 table write **之前**可以 table read
- 第一次 table write **之后**再 table read → 抛 `ReadAfterWrite`
- document 访问与 read-your-writes 在 table write 后仍可用
- 创建方法直接返回创建的 ID/record；调用方不要再读回来

Storage ownership：

- commit 参数借到 `Storage.commit()` settle
- settle 后保留的东西必须先 detach
- Memory：递归拷贝 JSON 容器；JSONL/SQLite：经序列化与解码索引 detach
- 每次 storage read 返回 detached JSON；不可变 string 可共享，可变 array/object 不行

Admission：

- close 会封印 mutation admission 与 task reservation
- 已 admitted 的 commit：调用方取消 **不**中断 storage settlement，也 **不**撤销 commit

## Task 状态机（§5.1）

```text
pending ──reserve──► running ──commit──► running | waiting | completing→terminal
                ▲                              │
                └── reopen: running→pending ───┘
```

| status | 含义 |
|---|---|
| `pending` | 可被 scheduler reserve |
| `running` | 有 invocation 在跑 phase |
| `waiting` | 无 invocation；等 `on` 里 tasks 终端后在 `checkpoint` 恢复 |
| `completing` | outcome 已定；等 ordinary owned work 排空后才 `terminal` |
| `terminal` | 最终收据 |

Outcomes：`completed` | `failed` | `aborted` | `orphaned` | `faulted`

Ownership：

```ts
TaskOwnership = { kind: "conversation" } | { kind: "task"; taskId }
```

- 每个 task 创建时必须命名 owner
- `background: true` 仅 conversation-owned；child task 不能 background
- owner 边不可变；`conversationId` 创建后不可变

## Effect sandwich（§5.2）

```text
commit intent phase
perform external effect
commit outcome or next phase
```

重开落在 intent phase = 副作用可能已经发生。handler 必须：安全重试、轮询外部 handle、或记录 interruption。Deferred provider 用 durable phase 存 handle + 下次 poll 时间。

`memo(name, candidate)`：小、first-writer-wins、与读 winner 同一次 commit；随 terminal 消失。大宗进度放 document。

## Phase 步进规则（压缩）

每次 phase 前后，scheduler 在 Session 线上跑 step：

1. 已是 terminal / completing / waiting → 停
2. Session closing → 停；保留 checkpoint 与 abort mark 供 reopen
3. run 模式且有 durable abort mark → 结束并 join；owned work 排空后开 abort invocation
4. 未捕获错误 → 写 terminal `faulted`
5. checkpoint 变了 → 刷新 registry snapshot；handover 或同 invocation 调下一 phase
6. checkpoint 没变 → `faulted`（无 durable 进展）

Abort invocation：abort handler 只跑一次；不提交 terminal outcome → fault。

## Scheduler / abort / ownership（§5.4）

Abort 协议：

```text
commit abortRequested
signal + join 活跃 run invocation
等 ordinary owned work 不再 live
开新的 abort invocation
abort handler 提交 terminal outcome
```

要点：

- 取消某个调用方 `Context` ≠ durable abort（除非 API 写入 abort mark）
- 定义缺失/过旧/migration 失败 → **blocked**（派生状态，非持久 status）；缺定义本身不会 orphan
- 只有 abort（直接 / conversation / cascade）才会把 blocked task **orphan**（`orphaned` ≠ `aborted`：后者表示 abort handler 跑过）
- cascade 对 foreground-owned work 幂等；background 是边界
- `close()` 停所有 invocation（含 background）但 **不写 outcome**；reopen 后续跑
- 正常 phase 边界若定义对象变了且新定义可 reserve → handover：commit 回 `pending`，下次 reservation 用新定义

## Structured concurrency（§5.5）— 必背

**Ordinary owned work**：从 task `T` 向下普通遍历到达的所有 task（`T` 拥有的 tasks、`T` 拥有的 conversations 里的 tasks，递归），**排除** background 及其整棵子树。

两条规则：

1. **Abort 向下流**
2. **Owned work 不能活过 owner 的 finish** → 于是有 `completing` hold

Hold 细节：

- task 写的 terminal commit：其它写入（如 tool result entry）在 hold 时落地；真正的 `terminal` 记录、task-document 退休、waiter 解决推迟到最终 commit
- 持有的非 `completed` outcome = 取消意图，会 abort 下面的 work
- 持有的 `completed` 只是等下面正常结束
- `completing` / `terminal` / abort-marked 的 owner 不能再 `createTask`/`createConversation` 以它为 owner

Waiting：

- `failFast`：`on` 里第一个非 completed 会给其余 live 的 `on` 成员打 abort mark；waiting task 自己不标，等全部 terminal 后 `outcomes()`
- `allSettled`：等全部 terminal
- `on` 不能指向缺失、自己、或 owner chain 上的 task
- 不拥有的 task 只能用 `allSettled`

Abort 顺序：**自底向上**。abort handler 看到的是下方最终 outcomes。abort handler **不能** `waiting`，也不能创建 owned children（自己已 abort-marked）；补偿用 inline 或 background conversation-owned tasks + `waitForTask()`。

## 终端转换原子动作（§5.3）

一次真正的 terminal：

1. 写 terminal task record
2. 追加任何 result entries
3. 退休该 task 作用域的全部 documents
4. resolve 由该 task settle 的 submissions

大结果不要塞进 terminal record；放 entry / 更长寿命 document，outcome 只留 ID。且不要引用同一次 outcome 刚退休的 task-scoped document。

## 学习检查题

1. 为什么 tool 的「intent commit」和「execute」必须拆开想？
2. 父 tool 拥有子 conversation 时，子还在跑，父 outcome 为何先变 `completing`？
3. `ReadAfterWrite` 在什么顺序下触发？document read 会吗？
4. 缺 extension 定义的 pending task，idle wait 会结束吗？abort 它会得到什么 outcome？
