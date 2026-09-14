# 06 · Sidecar 操作 COM 参考实现

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 覆盖原 v2.1 文档附录：connection / queue / ops-feature / server 四文件代码 + COM 调用具体示例 + 关键模式速查
> 关联：[04-sidecar.md](./04-sidecar.md)（结构）、[07-correctness.md](./07-correctness.md)（正确性语义）

四个文件展示完整链路：**连接 → 串行队列 → 复合操作（五级回退 + VBA 兜底）→ HTTP 端点**。所有 COM 细节（VARIANT_BOOL、单位换算、out 参数、时序）集中在这层。§5 另附 7 组最小示例（5.0–5.6），每组都注明与现有仓库（`src/`）的对应关系，可与真实实现对照阅读。

---

## 1. `kit/connection.ts` — winax 连接

```typescript
import winax from 'winax';
import { ISldWorksApp } from './types/com-types.js';

export class Connection {
  private swApp: ISldWorksApp | null = null;

  connect(): void {
    // ProgID：SW 已运行则附加，未运行则启动（耗时可达 30s+）
    this.swApp = new winax.Object('SldWorks.Application') as ISldWorksApp;
    this.swApp.Visible = true;
  }

  disconnect(): void {
    this.swApp = null;          // 只断引用，绝不关 SolidWorks
  }

  get app(): ISldWorksApp {
    if (!this.swApp) throw new Error('NOT_CONNECTED');
    return this.swApp;
  }

  // COM 存活探测（迁移自 com-lifecycle.ts）
  isAlive(): boolean {
    try { this.app.RevisionNumber; return true; }
    catch { return false; }
  }
}
```

---

## 2. `queue.ts` — 全局串行队列 + generation 版本

（正确性语义见 [07-correctness.md](./07-correctness.md) §1/§3）

```typescript
import { EventEmitter } from 'events';

interface OpJob {
  op: string;
  args: any;
  expectedGen?: number;              // 乐观并发：调用方持有的观察版本
  run: () => Promise<any>;           // 由 server 注入的执行体
  resolve: (v: any) => void;
  reject: (e: any) => void;
}

export class OpQueue extends EventEmitter {
  private q: OpJob[] = [];
  private running = false;
  private gen = 0;                    // 状态版本：SW 状态每变一次 +1
  runner: (op: string, args: any) => Promise<any> = async () => {
    throw new Error('runner not installed');
  };

  submit(job: Omit<OpJob, 'resolve' | 'reject'>): Promise<any> {
    return new Promise((resolve, reject) => {
      this.q.push({ ...job, resolve, reject });
      this.emit('depth', this.q.length);          // SSE: 队列深度上报（job_progress 家族）
      void this.drain();
    });
  }

  /** 外部变化检测调用（SSE doc_switched/state_changed 的来源，检测机制见 01 §3.2） */
  bumpGeneration(): void { this.gen++; this.emit('generation', this.gen); }
  /** 复合操作成功且改变了状态时调用（01 §3.1：agent 操作也自增） */
  commitGeneration(): void { this.gen++; this.emit('generation', this.gen); }
  get generation(): number { return this.gen; }

  private async drain(): Promise<void> {
    if (this.running) return;
    this.running = true;                           // 队列锁：复合操作在此槽内原子执行（01 §2）
    try {
      while (this.q.length > 0) {
        const job = this.q.shift()!;
        try {
          if (job.expectedGen !== undefined && job.expectedGen !== this.gen) {
            job.reject({                          // 版本不符：拒收过期观察（409 语义）
              code: 409, category: 'stale_state',
              hint: '模型状态已变化（可能被人工修改），请先 sw_get_state 重新观察',
              currentGen: this.gen,
            });
            continue;
          }
          const result = await job.run();
          this.commitGeneration();                 // 成功改状态 → 自增（01 §3.1 统一口径）
          job.resolve(result);
        } catch (e) { job.reject(e); }
      }
    } finally { this.running = false; }
  }
}
```

---

## 3. `ops/feature.ts` — 复合操作：拉伸

（内封自现有仓库 extrusion.ts 1147 行；粒度原则见 [07-correctness.md](./07-correctness.md) §2）

```typescript
import { COM } from '../kit/utils/com-boolean.js';   // TRUE=-1 / FALSE=0
import { renderExtrusionMacro } from '../kit/adapters/macro-generator.js';
import path from 'node:path'; import os from 'node:os'; import fs from 'node:fs';

const SW_SKETCH_SELECTED = 9;   // swSelSKETCHES

export async function opExtrude(conn: Connection, args: ExtrudeArgs) {
  const model = conn.app.ActiveDoc;
  if (!model) throw { code: 400, category: 'state_error', hint: '无活动文档' };

  // ① 退出草图（实测：InsertSketch(false) 才是"退出并提交"；版本分歧则换 true）
  const sketchMgr = model.SketchManager;
  if (sketchMgr.ActiveSketch) {
    try { sketchMgr.InsertSketch(COM.bool(false)); }
    catch { sketchMgr.InsertSketch(COM.bool(true)); }
  }
  model.ClearSelection2(COM.TRUE);

  const beforeCount = model.GetFeatureCount();    // 供步骤⑦ 的 VBA 结果差分查证

  // ② 重建，确保草图注册进特征树
  try { model.EditRebuild3(); } catch { /* 首个特征时重建失败可容忍 */ }

  // ③ 倒序特征树找草图并 Select2（名称可能中文"草图3"）
  const feat = findLatestSketch(model);
  if (!feat) throw { code: 400, category: 'state_error',
    hint: '未找到可拉伸的草图，请先创建闭合草图' };
  model.ClearSelection2(COM.TRUE);
  feat.Select2(false, 0);

  // ④ 执行时校验：选择类型 + 草图闭合轮廓（01 §3）
  const selMgr = model.SelectionManager;
  if (selMgr.GetSelectedObjectCount2(-1) === 0 ||
      selMgr.GetSelectedObjectType3(1, -1) !== SW_SKETCH_SELECTED) {
    throw { code: 422, category: 'state_error', hint: '草图选择失败，请 sw_get_state 检查' };
  }
  const closed = { value: 0 }, open = { value: 0 };   // {value:n} 模拟 out 参数
  try {
    selMgr.GetSelectedObject6(1, -1).CheckFeatureUse(0, open, closed);
    if (closed.value === 0) throw { code: 422, category: 'state_error',
      hint: `草图无闭合轮廓（${open.value} 段开放线段），请补线或加约束后重试` };
  } catch (e: any) { if (e?.category) throw e; /* CheckFeatureUse 不可用则跳过 */ }

  // ⑤ 单位换算：mm→米、度→弧度（LLM 永远用 mm/度）
  const depthM = args.depth / 1000;
  const draftR = (args.draft ?? 0) * Math.PI / 180;
  const featureMgr = model.FeatureManager;

  // ⑥ 五级 COM 回退链（winax 13+ 参数限制的系统性应对；链序与现有仓库 api.ts 一致）
  //    注意：FeatureExtrusion3 官方签名 24 参，winax ≥13 参数下大概率直接抛错——
  //    保留它只为链路完整，真实工作预期发生在 WithFeatureData 或 VBA 级
  let feature: any = null, lastErr: unknown = null;
  const singleDir = args.endCondition !== 'MidPlane';
  const isFirst = isFirstExtrusion(model);        // 倒序扫特征树判断（复用 findLatestSketch 同款遍历）
  for (const attempt of [
    () => featureMgr.FeatureExtrusion3(           // ① 24 参（此调用预期失败，占位说明参数布局）
      COM.bool(true),                             //    Sd 单方向 = true（仓库实测恒真）
      COM.bool(false), COM.bool(!singleDir),
      0, 0, depthM, 0, COM.bool(args.draft !== undefined), COM.FALSE,
      COM.bool(false), COM.FALSE, draftR, 0,
      COM.FALSE, COM.FALSE, COM.FALSE, COM.FALSE,
      COM.bool(!isFirst),                         //    Merge：首拉伸必须 false（extrusion.ts 实测）
      COM.FALSE, COM.TRUE, 0, COM.FALSE, COM.FALSE /*, T0/StartOffset/FlipStartOffset/UseFeatScope/UseAutoSelect 按官方签名补齐 */),
    () => featureMgr.FeatureExtrusion2(/* 参数递减，Merge 同上 */),
    () => featureMgr.FeatureExtrusion(/* 最老 API */),
    () => featureExtrusionWithFeatureData(featureMgr, depthM, isFirst),  // ④ 特征数据对象模式
    () => runVbaFallback(conn, args),             // ⑤ 宏兜底（见下）
  ]) {
    try { feature = attempt(); if (feature) break; }
    catch (e) { lastErr = e; }                    // 记录后继续降级
  }
  if (!feature && lastErr) throw { code: 500, category: 'permanent',
    hint: `拉伸失败（已穷尽 COM 与宏回退）：${String(lastErr)}` };

  // ⑦ finalize：结果查证——不信退出码（01 §6.1）
  //    COM 路径：feature.Name → FeatureByName 复核
  //    VBA 路径：RunMacro2 返回布尔不是特征对象，改用特征数差分 + 取最新特征名
  model.EditRebuild3();
  let name: string;
  if (feature && typeof feature.Name === 'string' && feature.Name) {
    name = feature.Name;
    if (!model.FeatureByName(name)) throw { code: 500, category: 'permanent',
      hint: `特征 ${name} 未出现在特征树中` };
  } else {
    const after = model.GetFeatureCount();
    if (after <= beforeCount) throw { code: 500, category: 'permanent',
      hint: '宏已执行但特征树未新增特征，判定失败' };
    name = String(model.FeatureByPositionReverse(0)?.Name ?? '最新特征');
  }

  return { ok: true, feature: name, state: snapshot(model) };
}

// VBA 宏兜底：拼宏 → 写临时文件 → RunMacro2→RunMacro
async function runVbaFallback(conn: Connection, args: ExtrudeArgs) {
  const vba = renderExtrusionMacro(args);          // 迁移自 macro-generator.ts
  const macroPath = path.join(os.tmpdir(), `sw_ext_${Date.now()}.swp`);
  fs.writeFileSync(macroPath, vba, 'utf-8');
  try {
    try { return conn.app.RunMacro2(macroPath, 'Module1', 'CreateExtrusion', 1, 0); }
    catch { return conn.app.RunMacro(macroPath, 'Module1', 'CreateExtrusion'); }
  } finally {
    setImmediate(() => { try { fs.unlinkSync(macroPath); } catch {} });
  }
}

function findLatestSketch(model: any): any | null {
  // 倒序最多 50 个特征，中英文类型名正则匹配：
  // /ProfileFeature|Sketch/i 测 GetTypeName2()，或 /草图/ 测 Name
  // 实现迁移自现有仓库 helpers/extrusion.ts 的 isSketchLikeFeature
  ...
}

function isFirstExtrusion(model: any): boolean {
  // 倒序扫特征树找 Extrude/拉伸/Boss/Cut 类型特征（迁移自 prepareForExtrusion 同款判断）
  ...
}

function snapshot(model: any) {
  return {
    doc: model.GetTitle(),
    path: model.GetPathName(),
    featureCount: model.GetFeatureCount(),
    units: 'mm',
  };
}
```

---

## 4. `server.ts` — Fastify 接线（`/op` → 队列 → 分发）

```typescript
import Fastify from 'fastify';
import { Connection } from './kit/connection.js';
import { OpQueue } from './queue.js';
import { opExtrude } from './ops/feature.js';

const conn = new Connection();
const queue = new OpQueue();

queue.runner = async (op, args) => {
  const dispatch: Record<string, (a: any) => any> = {
    feature: (a) => opExtrude(conn, a),           // op 名 = feature（与工具名 sw_feature 对齐）
    // sketch_entity / measure / validate / export ... 其余复合操作
  };
  const fn = dispatch[op] ?? ((a) => { throw { code: 404, hint: `unknown op ${op}` }; });
  return withComRetry(() => fn(args));   // 可恢复 COM 错误指数退避（迁移自 error-recovery.ts）
};

const app = Fastify({ logger: true });

// Bearer 鉴权（01 §1 硬要求，参考实现不省略安全件）
app.addHook('onRequest', async (req, reply) => {
  if (req.headers.authorization !== `Bearer ${process.env.SW_TOKEN}`) {
    return reply.code(401).send({ ok: false, hint: 'unauthorized' });
  }
});

app.post('/op', async (req, reply) => {
  const { op, args = {}, expectedGen } = req.body as any;
  if (!conn.isAlive()) conn.connect();              // 懒重连
  try {
    return await queue.submit({
      op, args, expectedGen,
      run: () => queue.runner(op, args),
    });
  } catch (e: any) {
    return reply.code(e?.code ?? 500).send({
      ok: false, category: e?.category ?? 'permanent',
      hint: e?.hint ?? String(e), currentGen: queue.generation,
      state: session.snapshot(),                    // 错误响应也带 state（01 §3④ 承诺）
    });
  }
});

app.listen({ port: 7654, host: '127.0.0.1' });
```

---

## 5. COM 调用具体示例（最小可运行）

> 各示例自底向上：连接 → 属性/方法 → 布尔 → out 参数 → 选择 → 轮询读 → 忙重试。
> 统一约定：`COM` 来自 `src/utils/com-boolean.ts`；**长度单位米、角度弧度**（mm/度只存在于 API 边界，见各示例的换算点）。

### 5.0 前置声明（每个示例都默认有）

```typescript
// src/types/winax.d.ts —— 现有仓库已提供，使 TS 接受任意 COM 成员访问
declare module 'winax' {
  export class Object {
    constructor(progId: string);
    [key: string]: any;    // COM 成员运行期绑定
  }
  export function release(obj: any): void;
}
```

```typescript
import winax from 'winax';
import { COM } from '../utils/com-boolean.js';

// ProgID 连接：SW 已运行 → 附加；未运行 → 启动新实例（冷启动可达 30s+，连接层需容忍）
// 等价于 new winax.Object('SldWorks.Application')，见 src/solidworks/operations/connection.ts
const swApp: any = new winax.Object('SldWorks.Application');
swApp.Visible = true;              // 属性赋值：直接 . 赋值，winax 转 VARIANT
const model = swApp.ActiveDoc;     // 属性读取：无参成员不加括号
if (!model) throw new Error('无活动文档');
```

### 5.1 属性 vs 方法（读 / 写 / 调用）

```typescript
// 读属性——不加括号
const title: string  = model.GetTitle();              // 方法带括号
const featureCount   = model.GetFeatureCount();       // 方法
const sketchActive   = model.SketchManager.ActiveSketch;  // 链式属性访问

// 写属性——直接赋值
swApp.Visible = true;
model.ShowInactiveSection = false;                    // VARIANT_BOOL 由 winax 自动转换

// 方法调用——参数顺序与官方 API 一致，JS number 映射 double/long 按签名自动匹配
model.ClearSelection2(COM.TRUE);                      // True= -1（VARIANT_BOOL，见 5.2）
```

### 5.2 VARIANT_BOOL（COM 布尔是 -1/0，不是 JS true/false）

```typescript
// COM 标准：VARIANT_TRUE = -1，VARIANT_FALSE = 0
// src/utils/com-boolean.ts 提供 COM.bool / COM.TRUE / COM.FALSE / COM.fromBool
model.ClearSelection2(COM.TRUE);                      // 传 -1

const ok = model.Extension.SelectByID2(
  '前视基准面', 'PLANE',                               // 中文名：SW 会随安装语言命名基准面
  0.0, 0.0, 0.0,                                      // x/y/z（仅用于实体拾取，按名选时给 0）
  COM.bool(false),                                    // Append=false = 替换当前选择
  0, null, 0,                                         // Mark / Callout / Options
);
if (!COM.fromBool(ok)) throw new Error('选择失败');     // 返回值同样按 -1 判真
```

> ⚠️ 现有仓库内部不统一：`extrusion.ts` 的 `FeatureExtrusion3` 实测**直传 JS boolean** 可用（winax 自动转换），而 `ClearSelection2` 等走 `COM.TRUE`。P1 真机 A/B 验证后统一口径（见 §6 速查表）。

### 5.3 out 参数——用 `{ value: 0 }` 容器模拟引用传参

```typescript
// Extension.SaveAs3 签名要求传 errors/warnings 的引用；winax 惯例是传 {value:n} 容器，
// 调用后从 .value 读取（现有仓库 src/solidworks/operations/export.ts 同款写法）
const errorRef   = { value: 0 };
const warningRef = { value: 0 };
const saved = model.Extension.SaveAs3(
  'D:\\part.SLDPRT',
  0,                        // Version：0 = 当前版本
  1,                        // swSaveAsOptions_Silent = 1（静默，不弹对话框）
  null,                     // exportData
  errorRef, warningRef,     // ← out 参数容器
);
if (!COM.fromBool(saved) || errorRef.value !== 0) {
  throw new Error(`保存失败：错误码 ${errorRef.value}，警告码 ${warningRef.value}`);
}
```

`CheckFeatureUse` 的闭合轮廓校验（07 §3 的执行时校验）是同款双 out 参数：

```typescript
// 迁移自 src/solidworks/helpers/extrusion.ts selectSketchForExtrusion
const selMgr = model.SelectionManager;
const sketch = selMgr.GetSelectedObject6(1, -1);      // 索引从 1 起；-1 = 含隐藏选择
const open = { value: 0 }, closed = { value: 0 };
sketch.CheckFeatureUse(0, open, closed);              // 0 = swSketchCheckFeatureBaseExtrude
if (closed.value === 0) {
  throw { code: 422, category: 'state_error',
          hint: `草图无闭合轮廓（${open.value} 段开放线段）` };
}
```

### 5.4 选择实体——名字可能中文，多策略回退

```typescript
// 迁移自 src/solidworks/operations/sketch.ts selectStandardPlane：
// 策略1 遍历特征树比 GetTypeName2；策略2 SelectByID2 枚举中英文名
const namesByPlane: Record<string, string[]> = {
  Front: ['Front Plane', '前视基准面', '前視基準面'],
  Top:   ['Top Plane',   '上视基准面', '上視基準面'],
  Right: ['Right Plane', '右视基准面', '右視基準面'],
};
let selected = false;
for (const name of namesByPlane.Front) {
  if (COM.fromBool(model.Extension.SelectByID2(
        name, 'PLANE', 0, 0, 0, COM.bool(false), 0, null, 0))) {
    selected = true; break;
  }
}
model.SketchManager.InsertSketch(COM.bool(true));     // true = 进入草图（false = 退出，见 §3 步骤①）

// 画一个圆心在原点、半径 30mm 的圆——CreateCircleByRadius 收米！
model.SketchManager.CreateCircleByRadius(0, 0, 0, 30 / 1000);
```

### 5.5 轮询读取长任务的进度（读端不阻塞 STA 队列）

```typescript
// 干涉检查/质量属性这类"读"操作可能耗时数秒；放在队列槽内同步做会饿死后续 /op。
// 模式：入队一个 job → 立即返回 jobId → 轮询端点读进度（对应 jobs.ts，见 04 §4）
const jobId = 'j_' + crypto.randomUUID();
jobs.set(jobId, { progress: 0, phase: 'checking' });  // 先占位再返回

// 队列槽内的实际 COM 读——IMassProperty 链式取值（迁移自 mass-properties.ts）
const modeler     = swApp.GetModeler?() ?? null;
const massProps   = modeler?.CreateMassProperty?.() ?? null;
massProps?.Update?.();                                 // 重新计算（部分版本叫 Recalculate）
const massInKg    = massProps.Mass;                    // 质量（kg，标准单位无需换算）
const volumeInM3  = massProps.Volume;                  // 体积（m³）
const comPoint    = massProps.CenterOfMass;            // 返回 SAFEARRAY → winax 转为 JS 数组 [x,y,z]（米）
```

### 5.6 COM 忙时的重试——`RPC_E_SERVERCALL_RETRYLATER` 是 STA 常态

SolidWorks 正在交互（用户开着对话框、正 rebuild）时，COM 调用可能被拒：

```typescript
// 迁移自 src/utils/error-recovery.ts recoverFromCOMError（指数退避 + 抖动）
const RETRYABLE = new Set<number>([
  0x80010001,  // RPC_E_CALL_REJECTED
  0x8001010A,  // RPC_E_SERVERCALL_RETRYLATER —— SolidWorks 忙，稍后重试
  0x800706BA,  // RPC_E_SERVER_UNAVAILABLE
]);

async function comRetry<T>(fn: () => T, maxRetries = 3, baseDelay = 1000): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      return fn();                                    // COM 调用是同步的，直接调
    } catch (e: any) {
      const code = e?.code ?? e?.hresult;
      const recoverable = code !== undefined &&
        (RETRYABLE.has(code) || /busy|timeout|rejected/i.test(String(e?.message ?? e)));
      if (!recoverable || attempt >= maxRetries - 1) throw e;
      const delay = baseDelay * 2 ** attempt + Math.random() * 200;
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

// 用法：把"必须成功的一串 COM 调用"整个包进去（server.ts 的 withComRetry 即此物）
await comRetry(() => {
  model.ClearSelection2(COM.TRUE);
  feat.Select2(false, 0);
  return model.FeatureManager.FeatureExtrusion3(
    /* Sd */ true, /* Flip */ false, /* Dir */ false,
    /* T1 */ 0, /* T2 */ 0,
    /* D1 */ 0.05, /* D2 */ 0,        // 50mm → 0.05m
    /* Dchk1 */ false, /* Dchk2 */ false, /* Ddir1 */ false, /* Ddir2 */ false,
    /* Dang1 */ 0, /* Dang2 */ 0,
    /* OffsetReverse1/2 */ false, false,
    /* TranslateSurface1/2 */ false, false,
    /* Merge */ false,                // 首个基体拉伸必须 false（extrusion.ts 实测）
    /* FlipSideToCut */ false, /* Update */ true,
    /* T0 */ 0, /* StartOffset */ 0, /* FlipStartOffset */ false,
  );
});
```

> FeatureExtrusion3 官方签名 24 参，winax ≥13 参数下大概率直接抛错（§3 步骤⑥ 的五级回退正源于此）；此处完整列出参数仅作签名对照。另注意本例 Merge 直传 JS `false`（与 5.2 的注一致）。

---

## 6. 关键模式速查

| 模式 | 位置 | 说明 |
|---|---|---|
| VARIANT_BOOL | `COM.bool(false)` / `COM.TRUE` | 按 -1/0 传参。⚠️ 现有仓库内部不统一（extrusion.ts 注释称部分方法须直传 true/false）——P1 真机 A/B 验证后统一 |
| 单位换算边界 | ops 层步骤⑤ | mm→米、deg→rad 只在此发生，LLM 永远用 mm/度 |
| out 参数模拟 | `{ value: 0 }` → `.value` | winax 惯例（RunMacro2 第 5 参传字面量 0 是仓库既有写法，留观） |
| 时序内封（01 §2） | 步骤①–④ 在一个队列槽内 | sidecar 通道内不可被打断；人机残余窗口见 01 §6.3 |
| 执行时校验（01 §3） | 步骤④ | 查闭合轮廓与选择类型，不信 agent 假设（closed=0 为有意收紧的硬拒绝） |
| 五级回退 | 步骤⑥ | Extrusion3→2→1→WithFeatureData→VBA 宏（与仓库 api.ts 链序一致） |
| 首拉伸 Merge=false | 步骤⑥ | extrusion.ts 实测：基体拉伸 Merge=true 导致内核验证失败 |
| 结果查证（01 §6.1） | 步骤⑦ | COM 路径 FeatureByName；VBA 路径特征数差分 + 取最新特征名 |
| 版本分歧兼容 | 步骤① catch | InsertSketch(false) 失败换 true |
| 过期版本拒收（01 §3） | queue.drain | expectedGen 不符 → 409 stale_state |
| 懒重连 | server.ts | 每请求前 isAlive() 探测 |

省略部分（captureView 截图、SSE 事件、其余复合操作）按 [04-sidecar.md](./04-sidecar.md) §4 结构补齐；`withComRetry` 与 `renderExtrusionMacro` 直接从现有仓库 的 `error-recovery.ts`、`macro-generator.ts` 迁移。

§5 各示例与现有仓库的对应索引：

| 示例 | 主题 | 现有仓库出处 |
|---|---|---|
| 5.0 | ProgID 连接 / 属性读写 | `src/solidworks/operations/connection.ts`、`src/types/winax.d.ts` |
| 5.1 | 属性 vs 方法调用惯例 | `src/solidworks/api.ts`（通篇） |
| 5.2 | VARIANT_BOOL (-1/0) | `src/utils/com-boolean.ts` |
| 5.3 | out 参数 `{value:n}` 容器 | `src/solidworks/operations/export.ts`（SaveAs3）、`src/solidworks/helpers/extrusion.ts`（CheckFeatureUse） |
| 5.4 | 中英文双语选择回退 | `src/solidworks/operations/sketch.ts`（selectStandardPlane） |
| 5.5 | IMassProperty 链式读 | `src/solidworks/operations/mass-properties.ts` |
| 5.6 | COM 忙重试（指数退避） | `src/utils/error-recovery.ts`（recoverFromCOMError / COMErrorCodes） |
