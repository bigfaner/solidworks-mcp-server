# 01 · Harness：CAD Agent 的闭环控制核心

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md) · 第一部分：核心概念（本篇是整个方案的组织中心）
> Harness 定义：**围绕 agent 的闭环控制层**——接收意图（DIL）、编译执行、观察验证、门控校准、失败恢复。其余所有架构组件（opencode、sidecar、通道、工具）都是 harness 某个环节的物理载体。
> 关联：[02-intent-language.md](./02-intent-language.md)（输入 IR）、[07-correctness.md](./07-correctness.md)（并发/时序正确性）、[06-tool-set.md](./06-tool-set.md)（执行表面）

---

## 1. 为什么以 harness 为核心

CAD agent 与代码 agent 的本质差异：**CAD 没有编译器和测试**——代码 agent 写完可以跑测试自证，CAD agent "写完"（发出建模指令）后必须有人替它"看结果、量尺寸、查干涉"。这个角色就是 harness。

三条设计公理：

1. **校验谓词在 sidecar 求值，无需 LLM 参与**——防止模型"看图自我说服"；LLM 只消费 pass/fail + 结构化 diagnostics（视觉提出假设，几何真值裁决）
2. **状态是唯一权威，对话只是缓存**——agent 对工件的认知永远可能过期，一切动作前以执行时校验为准
3. **失败是一等信号**——失败响应 = 诊断 + 当前状态 + 修复建议，驱动重规划而非盲目重试

由此，harness 是一个**闭环控制系统**，而非工具集合：

```
            ┌──────────────────────────────────────────────┐
            │                Harness 闭环                   │
            │                                              │
  意图 ──▶ 编译 ──▶ 执行 ──▶ 观察 ──▶ 门控 ──▶ 校准         │
 (DIL)   (adapter) (ops)   (L0-L3)  (gate)   (repair/升级/上报)
            ▲                                 │            │
            └────────── state diff ◀──────────┘            │
                （以 DIL 词汇回注 agent，触发下一轮）         │
            └──────────────────────────────────────────────┘
```

| 环节 | 职责 | 物理载体（文档） |
|---|---|---|
| 意图 | DIL 文档/片段 | [02-intent-language.md](./02-intent-language.md) |
| 编译 | DIL→后端操作序列；符号表（屏蔽原生命名） | sidecar ops 层 + adapter（[04](./04-sidecar.md)、[08](./08-com-impl.md)） |
| 执行 | 串行队列 + 复合操作（时序内封） | [07-correctness.md](./07-correctness.md) 层1/层2 |
| 观察 | L0–L3 状态谱系 + 快照 | 本文 §2–§3 |
| 门控 | assert/inspect/accept + 自动轻量 gate | 本文 §3–§4 |
| 校准 | repair 剧本 / 升级上报 / 补偿工具 | 本文 §4 + [07](./07-correctness.md) 层5 |
| 回注 | DIL 词汇 state diff + 截图 | [05-interaction.md](./05-interaction.md) |

---

## 2. SolidWorks 状态与校验 API 谱系（观察能力）

### L0 · 观察（免费~毫秒级，状态快照的主体）

| API | 返回 | 用途 |
|---|---|---|
| `IModelDoc2.GetType/GetPathName/GetTitle` | 文档身份 | 会话状态机（已在用） |
| 特征树遍历：`FirstFeature→GetNextFeature` / `FeatureByPositionReverse` + `IFeature.Name/GetTypeName2` | 全部特征及类型 | 特征清单、结构校验（已在用） |
| `IFeature.GetSuppressed2` 等抑制状态 | bool | 特征是否被抑制 |
| `EditRebuild3()` 返回值 | bool | **最便宜的全局合法性检查**——重建失败=模型有损坏特征 |
| `CheckFeatureUse(usage, openCnt, closedCnt)` | 轮廓计数 | 草图能否用于拉伸/旋转（DIL `closed` 约束的编译目标） |
| Open/Save 的 errors/warnings out 参数 + `swFileLoadError_e`/`swFileSaveError_e` 枚举 | 错误码 | 文档级异常（已在用） |

### L1 · 测量（10–100ms，数值真值）

| API | 返回 |
|---|---|
| `IModelDocExtension.CreateMassProperty` → `IMassProperty` | 质量/体积/表面积/质心/惯性矩/密度 |
| `IModelDocExtension.CreateMeasure` → `IMeasure` | 两实体间距离/角度 |
| `IPartDoc.GetBodies2` + `IBody2.GetBodyBox` | 实体数、包围盒 |
| `IModelDocExtension.CustomPropertyManager` | 自定义属性（材质等） |

### L2 · 检查（100ms~秒级，几何合法性审计）

| API | 检查内容 |
|---|---|
| `IAssemblyDoc.InterferenceDetection`（InterferenceDetectionManager） | 装配干涉：体积/零件对清单 |
| `IMassProperty.CheckInterference` | 零件级多实体干涉 |
| `IBody2.CheckBodySelfInterference` | 实体自相交（坏几何） |
| `IPartDoc.Check3dPrintable`（SW2019+） | 流形/水密性——兼做通用几何健康检查。注意：可能弹进度 UI（Silent 处理）；大零件秒级耗时（转后台任务覆盖） |
| Draft Analysis | 拔模角分析。**需核实是否存在公开 COM API**；若无公开 API，降级为逐面拔模角计算（`IFace2` + 参考向量） |

### L3 · 派生视图（视觉互补）

剖视图、临时三视图——作为"验证表示"而非状态源（视觉补偿策略见 [03](./03-architecture.md) §4.3 与 [05](./05-interaction.md) §4；[11](./11-fork-performance.md) 图像 GC 的降档素材）。

> **置信度标注**：L0/L1 条目均已在现有仓库出现或属最核心 API，置信度高；L2 的 `CheckBodySelfInterference`/`Check3dPrintable` 具体签名**实现前以 SolidWorks API Help 核实**；Draft Analysis 存在性存疑（见降级方案）。草图"完全定义"状态无直接属性，按启发式近似处理（`fullyDefined: 0.9`）。

---

## 3. 状态快照与校验操作

### 3.1 状态快照（`sw_get_state` / `dil_state` 的完整 schema）

```jsonc
{
  "generation": 43,   // ← 绑定文档的 docGen（多文档语义见 12 §4.2）
  "doc": { "id": "D3", "path": "...", "type": "Part", "config": "Default" },
  "dil": {                        // DIL 视图（12 文档 §6.2 符号表）
    "applied_ids": ["f_plate", "f_holes"],
    "skipped_ids": [], "failed_ids": [],
    "params": { "L": 50, "W": 30 }
  },
  "rebuild": { "ok": true, "ms": 120 },
  "features": [
    { "dilId": "f_plate", "native": "拉伸1", "type": "Extrusion", "suppressed": false, "status": "ok" }
  ],
  "sketches": [
    { "dilId": "s_base", "native": "草图1", "contours": { "open": 0, "closed": 1 }, "fullyDefined": 0.9 }
  ],
  "bodies": { "count": 1, "volume_mm3": 9000, "mass_kg": 0.0702, "bbox": [50,30,6] },
  "diagnostics": [   // ← harness 的核心：分级诊断（主键为 DIL id）
    { "severity": "error", "code": "OPEN_CONTOUR", "source": "s_holes", "hint": "补线或闭合约束" },
    { "severity": "warn",  "code": "UNDER_DEFINED", "source": "s_base", "hint": "欠定义，可能漂移" }
  ]
}
```

DIL 的 `verify.expr` 表达式上下文即本快照字段 + `params`。

### 3.2 校验操作（工具面）

| 操作 | 工具 | 成本 | 语义 |
|---|---|---|---|
| observe | `sw_get_state` / `dil_state` | 低 | 全量快照 + diagnostics（DIL id 视图） |
| measure | `sw_measure` | 低 | 精确数值（质量/距离/包围盒） |
| **inspect** | `sw_validate` | 高，按需 | 深度审计：干涉/自相交/水密/拔模——转后台任务，返回结构化报告 |
| **gate** | `sw_assert` | 低 | **断言求值**：sidecar 内判定 pass/fail |

### 3.3 `sw_assert` 语义

```jsonc
// 请求
{ "assert": [
    { "path": "bodies.mass_kg", "lt": 0.1 },
    { "path": "diagnostics", "no_error": true },
    { "dilId": "f_holes", "exists": true }
] }
// 响应：{ "pass": false, "failed": [{ "path": "bodies.mass_kg", "actual": 0.121 }] }
```

- 确定性布尔输出；generation 过期时同样拒收（409 stale_state，[07](./07-correctness.md) §3）
- **求值数据源约定**：L0 字段基于快照即可；**L1 数值是缓存值**——断言含 L1 路径时隐式触发轻量刷新（重建+重算），刷新 bump 该文档 gen
- DIL 文档中的 `verify` 块在 `dil_apply` 时自动编译为 assert/inspect 序列——准则与意图同文档演进

### 3.4 门控校准工作流（状态机）

```
plan → act → observe(L0) → gate(关键断言)
                              ├─ pass → 下一步
                              ├─ fail+可修复 → repair 剧本（skill）→ 回到 act
                              └─ fail+不可修复 → 上报人类（带 inspect 报告）
里程碑处（零件完成/装配前）→ inspect(L2) 全面审计
```

- 门控纪律写进 agent prompt 与 skills："每个特征后 gate，每个里程碑前 inspect"
- `tool.execute.after` 钩子在每个特征类工具后**自动追加轻量 gate**（rebuild ok + 无 error 级 diagnostics）——**复用复合操作 finalize 已有的"重建 + 结果查证"产物，零增量 COM 成本**
- diagnostics 的 `code`/`hint` 复用 [07](./07-correctness.md) 层 4 失败语义——观察、断言、修复共享同一套词汇表（且以 DIL id 为主键）

---

## 4. 校准：repair、补偿与升级

| 机制 | 工具/手段 | 场景 |
|---|---|---|
| repair 剧本 | skills（`sw-verification-discipline`）：按诊断码索引的修复套路 | OPEN_CONTOUR → 补线/闭合约束；UNDER_DEFINED → 补尺寸 |
| 补偿工具 | `sw_undo`（EditUndo2）/ `sw_delete_feature`（按 DIL id 删） | 建错特征回退 |
| 重同步 | `sw_get_state` / `dil_state` | LLM 迷失自愈；generation 失配后强制刷新 |
| 升级上报 | inspect 报告 + 失败前截图 → 人 | 不可自动修复 |

---

## 5. 与架构组件的映射（harness 视角看全文档集）

| 组件 | 在闭环中的角色 | 文档 |
|---|---|---|
| DIL 语言 | 意图载体 + 回注词汇 | 02 |
| opencode（plugin/agent/skills） | 意图生成器 + 校准决策者 | 03/05/06 |
| sidecar | 编译器 + 执行器 + 观察者 + 门控求值器 | 04/08 |
| 串行队列/复合操作 | 执行环节的时序与原子性保证 | 07 |
| generation | 观察结果与动作之间的因果链 | 07 §3 |
| 视觉闭环（截图） | 观察环节的补充通道（提出假设） | 03 §4.3/05 |
| 多工件（per-doc state） | 观察与门控的按文档隔离 | 12 |
| 桌面主程序面板 | 观察结果的人机呈现（通道C） | 03 §4 |

---

## 6. 实施切分

| 阶段 | 内容 |
|---|---|
| P1.5 | `sw_get_state` 扩展 diagnostics（OPEN_CONTOUR / REBUILD_FAIL / UNDER_DEFINED 三码起步）+ 自动轻量 gate 钩子 + DIL 符号表雏形（applied_ids 跟踪） |
| P2 | `sw_assert`（path 断言 + generation 联动）+ `sw_validate` 后台任务化（干涉+自相交+水密三查）+ repair 技能剧本 + `dil_apply`/`dil_state` 工具 |
| P3 | 特征级 status 细化（抑制/悬空引用）、拔模分析（无公开 API 则降级实现）、诊断面板（主程序）；DIL v1.0 跨后端验证（12 §9） |
