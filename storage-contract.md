---
title: Storage 合同与后端
type: design
status: published
updated: 2026-03-22
---

# Storage 合同与后端

对应 `docs/spec.md` §10–11，并衔接 README 的 Storage 表。深度学习重点是：**Storage 信任什么、强制什么**。

## Storage 信任 Session

Storage **信任** owning Session 提供语义上合法的 records、引用、祖先关系与状态转换。

它强制的是：

- 原子性
- 全局 ID 所有权
- conversation/entry 创建的不可变性
- document record 一致性
- detached values
- Session 序列化 commits

`Seq` 严格递增但允许 gap。`commit()` resolve 之后，经该 Storage 的后续读必须能观察到它。

## 写入类型（`StorageWrite`）

| type | 含义 |
|---|---|
| `conversation` / `entry` / `task` / `submission` | 写对应 record |
| `document.create` | 新 incarnation + base content |
| `document.copy` | 从 source id@point 拷出新 record |
| `document.change` | 对已有 id 写 base 或 delta |
| `document.retire` | 退休 |

Document content：

```text
{ version, kind: "base", value } | { version, kind: "delta", ops }
```

## 查询形态

- 有序扫描：cursor-based pages
- 精确查找：按 ID / address
- conversation / entry / task / submission / document 各有 query 形状（owner、kind、status、order…）
- document 可按 `at: Seq | "current"` 取点

实现自定义后端时，优先跑：

```ts
registerStorageConformance({ describe, expect, it }, "My Storage", async (use) => { ... })
```

## 后端对照

| 后端 | 持久？ | 崩溃语义要点 |
|---|---|---|
| Memory | 否 | 进程内；detach 靠递归拷贝 |
| SQLite (Node) | 是 | WAL + `synchronous=NORMAL`：进程崩溃后 commit 仍在；掉电可能丢最新 |
| JSONL (Node) | 是 | append-only；`{ fsync: true }` 提高耐久；默认模式主打普通进程崩溃 |
| Cloudflare DO SQLite | 是 | commit = DO storage 事务 |

约束：

- 同一时间一个进程拥有一个 storage；无跨进程锁
- 可移植核心（`/storage/sqlite`、`/storage/jsonl`）不绑 Node API

## SQLite adapter 纪律（再强调）

Promise 风格：`exec` / `run` / `get` / `all` / `transaction` / `close`。

- transaction 回调拿到 handle；事务内只能用 handle
- handle 在回调 settle 后失效
- adapter 必须排队其它操作/事务直到当前事务结束
- 在回调里再调外层 `database` 本身应永不 settle

## 与 Session 不变量的交界

把这三条钉死：

1. document 更新只在 `Storage.commit` 成功后发布
2. 不确定的 storage 失败 → Session 中毒，不发布，须 reopen
3. preparation/checkpoint 失败在 admission 前，正常回滚

## 学习检查题

1. Storage 要不要验证「这个 task 的 owner 边语义正确」？
2. 为什么 Memory 与 SQLite 的 detach 策略不同，但对调用方契约相同？
3. JSONL 在何种失败模式下可能丢「已向调用方成功返回」的 commit？
4. conformance suite 主要防止实现漏掉哪类保证？
