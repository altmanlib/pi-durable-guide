---
title: Agent、Settings、Environment
type: guide
status: published
updated: 2026-03-22
---

# Agent、Settings、Environment

## Per-Conversation Agent

每个 conversation 把「用什么跑」存在 `pi.agent`。`configure()` 一次 commit 改完；未设置字段跟随 host：

```typescript
await root.configure(
	{
		model: { provider: "openai", modelId: "gpt-6-sol" },
		thinkingLevel: "high",
		extensions: { remove: [Coding] }, // 相对 host 默认做编辑；传数组则精确选择这些，按序
		tools: [readTool, bashTool], // 数组 = 只提供这些；{ remove: [...] } = 去掉一些
		instructions: "Only read; never edit files.",
		cwd: "/work/repo",
	},
	context,
);
await root.configure({ tools: null }, context); // null 把字段清回 host 默认
const agent = await root.agent(context); // 解析后的：model、extensions、tools、sections、cwd
```

要点：

- extensions / tools 以对象传入、按名字存储；名字可在代码卸载后仍留在存储里，卸载后选中它的 conversation 暂时拿不到它，直到再次安装
- `createConversation()` / `fork()` / `root()` 接受与 `agent` 相同的变更
- 被 task 拥有的 conversation（如 subagent）从 owner conversation 的 agent 复制起步
- fork 从 fork entry 时刻父 conversation 的 agent 起步
- 一次请求的 model / prompt / offered tools 在准备请求时固定；变更从下一次请求起生效
- tool calls 与 hooks 在各自 task phase 解析 agent；environment 每次按当前 `cwd` 构建，因此 `cwd` / extension 变更可能影响模型已经发出的后续 calls

## Settings

所有 conversation 共享的运行策略通过 `settings` 传入。它每次使用时读取、从不入库，所以 getter 可以做成 live（例如读配置文件）：

```typescript
const harness = await Harness.open(storage, {
	models,
	registry,
	settings: {
		extensions: [CodingTools, Coding], // 默认选中；缺省 = 全部已安装
		stream: { timeoutMs: 120_000 },
		retry: { maxRetries: 3 },
		compaction: { reserveTokens: 16384 },
		progress: { partialIntervalMs: 100, outputIntervalMs: 100 },
		toolExecution: "parallel",
		contextRetentionMs: 600_000,
		get followUpMode() {
			return userSettings.followUpMode;
		},
	},
}, context);
```

## Environment

`env` 为每次 tool call、section 渲染、`runtime.env()` 构建执行环境。它收到 conversation ID、agent `cwd`、以及已提交的 reads，因此一个函数可以做到「每 conversation 一个目录」或「每 conversation 一个容器」：

```typescript
import { NodeExecutionEnv } from "@earendil-works/pi-durable/env/node";

const harness = await Harness.open(storage, {
	models,
	registry,
	env: ({ cwd }) => new NodeExecutionEnv({ cwd: cwd ?? process.cwd() }),
}, context);
```

- `env` 抛错 → 成为该 call 的 error result
- 没有 environment 时，内置工具以 error result 失败
- 每次 call 新建 env 对象没问题：`edit` / `write` 按 env 的 `id` + path 串行化对同一文件的变更
- 自定义 `ExecutionEnv` 设置 `id`，使相同 id 在相同路径看到相同文件（例如每容器一个 id）

宿主也可直接用 environment，例如展示项目文件：

- `openBinaryReader()`：按字节范围读已打开文件
- `openDirReader()`：分页读目录
- `exec(argv, ...)`：不经 shell 跑程序，并标明输出块来自 stdout 还是 stderr

取消 context 可停止单次 call；`cleanup()` 用于关闭环境。自定义 env 可用 `@earendil-works/pi-durable/testing` 的 `registerEnvConformance()` 做一致性检查。

## Reload

同名安装会一步替换已安装 extension：

```typescript
registry.install(await loadCodingExtension()); // 同名 "coding"：替换已安装的
```

`registry.uninstall(extension)` 按名字移除，不论对象是哪一个。

已开始的工作保留它拿到时的代码：

- 正在跑的 tool call 用旧实现跑完
- 每个 task phase 在 phase 开始时解析 hooks / agent 一次
- 下一 phase / request / call 用新代码
- 重启后需再次安装相同 extensions；extension 自带 `tasks` 的 pending tasks 在安装后才恢复
