---
title: 自定义状态与 Storage
type: guide
status: published
updated: 2026-03-22
---

# 自定义状态与 Storage

## Your Own State

Documents 是与 entries 一起提交的 typed JSON 对象。定义后在 commit 里编辑：

```typescript
import { defineDoc } from "@earendil-works/pi-durable";

const Todos = defineDoc<{ items: string[] }>({
	kind: "app.todos",
	version: 1,
	scope: "conversation",
	history: "latest", // 或 "rewindable"：可用 snapshotAsOf() 读旧值
	fork: "initial", // fork 起点："initial" | "current" | "asOf"
	initial: () => ({ items: [] }),
});

await root.commit(async (tx) => {
	(await tx.doc(Todos, root.id)).items.push("write docs");
}, context);
console.log(await harness.snapshot(Todos, root.id, context));
```

观察方式：

- `harness.watchDoc()` / `harness.documentState()`：观察单个 document，类似 conversation view
- `HarnessOptions.conversationCreated(tx, conversation)`：在每个创建或 fork conversation 的 commit 里跑（含 tool 里原始 `tx.createConversation()`），用来给每个 conversation 装上你的 documents
- `createConversation()` / `fork()` / `root()` 的 `init` 在同一 commit 写入 per-call 数据
- extension 的 tools / sections / hooks 通过 `api` 或 `input.read` 读自己的 documents；缺省时按默认值处理（见 `11-extension-state.ts`）

## Usage and Cost

每个 conversation 在 `docs["pi.usage"]` 累计 token 与费用：

- 按 `provider/model` 计模型回复
- 按 tool 名计上报了 usage 的 tool results
- 失败与中止的尝试也计入

整 Session：

```typescript
const usage = await harness.usage(context); // { models: { "openai/gpt-6-sol": Usage }, tools: {...} }
```

## Storage

| 后端 | Import | 说明 |
|---|---|---|
| Memory | 包根的 `MemoryStorage` | 不持久化 |
| SQLite | `@earendil-works/pi-durable/storage/sqlite/node` 的 `openNodeSqliteStorage(file)` | 单库文件。WAL + `synchronous = NORMAL`：进程崩溃后 commit 仍在；掉电/宿主机故障可能丢最新 |
| JSONL | `@earendil-works/pi-durable/storage/jsonl/node` 的 `openNodeJsonlStorage(directory, context)` | 目录内 append-only 文件。`{ fsync: true }` 在每个 commit marker 前 flush |
| Cloudflare Durable Object | `@earendil-works/pi-durable/storage/sqlite/cloudflare` 的 `openDurableObjectSqliteStorage(ctx.storage)` | 一个 SQLite-backed DO 的 SQLite storage；commit 即其 storage 事务 |

约束：

- 同一时间一个进程拥有一个 storage；没有跨进程锁
- 可移植的 SQLite / JSONL 核心（`/storage/sqlite`、`/storage/jsonl`）不依赖 Node API，可在 Bun 或 Cloudflare DO 上跑，只要提供异步 `SqliteDatabase` facade 或 `@earendil-works/pi-durable/env` 的 `FileSystem`

SQLite adapter 要实现基于 Promise 的 `exec` / `run` / `get` / `all` / `transaction` / `close`。`run` / `get` / `all` 接受 SQL 文本 + 位置绑定；可按 SQL 文本缓存 prepared statements。transaction 回调收到 transaction handle；事务内所有工作必须用它，回调结束后 handle 失效。adapter 必须排队无关操作与其他事务直到当前事务结束，因此在回调里调用 `database` 本身永远不会 settle：

```typescript
await database.transaction(async (transaction) => {
	await transaction.exec("CREATE TABLE example (value TEXT)");
	await transaction.run("INSERT INTO example (value) VALUES (?)", "stored atomically");
});
```

自定义后端可跑共享 conformance suite（Vitest / Jest 兼容）：

```typescript
import { registerStorageConformance } from "@earendil-works/pi-durable/testing";
import { describe, expect, it } from "vitest";

registerStorageConformance({ describe, expect, it }, "My Storage", async (use) => {
	const storage = await openMyStorage();
	try {
		await use(storage);
	} finally {
		await closeMyStorage(storage);
	}
});
```

包根会加载 TypeBox（tool task 用 pi-ai 的 `validateToolArguments()` 校验参数）。未打包时峰值 RSS 约 23 MB；tree-shaken bundle 约 4 MB。
