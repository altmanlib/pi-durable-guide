---
title: 示例与设计文档
type: guide
status: published
updated: 2026-03-22
---

# 示例与设计文档

## 怎么跑示例

示例在仓库 `packages/durable/test/examples`。在该包目录执行：

```bash
node --conditions=source --experimental-strip-types test/examples/14-chat.ts
```

需要真实 OpenAI 的示例依赖 `OPENAI_API_KEY`；多数示例用 faux provider。

## 示例索引

| 示例 | 展示内容 |
|---|---|
| `14-chat` | 一问一答 |
| `16-real-model` | 对流式 OpenAI 答案 |
| `17-coding-tools` | JSONL storage 上的工具轮次 |
| `18-print` | print 模式：提交 prompt，打印答案 |
| `19-json` | JSON 模式：agent events 或原始 view ops；SQLite / JSONL / memory |
| `20-inbox` | busy 时的 steer、follow-up、write、撤回 |
| `21-late-join` | 中途附着 view 与 event stream |
| `22-subagent-foreground` | replay-safe subagent；child 由 call 拥有；child events 缩进在 call 下 |
| `23-subagent-background` | 持久 subagents：spawn / steer / stop / list；答案回报；重启安全 |
| `24-child-tasks` | checkout 拥有并等待四笔 payment：failFast、abort、restart |
| `25-compaction` | 长聊天：后台 / 手动 / 上下文溢出后的 compaction |
| `26-coding-agent` | CodingTools + live settings + 跟随 conversation 目录的 environment |
| `27-plan-mode` | 只读 plan mode：自有 document，用 `configure()` 切换 |
| `28-reviewer` | reviewer conversation：独立 model / extensions / tools / 目录与 review loop |
| `29-sandbox-per-conversation` | 每 conversation 一个 environment，从 app document 查找 |
| `30-tool-override` | 部分 conversation 同名 bash；wrapper 给胜出的 bash 计时 |
| `31-reload-and-restart` | call 中途 reload extension；存储的选择跨重启存活 |
| `00`–`13` | 底层分层：sessions、documents、forks、watches、Harness、agent 配置、reload、extension state、tasks、recovery |

## 设计文档（英文原文）

| 文档 | 角色 |
|---|---|
| `packages/durable/docs/spec.md` | 规范性规格（约 4700 行） |
| `packages/durable/docs/pico-v5-handoff.md` | 实现计划 / handoff |
| `packages/durable/docs/pico-v5-chord-usage.md` | 本包如何使用 Chord |

## 上层 demo

`packages/coding-agent/src/experimental/durable/`：本机 TUI + SQLite 的 durable coding agent，复用 pi 的 model runtime、auth、settings、system prompt、keybindings、theme。会话目录大致在：

`~/.pi/agent/experimental/durable-sessions/<cwd-hash>/<session>/session.sqlite`

## Benchmarks

在 durable 包内：

- `npm run bench:storage`
- `npm run bench:storage:memory`
- `npm run bench:tool-output`

## License

MIT
