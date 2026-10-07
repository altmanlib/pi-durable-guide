---
title: 通过 Chord 观察 Document
type: design
status: published
updated: 2026-03-22
---

# 通过 Chord 观察 Document

对应 `docs/pico-v5-chord-usage.md`（与 `test/chord-guide.test.ts` 同步）。Chord 是观察与远程复制层；**Pico/Harness 仍是唯一写入者**。

## 角色对照

| 概念 | 含义 |
|---|---|
| Session / Harness | durable 容器；单线序列化 mutation |
| Facet | Chord host 里的功能安装单元；`setup` 同步声明服务与依赖，`onActivate` 在依赖就绪后跑 |
| Context | 显式传递的取消/调用上下文；转发调用方的 context |
| Service | typed token + 实现；`env.use` 获取。远程服务含 replicated state 与「JSON 参数 + 末尾 Context」的 async 方法 |
| DocumentState | 绑定一个 document incarnation 的可 dispose 只读 Chord state |
| ReplicatedState | `value` + `subscribe`；断连或未 hydrate 时 `value` 可为 `undefined`；delivery 有 `kind: hydrate\|update` 与 stream `sequence`（不是 Session Seq） |
| Transaction draft | `await tx.doc` 得到的 COW 对象；只在该 commit 回调内有效 |

## Adapter 边界

`documentState()` 在内部做 Chord adoption：

1. 原子捕获已提交 snapshot
2. 注册之后每个 exact frame
3. 返回**已经 hydrate** 的只读 state

snapshot 已覆盖的 ops **不会**再投递。dispose state = 释放观察，不是删 document。

## 模式 1：Session-wide canvas

- `defineDoc({ scope: "session" })`
- 先 `tx.doc()` **显式创建**，再 `documentState()`
- facet `setup` 里 `env.provide` + `env.own(() => state.dispose())`
- 消费者 `env.use` + `onActivate` 里 subscribe
- fork conversation **仍共享**同一 Session canvas
- reopen：再装 facet；加载 base+deltas，**不**重跑 `initial()`

关机顺序（反序）：

```text
detach clients → dispose facet host（撤回 services、dispose states）→ harness.close()
```

先关 Harness 会让仍连接的 client 停在最后值且永远不更新。

## 模式 2：Conversation-scoped family（diff reviews）

- `defineDocFamily`：逻辑键 `(kind, conversationId, key)`；数字 document ID = incarnation
- `fork: "current"`：fork 时拷贝当时逻辑存在的 reviews（新 ID，同 key），随后父子分叉
- `fork: "initial"`：不拷贝；子方首次 `tx.doc` 用自己的 seed
- `asOf` 需要 `history: "rewindable"`
- 再次访问忽略新 seed；改 patch 要显式 mutate 或换 key
- 多实例用 `env.provideMany` + `spawn`；消费者用 `env.observe`，不是 `env.use`

## 模式 3：Task-scoped output

- `scope: "task"`：寿命绑 task；terminal 时退休；不进 fork
- 用 `runtime.commit` + `tx.doc(JobOutputDoc, runtime.taskId)` 追加进度窗口
- 观察：`watchDoc` → 先读固定的 `watch.value` 初始化 → `start(callback)`
- 回调串行、不重叠；慢/未 start 最多缓冲 100 exact frames，溢出则用整份最新值替换未投递后缀
- producer terminal → 投递 retired（`null`）
- **永久结果**应在 terminal commit 退休前写入 entry 或 session/conversation document

## 模式 4：Built-in conversation view

`ConversationView` ≈：

```text
{ conversation, entries, docs: { "pi.live": ..., "pi.inbox": ..., ... } }
```

- 内置 singleton 按 **稳定 kind 字符串**挂到 `docs`，不是按数字 incarnation ID
- 每个完整 Session commit 一批：entries/heads 与变更的 mounted docs 一起
- **第三方 document 不会自动 mount**；用自己的 Chord service 或 trusted invocation watch

投递管线：

```text
mutate draft → callback 成功 → tracker prepare（immutable candidate + ops）
→ checkpoint 选 base 或 delta → Storage.commit
→ 成功则 adopt + enqueue（仍在 mutation line 上）
→ 释放 line → 投递给 Chord 消费者
```

## Chord 侧 footguns（短）

- 不要跨 commit 保留 draft / 嵌套 proxy / 绑定方法
- async commit 持有 line：里面只 await document/Tx 工作，不 await 模型/网络
- 只有 `tx.doc()` 创建；观察 API 从不创建
- watch 不是审计日志；要审计就写 immutable entry/journal 再扫描
- `stop()` 不 abort/join 已在跑的 callback
- kind / version / fork / 公开路径是协议；改 schema 要 migration，不能只改源码名

## 学习检查题

1. `ReplicatedState` 的 `sequence` 和 Session 的 `Seq` 是一回事吗？
2. 为什么 facet 要在 `setup` 里 provide，却常在 `onActivate` 前异步拿到 `documentState`？
3. task-scoped watch 在 task terminal 后还会继续收到业务更新吗？
4. 晚加入的 remote client 先看到什么？
