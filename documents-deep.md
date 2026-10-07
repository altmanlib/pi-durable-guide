---
title: Document 深度
type: design
status: published
updated: 2026-03-22
---

# Document 深度

对应 `docs/spec.md` §3。Document 是 Chord 跟踪的可变 JSON；它的 **scope / history / fork** 直接决定所有权与寿命。

## Scope 矩阵

| scope | 寿命 | history / fork | fork 时 |
|---|---|---|---|
| `session` | 属 Session；close/reopen 不退休 | 无（current-only） | 仍是同一份 |
| `conversation` + `history: "latest"` | 属 conversation | `fork: "current" \| "initial"` | 见下 |
| `conversation` + `history: "rewindable"` | 属 conversation | `fork: "asOf" \| "current" \| "initial"` | `asOf` 需要 rewindable |
| `task` | task 变 terminal 时原子退休 | 无 | **永不**被 conversation fork 拷贝 |

规则：

- version 必须是正整数
- typed access 在 token 的 scope/history/fork 与持久化 incarnation 不一致时拒绝；migration **不能**改写这些寿命语义
- `initial()` / `migrate()` 返回 JSON object
- `checkpointWhen()` **只**选择何时写完整 base 以限制 replay，不改变寿命语义

## Definition API

```ts
defineDoc({ kind, version, scope, history?, fork?, initial, migrate?, checkpointWhen? })
defineDocFamily({ family: true, kind, version, scope, ..., initial(seed) })
```

- singleton：逻辑身份 = `(kind, scope)`
- family：逻辑身份 = `(kind, scope, key)`；seed **不是**身份的一部分
- 持久化 `DocumentId` 标识 **incarnation**，永不复用
- 退休再创建同一逻辑地址 → 新 incarnation

## Record 与寿命区间

```text
DocumentRecord:
  id, kind, key?, createdAt: Seq, retiredAt?: Seq
  + scope 联合（session | conversation+history/fork | task）
```

成员关系是半开区间：`createdAt <= at < retiredAt`。未退休则无上界。同一次 commit 里创建又退休 → 空寿命。

## 访问与创建（关键纪律）

**没有**可变的 `session.document()` API。

- **只有** `tx.doc(...)` 会创建缺席的 document
- `snapshot` / `documentState` / `watchDoc` 缺席时返回 `undefined`，**从不写入**
- family 首次获取缺席地址时选用 seed；已存在的实例与之后的调用忽略 seed
- seed 既不是身份也不是更新

## Mutation ownership（摘要）

- draft 只在当前 commit 回调内有效（含嵌套对象）
- 回调 settle 时：要么 prepare 成不可变 candidate + ops，要么 abort
- 赋值进 draft 的容器按值拷贝
- 不要跨 commit 保留 draft / 嵌套 proxy / 绑定的 array method

## Bases 与 checkpoints

存储形态：`base`（完整值）+ 之后的 `delta`（Chord ops）。

- `checkpointWhen(value, ops, info)` 里 `info.deltasSinceBase` = 本变更之前已存的 deltas
- 若永远不返回 true：replay / current-only 存储可在 document 存活期间无界增长（footgun：checkpoint starvation）

## Versions 与 migrations

- 一个 incarnation 可含多个 definition version 写下的 records
- migration 在访问路径上发生（document）；task migration 在 reservation 时发生（另一章）
- **不能**靠 migration 改名 `kind`；换 kind 需要显式 copy + retire

## Fork 语义（产品语义，不是优化）

对 conversation document：

| fork | 子 conversation 起点 |
|---|---|
| `current` | fork commit 时逻辑上存在的实例，拷成新 incarnation（保留 family key） |
| `initial` | 不拷贝；子方首次 `tx.doc()` 用自己的 seed |
| `asOf` | 反映 `parent.at` 那个 commit 点的历史值（需 rewindable） |

改 fork 设置 = 改子 conversation 行为。

## 内置 document（用户面常见）

| kind | 作用 |
|---|---|
| `pi.agent` | 每 conversation 的 agent 选择 |
| `pi.provider` | `{ sessionId }`，给 pi-ai 做 cache / affinity |
| `pi.live` | run / generation partial / tools / compactions |
| `pi.inbox` | 排队的 steer / followUp / write |
| `pi.usage` | models / tools 花费账本 |

它们也走同一套 document 机制；通用机制不为它们开特例。

## 学习检查题

1. task-scoped document 在 fork 后还在吗？
2. 为什么 `watchDoc` 看不到的 document 不会被「观察时顺便创建」？
3. `fork: "current"` 与 `"initial"` 在子 conversation 第一次读 review family 时差在哪？
4. 若 `checkpointWhen` 恒 false，坏处是什么？
