---
title: Abort、Subagent 与 Tasks
type: guide
status: published
updated: 2026-03-22
---

# Abort、Subagent 与 Tasks

## Abort

`await root.abort(context)` 停止一个 conversation：

- 撤回排队中的 inputs（排队中的 writes 保留）
- abort 当前工作的每个 task
- 调用在 conversation 变为 idle 后才 resolve

## Subagents

conversation 可被某个 task **拥有**。subagent tool 在 `api.commit()` 内用 `ownership: { kind: "task", taskId: api.taskId }` 创建 child，再经 `api.conversation(id)` 驱动：

```typescript
const Subagent: Extension = defineExtension({
	name: "subagent",
	tools: [
		defineTool({
			name: "subagent",
			description: "Delegate a self-contained task to a subagent and get its answer back.",
			parameters: Type.Object({ task: Type.String() }),
			replay: "safe", // 崩溃后重跑时找到同一 child 与 submission
			execute: async (args, api, context) => {
				const child = await api.commit(async (tx) => {
					const existing = (await tx.scanConversations({ ownerTaskId: api.taskId }, 1)).items[0];
					if (existing !== undefined) return existing.id;
					const created = await tx.createConversation({ ownership: { kind: "task", taskId: api.taskId } });
					await configure(tx, created.id, { model: haiku, extensions: { remove: [Subagent] } });
					return created.id;
				}, context);
				await api.details({ conversationId: child }, context); // 让 UI 附着到 child
				const request = { type: "input", content: args.task, requestId: `subagent:${api.taskId}` } as const;
				const settled = await (await (await api.conversation(child, context))!.submit(request, context)).wait(context);
				return { content: [{ type: "text", text: settled.status }] };
			},
		}),
	],
});
```

所有权规则：

- abort 该 call 会 abort child；call 失败同样（`execute()` 抛错，或非 replay-safe 的中断）
- parent 只有在 child 结束后才算 idle
- 以 `{ background: true }` 创建的 task 是边界：它拥有的工作不受 parent abort 影响，也不让 parent 保持 busy。`root.abort(context, { background: true })` 才会一并 abort

产品级示例：

- `22-subagent-foreground.ts`：上面这种 tool，返回 child 答案；UI 经 call 的 `details` 找 child，并把 child events 缩进打在 call 下
- `23-subagent-background.ts`：持久 subagents；一个 `subagent` tool 负责 spawn / message（steer 或 follow-up）/ wait / stop / list。每个 child 由 background anchor task 拥有；答案由 background reporter task 作为 follow-up 投回 parent。request IDs 保证重启不重复发消息或报告

## Child Tasks

task 可拥有 child tasks（`ownership: { kind: "task", taskId }`），并通过提交 `waiting` 状态等待它们：

```typescript
pay: async (task, runtime, context) => {
	await runtime.commit(async (tx) => {
		const payments = [];
		for (const card of task.input.cards) {
			payments.push(await tx.createTask(Payment, { card }, { ownership: { kind: "task", taskId: task.id } }));
		}
		return { status: "waiting", checkpoint: { phase: "decide", payments }, on: payments, policy: "failFast" };
	}, context);
},
decide: async (task, runtime, context) => {
	const outcomes = await runtime.outcomes(task.state.checkpoint.payments, context);
	// ...提交 checkout 自己的 outcome
},
```

| 概念 | 行为 |
|---|---|
| **Waiting** | 等待期间不跑代码。`allSettled`：`on` 里全部结束后恢复；`failFast`：第一个失败的 child 也会 abort 其余。`on` 也可命名其他 tasks（配 `allSettled`） |
| **Finishing** | 自己结束但还拥有在跑工作 → 状态 `completing`：outcome 已定，但要等拥有的工作结束才终端，`waitForTask()` 才返回。失败/中止的 outcome 会先 abort 那些工作 |
| **Aborting** | 自底向上。abort 一个 task 先 abort 它拥有的工作；自己的 abort handler 等那些工作结束后才开始，以便各自撤销副作用 |

`24-child-tasks.ts`：一个 checkout 拥有并等待四笔 payment（拒卡、取消、运行中重启）。

## Task Graph

`harness.taskGraph(context)` 把 Session 里每个 live task 展成一份 Chord state，适合任务面板或调试。每个节点有：

- owner 边（owner task，或「由 conversation 拥有」）
- status
- 是否 `background` / 是否被标 abort
- 它拥有的 conversations

`harness.watchTaskGraph(context)` 以 watch 形式交付同样值。

```typescript
const graph = await harness.taskGraph(context);
graph.subscribe((value) => {
	for (const node of Object.values(value.tasks)) {
		const status = node.state.status === "waiting" ? `waiting on ${node.state.on.join(", ")}` : node.state.status;
		console.log(`${node.id} ${node.kind} ${status}`, node.owner ?? `conversation ${node.conversationId}`);
	}
});
```

补充：

- task 随创建它的 commit 出现，随令其终端的 commit 离开
- 已提交状态包括：`pending`、`running`、`waiting`（带 `on` 与 `policy`）、`completing`（带已持有 outcome 的 status）
- 重启后，原先 `running` 的显示为 `pending`，直到再次跑起来
- 「pending 是否因缺定义而卡住」不在 graph 里；看 `harness.inspect()`
- graph 只列 live tasks：subagent 的 owner task 终端后，其 conversation 里后续 task 会成为顶层节点；conversation 的 `ConversationRecord.owner`（也在 view 的 `conversation` 里）把它连回 parent
