---
title: 核心概念
type: guide
status: published
updated: 2026-03-22
---

# 核心概念

## 术语

| 术语 | 含义 |
|---|---|
| **Harness** | 一个打开的 storage，加上在其上跑 agent 的机制。所有变更走同一条原子 commit 线；未提交的东西不会展示 |
| **Conversation** | 一份 transcript。`root()` 首次创建根 conversation；还可以再建更多、fork。`Conversation` handle 本身不持有状态；用 `id` 比较 |
| **Entry** | 一条不可变 transcript 记录。例如用户消息 `pi.user`、模型回复 `pi.assistant`、工具结果 `pi.tool-result`、系统提示变更 `pi.system`、reset `pi.reset`，或你自定义的 kind。模型只看「最近一次 reset 之后」的 entries |
| **Commit** | 一次原子写。`conversation.commit((tx) => ...)` 可以同时追加 entries、改 documents、创建 tasks；要么全成功，要么全不落盘 |
| **Document** | 与 transcript 并存的 typed JSON 状态，在 commit 里修改。内置文档包括：`pi.agent`（agent 选择）、`pi.provider`（provider 侧 session 身份）、`pi.live`（进行中的 generation / tools）、`pi.inbox`（排队的 submissions）、`pi.usage`（花费） |
| **Task** | 耐用状态机，每一步都存 checkpoint；进程重启后从最近 checkpoint 继续。每个 task 有 owner：所属 conversation，或另一个 task。Harness 用内置 tasks 回答输入：`pi.generation` 调模型，并拥有其 tool call 对应的 `pi.tool` tasks，等待它们，再把 run 交给下一轮 generation |
| **Submission** | 交给 conversation 的东西：用户输入，或要写入的 entry；可以 wait |
| **Turn / Run** | Turn = 一次模型回复 + 它发出的 tool calls。Run = 从一次 input 到最终答案的多轮 turns。conversation 在 run 进行中为 busy |
| **Extension** | 具名打包：tools、system prompt sections、hooks、wraps、tasks |
| **Registry** | 本进程安装的 extensions。Harness 运行中可变化；新工作用新状态 |
| **Agent** | conversation 实际用什么跑：model、thinking level、选中的 extensions、tools、instructions、working directory。按 conversation 以名字存在 `pi.agent`，每次使用时对照 registry 解析 |

## 一次已回答输入的形状

```text
submit(input) → pi.user
  pi.generation → pi.system（仅当 prompt 或 tools 变了）、pi.assistant（带 tool calls）
    pi.tool × n → pi.tool-result × n   （由 generation 拥有并等待）
  pi.generation → pi.assistant（最终答案）→ submission done
```

## 持久化细节

- SQLite / JSONL 可跨重启保留 conversation
- 崩溃或 close 打断的工作保持 pending；`resume()` 启动 task scheduler。提交或 wait 也会启动它
- 每个 conversation 在 `pi.provider` 里有持久化的 UUIDv7，作为 pi-ai 的 `sessionId`，用于 provider prompt-cache / session affinity
- 该身份跨 reopen、retry、reset、compaction、model 变更存活；child / fork 拿到新身份
- 遗留 conversation 会在首次 generation 或 compaction 请求前生成并持久化一个身份
