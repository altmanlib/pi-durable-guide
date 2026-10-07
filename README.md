---
title: pi-durable 中文导读
type: guide
status: published
updated: 2026-03-22
---

# pi-durable 中文导读

基于仓库 `packages/durable` 的 README 与 `docs/spec.md` 等整理的中文阅读材料。包名：`@earendil-works/pi-durable`。

> **Experimental.** API 可能在版本之间无通知变更。本导读不是仓库内的官方中文文档；字段级合同以英文 `spec.md` 为准。

## 怎么用这套材料

- **只想会用：** 读「用户面」1–9
- **要深度学习：** 从 [study-path.md](study-path.md) 进入，按 A→H 推进，并把「学习检查题」当过关门槛

## 用户面（对齐 README）

| 顺序 | 文档 | 内容 |
|---|---|---|
| 1 | [overview.md](overview.md) | 它是什么、核心规则、安装 |
| 2 | [quick-start.md](quick-start.md) | 最小可运行示例 |
| 3 | [concepts.md](concepts.md) | Harness / Conversation / Entry / Task 等概念 |
| 4 | [extensions-and-tools.md](extensions-and-tools.md) | 扩展、内置工具、自定义工具、系统提示 |
| 5 | [agent-settings-env.md](agent-settings-env.md) | 每会话 Agent、Settings、Environment、Reload |
| 6 | [conversation-ops.md](conversation-ops.md) | 观察视图、忙碌队列、Reset、Compaction、Events、Hooks |
| 7 | [subagents-and-tasks.md](subagents-and-tasks.md) | Abort、Subagent、Child Tasks、Task Graph |
| 8 | [state-and-storage.md](state-and-storage.md) | 自定义 Document、Usage、Storage 后端 |
| 9 | [examples.md](examples.md) | 可运行示例索引与设计文档入口 |

## 深度学习面（对齐 spec / Chord 指南）

| 顺序 | 文档 | 对应原文 |
|---|---|---|
| 0 | [study-path.md](study-path.md) | 学习路径、心智模型、实验节奏 |
| 10 | [invariants-and-records.md](invariants-and-records.md) | `spec.md` §1–2 |
| 11 | [documents-deep.md](documents-deep.md) | `spec.md` §3 |
| 12 | [transactions-and-tasks.md](transactions-and-tasks.md) | `spec.md` §4–5 |
| 13 | [builtin-tasks.md](builtin-tasks.md) | `spec.md` §8 |
| 14 | [chord-usage.md](chord-usage.md) | `pico-v5-chord-usage.md` |
| 15 | [footguns-and-nongoals.md](footguns-and-nongoals.md) | `spec.md` §12–13 |
| 16 | [storage-contract.md](storage-contract.md) | `spec.md` §10–11 |

## 源材料位置

- 用户手册：`packages/durable/README.md`
- 规范：`packages/durable/docs/spec.md`
- 实现计划：`packages/durable/docs/pico-v5-handoff.md`
- Chord 用法：`packages/durable/docs/pico-v5-chord-usage.md`
- 示例：`packages/durable/test/examples/`
- Chord 指南测试：`packages/durable/test/chord-guide.test.ts`
- 上层 demo：`packages/coding-agent/src/experimental/durable/README.md`

## 一句话

一切可见进展都必须先提交到存储；进程中途挂掉后，重新打开 storage 就能从断点继续。副作用不进 mutation 事务；不确定的 storage 失败会使 Session 中毒，必须 reopen。
