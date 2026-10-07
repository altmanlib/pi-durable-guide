---
title: 对话操作
type: guide
status: published
updated: 2026-03-22
---

# 对话操作

## Watching a Conversation

UI 需要的一切都是已提交状态。`viewState()` 返回 conversation 的结构性视图（只读 Chord state），每次触及它的 commit 后更新：

```typescript
const view = await root.viewState(context);
view.subscribe((value) => {
	// value.entries: 活跃 transcript
	// value.docs["pi.live"]: 进行中的 generation（流式 partial、retry、deferred）与 tool calls（output、details）
	// value.docs["pi.inbox"] / ["pi.usage"] / ["pi.agent"] / ["pi.provider"]
	render(value);
});
// 之后: view.dispose();
```

`watch()` 交付同样视图，并附带每次 commit 的精确 Chord ops，一次一个回调：

```typescript
const watch = await root.watch(context);
render(watch.value); // 附着时的状态
watch.start(async (value, ops) => {
	await send(ops); // 例如发给远端客户端去 apply
});
// 之后: await watch.stop();
```

- 慢消费者最多积压 100 个未投递帧；之后待发帧会被「整份最新视图」的一帧替换
- 晚加入或重连的客户端从当前视图起步，不重放历史
- partial 答案与 tool output 默认最多每 100 ms 提交一次；崩溃最多丢这个窗口。`settings.progress` 可改间隔；远端 storage 可放慢，例如 `{ partialIntervalMs: 500, outputIntervalMs: 500 }`

## Busy Conversations

conversation 在 run 处理某个 input 时为 busy。向 busy conversation 提交会进入 inbox（视图里的 `docs["pi.inbox"]`）：

```typescript
await root.submit({ type: "input", content: "Also run the tests" }, context); // follow-up（默认）
await root.submit({ type: "input", content: "Use pnpm, not npm", whenBusy: "steer" }, context);
await root.submit({ type: "input", content: "Only if idle", whenBusy: "reject" }, context); // 抛 ConversationBusy
await root.submit({ type: "write", entry: { kind: "app.note", data: "user opened a file" } }, context);
```

| 模式 | 行为 |
|---|---|
| **Steer** | 放在当前 tool round 之后，并入正在进行的工作 |
| **Follow-up** | 等 run 回答后放置，开启下一轮 run |
| **Write** | 追加 entry，不请求模型 |
| `submission.abort` | 撤回排队中的 submission |

settings 里 `steeringMode: "all"` / `followUpMode: "all"` 会在一次边界上放置队列里全部同类项，而不是每 turn 一项。

若 run 失败，排队项留在 inbox，直到下一次 submission 放置它们（最旧优先）。

## Reset and Handoff

`reset()` 开启新上下文。模型不再看到更旧 entries，但它们仍留在 storage：

```typescript
await root.reset(undefined, context); // 从空开始
await root.reset("We were fixing the flaky login test. Continue.", context); // 带 handoff 说明
```

busy 时 reset 像 write 一样排队。若在 tool round 期间被放置，当前 run 结束。工具也可通过 `control: { handoff: "..." }` 请求同样效果。

## Compaction

Compaction 压缩模型可见内容：总结较旧 entries，追加 `pi.compaction` entry（含 summary，并指向它保留的第一条 entry）。更旧 entries 仍在 storage。

```typescript
const id = await root.compact("Keep the failing test names", context); // 手动，可选 instructions
const { outcome } = (await harness.waitForTask(id, context)).state;
if (outcome.status === "completed" && outcome.result.submissionId !== undefined) {
	const placed = await (await harness.submission(outcome.result.submissionId, context))!.wait(context);
	console.log(placed.status); // "done"，或带 reason "stale" 的 "unanswered"
}
```

- 做 summary 时 conversation 可继续工作
- idle 时 summary 立即放置；否则在下一个 turn 边界放置
- Esc（`abort()`）可取消手动 compaction
- generation 也会按 settings 自动 compaction

```typescript
settings: {
	compaction: {
		enabled: true, // 自动；手动 compact() 始终可用
		reserveTokens: 16384, // 超过 contextWindow - reserveTokens 时，下次请求等待 compaction
		keepRecentTokens: 20000, // 大致保留多少近期上下文原文
		backgroundTokens: 32768, // 再低这么多就开始后台 compaction；0 关闭
	},
}
```

其他行为：

- provider 因上下文过长拒请求时，generation 会 compact 并重试一次
- 若 summary 切点会切到当前上下文起点之前，放置时结算为 `stale`；多份并行时保留切得最远的那份
- 总结花费计入 `pi.usage`
- `CompactionTask` 上的 `beforeCompact` hook 可拒绝或自带 summary
- 进行中的 compaction 列在 `docs["pi.live"].compactions`（reason、attempt、retry backoff）
- agent events 另有 `compaction_start` / `compaction_end`，snapshot 里有 `comped` 字段

## Agent Events（Experimental）

若消费者想要 coding-agent 风格事件（`message_start`、`message_update`、`tool_execution_start` 等），而不是结构性状态：

```typescript
import { watchEvents } from "@earendil-works/pi-durable";

const stream = await watchEvents(harness, root.id, context);
initialize(stream.snapshot); // entries、run、进行中的 generation/tools/compactions、inbox、agent、usage
stream.start(async (events) => {
	for (const event of events) console.log(JSON.stringify(event));
});
```

- 事件从 commits 派生，每个 commit 一批，叠在 snapshot 上
- message / tool 更新带 deltas：文本与 thinking 追加、tool-call 参数文本追加、output 裁剪与追加
- 落后超过 100 批时，收到新的 `snapshot` 事件
- 完整一轮流见 `test/examples/19-json.ts`

## Hooks

hooks 让 extensions 观察或调整内置 tasks（仅对选中该 extension 的 conversations）：

```typescript
import { GenerationTask, hook, ToolTask } from "@earendil-works/pi-durable";

const Guard = defineExtension({
	name: "guard",
	hooks: [
		hook(ToolTask, { beforeTool: (call) => (call.name === "bash" ? { block: "bash is disabled here" } : undefined) }),
		hook(GenerationTask, { onYield: (answer) => (needsMoreWork(answer) ? { continue: "Keep going." } : undefined) }),
	],
});
```

| 作用对象 | hooks |
|---|---|
| **Generation** | `beforeRequest`（替换一次请求的 messages）、`afterResponse`、`onYield`（用另一条 user message 继续 run）、`afterTools`（一轮 tools 结束后） |
| **Tools** | `beforeTool`（拦截或改写参数）、`afterTool`（替换结果） |

要把 hook 限制到部分 conversations：只在那里选中其 extension，例如 `configure({ extensions: { add: [Guard] } })`。

## More Conversations and Forks

```typescript
const other = await harness.createConversation({ ownership: { kind: "ownerless" } }, context);
const fork = await root.fork(entryId, { ownership: { kind: "ownerless" } }, context);
```

- fork 看到父 conversation 到 `entryId` 为止的 entries，之后独立继续
- 保留该 entry 时刻父方的 agent，但拿到新的 provider session 身份
- 两者都接受 `agent` 与 `init`，在创建 commit 里生效
