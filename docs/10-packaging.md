# 08 · 打包分发、实施路线与风险管理

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 覆盖原 v2.1 文档 §12–§16：打包、P1–P3 路线、风险清单、fork 必要性、monorepo 布局、迁移对照表、术语表

---

## 1. 打包与分发

### 1.1 安装包内容（electron-builder，NSIS）

```
CADAgent-Setup-x.y.z.exe
└── resources/
    ├── app.asar                    # Electron 应用包
    ├── sidecar/
    │   ├── node.exe                # 锁定版本的标准 Node LTS（与 winax ABI 匹配）
    │   ├── dist/                   # sidecar 编译产物
    │   ├── node_modules/           # 含 winax 预编译 .node（CI 按 node.exe 版本构建）
    │   └── examples/vba-templates/
    ├── opencode/
    │   └── opencode.exe + 依赖     # 锁定版本的 headless opencode
    ├── vc-redist/                  # MSVC 运行库（winax node-gyp 产物依赖，干净机器未必有）
    └── workspace-template/         # opencode.json + AGENTS.md + .opencode/{plugins,agents,skills,commands}
```

> 目录名注意：opencode 实际加载 `.opencode/plugins/`、`.opencode/agents/`、`.opencode/commands/`（均为**复数**）；`opencode.json` 位于项目根目录，不在 `.opencode/` 内。

### 1.2 关键工程点

- **winax ABI**：CI 里用与打包的 node.exe 完全一致的版本编译 winax（node-gyp），预构建放 `node_modules`——用户机**零编译**
- **VC++ 运行库**：NSIS 捆绑 VC++ redist 安装器（或静态链接），首启检测
- **SolidWorks 宏信任目录**：VBA 回退把 .swp 写入临时目录再 RunMacro2，SW 宏安全会拒绝未信任位置——首次启动向导必须注册信任目录（或统一写入工作区 Macros 目录），否则拉伸的五级回退最终兜底失效
- **端口策略**：7654 被占用时改用随机端口并经 Main 分发（P1 即需，不等单实例锁）
- **杀软误报**：spawn node.exe + COM 自动化 + 临时写 .swp 是典型误报组合；签名前 SmartScreen/Defender 拦截需在文档中给用户预期与放行指引
- **首次启动**：向导做 SW 探测（注册表 ProgID/版本）、工作目录选择（复制 workspace-template）、宏信任目录注册、模型供应商配置（API key 或代理登录）
- **升级**：NSIS 全量替换；用户 workspace 与安装目录分离，升级不丢配置；sidecar 协议带 `protocol` 版本字段（请求头 `X-Protocol-Version`），主程序与 sidecar 版本强绑定同发布
- **目标环境**：Windows 10/11 + SolidWorks 2024（许可由用户提供，安装包不捆绑）

---

## 2. 分阶段实施路线

### P1（2–3 周）：端到端桌面 MVP

- **P0 spike 先行**（见 [00 待验证假设](./00-OVERVIEW.md)）：image part 注入、opencode Windows 原生稳定性、winax/Bun、SW 变化检测轮询
- sidecar：Fastify + Bearer 鉴权 + 串行队列 + 会话状态 + kit 迁移 + 6 个复合操作（new_doc/sketch/sketch_entity/feature/save_doc/state 端点）
- 主程序：Electron 骨架，Main 启动/监控两个子进程；对话面板（接 opencode 事件流）+ 视图镜像 + 审批对话框
- opencode：plugins/cad.ts（6 工具 + 截图回注）+ cad.md（vision 模型）+ 最小 AGENTS.md
- **验收**：安装包在干净 Win11+SW2024 机器上：对话输入"建一个 50×30×10 长方体"→ 面板看到视图变化与特征树更新 → 保存时弹审批框

### P2（4–6 周）：产品化

- 工具补全到核心 15 + 扩展（含补偿工具、校验工具）；generation 版本校验；长任务转后台任务 + 任务面板；特征树双向（人点选→sidecar select）
- skills 四件套、cad-inspector/drafting subagent、command 入口
- 截图历史时间线；单实例/托盘/会话恢复；日志聚合
- 安装包签名 + 自动更新通道

### P2.5（并入 P2 尾段或紧随其后）：多文档增量（[12](./12-multi-workpiece.md) §8）

- docId 寻址 + per-doc generation + 文档亲和调度 + sw_list_docs/sw_bind_session + 协调者-worker

### P3（按需）：进阶

- 多会话/多 SW 实例（选项 B）；局域网 worker 模式（SW 在工作站）
- 会话录制回放（审计/教学，复用现有仓库 macro/recorder 思路）
- 装配引导、参数化模板库、批处理工厂
- 状态 harness 的深化项（[01](./01-harness-model.md) §5 P3：特征级诊断细化、拔模分析、诊断面板）

---

## 3. 风险清单

| 风险 | 影响 | 缓解 | 详见 |
|---|---|---|---|
| SW 变化检测机制不可行/开销过大 | generation 与 SSE 失效（五层防御的两层悬空） | P0 spike：轮询字段与间隔验证；降级为仅工具调用间隙校验 | [07](./07-correctness.md) §3.2 |
| opencode Windows 原生运行不稳定 | 方案根基动摇（WSL 无法访问 COM） | P0 spike：干净 Win 环境验证 serve 模式；锁版本；问题上馈 | [03](./03-architecture.md) §3.3 |
| winax/Node ABI 不匹配 | sidecar 起不来 | CI 锁版本双端一致构建；启动自检失败即引导上报 | 本文 §1.2 |
| SW 宏信任目录未注册 | VBA 回退最终兜底失效 | 首次启动向导注册信任目录 | 本文 §1.2 |
| SW 模态框挂死 sidecar | 工具超时 | Silent 导出、VBA 去 MsgBox（改日志文件）；看门狗检测调用超时→提示用户处理前台 SW | [04](./04-sidecar.md) |
| opencode SDK/事件接口变更 | 主程序编译失败 | 锁定 opencode 版本打包；升级走回归测试 | [03](./03-architecture.md) |
| LLM 并发调用 COM | 队列堆积/状态错乱 | 串行队列 + 队列深度上报 + agent prompt 约束（AGENTS.md） | [07](./07-correctness.md) §1 |
| 人机操作竞态（用户直接动 SW） | agent 基于过期状态操作 | generation 过期版本拒收 + SSE doc_switched 对齐 + 残余窗口兜底 | [07](./07-correctness.md) §3/§6 |
| 序列中间态被打断 | 后续操作失效 | 复合操作原子化（内封进单队列槽） | [07](./07-correctness.md) §2 |
| 截图流性能（大装配） | UI 卡 | 截图降采样、事件触发而非轮询、面板不可见时暂停 SSE | [03](./03-architecture.md) §4.2 |
| SW 语言/版本差异 | 命名兼容问题 | 迁移现有仓库的中英文兼容与启发式选择；启动探测语言并告警 | [08](./08-com-impl.md) |
| 用户绕过主程序直接操作 SW | 状态漂移 | SSE 事件对齐 + generation 版本校验 | [07](./07-correctness.md) §3 |

---

## 4. fork opencode 的必要性重估

桌面形态下，fork 动机进一步收缩：

| 动机 | v1 判断 | v2 判断 |
|---|---|---|
| 做自定义 UI/面板 | 需 fork TUI | **消失**：自建客户端即主程序 |
| 改 agent loop（如每特征强制 checkpoint） | 需 fork | 仍需 fork；但可先用 plugin 钩子（tool.execute.after 校验）逼近，最后才 fork |
| 其他一切 | 插件层 | 插件层 |

**结论：本形态下默认永不 fork。** 主程序本身就是"完全自定义的 opencode 客户端"。

---

## 5. 仓库布局（monorepo）与迁移对照

### 5.1 布局（pnpm workspace）

```
cad-agent/                        # pnpm workspace
├── apps/
│   └── desktop/                  # Electron 桌面应用（main + renderer/React）
├── packages/
│   ├── sw-com-kit/               # sidecar（含迁移的 kit/）
│   ├── sw-protocol/              # 三通道共享类型（op 请求/响应/事件/SSE payload）
│   └── opencode-workspace/       # opencode.json + AGENTS.md + .opencode/{plugins,agents,skills,commands} 模板
├── scripts/
│   ├── build-winax.ps1           # CI：锁定 Node 版本编译 winax
│   └── package.ts                # 组装安装包资源
└── docs/                         # 本设计文档集
```

### 5.2 现有仓库迁移对照表

| 源（solidworks-mcp-server） | 目标 | 处置 |
|---|---|---|
| src/solidworks/{operations,helpers,types} | packages/sw-com-kit/src/kit/ | 原样迁移（extrusion.ts/selection.ts 为核心资产） |
| src/utils/{com-boolean,com-lifecycle,error-recovery,feature-utils} | 同上 utils/ | 原样迁移 |
| src/utils/{logger,config,environment,solidworks-config} | 同上 utils/ | **重写**（winston→sidecar 日志方案；config 并入 sw-protocol） |
| src/adapters/{types,macro-generator} | 同上 adapters/ | 原样迁移（VBA 回退） |
| src/shared/constants | 同上 constants/ | 原样迁移 |
| examples/vba-templates | sw-com-kit/examples/ | 原样迁移 |
| src/macro/ | P3 评估 | 会话录制回放 |
| src/index.ts、src/tools/、src/resources/、src/prompts/ | — | 丢弃（opencode 扩展点替代） |

---

## 6. 术语表（方案核心概念）

| 术语 | 含义 |
|---|---|
| **主程序** | CADAgent.exe，Electron 桌面应用：用户唯一入口，编排 opencode 与 sidecar 两个子进程 |
| **Sidecar（伴随进程）** | 独立辅助进程模式：主程序旁边挂一个独立运行时进程，IPC 通信、崩溃相互隔离、可单独重启。本项目 = sw-com-kit sidecar（Node 20 + Fastify + winax） |
| **通道A/B/C** | 主程序↔opencode（会话通道）/ plugin↔sidecar（工具通道）/ 主程序↔sidecar（面板数据通道）三条职责分离的通信通道 |
| **复合操作（op）** | 内封了完整 COM 时序的意图级原子单元（如 feature 内封退出草图→选草图→校验→五级回退），在队列锁内一口气执行 |
| **generation（状态版本号）** | sidecar 维护的状态版本号，SW 状态每变一次 +1；工具调用携带 expectedGen，不符即拒收（乐观并发） |
| **stale_state** | 版本不符的错误类别：调用基于过期观察，拒收并要求重新 sw_get_state |
| **结果查证** | 不信 API/宏退出码，执行后查证效果（COM 路径 `FeatureByName`；VBA 路径特征数差分） |
| **五级回退** | FeatureExtrusion3 → 2 → 1 → WithFeatureData → VBA 宏，应对 winax 13+ 参数限制 |
| **截图单源复用** | 同一份 sidecar 截图提供给 agent（image part）、推送给视图面板（SSE）、展示于审批框 |
| **补偿工具** | sw_undo / sw_delete_feature / sw_get_state，失败后的修复手段（无真事务） |
| **交互优先通道** | `/select` 端点：人在特征树面板的点击先于 agent 队列执行 |
| **看门狗** | sidecar 调用超时检测，提示用户处理前台 SW（模态框挂死场景） |
| **懒重连** | 每次请求前 isAlive() 探测 COM 连接，SW 重启后自动恢复 |
| **执行时校验** | 复合操作执行时刻对着真实 COM 状态重新验证前置条件（01 §3） |
| **Workbench（项目）** | 多工件工件集 + 派发关系的持久化概念（12 §7） |
| **文档亲和调度** | 多文档队列调度：连续同文档操作聚批执行，避免激活抖动（12 §4.3） |
| **薄 fork** | 改动 <3000 行、全特性挂 flag 的 opencode fork 维护策略（01 §4） |
| **状态 Harness** | 以 sidecar 求值的校验体系：observe/measure/inspect/gate（11） |
