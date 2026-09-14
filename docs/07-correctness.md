# 05 · 时序正确性保障：五层防御模型

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 覆盖原 v2.1 文档 §9：五层防御、generation 乐观并发、闭环观察、补偿工具、残余风险
> 关联：[04-sidecar.md](./04-sidecar.md)（队列/协议）、[08-com-impl.md](./08-com-impl.md)（实现代码）

---

## 0. 问题陈述与总纲

**问题**：前一个操作改变了工件状态，后序操作可能失去执行条件；并行调用可能乱序；人还可能同时手动操作 SolidWorks。

**没有银弹**（SolidWorks 没有事务 API）。正确思路不是"保证不出错"，而是**把开环脚本变成闭环控制系统**——三层保正确、两层用于错误恢复：

```
层1 串行队列        → 保证"顺序"本身不乱（确定性）
层2 复合操作        → 把脆弱序列内封成原子单元（顺序内部无懈可击）
层3 执行时校验      → 不信任 agent 的想象，动手前查真实状态（乐观并发）
层4 闭环观察        → 每步结果（截图+state）回注，失败带诊断 → LLM 重规划
层5 补偿工具        → 真出事了能修（undo / 删特征 / 重同步）
```

**一句话总结：顺序靠队列，原子靠复合，正确靠执行时校验，生存靠闭环 + 补偿。**

---

## 1. 层1：串行队列——顺序确定性

sidecar 全局 FIFO，所有 `/op` 排队执行。LLM 并行发 3 个调用 → 到 sidecar 变成严格串行。COM STA 特性反而由风险转为保障。

注意：队列只保证"到达顺序"正确。模型在一条消息里发并行调用、而其本意是顺序依赖——这是模型层的错，靠 AGENTS.md 约束 + 响应自带 state 养成串行习惯（"有依赖的操作必须分步发"，见 [05-interaction.md](./05-interaction.md) §7）。

---

## 2. 层2：复合操作——序列原子化（关键设计）

"前一步改变状态、后一步失去条件"的**最大风险源是序列内部**。现有仓库的 extrusion.ts 写了 1147 行 workaround，根源就是 MCP 层把"退出草图→选草图→验证→拉伸"拆成了可被外部打断的独立调用。

解法：**把脆弱序列内封进一个队列槽**，在队列锁内一口气执行——sidecar 通道内无法插入执行；人工操作仍可能在两次 COM 调用的间隙发生（SolidWorks 是 STA 消息泵），该残余窗口由层3 generation 版本与 SSE 对齐兜底（见 §6.3）：

```
sw_feature("boss") 入队
└─ 队列锁内原子执行：
   ① 退出草图 InsertSketch(false)
   ② 清选择 ClearSelection2(-1)
   ③ 倒序找草图特征 → Select2
   ④ 校验闭合轮廓 CheckFeatureUse     ← 任一步失败，整体失败并报告
   ⑤ FeatureExtrusion3 → 2 → 1 → WithFeatureData → VBA 五级回退
   ⑥ finalize（重建 + 结果查证 + 截图 + state）
```

**粒度原则**：凡是"中间状态无意义、后步依赖前步"的序列，一律内封进复合操作；暴露给 LLM 的每个工具都是自足的意图单元。

---

## 3. 层3：执行时校验——永不信任 agent 的世界模型

LLM 对状态的认知永远可能是旧的。每个复合操作在**执行时刻**对着真实 COM 状态重新验证前置条件（现有仓库已有现成模式）：

| 校验 | 来源（现有仓库） | 约束落点 |
|---|---|---|
| 草图是否激活 | `ensureActiveSketch` | 06 步骤① |
| 草图是否有闭合轮廓 | `CheckFeatureUse(0, open, closed)`，closed=0 即拒绝（现有仓库为告警继续，此处**有意收紧**为硬拒绝） | 06 步骤④ |
| 选中对象类型对不对 | `GetSelectedObjectType3 == 9 (swSelSKETCHES)` | 06 步骤④ |
| 是不是首个拉伸 | 倒序扫特征树找 Extrude 类型；**首拉伸 Merge 必须为 false**（extrusion.ts 实测：基体拉伸 Merge=true 导致内核验证失败） | 06 步骤⑥ |

再加 **generation 版本计数**（乐观并发控制，HTTP ETag/CAS 思想）解决"人到 SolidWorks 里动了手"的竞态：

```
① agent 读 state       → gen=5, "草图3 激活"
② 人切到别的文档/删特征 → sidecar 检测到，gen=6, SSE 推 doc_switched
③ agent 发 extrude(expectedGen=5)
④ sidecar: 5 ≠ 6 → 拒绝执行（409 stale_state），返回 currentGen + hint + 当前 state
⑤ LLM 拿到新鲜状态 → 重新规划
```

本质：**基于过期观察的操作直接拒收**，而不是执行后炸出莫名错误。

### 3.1 generation 语义约定（全文统一口径）

- **自增时机**：任何改变 SW 状态的事件（agent 操作成功、人工操作被检测到、文档切换）都 +1；`bumpGeneration()` 供变化检测调用，`job.run()` 成功且改状态后 sidecar 也自增
- **sidecar 重启**：gen 归零会导致旧 expectedGen 永远 409——state 附带 `bootId`（重启计数），LLM 可区分"过期观察"与"服务重启"
- **多文档扩展**：本文为单文档语义；多文档下升级为 globalGen + per-doc gen 二元组，"切文档不再必然拒收"，见 [12](./12-multi-workpiece.md) §4.2

### 3.2 变化检测机制（generation 与 SSE 的地基）

winax 对 SolidWorks COM 事件（连接点）支持有限，`doc_switched`/`state_changed` 实际依赖**轮询差分**：

- 轮询字段：`ActiveDoc.GetTitle()`（文档切换）+ `GetFeatureCount()`（特征增删）+ `SelectionManager.GetSelectedObjectCount2`（选择集变化）
- 轮询时机：每次 `/op` 执行前强制一次（零成本拿到最新差分）+ 空闲期低频定时（如 500ms，可配）
- 轮询间隔决定层3 检测延迟的下限；大装配下 GetFeatureCount 开销需实测——**该机制为 P0 spike 项**（[00 待验证假设](./00-OVERVIEW.md) #4）

---

## 4. 层4：闭环观察——失败也是信息

每个响应（成功/失败）都带 `state` + 截图 + 结构化诊断：

```jsonc
// 失败响应
{ "ok": false,
  "category": "state_error",        // recoverable / state_error / permanent / stale_state
  "hint": "草图无闭合轮廓，无法拉伸。当前草图3含2条开放线段，建议补线或加闭合约束",
  "state": { "generation": 6, "activeSketch": "草图3", ... },
  "screenshot": "..." }
```

模型看到的是"操作失败 + 为什么 + 现在什么样"，自然走重规划而不是盲目重试。可恢复类错误（RPC_E_CALL_REJECTED 等）sidecar 内部指数退避重试，无需 LLM 参与。

视觉闭环的"单源复用"（提供给 agent / 推送给面板 / 展示于审批框）见 [05-interaction.md](./05-interaction.md) §4 与 [03-architecture.md](./03-architecture.md) §4.3。

---

## 5. 层5：补偿——真出事了的修复工具

| 工具 | 机制 | 场景 |
|---|---|---|
| `sw_undo` | `IModelDoc2.EditUndo2(n)` | agent 建错特征，回退 |
| `sw_delete_feature` | 按名删除特征 | 外科手术式撤销（比 undo 精确） |
| `sw_get_state` | 强制重同步 | LLM 迷失时自愈 |

---

## 6. 诚实的残余风险

1. **部分应用**：VBA 宏回退执行到一半失败——宏无返回值。缓解：宏设计成"先验证后执行"，sidecar 执行后**查证结果**——以执行前后 `GetFeatureCount()` 差分 + 取最新特征名（或宏内将新特征重命名为固定标记后 `FeatureByName`），不信退出码，查实了才报成功（见 06 步骤⑦）
2. **没有真事务**：两个 `sw_feature` 之间不存在原子性，第二个失败时第一个已生效——靠 undo/delete 补偿，不承诺回滚
3. **generation 拒收的粒度**：防的是"过期观察"，防不了"校验通过后、执行前的微秒级竞态"——该窗口在队列锁内，只有人直接操作 SW 才可能触发，靠 SSE 事后对齐兜底
