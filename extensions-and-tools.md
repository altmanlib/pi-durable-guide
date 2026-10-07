---
title: 扩展与工具
type: guide
status: published
updated: 2026-03-22
---

# 扩展与工具

## Extensions

Harness 除内置 tasks 外要跑的代码，都放进你进程拥有的 registry 里的具名 extension：

```typescript
import { createRegistry, defineExtension, defineTool, hook, section, ToolTask } from "@earendil-works/pi-durable";
import { CodingTools } from "@earendil-works/pi-durable/tools";

const Coding = defineExtension({
	name: "coding",
	sections: [section("preamble", () => "You are a concise coding assistant.", { tag: false })],
	hooks: [hook(ToolTask, { beforeTool: (call) => (isDangerous(call) ? { block: "Needs approval" } : undefined) })],
});

const registry = createRegistry();
registry.install(CodingTools);
registry.install(Coding);
```

一个 extension 可以带：

- `tools`
- `sections`
- `hooks`
- `wraps`（按名字装饰某个 tool 或 section）
- `tasks`

默认每个 conversation 按安装顺序选中全部已安装 extensions。registry 内容不入库；conversation 只存 extension 名字。

## 内置工具

来自 `@earendil-works/pi-durable/tools`：

| 工具名 | 工厂 | 说明 |
|---|---|---|
| `read` | `createReadTool()` | 读文本文件；支持 `offset` / `limit`；暂不支持读图 |
| `write` | `createWriteTool()` | 写/覆盖文件 |
| `edit` | `createEditTool()` | 按精确文本做定向替换 |
| `bash` | `createBashTool()` | 在 call 的环境中执行 bash |
| `powershell` | `createPowerShellTool()` | PowerShell；**不在** `CodingTools` 里，需单独安装 |

`CodingTools` 扩展一次性装入 `read` / `write` / `edit` / `bash`（`name: "coding-tools"`）。它**不会**自动安装，需要 `registry.install(CodingTools)`。

这些工具只通过 call 的 environment 碰文件与进程。

## 自定义工具

用 TypeBox schema 定义。`defineTool()` 会从 `parameters` 推导 `args` 类型；Harness 在 `execute()` 前校验。`api.output()` 流式输出运行中内容；若 `execute()` 不返回 `content`，这些输出会成为结果：

```typescript
import { Type } from "@earendil-works/pi-ai";

const count = defineTool({
	name: "count",
	description: "Count from 1 to n",
	parameters: Type.Object({ n: Type.Number() }),
	execute: async (args, api) => {
		for (let i = 1; i <= args.n; i++) api.output(`${i}\n`);
		return {};
	},
});
registry.install(defineExtension({ name: "count", tools: [count] }));
```

行为要点：

- 每次调用是一个独立 durable task；intent 在 `execute()` 前已提交
- 进程中途挂掉后：仅当声明 `replay: "safe"` 才会重跑；否则模型收到 `interrupted` 错误结果，并带上已提交的输出
- `execute()` 抛错 → 模型收到 error result
- 结果可带 `usage`，计入 conversation 的 usage
- 结果可带 `control: { terminate: true }`：当本轮所有结果都要求 terminate 时，run 结束，不再请求模型

后装同名 tool 会替换先装的（在两者都被选中时）。`wrapTool()` 装饰最终胜出的那个：

```typescript
const Venv = defineExtension({
	name: "venv",
	tools: [createBashTool({ commandPrefix: "source .venv/bin/activate" })],
});
const Timing = defineExtension({
	name: "timing",
	wraps: [wrapTool(createBashTool(), (bash) => ({
		...bash,
		execute: (args, api, ctx) => timed(() => bash.execute(args, api, ctx)),
	}))],
});
```

## System Prompt

系统提示由已选 extensions 的 sections 按顺序渲染，在每次请求前生成。section 能看到解析后的 agent、为请求构建的 environment、以及已提交的 documents：

```typescript
section("cwd", (input) => input.env?.cwd); // 渲染为 <cwd>\n...\n</cwd>；undefined 则省略
```

conversation 的 `instructions` 最后渲染，section 名为 `instructions`。sections 与 tool 变更以位置化 system entries 写入 transcript。只有变化的部分会再次发送，以保持 provider prompt cache 热度。每次返回都不同的 section（例如当前时间）会破坏缓存效果。
