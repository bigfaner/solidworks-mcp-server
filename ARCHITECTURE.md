# SolidWorks MCP Server 架构、术语与业务流程详解

> 本文档基于对源码的实际阅读整理（2026-08），重点剖析项目架构、核心术语，以及与 SolidWorks 协作的完整机制。
> 文档中的 `文件:行号` 均可直接定位到源码。

---

## 目录

1. [项目定位与技术栈](#1-项目定位与技术栈)
2. [整体分层架构](#2-整体分层架构)
3. [目录结构详解](#3-目录结构详解)
4. [MCP 协议层：服务器启动与工具调用链路](#4-mcp-协议层服务器启动与工具调用链路)
5. [与 SolidWorks 协作机制（核心）](#5-与-solidworks-协作机制核心)
6. [SolidWorksAPI 门面层与操作层](#6-solidworksapi-门面层与操作层)
7. [辅助层：拉伸与选择两大难点](#7-辅助层拉伸与选择两大难点)
8. [COM 桥的类型系统与数据转换](#8-com-桥的类型系统与数据转换)
9. [错误恢复与可靠性机制](#9-错误恢复与可靠性机制)
10. [宏体系：三条独立的宏链路](#10-宏体系三条独立的宏链路)
11. [Prompts 与 Resources](#11-prompts-与-resources)
12. [配置与环境变量](#12-配置与环境变量)
13. [测试体系](#13-测试体系)
14. [典型业务流程时序](#14-典型业务流程时序)
15. [术语表](#15-术语表)
16. [README 宣称与实际实现的差异](#16-readme-宣称与实际实现的差异)
17. [关键代码位置索引](#17-关键代码位置索引)

---

## 1. 项目定位与技术栈

一个运行在 **Windows** 上的 **Node.js（≥20）MCP 服务器**，把 SolidWorks 2024 的 COM 自动化能力暴露给 Claude Desktop 等 MCP 客户端，使 AI 能直接驱动 CAD 完成建模、草图、工程图、导出、质量分析、宏录制等操作。

| 技术 | 用途 |
|---|---|
| TypeScript 5.5 + ESM | 主体语言，`tsc` 编译到 `dist/` |
| `@modelcontextprotocol/sdk` | MCP 协议实现（stdio 传输、高层 `McpServer` API） |
| `zod` | 所有工具入参/出参 schema，注册时自动校验 |
| `winax` 3.4 | Node.js ↔ Windows COM 桥（唯一与 SolidWorks 通信的通道，原生模块需编译） |
| `handlebars` | 编译 `examples/vba-templates/*.vba` 模板生成 VBA 代码 |
| `winston` | 日志（避免 console 干扰 stdio 上的 JSON-RPC） |
| `uuid` | 宏录制会话 ID |
| `vitest` | 单元测试 + 集成测试 |

**平台约束**：只能在 Windows 上运行；SolidWorks 需已安装且有许可证；MCP 客户端通过 stdio 与本服务器同机通信。

---

## 2. 整体分层架构

```
┌────────────────────────────────────────────────────────────────┐
│  MCP 客户端（Claude Desktop / 其他 MCP Host）                    │
└───────────────────────────┬────────────────────────────────────┘
                            │ stdio + JSON-RPC（MCP 协议）
┌───────────────────────────▼────────────────────────────────────┐
│ ① MCP 协议层   src/index.ts  SolidWorksMCPServer                │
│    注册 Tools / Resources / Prompts；统一日志、宏录制、懒连接     │
├────────────────────────────────────────────────────────────────┤
│ ② 工具层       src/tools/    10 个模块 ≈ 84 个工具              │
│    ToolDefinition = { name, description, zod inputSchema,       │
│                       outputSchema, handler(args, swApi) }      │
├────────────────────────────────────────────────────────────────┤
│ ③ API 门面层   src/solidworks/api.ts  SolidWorksAPI（facade）    │
│    持有 currentModel，委托给 operations/*，复用 helpers/*        │
├────────────────────────────────────────────────────────────────┤
│ ④ 操作层       src/solidworks/operations/                       │
│    ConnectionManager / Model / Sketch / Dimension / Export /    │
│    VBA / MassProperties / ConfigSystem（均为 static 类方法）     │
├────────────────────────────────────────────────────────────────┤
│ ⑤ 辅助层       src/solidworks/helpers/                          │
│    extrusion（1147 行，拉伸全流程+多级回退）、selection（多策略  │
│    选实体）、sketch、model（ModelHelpers）                       │
├────────────────────────────────────────────────────────────────┤
│ ⑥ COM 桥       winax：new winax.Object('SldWorks.Application')   │
│    src/solidworks/types/com-types.ts（740 行手写 COM 接口声明）  │
└───────────────────────────┬────────────────────────────────────┘
                            │ COM Automation（同步调用）
                ┌───────────▼──────────────┐
                │  SolidWorks 进程（SldWorks）│
                └──────────────────────────┘

横切模块：
  src/adapters/    适配器抽象（ISolidWorksAdapter + MacroGenerator 动态 VBA 生成）
  src/macro/       MCP 侧宏录制器（MacroRecorder）
  src/utils/       logger / error-recovery / com-boolean / com-lifecycle /
                   config / feature-utils / environment / solidworks-config
  src/shared/constants/solidworks-constants.ts   SolidWorks 枚举常量（替代魔法数字）
  src/prompts/     5 个工作流 Prompt
  src/resources/   4 个只读 Resource
```

---

## 3. 目录结构详解

```
solidworks-mcp-server/
├── src/
│   ├── index.ts                     # 入口：SolidWorksMCPServer，stdio transport
│   ├── adapters/
│   │   ├── types.ts                 # ISolidWorksAdapter / AdapterConfig / 各特征参数接口
│   │   └── macro-generator.ts       # MacroGenerator：动态拼 VBA 字符串（537 行）
│   ├── solidworks/
│   │   ├── api.ts                   # SolidWorksAPI 门面（511 行）
│   │   ├── types/
│   │   │   ├── com-types.ts         # ISldWorksApp/IModelDoc2/IFeatureManager 等 COM 声明
│   │   │   ├── business-types.ts    # SolidWorksModel/Feature/Dimension 等业务类型
│   │   │   ├── interfaces.ts        # 草图等参数接口
│   │   │   └── index.ts
│   │   ├── operations/              # 按领域拆分的静态操作类
│   │   │   ├── connection.ts        # ConnectionManager（winax 连接）
│   │   │   ├── model.ts             # 打开/关闭/新建文档
│   │   │   ├── sketch.ts            # SketchOperations（草图 CRUD、选基准面）
│   │   │   ├── dimension.ts         # DimensionOperations（5 级取尺寸回退）
│   │   │   ├── export.ts            # ExportOperations（多格式导出+临时保存）
│   │   │   ├── vba.ts               # VBAOperations（RunMacro2→RunMacro 回退）
│   │   │   ├── mass-properties.ts   # 质量属性
│   │   │   └── config-system.ts     # 配置/材料/模板/系统状态（供 Resources 用）
│   │   └── helpers/
│   │       ├── extrusion.ts         # ★ 拉伸核心难点（退草图→选草图→4 级 API 回退）
│   │       ├── selection.ts         # ★ 多策略实体选择
│   │       ├── sketch.ts            # 草图段枚举/命名
│   │       ├── model.ts             # ModelHelpers（ensureCurrentModel 等）
│   │       └── index.ts
│   ├── tools/                       # MCP 工具（见 §4 的工具清单）
│   │   ├── sketch.ts (20)  drawing.ts (8)  export.ts (4)  analysis.ts (6)
│   │   ├── template-manager.ts (6)  native-macro.ts (8)  diagnostics.ts (1)
│   │   ├── macro-security.ts (2)    macro-recording.ts (3)
│   │   ├── vba/ base.ts (5) part.ts (5) assembly.ts (4) drawing.ts (5)
│   │   │        file-management.ts (2) advanced.ts (5)
│   │   ├── title-map.ts             # 工具名 → 人读标题
│   │   └── types.ts                 # ToolDefinition + createToolResult
│   ├── resources/index.ts           # 4 个 MCP Resource
│   ├── prompts/index.ts             # 5 个 MCP Prompt
│   ├── macro/                       # MacroRecorder + 类型
│   ├── utils/                       # 见 §9
│   ├── shared/constants/            # SolidWorks 枚举（SwDocumentType 等）
│   └── types/winax.d.ts             # winax 模块声明
├── examples/vba-templates/          # 9 个 Handlebars VBA 模板
│   ├── create_part / create_assembly / create_drawing / create_sketch
│   ├── modify_dimensions / modify_properties
│   └── batch_process / batch_export / common
├── tests/                           # vitest 单测+集成（见 §13）
├── .env.example                     # 环境变量样例
└── package.json                     # bin: solidworks-mcp-server → dist/index.js
```

括号内数字为该文件定义的 MCP 工具数量，合计 **84 个**（与 README 宣称的 83/87 均有出入）。

---

## 4. MCP 协议层：服务器启动与工具调用链路

### 4.1 启动流程（src/index.ts）

1. `dotenv.config()` 读取 `.env`
2. `zod` 解析 `ConfigSchema`（solidworksPath / enableMacroRecording / logLevel）
3. 构造 `SolidWorksAPI` 与 `MacroRecorder` —— **注意此时并不连接 SolidWorks**（懒连接）
4. `new McpServer({ name: 'solidworks-mcp-server', version })`
5. 依次 `registerAllTools()` / `registerAllResources()` / `registerAllPrompts()` / `setupMacroHandlers()`
6. `start()`：绑定 `StdioServerTransport`，监听 SIGINT 优雅关闭（清空录制、断开连接）
7. 测试环境（`NODE_ENV=test` 或 `VITEST`）不自动启动

### 4.2 工具注册（src/index.ts:105-182）

- 10 个工具模块的数组合并后逐个 `server.registerTool(name, { title, description, inputSchema, outputSchema }, handler)`
- ZodObject 会拆成 `.shape` 传给 SDK；无 outputSchema 时降级为 `z.any()`
- 已注册错误（测试场景）被优雅吞掉
- 工具标题统一来自 `title-map.ts`

### 4.3 每次工具调用的统一链路（src/index.ts:131-170）

```
MCP 请求
 → logOperation(name, 'started', args)            # winston 日志
 → macroRecorder.recordAction(...)                 # 若正在录制（异常静默忽略）
 → if (!api.isConnected()) await api.connect()     # ★ 懒连接兜底
 → result = await tool.handler(args, this.api)     # 业务逻辑
 → logOperation(name, 'completed', { result })
 → createToolResult(result)                        # text + structuredContent
 →（失败）logError + re-throw → SDK 转为 MCP 协议错误
```

`createToolResult`（src/tools/types.ts:24）：对象结果同时放入 `content[0].text`（JSON 字符串）与 `structuredContent`（结构化输出），兼容两类客户端。

### 4.4 工具清单（按模块）

| 模块 | 数量 | 代表工具 |
|---|---|---|
| sketch.ts | 20 | `create_sketch`、`sketch_line/circle/arc/rectangle/polygon/spline/ellipse`、`add_sketch_constraint/dimension`、`sketch_linear_pattern/circular_pattern/mirror/offset`、`edit_sketch`、`exit_sketch`、`get_sketch_context` |
| drawing.ts | 8 | `create_drawing_from_model`、`add_drawing_view`、`add_section_view`、`add_dimensions`、`update_sheet_format`、`create_configurations_batch` 等 |
| export.ts | 4 | `export_file`、`batch_export`、`export_with_options`、`capture_screenshot` |
| analysis.ts | 6 | `get_mass_properties`、`check_interference`、`measure_distance`、`analyze_draft`、`check_geometry`、`get_bounding_box` |
| template-manager.ts | 6 | `extract/apply/batch_apply/compare_drawing_templates`、`save/list_template_library` |
| native-macro.ts | 8 | `start/stop_native_macro_recording`、`pause_resume_macro_recording`、`run_macro`、`edit_macro`、`create_initialized_macro`、`convert_text_to_native_macro`、`batch_run_macros` |
| vba/（6 文件） | 26 | `generate_vba_script`、`create_feature_vba`、`create_batch_vba`、`run_vba_macro`、`create_drawing_vba` + 21 个 `vba_*` 生成器（参考几何/高级特征/阵列/钣金/曲面/装配配合/工程图/批量/配置/方程式/仿真/错误处理） |
| diagnostics.ts | 1 | `diagnose_macro_execution` |
| macro-security.ts | 2 | `macro_set_security`、`macro_get_security_info` |
| macro-recording.ts | 3 | `macro_start_recording`、`macro_stop_recording`、`macro_export_vba` |
| **合计** | **84** | |

---

## 5. 与 SolidWorks 协作机制（核心）

### 5.1 连接机制

`ConnectionManager.connect()`（src/solidworks/operations/connection.ts:12-29）：

```typescript
this.swApp = new winax.Object('SldWorks.Application') as ISldWorksApp;
this.swApp.Visible = true;    // 置为可见便于人工观察
```

- `SldWorks.Application` 是 SolidWorks 的 **COM ProgID**：已在运行则附加到现有实例，未运行则启动新进程
- 连接失败会用函数式 `winax.Object(...)` 再试一次
- `disconnect()` 只把引用置 null，**绝不关闭 SolidWorks**（保护用户会话）
- `isConnected()` 仅判断引用非空，不代表进程一定存活（真实存活靠 `com-lifecycle.ts` 的探测或调用报错）
- 工具层每次调用前的懒连接兜底意味着：用户可以先重启 SolidWorks，再继续用 MCP

### 5.2 双通道执行模型（解决 winax 参数限制）

这是本项目存在的根本理由。**winax 在 COM 方法参数 ≥13 个时调用失败**，而 SolidWorks 关键 API 参数极多：

| API | 参数个数 | 影响 |
|---|---|---|
| `FeatureExtrusion3` | 24+ | 拉伸（凸台/切除） |
| `FeatureRevolve2` | 12+ | 旋转 |
| `FeatureManager.FeatureSweep` | 更多 | 扫描 |
| `FeatureManager.FeatureLoft` | 更多 | 放样 |

**通道 A：直连 COM（简单操作）**
工具 handler → `swApi.getCurrentModel()` 得到 `IModelDoc2` → 直接调用 `SketchManager.CreateLine`、`FeatureManager.FeatureExtrusion3` 等。快、实时返回 COM 对象。

**通道 B：VBA 宏回退（复杂操作）**
1. `MacroGenerator.generateExtrusionMacro(params)`（src/adapters/macro-generator.ts:20）拼出完整 VBA 子程序字符串：
   - `Option Explicit` + `On Error GoTo ErrorHandler`
   - `Set swApp = Application.SldWorks` / `swApp.ActiveDoc`
   - 自动选草图：先试 `Sketch1..Sketch5` 名称，再按 `FeatureByPositionReverse` 倒序找 `ProfileFeature/Sketch` 类型特征
   - 单位换算：深度 `params.depth / 1000`（mm→m）、拔模角 `draft * π/180`（度→弧度）
   - `FeatureExtrusion3` 的 24 个参数逐个填入，含注释
   - 支持薄壁（thinFeature）、终止条件映射（Blind=0 … MidPlane=6）
   - ErrorHandler 弹 MsgBox 报告失败
2. 写入临时 `.swp/.vba` 宏文件
3. `VBAOperations.runMacro` 执行（见下）

**RunMacro 的二级回退**（src/solidworks/operations/vba.ts:10-42）：
```
swApp.RunMacro2(path, module, proc, swRunMacroUnloadAfterRun=1, err)   // 首选
  失败 → swApp.RunMacro(path, module, proc)                            // 旧版 API
  再失败 → 抛错
```

此外 `extrusion.ts` 中的直连通道自身也有**四级 API 回退**（见 §7.1）。

### 5.3 同步调用模型

winax 的 COM 调用是**同步阻塞**的。工具 handler 虽然写成 async，但实际执行期间事件循环被阻塞；SolidWorks 弹模态对话框（如 VBA 的 MsgBox）时可能造成 MCP 客户端超时。这也是为什么导出强制用 `swSaveAsOptions_Silent`（静默、禁对话框）。

### 5.4 实体选择模型（SolidWorks API 的"隐式上下文"）

SolidWorks 大量 API 依赖**当前选择集**而非显式传参，因此选择是所有操作的前置步骤。项目实现了多级选择策略（`SolidWorksAPI.selectSketchEntity`，src/solidworks/api.ts:416-437）：

```
spec === 'last'                     → selectLastSketchSegment()
spec ∈ {Front, Top, Right}          → selectStandardPlane()
正则匹配 "Line3/Arc2/直线1/圆2@草图1" → selectByHeuristicMatch()   # 枚举草图段按序号选
兜底                                 → selectByID2Fallback()        # 多类型名逐一尝试
```

关键细节（src/solidworks/helpers/selection.ts:38-89）：
- 草图**未激活**时实体名要写成 `段名@草图名`、类型用 `EXTSKETCHSEGMENT`；激活时用短名 + `SKETCHSEGMENT`
- `append` 参数必须用 `toVariantBool()` 转 -1/0
- 枚举 `GetSketchSegments()` 返回值时兼容 Array / 类数组 length / Count+Item 三种形态（winax 对 VARIANT 数组的封装不确定）

### 5.5 单位与数据转换约定

| 层面 | 约定 |
|---|---|
| MCP 工具入参 | mm、度、常规数值（对用户/AI 友好） |
| SolidWorks COM API | **米**（长度）、**弧度**（角度）（API 是 SI 单位制） |
| 转换位置 | 统一在 API/操作层完成：`depth / 1000`、`draft * Math.PI / 180` |
| VBA 模板 | 在模板里显式写 `{{this.x1}}/1000`（见 create_part.vba:44-52） |
| 布尔 | COM `VARIANT_BOOL`：**-1 = true，0 = false**（见 §8.2） |
| out 参数 | 用 `{ value: 0 }` 引用对象模拟指针（winax 约定），如 `SaveAs3(..., errorRef, warningRef)` |

---

## 6. SolidWorksAPI 门面层与操作层

### 6.1 SolidWorksAPI（src/solidworks/api.ts）

- 门面模式：持有 `connectionMgr` 与 `currentModel`
- `getCurrentModel()` 每次先 `ensureCurrentModel()`：若本地引用为空则从 `swApp.ActiveDoc` 取（用户手动切换文档后 MCP 能自动跟上）
- `createExtrude` 是最复杂的方法（api.ts:128-241），完整编排：`prepareForExtrusion` → `selectSketchForExtrusion` → 判断是否首个拉伸 → `validateSketchSelectionBeforeExtrusion` → 单位换算 → **四级 API 回退** → `finalizeExtrusion`
- `createExtrudeCut`（切除拉伸）复用同一套流程，走 `tryFeatureCut3`
- `extrude()/extrudeCut()` 是带默认值（depth=25mm）的简化包装，返回 `{success, featureId?, error?}`

### 6.2 操作层（均为静态类）

| 类 | 职责 | 值得注意的细节 |
|---|---|---|
| `ConnectionManager` | 连接生命周期 | §5.1 |
| `ModelOperations` | openModel/createPart/closeModel | `OpenDoc6` 带 errors/warnings out 参数 |
| `SketchOperations` | 草图创建/实体/上下文/选基准面 | `selectStandardPlane` 需兼容中英文基准面名（"前视基准面"/"Front Plane"） |
| `DimensionOperations` | getDimension/setDimension | 取尺寸有 **5 级回退**（dimension.ts:14-90）：`model.Parameter` → `GetParameter` → `Extension.GetParameter` → 按 `名@特征` 拆分遍历特征的 DisplayDimension → SelectByID2 + SelectionManager；遍历有 `PerformanceLimits.MAX_DIMENSION_ITERATIONS` 上限防死循环 |
| `ExportOperations` | 多格式导出 | §14.3 |
| `VBAOperations` | 运行宏 | RunMacro2→RunMacro 回退 |
| `MassPropertiesOperations` | 质量属性 | 返回质量/体积/表面积/质心 |
| `ConfigSystemOperations` | 配置/材料/模板/系统状态 | 主要供 Resources 层使用；单位制用 `GetUserPreferenceIntegerValue(1)` 读 swUnitSystem，映射到 {mm,cm,m,in,ft}；模板路径用 `GetUserPreferenceStringValue(8/9/10)` |

---

## 7. 辅助层：拉伸与选择两大难点

### 7.1 extrusion.ts（src/solidworks/helpers/extrusion.ts，1147 行）

这是全项目经验密度最高的文件，中文注释记录了大量实测结论。

**prepareForExtrusion（:22-144）— 拉伸前的时序准备**
```
1. 若 ActiveSketch 存在：
   InsertSketch(false) 退出草图并提交特征树
     ↳ 实测结论：false 才是"退出"（注释明确指出与部分文档相反）
     ↳ 失败则试 InsertSketch(true)（不同 SW 版本行为相反）
   ClearSelection2(-1)
   判断是否首个拉伸（倒序找 Extrude/拉伸/Boss/Cut 类型特征）
   EditRebuild3() 重建（首/后续拉伸都重建，重建后重新选择的代价被接受）
   复查 ActiveSketch，仍激活则再走一遍 退出+清选+重建
2. 无活动草图：直接 EditRebuild3()
3. 最终 ClearSelection2(-1)
```

**selectSketchForExtrusion（:157-…）— 拉伸用草图选择**
```
ClearSelection2(-1)
倒序遍历 FeatureByPositionReverse(0..49)：          # 最多 50 个特征
  isSketchLikeFeature(名, 类型) 判断（兼容中英文与 ICE/ProfileFeature 等内部类型名）
  → 首选 feat.Select2(false, 0)，用 SelectionManager.GetSelectedObjectCount2(-1) 验证
  → 尝试升级为 REGION（草图轮廓）选择（空名 + 坐标 0,0,0 定位）
  → 回退 SelectByID2(featName, 'SKETCH', ...)
验证阶段：
  GetSelectedObjectType3(1,-1) == 9 (swSelSKETCHES) 时确认类型正确
  CheckFeatureUse(0, openCount, closedCount) 验证草图闭合轮廓数
    ↳ closedCount == 0 时告警"拉伸可能失败"
全部失败 → 抛错并列出 attemptedSketches
```

**四级拉伸 API 回退（api.ts 编排，helpers 实现）**
```
tryFeatureExtrusion3     # 最完整，24+ 参数（winax 大概率失败，仍先试）
  ↓ null/异常
tryFeatureExtrusion2     # 参数略少
  ↓
tryFeatureExtrusion      # 最老最少参数
  ↓
tryFeatureExtrusionWithFeatureData   # 特征数据对象模式
  ↓ 全失败
抛 "All extrusion methods failed. Last error: ..."
```

`finalizeExtrusion`：拉伸成功后重建、取特征信息、清选择。

### 7.2 selection.ts（§5.4 已述）

要点重申：`last` 关键字、中英文段名正则（`/^(Line|Arc|Circle|直线|弧|圆|矩形|Point|点)(\d+)(@.*)?$/i`）、三种数组形态兼容、短名/全名与类型联动。

---

## 8. COM 桥的类型系统与数据转换

### 8.1 手写 COM 接口声明（src/solidworks/types/com-types.ts，740 行）

winax 返回的是 `any`，项目为常用接口手写 TS 声明获得类型安全：

- `ISldWorksApp`：文档操作（OpenDoc6/NewPart/NewAssembly/NewDrawing/ActivateDoc2）、宏（RunMacro2/RunMacro/RecordMacro/StopMacroRecording/Pause/Resume/EditMacro）、偏好（SetUserPreferenceToggle 等）
- `IModelDoc2`：Extension / SelectionManager / FeatureManager / SketchManager 四大管理器属性 + Save3/SaveAs3/SaveAs4、EditRebuild3、InsertSketch2、ClearSelection2、特征遍历（FirstFeature/FeatureByName/FeatureByPositionReverse/GetFeatureCount）
- 所有接口末尾都有 `[key: string]: unknown` **索引签名**——为未声明的方法放行（winax 动态分派）

### 8.2 VARIANT_BOOL（src/utils/com-boolean.ts）

COM 的布尔是 16 位 `VARIANT_BOOL`：**-1 = TRUE，0 = FALSE**（按位取反的惯例）。JavaScript 的 `true` 经 winax 传入后行为不可靠，因此：

```typescript
COM.TRUE === -1; COM.FALSE === 0;
toVariantBool(true)  → -1   // 传参前转换
fromVariantBool(-1)  → true // 接收返回值
```

文件头注释称其为"最常被忽视的问题"。全项目 COM 布尔传参统一走此模块。

### 8.3 winax 类型声明（src/types/winax.d.ts）

`Object` 类（构造即创建 COM 对象）、函数式 `Object()`、`release()`、`Variant()`。成员访问全靠动态分派。

### 8.4 COM 生命周期（src/utils/com-lifecycle.ts）

- `isCOMObjectValid(obj)`：尝试访问 toString/Name/GetTypeName2 探测对象是否仍存活（RPC 断连后访问会抛错）
- `SafeCOMRef<T>`：带 released 标志的引用包装，`release()` 幂等，尝试调用原生 `Release()`

---

## 9. 错误恢复与可靠性机制

### 9.1 错误分类（src/utils/error-recovery.ts）

```
ErrorCategory.RECOVERABLE  # 可重试：RPC_E_CALL_REJECTED / busy / timeout / connection /
                           #        "No active model" 等（按错误码或消息正则判定）
ErrorCategory.STATE_ERROR  # 需状态恢复：no active sketch / selection failed / invalid state
ErrorCategory.PERMANENT    # 不可恢复：直接抛出
```

COM 错误码表（`COMErrorCodes`）：RPC_E_CALL_REJECTED(0x80010001)、RPC_E_DISCONNECTED、RPC_E_SERVERCALL_RETRYLATER、RPC_E_SERVER_UNAVAILABLE(0x800706BA)、E_FAIL、E_INVALIDARG 等。

### 9.2 重试与状态恢复

- `recoverFromCOMError()`：按错误码判断可恢复性，**指数退避 + 抖动**（`delay = base × 2^attempt + rand(200ms)`），默认 3 次
- `withRetry()`：通用版（消息正则判定）
- `recoverSketchState(model)`：状态恢复三板斧——ClearSelection2 → 若草图激活则 InsertSketch(true) 退出 → EditRebuild3
- `withErrorRecovery()`：组合技，工具层（如 sketch.ts）直接包裹业务函数；STATE_ERROR 在每次重试前先做状态恢复，最终失败后再做一次

### 9.3 未实现的可靠性特性

README 宣称的 **断路器（Circuit Breaker）、连接池（Connection Pooling）、复杂度分析器** 在代码中只有配置项占位（`AdapterConfig.enableCircuitBreaker` 等，src/adapters/types.ts:182-196），**没有实现**。Edge.js 适配器与 PowerShell 桥同样仅是 Roadmap。

---

## 10. 宏体系：三条独立的宏链路

项目里"宏"一词出现在三个互不相同的体系里，极易混淆：

### 链路 1：MCP 侧动作录制（src/macro/recorder.ts + src/tools/macro-recording.ts）

- `macro_start_recording(name)` 开始会话 → 之后**每次 MCP 工具调用**都被 index.ts 自动 `recordAction(tool.name, description, args)` 记录（录制的是 MCP 动作序列，不是 SolidWorks 操作）
- `macro_stop_recording()` 结束并返回动作列表
- `macro_export_vba(macroId)` 把动作序列转译为 VBA 代码
- 录制结果存内存 Map，可 `executeMacro` 回放（按 action type 查 `actionHandlers`；index.ts 里注册了 create-sketch/add-line/extrude 三个 handler）

### 链路 2：SolidWorks 原生宏录制（src/tools/native-macro.ts）

直接映射 SolidWorks 自带录制器：`start_native_macro_recording` → `swApp.RecordMacro(path)`；`stop_native_macro_recording` → `StopMacroRecording()`；还有 `pause/resume`、`run_macro`（→ VBAOperations）、`edit_macro`（打开 VBA 编辑器）、`create_initialized_macro` / `convert_text_to_native_macro`（把纯文本 VBA 包装成带 swApp 获取、错误处理的合规宏）、`batch_run_macros`。辅助以 `macro-security.ts`（读设安全级别）与 `diagnostics.ts`（宏执行失败分步诊断）。

### 链路 3：模板化 VBA 生成（src/tools/vba/ + examples/vba-templates/）

- `generate_vba_script` 等 26 个工具用 Handlebars 编译 `.vba` 模板，`compileTemplate` 里有模板名映射（如 batch_export → batch_process），模板缺失时回退直读
- base.ts 注册了 eq/ne/lt/gt/and/or/not 等 Handlebars helper（"CRITICAL FIX"注释）
- 模板示例（create_part.vba）：`{{#each features}}` 循环内 `{{#eq this.type "rectangle"}}` 分支生成 `CreateCornerRectangle {{x1}}/1000, ...`——**生成的是代码文本，不直接执行**；配合 `run_vba_macro` 才落地
- 这是处理"13+ 参数复杂操作"的静态模板方案；`MacroGenerator`（adapters）则是动态字符串拼接方案，二者互补

---

## 11. Prompts 与 Resources

### Resources（src/resources/index.ts，只读）

| 名称 | URI | 实现要点 |
|---|---|---|
| `sw-config` | `solidworks://config`（静态） | `ConfigSystemOperations.getConfiguration`：版本 RevisionNumber、单位制映射、模板路径 |
| `material` | `solidworks://materials/{name}`（模板） | 材料库属性：密度/弹性模量/泊松比/屈服强度 |
| `templates` | `solidworks://templates/{type}`（模板） | part/assembly/drawing 可用模板清单 |
| `sw-system` | `solidworks://system`（静态） | 连接状态、版本、活动文档、已打开文档列表 |

错误统一降级为 `{error, message}` JSON 而非抛出（Resource 读取不应让客户端崩）。

### Prompts（src/prompts/index.ts，5 个）

`create-part-workflow`（simple/medium/complex 三档步骤树）、`create-assembly-workflow`（先固定基座组件、最少配合、查干涉）、`analyze-model`（质量/干涉/几何/全部）、`export-workflow`（按格式给提示）、`sketch-workflow`（完全定义草图等最佳实践）。实现均为返回预组装的 user message 文本，指导 AI 按步骤调用工具。

---

## 12. 配置与环境变量

`.env.example` + `src/utils/config.ts`：

| 变量 | 默认 | 用途 |
|---|---|---|
| `SOLIDWORKS_PATH` | `C:/Program Files/SOLIDWORKS Corp/SOLIDWORKS` | 安装路径（推 SLDWORKS.exe 用） |
| `SOLIDWORKS_VERSION` | `2024` | 版本声明 |
| `SOLIDWORKS_MODELS_PATH` | `~/Documents/SolidWorks` | 默认模型目录 |
| `SOLIDWORKS_MACROS_PATH` | `~/Documents/SolidWorks/Macros` | 默认宏目录 |
| `LOG_LEVEL` | `info` | winston 级别 |
| `ENABLE_MACRO_RECORDING` | 非 `'false'` 即开 | index.ts 是否对每次调用 recordAction |
| `CHROMA_HOST/PORT` | — | `.env.example` 遗留项，代码未使用 |

Claude Desktop 接入（README）：`claude_desktop_config.json` → `mcpServers.solidworks` → `node dist/index.js`，env 里可设 `SOLIDWORKS_PATH`、`ADAPTER_TYPE=winax-enhanced`（后者实际代码未消费）。

---

## 13. 测试体系

vitest，`npm run test:unit`（排除 integration）/ `test:integration` 分离，`USE_MOCK_SOLIDWORKS=true` 可在无 SolidWorks 的 CI 上跑单测。

```
tests/
├── index.test.ts / version.test.ts            # 服务器装配
├── adapters/       macro-generator、types
├── solidworks/
│   ├── api.test.ts
│   ├── operations/ connection/model/sketch/dimension/export/mass-properties/vba/config-system
│   └── helpers/    extrusion（含 extrusion.integration.test.ts）、selection、sketch、model
├── macro/recorder.test.ts
├── prompts/ resources/                        # MCP 三件套测试
├── utils/         com-boolean/com-lifecycle/config/environment/error-recovery/
│                  feature-utils/logger/solidworks-config
├── helpers/       check-solidworks.ts（探测真实 SW 环境）、solidworks-setup、test-models、test-utils
└── integration/   mcp-tools-comprehensive、motor-bracket（端到端建模用例）
```

---

## 14. 典型业务流程时序

### 14.1 建模主流程（最常用）

```
AI/用户
 │ create_part
 │   └→ api.createPart() → swApp.NewPart()（或默认模板 NewDocument）→ currentModel
 │ create_sketch {plane: 'Top', offset: 10}
 │   └→ selectStandardPlane('Top')   # 兼容 "Top Plane"/"上视基准面"
 │      offset≠0 时 FeatureManager.InsertRefPlane(8,0,4,offset/1000,0,0) 造偏移基准面
 │      SketchManager.InsertSketch(true)
 │      返回 sketchName（可能是 "草图1"/"Sketch1"，无名字时回退 MCP_Sketch_<ts>）
 │ sketch_rectangle {corner1, corner2}
 │   └→ ensureActiveSketch 校验 → SketchManager.CreateCornerRectangle(x1/1000,…)
 │ add_sketch_constraint / add_sketch_dimension
 │   └→ selectSketchEntity('last'/'Line2'/…) → SketchAddConstraints / AddDimension
 │ extrude {depth: 50}
 │   └→ createExtrude(50, 0, false)
 │      ├─ prepareForExtrusion       # §7.1：退草图→清选→判断首拉伸→重建
 │      ├─ selectSketchForExtrusion  # §7.1：倒序找草图→Select2→REGION 尝试→闭合性验证
 │      ├─ validateSketchSelectionBeforeExtrusion
 │      ├─ depth/1000
 │      ├─ FeatureExtrusion3 → 2 → 1 → WithFeatureData 四级回退
 │      └─ finalizeExtrusion（重建+取特征信息）
 │ get_mass_properties / check_geometry / export_file …
```

### 14.2 复杂拉伸（宏回退路径）

`create_extrusion_advanced`（多参数）→ 判定超 winax 参数上限 → `MacroGenerator.generateExtrusionMacro`（拼 24 参数 VBA，含终止条件映射 mm→m、deg→rad）→ 写临时宏文件 → `RunMacro2`（失败→`RunMacro`）→ VBA 内部自选草图并执行 → 返回结果。VBA 的 MsgBox 错误提示在无人值守时是已知隐患。

### 14.3 导出流程（ExportOperations.exportFile）

```
路径规范化（相对→绝对）→ 目录不存在则 mkdirSync(recursive)
模型未保存过？→ 先临时保存到导出目录（Extension.SaveAs3 优先，回退 model.SaveAs3）
     ↳ 某些格式（如 STEP）要求文档已落盘
EditRebuild3() 重建
按格式分支（step/stp、iges/igs、stl、pdf、dxf、dwg…）
  统一策略：Extension.SaveAs3(路径, 0, swSaveAsOptions_Silent=1, null, errRef, warnRef)
  ↳ Silent = 1：禁止一切对话框（防 stdio 服务被模态框挂死）
  失败 → 各格式专属备用方法
验证文件存在 → 返回
```

### 14.4 工程图流程

`create_drawing_from_model`（按模板新建 SLDDRW + 基础视图）→ `add_drawing_view`（front/top/iso/section…，坐标单位 mm）→ `add_dimensions`（自动标注）→ `update_sheet_format`（标题栏属性）→ `export_file`(pdf/dxf/dwg)。模板管理器支持从父图提取（extract）、应用到子图（apply/batch_apply）、对比（compare）、入库（save_to_library）。

---

## 15. 术语表

| 术语 | 含义 | 出处/位置 |
|---|---|---|
| **MCP** | Model Context Protocol，Anthropic 的 AI-工具连接协议（JSON-RPC over stdio） | `@modelcontextprotocol/sdk` |
| **Tool / Resource / Prompt** | MCP 三类能力：可调用工具 / 只读资源 / 工作流提示模板 | src/tools,resources,prompts |
| **winax** | Node.js 的 Windows COM 自动化原生桥，本项目唯一 SW 通信通道 | 依赖 + src/types/winax.d.ts |
| **COM** | Component Object Model，Windows 进程间组件协议；SolidWorks 全部 API 以 COM 暴露 | — |
| **ProgID** | COM 程序标识符，本项目用 `SldWorks.Application` | connection.ts:15 |
| **SldWorks / IModelDoc2** | SW 应用对象 / 打开的文档对象（零件/装配/工程图统一接口） | com-types.ts |
| **FeatureManager / SketchManager / SelectionManager / Extension** | IModelDoc2 的四大管理器：特征树 / 草图 / 选择 / 扩展（SaveAs、SelectByID2 等） | com-types.ts |
| **VARIANT_BOOL** | COM 布尔：-1=true, 0=false | com-boolean.ts |
| **out 参数** | COM 引用传参，winax 用 `{value: n}` 对象模拟 | SaveAs3/RunMacro2 等 |
| **Macro Fallback** | 复杂操作（13+ 参数）降级为生成 VBA 宏在 SW 内执行 | adapters/macro-generator.ts |
| **FeatureExtrusion3** | 24+ 参数的拉伸 API，winax 参数限制的典型受害者 | extrusion.ts / macro-generator.ts |
| **REGION** | 草图轮廓（面域）选择类型，拉伸时选轮廓比选草图特征更可靠 | extrusion.ts:218-228 |
| **swSelSKETCHES (9)** | SelectionManager 对象类型枚举，9 = 草图特征 | extrusion.ts:340 |
| **ProfileFeature / ICE** | 草图特征的内部类型名（isSketchLikeFeature 兼容） | feature-utils.ts |
| **基准面** | Front/Top/Right 前视/上视/右视，中英文命名均需兼容 | SketchOperations |
| **Native Macro** | SW 自带录制器产出的 .swp VBA 宏 | native-macro.ts |
| **MCP 宏录制** | 记录 MCP 工具调用序列并可回放/导出 VBA（区别于 Native） | macro/recorder.ts |
| **Blind / ThroughAll / MidPlane…** | 拉伸终止条件枚举（SwEndCondition 0-9） | solidworks-constants.ts |
| **RPC_E_CALL_REJECTED 等** | COM/RPC 错误码，决定重试策略 | error-recovery.ts |
| **懒连接** | 首次工具调用才建立 SW COM 连接 | index.ts:147-149 |
| **静默保存 (Silent=1)** | 禁止保存/导出时弹对话框，防 stdio 服务挂死 | export.ts |

---

## 16. README 宣称与实际实现的差异

| README 宣称 | 实际情况 |
|---|---|
| "Feature Complexity Analyzer 智能路由" | **未实现独立组件**。实际路由是硬编码的：简单操作直连，复杂操作由开发者预先写成宏回退路径 |
| "Circuit Breaker 断路器" | 仅 `AdapterConfig` 中有配置项字段，无实现 |
| "Connection Pooling 连接池" | 同上，未实现；实际单连接（ConnectionManager 单实例） |
| "Edge.js Adapter / PowerShell Bridge" | Roadmap 项，架构图里画了但代码不存在 |
| "87 Tools"（头部）/ "83 Tools"（统计节） | 实际统计 **84 个**（含 macro-recording 3 个） |
| ADAPTER_TYPE 环境变量 | README 示例中出现，代码未消费 |
| `.env.example` 的 CHROMA_HOST/PORT | 遗留项，无代码引用 |
| CHANGELOG 提到 CHANGELOG.md | 仓库实际无该文件（package.json files 列表里引用了它） |

另注意：代码中大量 `console.log('[DEBUG] ...')` 中英混排调试输出（尤其 extrusion.ts），与 winston 并存，生产环境可能污染 stdout——**MCP 的 stdio 传输下 stdout 被 JSON-RPC 独占，console.log 写入 stdout 属于隐患**（logger 专门用 winston 规避了这一点，console.log 是漏网之鱼）。

---

## 17. 关键代码位置索引

| 主题 | 位置 |
|---|---|
| 服务器入口 / 工具注册 / 调用链 | src/index.ts:55-292（调用链 :131-170） |
| 工具定义接口 / 结果封装 | src/tools/types.ts:13-43 |
| COM 连接 | src/solidworks/operations/connection.ts:12-29 |
| 拉伸全流程编排 | src/solidworks/api.ts:128-241 |
| 退草图时序 | src/solidworks/helpers/extrusion.ts:22-144 |
| 拉伸草图选择+验证 | src/solidworks/helpers/extrusion.ts:157-370 |
| 四级拉伸 API 回退 | src/solidworks/helpers/extrusion.ts（tryFeatureExtrusion3/2/1/WithFeatureData） |
| 多策略实体选择 | src/solidworks/helpers/selection.ts:14-197 |
| 尺寸 5 级回退 | src/solidworks/operations/dimension.ts:14-90 |
| 导出（临时保存+Silent） | src/solidworks/operations/export.ts:27-130 |
| RunMacro2→RunMacro | src/solidworks/operations/vba.ts:10-42 |
| 动态 VBA 生成（24 参数拉伸） | src/adapters/macro-generator.ts:20-150 |
| Handlebars VBA 模板编译 | src/tools/vba/base.ts:26-89 |
| VARIANT_BOOL | src/utils/com-boolean.ts:14-35 |
| 错误分类/重试/状态恢复 | src/utils/error-recovery.ts:93-137, 249-327 |
| COM 对象存活探测/安全引用 | src/utils/com-lifecycle.ts:11-80 |
| SW 枚举常量 | src/shared/constants/solidworks-constants.ts |
| Resources | src/resources/index.ts:14-169 |
| Prompts | src/prompts/index.ts:13-309 |
| MCP 侧宏录制 | src/macro/recorder.ts:9-120 |
| VBA 模板库 | examples/vba-templates/*.vba（9 个） |

---

*本文档由源码静态分析生成；行号对应 2026-08 的 main 分支状态，后续演进请以源码为准。*
