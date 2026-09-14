# 03 · Agent 与 Sidecar 交互机制（通道B详解）

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 覆盖原 v2.1 文档 §7：九步调用链、发起方、插件桥接代码、报文示例、边界接口、并发语义
> 关联：[06-tool-set.md](./06-tool-set.md)（工具 schema）、[04-sidecar.md](./04-sidecar.md)（协议端点）

---

## 1. 认知前提：LLM 不知道 sidecar 的存在

模型不感知 HTTP、进程、COM。它唯一会做的事：在回复文本里发出**工具调用**（tool_use，如 `sw_feature({type:'boss', depth:50})`）。从 tool_use 到 SolidWorks 模型变化的全部环节都是 opencode 与插件的机械配合。

---

## 2. 完整调用链（九步）

```
用户: "建一个 50×30×10 的长方体"
  │
  ▼
┌─ opencode agent loop ─────────────────────────────────────┐
│ ① 消息+工具 schema 发给 LLM                                │
│ ② LLM 生成: tool_use { name:"sw_feature", args:{...} }    │
│ ③ opencode 查工具注册表 → 命中 plugin 注册的 sw_feature     │
│ ④ 执行该工具的 execute 函数 ← 就是一个普通 TS 函数          │
└──────────────┬────────────────────────────────────────────┘
               │ ⑤ execute 内部: fetch('http://127.0.0.1:7654/op', ...)
┌──────────────▼─────────────┐
│ sidecar: 入串行队列 → 执行 COM 时序 → 截图                  │
└──────────────┬─────────────┘
               │ ⑥ JSON 响应 {ok, feature, screenshot, state}
               ▼
⑦ tool.execute.after 钩子: 把 screenshot 转成 image part
⑧ 组装 tool_result 消息（文本+图+state行）塞回对话
  ▼
⑨ LLM 看到结果（和图！），决定下一步或汇报完成
```

---

## 3. 发起方只有两种

| 发起者 | 机制 | 例子 |
|---|---|---|
| **LLM 自主决定** | agent loop 检测 tool_use → 执行 | 对话中模型判断"该拉伸了" |
| **确定性入口** | command/skill 预制 prompt 引导 | 用户敲 `/model-part 支架` |

sidecar **永远不会主动发起** agent 调用；它推的 SSE 事件只给主程序面板（通道C），不进对话。

---

## 4. 插件即桥：`.opencode/plugins/cad.ts` 核心

> 以下代码已对照 opencode 官方插件文档（2026-08）修正：自定义工具用 `tool()` helper + **Zod** `args`（不是 JSON Schema `parameters`）；`execute(args, context)` 带上下文（多工件会话绑定依赖 `context.sessionID`，见 [12](./12-multi-workpiece.md) §5.2）。

```typescript
import type { Plugin } from "@opencode-ai/plugin"
import { tool } from "@opencode-ai/plugin"
import { z } from "zod"

const SIDECAR = "http://127.0.0.1:7654"
const HEADERS = { Authorization: `Bearer ${process.env.SW_TOKEN}` }

export default (async () => {
  return {
    // 注册自定义工具（execute = 无状态转发层，一次 fetch）
    tool: {
      sw_feature: tool({
        description: "创建实体特征：凸台/切除/旋转/薄壁拉伸",
        args: {
          type: z.enum(["boss", "cut", "revolve", "revolve_cut", "thin"]),
          depth: z.number().describe("mm"),
          endCondition: z.string().optional(),
        },
        async execute(args, context) {          // context 含 sessionID 等
          const res = await fetch(`${SIDECAR}/op`, {
            method: "POST",
            headers: { ...HEADERS, "Content-Type": "application/json" },
            body: JSON.stringify({
              op: "feature",
              args,
              sessionKey: context.sessionID,     // 多工件：会话→文档映射（12 §5.2）
            }),
          })
          return await res.json()   // {ok, feature, screenshot, state}
        },
      }),
      // ...其余 sw_* 工具同构
    },

    // 特征类工具成功后：截图转 image part 回注给 LLM（视觉闭环）
    // ⚠️ image part 注入是待验证 API（00 待验证假设 #1）：
    //    若 output 无 parts 数组，Plan B = 截图落盘 + 经消息 parts 旁路注入
    "tool.execute.after": async (input, output) => {
      if (input.tool.startsWith("sw_") && output.result?.screenshot) {
        output.parts.push({ type: "image", data: output.result.screenshot })
      }
    },
  }
}) satisfies Plugin
```

要点：

- 工具的 `execute` 是**无状态转发层**——一次 fetch，零业务逻辑；时序/重试/VBA 回退全在 sidecar
- `tool.execute.after` 是视觉闭环的挂载点：sidecar 响应自带的 screenshot 在此转成 image part（**该注入方式未经官方文档确认，P0 spike 验证，失败走 Plan B**）
- 工具注册机制（`tool()` helper + Zod args）已对照官方文档；image part 注入是唯一悬而未决的 API 假设

---

## 5. 一次调用的实际报文

```http
POST /op HTTP/1.1
Host: 127.0.0.1:7654
Authorization: Bearer eyJ...
{ "op": "feature",
  "args": { "type": "boss", "depth": 50, "endCondition": "Blind" },
  "expectedGen": 42 }

HTTP/1.1 200 OK
{ "ok": true,
  "feature": "拉伸1",
  "screenshot": "iVBORw0KGgo...",       // ← 插件转 image part，LLM 下一轮看到
  "state": { "doc": "Part1", "features": 6, "units": "mm", "queueDepth": 0 },
  "generation": 43 }
```

---

## 6. 每一环的边界接口

| 边界 | 传递物 | 为什么在这断开 |
|---|---|---|
| agent → 工具 | **JSON 参数**（意图） | LLM 只表达"要什么"，不懂怎么做的 |
| 工具 → sidecar | **HTTP /op**（Bun → Node 进程边界） | winax 进不了 Bun；fetch 跨进程 |
| sidecar → COM | **winax 方法调用**（JS → 原生边界） | 单位换算/VARIANT_BOOL/out 参数在这消化 |
| COM → SolidWorks | **COM 自动化**（进程界，RPC） | STA 单线程，串行队列在这守门 |

两个易误解点：

1. **agent 不是"持续连接" sidecar**——每次工具调用就是一次独立 fetch；连接状态活在 sidecar 的会话状态机里，这正是选择集能跨调用维持的原因
2. **工具层和 agent 同进程**——plugin 的 execute() 跑在 opencode 里，是 agent loop 的一部分；真正跨进程的只有 fetch 那一跳

---

## 7. 并发语义

模型在一条消息里并行发 3 个工具调用 → opencode 并发执行 3 次 fetch → sidecar FIFO 依次执行：

- **串行队列天然化解 COM 并发冲突**（COM STA 由队列守门）
- 有依赖关系的操作由 AGENTS.md 约束模型分步发（"有依赖的操作必须分步发"）
- 配合响应自带 state 让模型养成串行习惯（每次看到最新状态再决定下一步）

并发正确性的完整模型（含人机竞态、过期版本拒收）见 [07-correctness.md](./07-correctness.md)。
