# 02 · Sidecar（sw-com-kit）设计

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 覆盖原 v2.1 文档 §6：技术栈、私有协议、内部结构、串行队列、可靠性迁移清单
> 关联：[07-correctness.md](./07-correctness.md)（队列与版本的正确性语义）、[08-com-impl.md](./08-com-impl.md)（参考实现）

---

## 1. 定位

Sidecar 是**所有 COM 状态与领域时序的唯一持有者**：

- 唯一通过 winax 连接 SolidWorks 的进程
- 会话状态（活动文档/草图/选择集/单位）跨工具调用维持
- 复合操作（意图级原子单元）在此实现，脆弱 COM 时序不外泄
- 同时服务两条通道：通道B（opencode 插件的 `/op`）与通道C（主程序面板的只读端点 + SSE）

存在理由（为什么必须是独立进程）：winax 装不进 Bun、COM 同步阻塞不能冻结 opencode/UI、状态需跨调用维持、关桌面应用不应关 SolidWorks。

---

## 2. 技术栈

- Node 20 + TypeScript（直接复用现有仓库 代码，ESM）
- Fastify（HTTP + SSE），监听 `127.0.0.1:7654`，Bearer token 鉴权
- 从 solidworks-mcp-server 迁移：operations / helpers / utils / adapters 四个目录（对照表见 [10-packaging.md](./10-packaging.md) §5）

---

## 3. 私有协议

### 3.1 端点一览

```http
POST /op                     # 执行复合操作（通道B，入串行队列）
{ "op": "feature", "args": { "type": "boss", "depth": 50, "endCondition": "Blind" },
  "expectedGen": 42 }        # 乐观并发：调用方持有的观察版本
→ { "ok": true, "feature": "拉伸1", "screenshot": "<base64>", "state": {...}, "generation": 43 }

POST /op  (长任务)
{ "op": "export_batch", "args": {...} }
→ { "ok": true, "jobId": "j_123" }        # 立即返回

GET  /job/:id               # 轮询长任务状态
GET  /state                 # 当前会话快照（activeDoc/sketch/特征树摘要/单位/队列深度/generation）
GET  /screenshot?w=800      # 当前视图截图（image/png，Buffer 直传）
GET  /events                # SSE：view_updated/state_changed/job_progress/doc_switched/sw_exited
POST /select                # 主程序面板直发选择（通道C，交互优先，见 §5）
GET  /health                # 主程序心跳
```

### 3.2 设计原则

- **意图级端点**，不是 COM 方法的 1:1 镜像
- 每个会改状态的操作响应里**自带 state 快照 + 可选截图**，插件层不用二次请求
- 错误响应携带 `category`（recoverable / state_error / permanent / stale_state）与 `hint`（给 LLM 的下一步建议）
- 协议带 `protocol` 版本字段，主程序与 sidecar 版本强绑定同发布：请求头 `X-Protocol-Version` 不匹配时 sidecar 返回 426 并拒绝服务

---

## 4. 内部结构（迁移自 solidworks-mcp-server）

```
sidecar/
├── server.ts                # Fastify 入口 + SSE + Bearer 鉴权
├── queue.ts                 # 全局串行队列 + generation 版本（COM STA 强制）
├── session.ts               # 会话状态机（activeDoc/sketch/选择集）
├── jobs.ts                  # 长任务管理（jobId → progress）
├── dil/                     # DIL 前端与编译器（新增，规范见 02 文档）
│   ├── parser.ts            # YAML→AST、词汇/引用/DAG 前置校验
│   ├── expr.ts              # params 表达式求值
│   ├── symbols.ts           # 符号表：DIL id ↔ 原生句柄/名称（屏蔽中英文命名差异）
│   └── adapters/sw.ts       # SolidWorks adapter：DIL 动词→复合操作序列 + 能力矩阵
├── ops/                     # 复合操作层（意图级；参考实现见 08 文档）
└── kit/                     # ← 从 solidworks-mcp-server 迁移
    ├── operations/          # connection/model/sketch/dimension/export/vba/...
    ├── helpers/             # extrusion/selection/sketch/model
    ├── types/               # COM 接口与业务类型声明
    ├── utils/               # com-boolean/error-recovery/com-lifecycle/...
    └── adapters/            # macro-generator（VBA 回退）
```

**DIL 编译器是 sidecar 的新前端**（[02](./02-intent-language.md) §6）：`/op` 收到的 `sw_*` 快捷调用在入口处即编译为 DIL 片段，与 `dil_apply` 走同一条 编译→执行→门控 流水线——两套入口、一个 IR。符号表让 state/diagnostics 以 DIL id 为主键输出，原生名（"草图1"/"Sketch3"）被封在 sidecar 内部。

---

## 5. 全局串行队列

LLM 会并行发出工具调用，而 COM 是 STA 单线程。队列规则：

- 所有 `/op` 请求入队，FIFO 串行执行
- 队列深度上报到 `/state`（`pendingOps: 3`），插件层可在状态行展示
- 长任务（rebuild/批量导出/干涉检查）不占队列——转为后台任务（job）执行；**同一时刻只允许一个后台 job**（rebuild 类 job 独占 COM/CPU；多 job 并存属 [12](./12-multi-workpiece.md) §4.2 的多文档决策）
- `/select`（人点特征树）走**交互优先通道**：先于队列中下一个 `/op` 执行——**人的操作永远优先于 agent 队列**。select 是通道C 唯一的写端点（白名单轻量写），仅修改选择集，不改几何，与 agent 的潜在冲突由 generation 版本校验兜底（[07](./07-correctness.md) §3）

队列的正确性语义（为何它同时是"原子化容器"与"乐观并发检查点"）见 [07-correctness.md](./07-correctness.md) §2/§3。

---

## 6. 可靠性迁移清单（原样搬，与传输/形态无关）

| 机制 | 来源（solidworks-mcp-server） |
|---|---|
| COM 错误码分类 + 指数退避重试 | utils/error-recovery.ts |
| 13+ 参数 → VBA 宏回退 | adapters/macro-generator.ts + operations/vba.ts |
| VARIANT_BOOL(-1/0) 转换 | utils/com-boolean.ts |
| `{value:n}` out 参数模拟 | 各 operations |
| Silent 导出（禁模态框） | operations/export.ts |
| 多策略实体选择 | helpers/selection.ts |
| 退草图时序 | helpers/extrusion.ts |
| COM 对象存活探测 | utils/com-lifecycle.ts |

各机制在代码中的落点示例见 [08-com-impl.md](./08-com-impl.md)。
