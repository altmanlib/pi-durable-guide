---
title: 快速开始
type: guide
status: published
updated: 2026-03-22
---

# 快速开始

```typescript
import { BACKGROUND_CONTEXT } from "@earendil-works/chord/context";
import { createModels } from "@earendil-works/pi-ai/models";
import { openaiProvider } from "@earendil-works/pi-ai/providers/openai";
import { AssistantEntry, createRegistry, Harness, MemoryStorage } from "@earendil-works/pi-durable";

const context = BACKGROUND_CONTEXT;

const models = createModels();
models.setProvider(openaiProvider()); // 读取 OPENAI_API_KEY

const harness = await Harness.open(new MemoryStorage(), { models, registry: createRegistry() }, context);
const root = await harness.root(context, { agent: { model: { provider: "openai", modelId: "gpt-6-sol" } } });

const submission = await root.submit({ type: "input", content: "What is the capital of France?" }, context);
const settled = await submission.wait(context);
if (settled.status === "done" && settled.type === "input") {
	const answer = await root.commit((tx) => tx.entry(AssistantEntry, settled.answer), context);
	console.log(answer?.model?.[0]);
}
await harness.close(context);
```

## 这段代码做了什么

1. `Harness.open()` 在某个 storage 上打开 Session。`MemoryStorage` 只放内存
2. `root()` 返回根 conversation；首次使用时按给定 agent 选择创建它。conversation 是不可变 entries 的 transcript
3. `submit()` 持久化接纳你的输入，返回 `Submission`。内置 generation task 调模型并追加答案
4. `wait()` 等到输入被回答（`done`）或失败（`unanswered`，带 reason）

## Context

每个异步调用都带 Chord `Context`，用来携带取消信号：

- `BACKGROUND_CONTEXT`：永不取消
- 取消一次 `wait` 只取消这次等待，不会取消后台工作本身

## 持久化与恢复（最短版）

```typescript
import { openNodeSqliteStorage } from "@earendil-works/pi-durable/storage/sqlite/node";

const harness = await Harness.open(await openNodeSqliteStorage("./session.sqlite"), { models, registry }, context);
const root = await harness.root(context); // 与上次同一个 root
harness.resume(); // 继续上次未完成的 run
```

相同 `requestId` 的重试提交会返回已有 submission，而不会提交两次：

```typescript
const submission = await root.submit({ type: "input", content: "Hello", requestId: "greeting-1" }, context);
const again = await root.submit({ type: "input", content: "Hello", requestId: "greeting-1" }, context);
// again.id === submission.id
```

`harness.submission(id)` 可在重启后按 ID 重新拿到 submission，例如继续 wait。
