# 02 · DIL：设计意图语言（Design Intent Language）

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md) · 第一部分：核心概念
> 状态：v0.1 规范草案（P1 用例集验证词汇覆盖度后升 v0.2）
> 关联：[01-harness-model.md](./01-harness-model.md)（DIL 是 harness 的输入/输出中间表示）、[06-tool-set.md](./06-tool-set.md)（工具面 = DIL 的执行表面）

---

## 1. 业界考察：存在通用的设计意图语言吗？

**结论：不存在被跨厂商采纳的中性设计意图语言。** 这是行业公认的能力缺口，也是 DIL 的立项依据。

### 1.1 现有格式/语言盘点

| 类别 | 代表 | 表达设计意图？ | 中性/通用？ |
|---|---|---|---|
| 几何交换标准 | STEP（ISO 10303，AP203/214/242） | ❌ 交换的是 B-rep 结果几何 + PMI 标注；ISO 的参数与约束扩展模块（10303-108/-111 系）在商业 CAD 中几乎没有实现 | 中性但**无意图** |
| 内核私有格式 | Parasolid、ACIS(SAT)、OCCT(BRep) | ❌ 特征历史与参数随内核/版本绑定，不跨厂商 | ❌ |
| 厂商意图语言 | PTC Creo **Pro/PROGRAM**（文本特征清单）、CATIA **EKL**（Knowledgeware 规则）、Siemens NX **Knowledge Fusion**（规则式 KBE，源自 ICAD/Intent 一脉）、Autodesk Inventor **ETO/Intent**（已停维护） | ✅ 规则/参数/特征意图 | ❌ 全部绑定宿主 |
| 脚本式 CAD DSL | **OpenSCAD**（CSG）、**CadQuery / build123d**（Python + OCCT）、FreeCAD Python 脚本 | 部分：表达操作序列，非工程语义（无约束/PMI/验证） | ❌ 绑定范式或内核 |
| 可视化数据流 | Grasshopper（Rhino）、Dynamo DesignScript（Revit） | 部分 | ❌ 绑定宿主且非文本 |
| 研究本体 | Gero FBS（Function-Behavior-Structure）等 | 概念层有意图建模，无可执行落地产物 | 学术 |

近年 text-to-CAD 研究普遍借用 OpenSCAD/Python DSL 作为生成目标——进一步印证"缺一个可执行的意图层中间表示"。

### 1.2 DIL 填的空位

```
      设计意图（参数、约束、特征语义、验证准则）
        ↑ 谁来表达？ —— 没有通用者 → DIL
      几何结果（B-rep、PMI）
        ↑ STEP 已解决
```

**DIL 不是又一个几何格式，而是"可执行的意图中间表示（IR）"**：agent 读写的统一格式，harness 的编译输入，CAD 后端（SolidWorks/FreeCAD/Onshape/…）可替换。

---

## 2. 设计目标与原则

| # | 原则 | 落点 |
|---|---|---|
| 1 | **意图先于操作** | 每个 feature 必须带 `intent`（人话描述）；op 只是实现手段 |
| 2 | **逻辑引用，屏蔽命名差异** | 一切实体用 DIL 逻辑 id 引用；"草图1/Sketch3"等原生名被封在 harness 符号表内（§6.2） |
| 3 | **单位显式** | 文档级默认单位 + 量级可带单位；永不假设内核单位 |
| 4 | **验证内嵌** | `verify` 块是一等公民，直接映射 harness 门控（observe/gate/inspect） |
| 5 | **可增量** | 文档可整编可片段（fragment）；按 id 跟踪满足状态，支持断点续跑 |
| 6 | **后端无关词汇 + 能力协商** | ~20 个意图动词封闭词汇表；adapter 声明能力矩阵，不可表达时语义降级而非失败 |
| 7 | **对 agent 友好** | YAML 序列化（token 高效、可 diff、可注释）；小封闭词汇；失败语义显式 |

**不是什么**：不是编程语言（无循环；条件仅 `when` 守卫；重复用 pattern 类动词表达）；不是几何交换格式；不追求覆盖各后端全部专有功能（逃生口：`native` 扩展块，标注后端绑定，不承诺可移植）。

---

## 3. 语法总览

一个 DIL 文档（`.dil.yaml`）由七个块组成，全部可选除 `dil_version`：

```yaml
dil_version: 0.1
unit: mm                      # 文档默认单位
part: <名称>                  # 或 assembly / drawing
intent_desc: <一句话功能描述>

params:        { ... }        # 参数与表达式
references:    [ ... ]        # 参考几何（平面/轴/点）
sketches:      [ ... ]        # 草图（几何原语 + 约束）
features:      [ ... ]        # 特征程序（有序）
verify:        [ ... ]        # 验证准则（assert / inspect / accept）
provenance:    { ... }        # harness 回填：后端、已应用 id、generation
```

---

## 4. 完整示例（同时也是规范基准用例）

```yaml
dil_version: 0.1
unit: mm
part: mounting_bracket
intent_desc: 矩形安装板，四角螺栓孔贯通，垂直边倒圆去毛刺

params:
  L:       { value: 50 }
  W:       { value: 30 }
  T:       { value: 6,  min: 3, max: 12, desc: 板厚 }
  hole_d:  { value: 5.5, desc: M6 螺栓过孔 }
  edge:    { value: 8,  desc: 孔中心到板边距离 }
  fr:      { value: 3,  max: "min(params.T, 5)", desc: 倒圆半径 }

references:
  - { id: top_plane, kind: plane, def: { canonical: XY } }

sketches:
  - id: s_base
    intent: 底板外形
    plane: top_plane
    geometry:
      - { prim: rectangle, p1: [0, 0], p2: "[params.L, params.W]" }
    constraints:
      - { type: closed }
  - id: s_holes
    intent: 四角螺栓孔
    plane: top_plane
    geometry:
      - { prim: circle, center: "[params.edge, params.edge]",                d: params.hole_d }
      - { prim: circle, center: "[params.L - params.edge, params.edge]",     d: params.hole_d }
      - { prim: circle, center: "[params.edge, params.W - params.edge]",     d: params.hole_d }
      - { prim: circle, center: "[params.L - params.edge, params.W - params.edge]", d: params.hole_d }
    constraints:
      - { type: equal, targets: ["#0", "#1", "#2", "#3"] }    # 四孔等径

features:
  - id: f_plate
    intent: 底板基体
    op: extrude
    profile: s_base
    depth: params.T
    on_fail: abort
  - id: f_holes
    intent: 螺栓过孔
    op: cut_extrude
    profile: s_holes
    depth: { thru: true }
    on_fail: ask
  - id: f_fillet
    intent: 垂直边倒圆去毛刺
    op: fillet
    target: "@f_plate.edges.vertical"
    radius: params.fr
    when: "params.fr > 0"
    on_fail: skip

verify:
  - { id: v_mass,  kind: assert,  expr: "mass_kg < 0.1" }
  - { id: v_holes, kind: assert,  expr: "count_features(op='cut_extrude', thru=true) == 1" }
  - { id: v_geom,  kind: inspect, checks: [self_intersect, interference] }

provenance:                     # harness 自动回填，agent 不写
  backend: solidworks-2024
  applied_ids: [f_plate]
  generation: 43
```

---

## 5. 各块语义规范

### 5.1 params

```yaml
params:
  <name>: { value: <num|expr>, min?: <num|expr>, max?: <num|expr>, unit?: <str>, desc?: <str> }
```

- **表达式语法**：四则运算、`min/max/abs/floor/ceil/sqrt`、`params.*` 引用、比较与逻辑（`when`/`verify` 中使用）。表达式一律为字符串形式（YAML 类型安全）
- 参数图必须无环（DAG）；harness 编译期做拓扑求值与 min/max 越界诊断（越界 → warn 级 diagnostic，不阻塞）
- 修改 `params` 再重放 = 参数化驱动的模型变体（配合 [12](./12-multi-workpiece.md) 的 Workbench 批处理）

### 5.2 references / sketches

```yaml
references:
  - { id: <id>, kind: plane|axis|point, def: { canonical: XY|YZ|XZ } | { offset: { from: <id|canonical>, dist: <expr>, normal?: [x,y,z] } } }

sketches:
  - id: <id>
    intent: <人话>
    plane: <ref-id>
    geometry:                    # 局部寻址：块内 "#<序号>"
      - { prim: point|line|rectangle|circle|arc|slot|polygon|spline|ellipse, ... }   # 每原语的具体参数见词汇表附录
    constraints:
      - { type: <约束词>, targets: ["#0", "#1", ...], value?: <expr> }
```

- **几何原语词汇（v0.1，9 个）**：point、line（p1/p2）、rectangle（p1/p2）、circle（center/d）、arc（center/start/end, ccw）、slot（center/length/width, angle）、polygon（center/sides/r, inscribed）、spline（points, closed）、ellipse（center/major/minor, angle）
- **约束词汇（12 个）**：closed、coincident、parallel、perpendicular、tangent、concentric、horizontal、vertical、equal、symmetric、midpoint、fix
- `closed` 是意图声明：harness 编译为闭合性校验（`CheckFeatureUse`），违反即 422（[07](./07-correctness.md) §3 有意收紧语义）
- construction geometry：原语加 `construction: true`

### 5.3 features

```yaml
- id: <id>            # 文档内唯一；断点续跑与符号引用的锚点
  intent: <人话必填>
  op: <动词>
  ...: <该动词的参数>
  when?: <expr>       # 条件包含（配置变体）
  on_fail?: abort|skip|ask    # 缺省 abort
  native?: { ... }    # 后端逃生口：显式标记不可移植
```

**意图动词词汇表（v0.1 核心 18 个）**：

| 类 | 动词 | 关键参数 |
|---|---|---|
| 实体成形 | extrude | profile, depth \| thru \| up_to, draft?, direction? |
| | cut_extrude | profile, depth \| thru, draft? |
| | revolve / cut_revolve | profile, axis, angle |
| | sweep | profile, path, twist? |
| | loft | profiles[], guides[]? |
| | hole | position, d, depth \| thru, counterbore?/countersink?（标准孔优先用 hole 而非 cut+circle，语义更强） |
| 细节 | fillet / chamfer | target, radius / dist |
| | shell | thickness, open_faces |
| | draft | target, angle, pull_dir |
| 变换 | pattern_linear / pattern_circular | target, count, spacing/angle |
| | mirror | target, plane |
| 参考 | ref_plane / ref_axis / ref_point | def |
| 工程 | sheet_metal_bend（占位，v0.2 细化） | ... |

`target` 的引用语法：`@<feature-id>`、`@<id>.faces.top`、`@<id>.edges.vertical`、`@<sketch-id>.#n`——**拓扑选择器**是封闭子词汇（faces: top/bottom/side/cylindrical…；edges: vertical/horizontal/circular…），由 adapter 翻译为后端选择逻辑（SolidWorks 侧即 [08](./08-com-impl.md) 的多策略选择器）。禁止在 DIL 中出现原生实体名。

### 5.4 verify

```yaml
- { id: <id>, kind: assert,  expr: <布尔表达式> }          # → sw_assert（[01] §3.2 gate）
- { id: <id>, kind: inspect, checks: [self_intersect, interference, printable, draft(angle=..)] }  # → sw_validate（job）
- { id: <id>, kind: accept,  note: <里程碑说明> }          # 人工/agent 确认点
```

- 表达式上下文 = `params` + harness 状态快照字段（`mass_kg`、`volume_mm3`、`bbox`、`count_features(...)`、diagnostics 计数等，字段表见 [01](./01-harness-model.md) §3.1）
- 含 L1 数值（质量/体积）的断言触发隐式轻量刷新（[01](./01-harness-model.md) §3.3 数据源约定）
- 验证准则随文档版本化：改设计必须同步改 verify——意图与验收永不分离

---

## 6. 执行语义

### 6.1 编译与两种运行模式

```
DIL 文档/片段
   ↓ harness 前端：表达式求值 → 引用解析 → 前置校验（闭合性/DAG/词汇合法性）
   ↓ 编译：DIL 动词 → 后端操作序列（adapter）
   ↓ 执行：走串行队列，按复合操作内封时序（01 §2）
   ↓ 每特征后：轻量 gate（rebuild + diagnostics），失败按 on_fail 处置
   ↓ 返回：DIL 词汇的 state diff（applied_ids、诊断、generation）
```

| 模式 | 场景 | 说明 |
|---|---|---|
| **compile-run** | 整文档/参数变体重放 | 全编译全执行，断点续跑按 `provenance.applied_ids` 跳过已满足项 |
| **repl（片段）** | 交互对话 | agent 单发一个 feature/sketch 片段（`dil_apply`）；**每次 `sw_*` 工具调用在插件层等价编译为一个 DIL 片段**——工具调用与 DIL 是同一 IR 的两种糖 |

### 6.2 符号表：屏蔽命名的机制落点

```
harness 符号表（session 内）
  DIL id        ↔  后端原生句柄/名称
  s_base        ↔  "草图3"（中文版 SW）/ "Sketch1"（英文版）
  f_holes       ↔  "切除-拉伸1" / "Cut-Extrude1"
```

- state/diagnostics/verify 结果**一律以 DIL id 为主键**输出（原生名附注）——LLM 上下文永不接触中英文原生命名差异
- 后端重命名/重排序（特征树拖动）不影响引用：harness 句柄跟踪 + generation 失配触发重绑定

### 6.3 失败与恢复语义

- 片段失败：按 `on_fail`（abort 整批 / skip 留洞 / ask 上抛给 agent+人）
- `when` 不满足：记 `skipped_ids`，不算失败
- generation 过期（[07](./07-correctness.md) §3）：整片段拒收（409），重观察后重发
- 幂等约定：同 id 片段重复 apply = no-op（已满足即跳过），agent 重试安全

---

## 7. 后端适配（可移植性的实现）

### 7.1 adapter 契约

```typescript
interface DilAdapter {
  capabilities(): CapMatrix;                        // 动词×能力 三态：native | degraded | unsupported
  compile(feature: DilFeature, ctx: SymbolTable): OpSequence;   // → 该后端的复合操作
  select(topoRef: TopoRef, ctx): SelectionPlan;     // 拓扑选择器 → 后端选择策略
}
```

### 7.2 词汇映射示例（extrude 一行对照）

| DIL | SolidWorks adapter（现实现） | FreeCAD adapter（未来） | Onshape adapter（未来） |
|---|---|---|---|
| `op: extrude, depth: params.T` | 五级回退（FeatureExtrusion3→…→VBA，[08](./08-com-impl.md)） | PartDesign Pad | feature "extrude" via REST |
| `constraints: [closed]` | CheckFeatureUse 422 | Sketch.FullyConstrained 检查 | … |

### 7.3 语义降级规则

能力矩阵声明 `degraded` 时 adapter 必须给出降级映射与保真度标注（如 `slot` 在无原生槽口特征的后端 → 两圆+两线约束组）。`unsupported` 时 harness 返回结构化诊断（"后端 X 不支持动词 Y，建议替代 Z"）——**降级是显式信息，不是静默畸变**。

---

## 8. 与 harness / 工具面的闭环（本语言存在的理由）

```
agent ──DIL 片段──▶ dil_apply / sw_* 工具（插件层统一编译为片段）
                        │
                        ▼
              harness：编译 → 执行 → 观察 → 门控
                        │
agent ◀──DIL 词汇──── state diff + diagnostics + 截图
（上下文全程停留在 DIL 语义空间，原生差异被隔离在 harness 内）
```

对工具面的影响（详见 [06](./06-tool-set.md) §6）：新增 `dil_apply`（提交片段/文档）与 `dil_state`（DIL id 满足视图）两个工具；既有 `sw_feature` 等保留为快捷语法糖。

---

## 9. 版本与演进

| 版本 | 内容 |
|---|---|
| v0.1（本文） | 零件域：params/sketches/features/verify；18 动词、12 约束、9 原语、拓扑选择器子集 |
| v0.2（P1 后） | 用例集验证补词；sheet_metal 细化；`native` 块治理规范 |
| v0.3（P2） | 装配域（mate 词汇）、配置域（configurations）、PMI/公差块 |
| v1.0（P3） | 首个跨后端验证：同一 DIL 文档在 SolidWorks + 一个开源后端（FreeCAD/CadQuery）编译出等价模型 |

**验证准则**：DIL 的成功标准不是"能表达一切"，而是"95% 常见建模意图可用 ≤20 个动词表达，且跨后端可编译"。
