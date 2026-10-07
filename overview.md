---
title: 概述与安装
type: guide
status: published
updated: 2026-03-22
---

# 概述与安装

## 它是什么

`@earendil-works/pi-durable` 是一个**可持久化的 agent harness**。

它负责把这些东西在展示之前先提交进 storage：

- 对话（conversation）
- 模型轮次（model turns）
- 工具调用（tool calls）
- 你自己的状态（documents / tasks）

如果进程在一轮中间崩溃，重新打开同一个 storage，工作会从上次停下的地方继续。

依赖：

- `@earendil-works/pi-ai`：模型访问
- `@earendil-works/chord`：文档状态与观察

## 核心规则

> Session 原子地提交不可变 entries、完整 task records，以及 Chord 跟踪的 documents。只有已提交状态才可被观察。

要点：

1. 一次 commit 对所有记录与文档写入是原子的
2. 文档更新只在 storage commit 成功后发布
3. 没有「先显示、后落盘」的易失路径
4. 外部副作用不在 Session mutation 事务内执行

## 安装

```bash
npm install @earendil-works/pi-durable @earendil-works/pi-ai @earendil-works/chord
```

## 包状态

包标记为 **Experimental**。写产品代码前应对照当前 `packages/durable/README.md` 与 `docs/spec.md`，不要把本导读当成冻结 API 合同。
