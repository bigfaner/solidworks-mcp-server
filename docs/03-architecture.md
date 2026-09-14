# 01 · 总体架构与关键决策

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 覆盖原 v2.1 文档 §1–§5：产品形态、四层拓扑、关键决策、主程序设计、opencode 层设计

---

## 1. 产品形态定义

### 1.1 v1 → v2 的关键变化

v1 方案假设用户在终端里用 opencode TUI，插件挂工具。v2 明确为**独立桌面应用**：

| 维度 | v1（TUI 插件形态） | v2（桌面应用形态） |
|---|---|---|
| 用户界面 | opencode TUI | 自研 GUI：对话面板 + CAD 专用面板 |
| opencode 角色 | 用户直接操作的工具 | **被嵌入的 agent 引擎**（headless 服务） |
| 特征树/进度/视图 | 无处展示，挤在聊天里 | 专用面板 |
| 权限审批 | TUI 内 ask | 原生对话框/通知 |
| 分发 | `npm i` + 手工配置 | 安装包（exe），开箱即用 |
| fork TUI 的动机 | 存在（做面板要改 TUI） | **消失**（自建客户端） |

### 1.2 产品定义

**一个 Windows 桌面应用，内置并托管 opencode 服务 作为 agent 决策层、sw-com-kit 作为 SolidWorks 执行层，用户用自然语言驱动建模，在专用面板里看到特征树、实时视图、任务进度，破坏性操作弹原生审批框。**

用户心智模型：

```
┌──────────────────────────────────────────────────────┐
│  CAD Agent (桌面应用)                                  │
│  ┌────────────┐ ┌──────────────┐ ┌────────────────┐ │
│  │            │ │  SolidWorks  │ │   特征树面板    │ │
│  │  对话面板   │ │  视图镜像     │ │   (实时同步)    │ │
│  │  (含工具    │ │  (截图流)     │ ├────────────────┤ │
│  │   调用卡片) │ │              │ │   任务/队列面板  │ │
│  │            │ │              │ ├────────────────┤ │
│  │            │ │              │ │   审批通知      │ │
│  └────────────┘ └──────────────┘ └────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## 2. 总体架构：四层拓扑

```
┌──────────────────────────────────────────────────────────────────┐
│ ④ 主程序 CADAgent.exe（Electron）                                 │
│    Renderer(React)：对话面板 / 视图镜像 / 特征树 / 任务 / 审批      │
│    Main：子进程编排 · 三通道路由 · 生命周期与自愈 · 托盘            │
└──────┬──────────────────────┬────────────────────────────────────┘
       │ 通道A（会话通道）：opencode SDK    │ 通道C：直连 sidecar（只读+SSE）
       │ (HTTP + 事件流)        │ （面板数据，不经 agent）
┌──────▼──────────────┐  ┌─────▼──────────────────────────────────┐
│ ③ opencode 服务      │  │ ② sw-com-kit sidecar（Node 20）        │
│   (headless, Bun)   │  │   Fastify 127.0.0.1:7654 + token        │
│   agent loop · LLM  │  │   串行队列 · 会话状态机 · job 管理        │
│   加载 .opencode/   │  │   可靠性层（错误重试/VBA回退/单位换算）    │
│   ├ plugin/cad.ts ──┼──┤ 通道B：sw_* 工具 → fetch sidecar /op     │
│   ├ agent/*.md      │  │   SSE：state_changed / job_progress /   │
│   ├ skills/sw-*/    │  │        doc_switched / sw_exited         │
│   └ opencode.json   │  └─────┬───────────────────────────────────┘
└─────────────────────┘        │ winax → SldWorks.Application
                        ┌──────▼─────────┐
                        │ ① SolidWorks   │
                        └────────────────┘
```

### 2.1 四层职责一句话

1. **SolidWorks**：被控对象（用户已有，需许可证）
2. **sidecar**：所有 COM 状态与领域时序的唯一持有者（从现有仓库 迁移）
3. **opencode 服务**：agent 决策层；plugin 注册的 `sw_*` 工具是打到 sidecar 的无状态转发层
4. **主程序**：用户唯一入口；既是 opencode 的客户端（通道A），也是 sidecar 的直连客户端（通道C，面板只读数据不占用 agent 工具通道）

### 2.2 三条通道（职责严格区分）

| 通道 | 谁连谁 | 走什么 | 为什么分开 |
|---|---|---|---|
| A（会话通道） | 主程序 → opencode | SDK/HTTP：会话、消息、事件订阅、权限应答 | agent 交互的正式通道 |
| B（工具通道） | opencode plugin → sidecar | HTTP `/op`：工具执行 | agent 之手；与 UI 无关 |
| C（面板数据通道） | 主程序 → sidecar | HTTP 只读端点 + SSE：视图截图流、特征树、job 进度 | **面板刷新不产生 agent 消息**，不污染上下文、不占串行队列 |

### 2.3 Sidecar（伴随进程）模式说明

"Sidecar"（边车）是进程架构模式：主程序旁边挂一个独立辅助进程，替主程序承担它干不了或不该干的工作。独立进程、独立运行时、IPC 通信、崩溃相互隔离、可单独重启。本项目存在 sidecar 的四个硬理由：

1. winax 是原生 Node 插件，**装不进 Bun**（opencode 运行时）
2. COM 调用**同步阻塞**，不能冻结 opencode / 桌面 UI
3. COM 选择集/文档状态需**跨调用维持**
4. 关桌面应用**不应**关 SolidWorks

同类实例：VS Code 的语言服务器（LSP server）、Istio 的 Envoy proxy sidecar。

---

## 3. 关键决策

### 3.1 主程序 = opencode 的又一个客户端（不是 fork）

opencode 本身是 server/client 架构：核心是 headless 服务，TUI 只是官方客户端之一。主程序通过官方 SDK/HTTP 对接 opencode 服务，**零 fork**。由此：

- opencode 升级 = 换一个打包进安装包的版本，无合并成本
- TUI 用户的全部能力（agent/skill/command/config）在主程序里同样生效
- v1 里"做面板要 fork TUI"的动机彻底消失

### 3.2 桌面框架选型：Electron（推荐）vs Tauri

| 考量 | Electron | Tauri |
|---|---|---|
| 团队技能匹配（本仓库为 TS） | ✅ 全 TS | 后端 Rust |
| 子进程管理（opencode/sidecar） | ✅ Node `child_process` 原生 | 可行（官方 sidecar/shell 插件），但子进程编排生态弱于 Node child_process |
| 体积/内存 | 差（~150MB） | 好（~10MB） |
| 与 sidecar 同构（都是 Node 生态） | ✅ | 无关 |
| 生态（聊天 UI、虚拟列表、图表） | ✅ | ✅（同一 Web 生态） |

**结论：Electron。** 理由：整个交付物已是多进程 Node 系（opencode + sidecar + winax），Electron 的 Main 进程天然是这些子进程的"编排器"；体积劣势对工作站级 CAD 用户不敏感。Tauri 留作未来轻量版选项。

**重要：Electron 不运行 winax。** 理由：① winax 的 Electron 构建（electron-rebuild）无人维护；② 即便用 utilityProcess 承载，COM 同步阻塞仍会卡死该进程内全部逻辑；③ sidecar 作为标准 Node 进程可零改动复用现有仓库代码。Electron 只负责 spawn/监控它。

### 3.3 已知风险：opencode 的 Windows 原生运行

opencode 官方对 Windows 的表述是"可直接运行但推荐 WSL"。本方案**必须在 Windows 原生运行**（WSL 无法访问 COM/winax），属于官方非推荐路径。缓解：P0 阶段在干净 Windows 环境对 opencode serve 模式做稳定性验证（见 [00 待验证假设](./00-OVERVIEW.md) #2），锁定经验证的版本号打包，问题上报上游。

### 3.4 SolidWorks 视图呈现：截图镜像（而非 HWND 内嵌）

| 方案 | 评估 |
|---|---|
| **截图流镜像**（sidecar 定时/事件触发截图，SSE 推到面板） | ✅ 推荐：稳定、跨 DPI 安全、同源数据可直接回注给 agent（视觉闭环复用） |
| HWND 重父（SetParent 把 SW 窗口塞进面板） | ❌ 已知 hack：DPI/菜单/弹窗/崩溃传导问题多，仅作远期实验项 |

用户想直接操作时点"聚焦 SolidWorks"按钮把 SW 窗口置前——**agent 与人共用同一个 SolidWorks 实例**，这是本架构的产品优势。

### 3.5 进程创建与归属

| 进程 | 谁启动 | 生命周期 |
|---|---|---|
| CADAgent.exe（Electron） | 用户/开机自启 | 宿主进程，托盘常驻 |
| opencode 服务 | 主程序 Main 进程 spawn（bundled） | 随主程序；崩溃自动重启，会话不丢 |
| sidecar（node.exe + kit） | 主程序 Main 进程 spawn（bundled） | 随主程序；崩溃自动重启，COM 重连 |
| SolidWorks | 用户启动，或 sidecar 经 COM 激活 | **独立于主程序**——关掉 CADAgent 不关 SW |

---

## 4. 主程序设计

### 4.1 Main 进程：进程编排器

```
职责：
├── 启动顺序：sidecar → 健康检查 → opencode → 健康检查 → Renderer 就绪
├── 健康监督：心跳轮询（sidecar /health、opencode /global/health）；退避重启（3次失败转托盘告警）
├── SolidWorks 探测：注册表查 SldWorks.Application ProgID/版本 → 未装则首次启动引导页
├── 凭据分发：生成 SW_TOKEN（sidecar Bearer）与 OPENCODE_SERVER_PASSWORD（opencode Basic Auth），注入子进程 env 与 Renderer 配置
├── 托盘：常驻、全局快捷键呼出、SW 连接状态角标
└── 单实例锁 + 日志聚合（三进程日志统一滚动上传入口）
```

### 4.2 Renderer：五个面板

| 面板 | 数据源 | 交互 |
|---|---|---|
| **对话** | 通道A（opencode 事件流） | 消息、工具调用卡片（可折叠：工具名/参数/结果摘要/截图缩略）、重试/中断 |
| **视图镜像** | 通道C：SSE `view_updated` → 拉取 PNG | 缩放；"聚焦 SolidWorks"按钮；截图历史时间线（回看每步操作后的样子） |
| **特征树** | 通道C：`/state` + SSE `state_changed` | 特征列表（名/类型/suppressed）、点击选中（经通道C `/select`，非 agent）、右键抑制/回滚（P2） |
| **任务/队列** | 通道C：`/jobs` + SSE `job_progress` | 串行队列深度、长任务进度条、失败原因 |
| **审批通知** | 通道A：permission 事件 → 原生对话框 | allow/deny/always（always 仅本会话生效，持久化由主程序写回 opencode.json） |

> Renderer 访问两个后端的方式：opencode 需 `--cors` 启动（或经 Main 代理）；sidecar（Fastify）配 CORS 或统一经 Main 的 IPC 代理，避免暴露凭据给 Renderer。

### 4.3 关键 UX 细节

- **审批对话框**必须给上下文：工具名 + 参数 diff（如 `sw_save_doc → D:\proj\part1.SLDPRT`）+ 操作前截图，一键 allow/deny
- **会话恢复**：SW 文档路径随 opencode session 持久化；主程序重启后自动 `/state` 对齐
- **首次启动向导**：检测 SW → 检测许可证 → 选工作目录（写入 opencode project）→ 模型 API key/登录
- **视觉闭环**（截图单源复用：agent / 视图面板 / 审批框三端共用同一截图流）：同一份 sidecar 截图回注给 agent（image part，**依赖待验证 API，见 [00 待验证假设](./00-OVERVIEW.md) #1**）、推送给视图面板（SSE）、展示于审批框——详见 [07-correctness.md](./07-correctness.md) 与 [05-interaction.md](./05-interaction.md)

---

## 5. opencode 层设计

### 5.1 扩展点使用清单

- **plugin**（`.opencode/plugins/cad.ts`，注意目录为复数 `plugins`）：用官方 `tool()` helper 注册 `sw_*` 工具（Zod `args` 定义参数）；`tool.execute.after` 把 sidecar 响应里的截图/state 以 image/text part 回注消息（agent 的视觉闭环，image part 注入为待验证项）
- **agent 文件**：`cad`（primary，vision 模型）、`cad-inspector`（只读测量 subagent）、`drafting`（工程图 subagent）
- **skills**：`sw-part-modeling` / `sw-sketch-discipline` / `sw-drawing-workflow` / `sw-export-guide`，按需注入
- **command**：`/model-part`、`/export-batch`、`/drawing-update`
- **permission + AGENTS.md**：单位纪律（mm/度）、状态行解读、建模前先 `sw_get_state`

区别于 v1：这些文件**由安装包写入主程序管理的项目目录**（如 `Documents/CADAgent/workspace/`），用户无感；高级用户仍可手改。

工具本身的定义与收敛设计见 [06-tool-set.md](./06-tool-set.md)；调用链细节见 [05-interaction.md](./05-interaction.md)。

### 5.2 主程序如何驱动 opencode（通道A）

- 通过官方 SDK/HTTP 创建会话、发送消息、订阅事件流（消息更新、工具调用、permission 请求）
- 权限应答：主程序收到 permission 事件 → 弹原生对话框 → 应答回传
- 多会话（P2）：每个会话绑定一个 sidecar session key；v1 单会话

> 具体 SDK 方法名以 opencode 官方文档/`@opencode-ai/sdk` 为准，实现时核对，不在此臆写。
