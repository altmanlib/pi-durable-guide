---
title: 深度学习路径
type: guide
status: published
updated: 2026-03-22
---

# 深度学习路径

目标：不只会用 API，还能在脑中推演「一次 commit / 一次崩溃 / 一次 abort」之后系统处于什么状态。

## 建议顺序

| 阶段 | 读什么 | 练什么 | 过关标准 |
|---|---|---|---|
| A. 用户面 | 本目录 `overview` → `examples` | 跑 `14-chat`、`17-coding-tools`、`19-json` | 能解释 submit → generation → tool → answer |
| B. 持久化语义 | [invariants-and-records.md](invariants-and-records.md) | 对照 `00`–`06` 示例 | 能背出 8 条 invariants，并指出「可见进度」从哪来 |
| C. Document | [documents-deep.md](documents-deep.md) | 跑 `01-documents`、`11-extension-state` | 能说清 scope / history / fork / incarnation |
| D. Task 内核 | [transactions-and-tasks.md](transactions-and-tasks.md) | 跑 `12-tasks`、`13-recovery`、`24-child-tasks` | 能画出 effect sandwich 与 `completing` hold |
| E. 内置 agent loop | [builtin-tasks.md](builtin-tasks.md) | 跑 `20-inbox`、`22`/`23`、`25-compaction` | 能逐步说出 generation / tool / compaction 的 phase |
| F. Chord 观察层 | [chord-usage.md](chord-usage.md) | 读 `test/chord-guide.test.ts` | 能区分 Session commit 线与 Chord 投递 |
| G. 踩坑与边界 | [footguns-and-nongoals.md](footguns-and-nongoals.md) | 故意踩 2–3 个 footgun（只在实验目录） | 遇到症状能映射回对应条款 |
| H. Storage | [storage-contract.md](storage-contract.md) | 扫 conformance 测试名 | 知道 Storage 信任什么、强制什么 |

## 心智模型（先记住这张图）

```text
Host / Extension / Tool code
        │  (外部副作用：模型、进程、网络)
        ▼
Task phase handler  ── commit intent / outcome ──► Session mutation line
                                                        │
                                                        ▼
                                                   Storage.commit
                                                        │
                                                        ▼
                                              已提交状态被采纳
                                                        │
                         ┌──────────────────────────────┼──────────────────────────────┐
                         ▼                              ▼                              ▼
                   view / watchDoc                 agent events                   taskGraph
                   (Chord 投递，离线)              (派生事件)                    (live tasks)
```

三条硬边界：

1. **副作用不进 mutation 事务**
2. **只有已提交状态可观察**
3. **不确定的 storage 失败 = Session 中毒，必须 reopen**

## 与英文原文的关系

| 导读 | 对应原文 |
|---|---|
| invariants / records | `docs/spec.md` §1–2 |
| documents | `docs/spec.md` §3 |
| transactions / tasks | `docs/spec.md` §4–5 |
| builtin tasks | `docs/spec.md` §8 |
| chord | `docs/pico-v5-chord-usage.md` |
| footguns / non-goals | `docs/spec.md` §12–13 |
| storage | `docs/spec.md` §10–11 |

导读是压缩与中文化；字段级合同以 `spec.md` 为准。

## 推荐实验节奏

1. 一天：跑完 `14`–`21`，对照用户面文档
2. 一天：只盯 recovery——杀进程、`--continue` / reopen、看 `interrupted` 与 checkpoint
3. 一天：只盯 ownership——`22`/`23`/`24`，画 owner 树与 abort 方向
4. 一天：只盯 compaction + prompt cache——`25` + system sections 稳定性
5. 之后：对着 `spec.md` 把本导读当索引查，不要反向用导读替代规范
