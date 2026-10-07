---
title: 内置 Tasks（Generation / Tool / Compaction）
type: design
status: published
updated: 2026-03-22
---

# 内置 Tasks

对应 `docs/spec.md` §8。Harness 的 agent loop 就是这三个 durable task。

## 总表

| kind | 职责 |
|---|---|
| `pi.generation` | 准备 system prompt/tools、请求或轮询模型、retry、分类响应 |
| `pi.tool` | 校验、hook、execute、持久化 output/details、追加 result |
| `pi.compaction` | 选 transcript 范围、总结、放置带 head 的 summary |

进度用 throttled durable document commits。崩溃最多丢未提交 throttle 窗口。Recovery 把已提交 partial 转成 interrupted/aborted transcript entries，清 presentation state，再按 phase 重试或终止。

## 内置 entries（§8.1）

| kind | model | 谁写 |
|---|---|---|
| `pi.user` | `[UserMessage]` | submissions、`onYield` 续跑 |
| `pi.assistant` | `[AssistantMessage]`（任意 stop reason） | generation |
| `pi.system` | `[SystemMessage]` 且 `content: ""` | generation preparation |
| `pi.tool-result` | `[ToolResultMessage]`；`data.diagnostics` | tool tasks；generation 对「请求未提供的 tool」 |
| `pi.reset` | 无，或 handoff 的 `[UserMessage]`；恒 `head: "self"` | `reset()`、generation tools 阶段的 handoff |
| `pi.compaction` | 包着 summary 的 `[UserMessage]`；`head` = 第一条保留 entry | compaction |

失败/aborted 的 assistant 仍写入 transcript；上下文推导会把它们排除出后续请求（§2.1 rule 9）。

## `pi.live`（§8.2）

conversation / `latest` / `initial`。idle 时多为 `{}`。

关键字段：

- `run?: { taskId, inputs[] }` — 恰在 busy 时存在
- `generation?` — attempt、throttled partial `message`、retry backoff、deferred poll
- `tools?` — 当前 tool round 的 slots：`pending|running|done`、output、details、diagnostics、`entry?`
- `compactions?` — 活着的 compaction 状态

checkpoint 策略：每当「什么都不在跑」（无 `generation` 且无 `running` tool slot）写完整 base。delta 链最多跨一轮 generation（含 retry/deferred）或一轮 tools 的重叠执行。

## Generation phases（§8.3）

```text
prepare → request ⇄ retry
            ↓
           poll（deferred）
            ↓
          tools（waiting on pi.tool tasks）
```

Checkpoint 要点：

- **prepare**：解析 agent/settings；无 model → `no_model`。可因 threshold 创建 **blocking** compaction 并 `waiting`；或在转 request 时顺带创建 **background** compaction。成功则追加计划好的 `pi.system`，带着 `cutoff`/model/thinking/streamOptions 进 request
- **request**：先把中断留下的 partial 转成 aborted assistant；再 stream；`beforeRequest` 可只替换本请求 messages。分类前停 partial throttle 并等在飞 partial commit
- 分类：
  - `stop`/`length`：跑 `onYield`；可续跑（新 `pi.user` + successor generation）或 settle inputs `done`
  - `toolUse`：追加 assistant，开 tool round，进入 `tools` waiting
  - context overflow（且可 compact）：建 owned compaction，`waiting` 回 prepare（带 `overflow`）；不计入 retry 次数。第二次 overflow 不重试
  - retryable error：进 `retry` sleep
  - 其它 error / 无 abort mark 的 aborted：inputs `unanswered`（`model_error`），fail
  - `deferred`：进 `poll`
- **abort handler**：cancelDeferred、转换 partial、inputs `unanswered`/`aborted`、清 `run`/`generation`

Token 估计（threshold）：从 head 后最近一条有非零 usage 的 assistant 起算 `calculateContextTokens` + 其后消息与计划中 system 的 `estimateMessageTokens`。

## Tool task（§8.4）

```text
call →（intent commit）→ execute（仅 recovery 直接进入）
```

- **call**：读 assistant 上的 call → 在 phase agent 的 tools 里解析 → `validateToolArguments` → `beforeTool` → 再校验 → **一条 commit** 记 intent（phase=`execute` + args + replay 策略）并 slot=`running` → 同 handler 内执行 → `afterTool` → 提交 result
- **execute**（recovery）：两边策略都是 `safe` 才重跑（不再跑 beforeTool）；否则用 slot 里已提交的 partial/details/diagnostics 写 `interrupted` error result
- result commit：bound content、diagnostics、`pi.tool-result` entry、slot=`done`、完成 `{ entryId, control? }`
- `execute()` 或构建 env 抛错 → `tool_error` result，task `failed`（这是取消意图，会 abort 它拥有的 conversations）
- 返回的 `isError` result 仍算 **completed**（拥有的工作可存活）
- abort handler：从 slot 写 `aborted` error result

## Tool rounds（§8.5）

generation 在追加 tool-calling answer 的 commit 里：

1. 用 `getCurrentTools()` 重放 cutoff 处的 offered tools；未提供的 call 当场写 `tool_unavailable` result，不建 task
2. 其它 call 建 owned `pi.tool`。`toolExecution: "sequential"` 或任一 called tool `executionMode: "sequential"` → 顺序，否则并行
3. generation `waiting`（`allSettled`）在这些 tool tasks 上
4. `pi.live.tools` 填 slots，去掉 `generation`

tools 阶段恢复后：

- 跑 `afterTools`
- 应用结果里的 `addTools` 到存储的 `pi.agent.tools`
- 全员 `terminate` 或任一 `handoff` → 可能写 `pi.reset`，settle run，final boundary
- 否则 `postTools` boundary：可能因 reset 结束；否则把 selected steers 并入 run，创建下一 generation

## Compaction（§8.7）

同一 task，三种启动方式：

| 启动方 | owner | background | generation 等待？ | 放置方式 |
|---|---|---|---|---|
| `Conversation.compact()` | conversation | 否 | 否 | write submission |
| generation 过 background 阈值 | conversation | 是 | 否 | write submission |
| generation 过 blocking 阈值 / overflow | generation | 否 | 是 | 直接 append |

Range selection（纯函数）：

1. cut 候选 = 贡献以 user/assistant 开头的非 marker entries（tool result / system 永不做候选；且不能切开 assistant 与其 tool results）
2. 从新到旧累加 `estimateMessageTokens`；达到 `keepRecentTokens` 时，cut = 该点及之后第一个候选
3. 达不到或 cut 前无模型消息 → 无事可 compact

Phases：`select` → `summarize` ⇄ `retry`

- `beforeCompact` 可 `{ decline }` 或直接 `{ summary }`
- summarize 请求：无 tools；内置 summarization prompt；`cacheRetention: "none"`；usage 计入 `pi.usage`
- 放置的 entry：`kind: "pi.compaction"`，`head: firstKept`，user message 包着 `<summary>...`

并行 compaction 不互相协调；section 6 的 boundary / stale 规则决定最终哪份 summary 生效。

## 一条完整 run 的时序（背这个）

```text
submit input
  → inbox/boundary 放置 → pi.user
  → create pi.generation (conversation-owned)
      prepare: 可能 wait compaction；写 pi.system
      request: stream → 写 throttled pi.live.generation
      toolUse:
        append pi.assistant
        create pi.tool×n (owned by generation)
        wait allSettled
        each tool: intent → effect → pi.tool-result
      postTools boundary → next generation 或 settle done
  → submission done
```

## 学习检查题

1. 为什么 `replay: "unsafe"` 的工具崩溃后不能静默重跑？
2. blocking compaction 与 background compaction 在 ownership 与 placement 上差在哪？
3. tool 返回 `isError: true` 会不会 abort 它拥有的子 conversation？`execute()` throw 呢？
4. `pi.live` 何时倾向于写完整 base？为什么？
