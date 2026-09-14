# 06 · 工具面：DIL 的执行表面（84 → ~19 收敛）

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 收敛（本文用法）＝将大量细粒度工具整合为少量意图级工具
> 关联：[02-intent-language.md](./02-intent-language.md)（工具调用在插件层等价编译为 DIL 片段）、[05-interaction.md](./05-interaction.md)（工具如何被执行）、[07-correctness.md](./07-correctness.md)（expectedGen 与补偿工具的语义）、[01-harness-model.md](./01-harness-model.md)（校验工具）

---

## 1. 为什么要收敛

现有仓库的 84 个扁平工具直接挂给 LLM 存在三个问题：

- schema 总量占满上下文窗口
- 模型选择准确率显著下降（近似工具太多）
- 程序性知识被迫写进 description，随 schema 常驻

---

## 2. 收敛原则

1. **意图级合并**：一个工具覆盖一个建模意图，参数用判别联合（discriminated union）
2. **时序内封**：多步 COM 时序（退出草图→选草图→验证→拉伸）是代码的事，不是模型的事——LLM 编排意图，代码编排时序
3. **知识下沉到 skills**：何时选哪种终止条件、草图约束顺序——写成 SKILL.md 按需注入，而非常驻 schema

---

## 3. 工具清单（核心 15 个 + 扩展 4 个）

### 3.1 核心清单（P1–P2）

| 工具 | 覆盖 | 要点 |
|---|---|---|
| `sw_get_state` | 状态/上下文 | 文档/草图/特征树摘要/单位/队列；含 `generation` 版本 |
| `sw_new_doc` / `sw_open_doc` / `sw_save_doc` / `sw_close_doc` | 文档（4 个） | close 一律 ask |
| `sw_sketch` | enter/edit/exit | 内封中英文基准面兼容 |
| `sw_sketch_entity` | 9 种几何（line/circle/arc/rectangle/polygon/spline/ellipse/centerline/point） | type 判别联合，mm 坐标 |
| `sw_sketch_mod` | 约束/尺寸/阵列/镜像/偏移（7 个原工具） | action 判别；**带 `sw_` 前缀**（权限通配与插件 `startsWith("sw_")` 判断需要） |
| `sw_feature` | 凸台/切除/旋转/薄壁 | 终止条件枚举；13+ 参数自动 VBA 回退；支持 `expectedGen` |
| `sw_feature_mod` | 圆角/倒角/抽壳/拔模/阵列/镜像特征 | |
| `sw_reference` / `sw_assembly` / `sw_drawing` | 参考几何/装配/工程图 | |
| `sw_measure` | 纯测量：质量属性/距离/包围盒（3 个原工具） | inspector 专用；**检查类（干涉/拔模分析/几何检查）归 `sw_validate`（01 文档裁决）** |
| `sw_export` | export_file/batch_export/with_options/screenshot（4 个原工具） | 转后台任务（job） |
| `sw_undo` / `sw_delete_feature` | 补偿（见 01 文档 §5） | delete 可 ask |
| `sw_macro` / `sw_job` | 宏兜底通道/长任务轮询 | macro 一律 ask |

### 3.2 扩展清单（+4，见对应文档）

| 工具 | 来源 | 说明 |
|---|---|---|
| `sw_list_docs` / `sw_bind_session` | [12](./12-multi-workpiece.md) §4.1/§5.2 | 多工件：文档枚举与 worker 会话绑定 |
| `sw_validate` / `sw_assert` | [01](./01-harness-model.md) §3.2 | 状态 harness：深度审计 / 断言求值（均 allow，只读校验） |

### 3.3 原 84 工具映射（分项之和 = 84）

| 原模块（工具数，已对照源码核实） | 去向 |
|---|---|
| sketch.ts（20，其中 extrude/extrude_cut 2 个） | 18 个 → sw_sketch / sw_sketch_entity / sw_sketch_mod；2 个 → sw_feature |
| drawing.ts（8） + template-manager.ts（6） | sw_drawing |
| export.ts（4） | sw_export |
| analysis.ts（6） | 3 个测量类 → sw_measure；3 个检查类 → sw_validate |
| vba/*（26） | sw_macro 承接执行类；各域生成器由对应域工具吸收 |
| native-macro.ts（8） + macro-recording.ts（3） + macro-security.ts（2） + diagnostics.ts（1）＝14 | sw_macro |
| 新增（不属于原 84） | sw_get_state / sw_undo / sw_delete_feature / sw_job / sw_validate / sw_assert / sw_list_docs / sw_bind_session |

---

## 4. 例：`sw_feature` 的 schema 骨架

```typescript
{
  type: z.enum(['boss', 'cut', 'revolve', 'revolve_cut', 'thin']),
  depth: z.number().describe('mm'),
  endCondition: z.enum(['Blind','ThroughAll','MidPlane','UpToNext',...]).default('Blind'),
  draft: z.number().optional(),          // 度
  bothDirections: z.boolean().optional(),
  thinThickness: z.number().optional(),  // thin 类型时必填
  expectedGen: z.number().optional(),    // 乐观并发版本，由上一响应的 state 获得（01 §3）；
                                         // 多文档下升级为 (docId, expectedGen) 二元组（12 §4.2）
  // 写类工具应总是携带 expectedGen；读类工具可省略
  // 内部：13+ 参数 → 自动拼 VBA 宏 → RunMacro2→RunMacro 回退
}
```

模型只说"我要一个 50mm 盲拉伸凸台"，退出草图/选草图/轮廓（REGION）选择回退/闭合验证/五级 API 回退全部发生在 sidecar（参考实现见 [08-com-impl.md](./08-com-impl.md)）。

---

## 5. DIL 绑定：工具面与意图语言的统一

工具面是 DIL 的**执行表面**（[02](./02-intent-language.md) §8）：

- **每次 `sw_feature`/`sw_sketch_entity` 等调用，插件层等价编译为一个 DIL 片段**（如 `sw_feature({type:'boss', depth:50})` → `features: [{id:<auto>, op: extrude, profile:<最近草图>, depth:50}]`）——工具调用与 DIL 文档是同一中间表示的两种糖
- `sw_*` 快捷工具保留：交互对话中比写 YAML 片段更省 token；DIL 片段用于复杂/可复用/需参数化重放的意图

新增两个 DIL 原生工具（合计 ~21）：

| 工具 | 语义 | 成本 |
|---|---|---|
| `dil_apply` | 提交 DIL 片段或整文档：harness 编译→执行→按 verify 自动门控→返回 applied_ids/诊断 | 按片段内容 |
| `dil_state` | DIL id 满足视图（applied/skipped/failed + params 当前值） | 低 |

两个工具均 allow（dil_apply 落盘语义由片段内容决定；含 `native` 块时升 ask）。

---

## 6. 权限级别速记

完整权限矩阵见 [09-security.md](./09-security.md)：

- **allow**：读/测量/校验/建模（可撤销：SW 特征树）——`sw_get_state` `dil_state` `sw_measure` `sw_validate` `sw_assert` `sw_job` `sw_new_doc` `sw_open_doc` `sw_sketch`* `sw_feature`* `sw_reference` `sw_assembly` `sw_drawing` `sw_undo` `sw_list_docs` `sw_bind_session`
- **ask**：落盘/宏/删除/关闭——`sw_save_doc` `sw_export` `sw_macro` `sw_delete_feature` `sw_close_doc`；`dil_apply` 按内容定（含 `native` 块或落盘时 ask）
