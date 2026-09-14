# 00 · 总览与导读

> 基于 opencode 的桌面端 SolidWorks CAD Agent —— 设计文档集 v2.5（2026-08）
> 本文件是文档集的入口：harness 核心叙事、全局目录、阅读路径、共享约束与待验证假设。

---

## 项目一句话

**以 harness 为核心的 CAD agent**：agent 用 **DIL**（设计意图语言，[02](./02-intent-language.md)）表达意图；**harness**（[01](./01-harness-model.md)）负责编译执行、观察验证、门控校准、失败恢复的闭环；opencode 提供 agent 决策层，sidecar 提供 SolidWorks 执行层；桌面主程序（CADAgent.exe，Electron）是用户唯一入口。CAD 后端可替换——SolidWorks 是第一个后端，不是架构的中心。

## 核心闭环（整个方案的组织原则）

```
 agent ──意图(DIL)──▶ 编译 ──▶ 执行 ──▶ 观察 ──▶ 门控 ──▶ 校准
   ▲                                                            │
   └──────────── state diff + diagnostics（DIL 词汇）◀──────────┘
```

三条公理：**校验谓词在 sidecar 求值，无需 LLM 参与**（防看图自我说服）；**状态是唯一权威，对话只是缓存**；**失败是一等信号**（诊断+状态+建议，驱动重规划）。

---

## 全局约束（所有文档默认遵守）

- 排除 MCP 方案，理由：① MCP stdio 服务器生命周期与 opencode 进程绑死，无法满足"关主程序不关 SolidWorks 会话"；② 面板需要独立的只读数据通道（通道C）与推送事件，MCP 请求-响应模型不提供；③ 长任务后台化 / SSE 进度 / 图片直传 / 串行队列治理均超出 MCP 工具语义，最终仍需一个私有服务进程——不如统一为一套私有 HTTP/SSE 协议（未来需要 MCP 客户端时可在 sidecar 前加薄适配层复用同一 kit）
- agent 运行时为 opencode（Bun）；桌面应用主程序基于 Electron
- SolidWorks 2024 / Windows 10/11 / COM 自动化（winax 桥）；**CAD 后端可替换**（DIL adapter，02 §7）
- 单位约定：所有 `sw_*`/DIL 入参出参统一 **mm / 度**，换算只发生在 sidecar ops 层
- 术语约定：**主程序**指 CADAgent.exe（Electron 桌面应用）；**sidecar** 指伴随的 Node 服务进程；**harness** 指闭环控制层（01）；**DIL** 指设计意图语言（02）；通道A/B/C 分别为会话通道/工具通道/面板数据通道（01 §2.2）

---

## 文档目录

### 第一部分 · 核心概念（先读）

| # | 文件 | 内容 | 主要读者 |
|---|---|---|---|
| 01 | [harness-model.md](./01-harness-model.md) | **闭环控制核心**：意图→执行→观察→门控→校准；L0–L3 API 谱系；状态快照；校验操作；repair/补偿；全文档集的 harness 视角映射 | 全员必读 |
| 02 | [intent-language.md](./02-intent-language.md) | **DIL 设计意图语言**：业界考察（无通用语言）、语法与语义规范、词汇表（18 动词/12 约束/9 原语）、符号表屏蔽命名差异、后端 adapter 与能力协商、执行语义 | 全员必读 |

### 第二部分 · 落地架构

| # | 文件 | 内容 | 主要读者 |
|---|---|---|---|
| 03 | [architecture.md](./03-architecture.md) | 四层拓扑、三通道、关键决策（Electron/截图镜像/进程归属）、主程序与 opencode 层设计 | 全员必读 |
| 04 | [sidecar.md](./04-sidecar.md) | sw-com-kit 设计：私有协议、串行队列、DIL 编译器与符号表、可靠性迁移清单 | sidecar/后端开发 |
| 05 | [interaction.md](./05-interaction.md) | 通道B详解：九步调用链、插件桥接代码、报文示例、并发语义 | 插件/opencode 层开发 |
| 06 | [tool-set.md](./06-tool-set.md) | 工具面 = DIL 执行表面：84→~19 收敛映射、schema 骨架、dil_apply/dil_state | agent/插件开发 |

### 第三部分 · 正确性与实现

| # | 文件 | 内容 | 主要读者 |
|---|---|---|---|
| 07 | [correctness.md](./07-correctness.md) | 时序正确性五层防御：队列/原子化/执行时校验+generation/闭环/补偿，残余风险 | sidecar/后端开发 |
| 08 | [com-impl.md](./08-com-impl.md) | Sidecar 操作 COM 参考实现：connection/queue/ops-feature/server 四文件 + 模式速查 | sidecar 开发 |

### 第四部分 · 工程化

| # | 文件 | 内容 | 主要读者 |
|---|---|---|---|
| 09 | [security.md](./09-security.md) | 网络面（凭据/localhost）、权限矩阵、审批 UX | 全员 |
| 10 | [packaging.md](./10-packaging.md) | 打包分发（winax ABI/NSIS）、P1–P3 路线、风险清单、fork 必要性结论、monorepo 布局、迁移对照表、术语表 | 架构/发布 |

### 第五部分 · 进阶

| # | 文件 | 内容 | 主要读者 |
|---|---|---|---|
| 11 | [fork-performance.md](./11-fork-performance.md) | 性能导向的深度改造：瓶颈分析、七个方向、薄 fork 策略与触发信号 | 性能/核心开发 |
| 12 | [multi-workpiece.md](./12-multi-workpiece.md) | 多工件并行：单实例多文档（文档亲和调度/协调者-worker）→ 多实例、Workbench 项目概念 | sidecar/agent/产品 |

---

## 阅读路径建议

- **理解核心思想**：00 → 01 → 02（约 25 分钟，读不懂闭环与 DIL，后面的实现细节没有意义）
- **写 sidecar**：01 → 04 → 07 → 08 →（多工件：12）
- **写 opencode 插件**：01 → 02 → 03 §opencode层 → 05 → 06
- **做主程序**：00 → 03（主程序章节为主）→（项目视图：12 §7）
- **评估可行性/立项**：00 → 01 → 03 → 10（风险与路线）
- **性能优化/深度改造**：00 → 11（先看 §5 决策表）

---

## 待验证假设（P0/P1 spike 清单）

| # | 假设 | 等级 | 失败时的 Plan B |
|---|---|---|---|
| 1 | `tool.execute.after` 钩子可向工具结果注入 image part（视觉闭环承重墙） | **P0** | 截图落盘 + 消息 parts 旁路注入；或验证工具结果原生支持 image content |
| 2 | opencode 在 Windows 原生稳定运行（官方推荐 WSL，但 WSL 无法访问 COM） | **P0** | 提前埋点验证；异常则评估 serve 模式部署在 Windows 原生 + 客户端分离 |
| 3 | winax 无法在 Bun 运行时加载 | P0（按最坏情况设计） | 已用 sidecar 规避；实测留档 |
| 4 | SolidWorks 状态变化检测可行：轮询 ActiveDoc 标题/FeatureCount 的开销与延迟可接受（generation 与 SSE 的地基） | **P0** | 降低轮询频率 + 仅在工具调用间隙校验 |
| 5 | 自定义工具的 permission pattern 粒度（能否按参数/路径匹配） | P1 | 目录白名单下沉 sidecar 强制校验（09 已按此设计） |
| 6 | DIL v0.1 词汇表对真实建模任务覆盖率 ≥95%（18 动词够用） | P1 | 用例集驱动补词升 v0.2；`native` 块兜底（12 §9） |

---

## 关联文档（本仓库之外）

- [../ARCHITECTURE.md](../ARCHITECTURE.md) —— 现有 solidworks-mcp-server 的源码级架构剖析（sidecar 迁移的资产来源）

---

## 变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v2.0 | 2026-08 | 产品形态明确为桌面应用；四层三通道架构 |
| v2.1 | 2026-08 | 新增 agent↔sidecar 交互、时序正确性五层防御、COM 参考实现；文档集拆分 |
| v2.2–v2.3 | 2026-08 | 新增多工件并行、状态 harness |
| v2.4 | 2026-08 | 评审修订：术语统一（主程序/通道/版本号/迁移）、权限矩阵与插件 API 对齐官方、参考实现修正、待验证假设清单 |
| v2.5 | 2026-08 | **以 harness 为核心重组**：01 提炼闭环模型（吸收原状态 harness 文档）、新增 02 DIL 设计意图语言（业界无通用意图语言，自建）、全部文档重新编号为五部分结构 |
