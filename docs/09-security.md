# 07 · 安全与权限模型

> 所属文档集：[00-OVERVIEW](./00-OVERVIEW.md)
> 覆盖原 v2.1 文档 §11：网络面、权限矩阵、审批 UX

---

## 1. 网络面

- sidecar、opencode 服务均绑 **127.0.0.1**，各自认证凭据（主程序生成，env 注入）：sidecar 用 Bearer token（`SW_TOKEN`）；opencode 用 `OPENCODE_SERVER_PASSWORD`（Basic Auth）
- 主程序是凭据的唯一分发者；Renderer 经 Main 代理访问，不落盘明文
- 未来局域网模式（SW 在工作站）：sidecar 加 mTLS/P2P 通道，P3 再议

### 1.1 进程/端口/令牌约定

| 进程 | 监听 | 令牌 | 父进程 |
|---|---|---|---|
| sidecar | 127.0.0.1:7654 | `SW_TOKEN`（env） | 主程序 Main |
| opencode 服务 | 127.0.0.1:随机（SDK 发现） | `OPENCODE_SERVER_PASSWORD` | 主程序 Main |
| Electron Renderer | — | 无（经 Main IPC 代理） | — |

---

## 2. 权限矩阵（opencode permission，审批走原生对话框）

> ⚠️ opencode 权限规则按**最后匹配生效**：宽泛规则（`"*"`）必须放**最前**，具体规则放后面，否则 allow/deny 会被兜底规则覆盖。自定义工具是否支持按参数/路径的模式匹配未经官方文档确认（[00 待验证假设](./00-OVERVIEW.md) #5），故**目录白名单下沉到 sidecar 强制校验**，opencode 层只做工具级 ask。

```jsonc
{
  "permission": {
    "*": "ask",                                  // 全局默认收紧（opencode 默认大多 allow，不写即静默放行）
    "sw_get_state": "allow",  "sw_measure": "allow",
    "sw_validate": "allow",   "sw_assert": "allow",
    "sw_job": "allow",        "sw_list_docs": "allow", "sw_bind_session": "allow",
    "sw_new_doc": "allow",    "sw_open_doc": "allow",
    "sw_sketch": "allow",     "sw_sketch_entity": "allow",
    "sw_sketch_mod": "allow", "sw_feature": "allow",
    "sw_feature_mod": "allow","sw_reference": "allow",
    "sw_assembly": "allow",   "sw_drawing": "allow",
    "sw_undo": "allow",
    "sw_delete_feature": "ask",
    "sw_close_doc": "ask",
    "sw_save_doc": "ask",     // 目录白名单由 sidecar 强制（见 §2.1）
    "sw_export": "ask",
    "sw_macro": "ask",
    "bash": "deny",           // CAD agent 无需 shell；彻底杜绝绕过 sidecar 直连 COM
    "edit": "ask",
    "read": "allow", "glob": "allow", "grep": "allow", "list": "allow",
    "todowrite": "allow"
  }
}
```

### 2.1 分级原则

- **读取/测量/校验/建模放行**（可撤销：SW 特征树本身就是 undo 机制）
- **落盘/宏/删除/关闭 ask**——工作目录白名单（如 `D:/cad-work/**` 内静默放行）由 **sidecar ops 层强制校验**：它是唯一落盘通道，在 sidecar 校验比在 opencode 层更可靠（双保险，即使 opencode pattern 粒度不支持路径匹配也不失效）
- **bash 整体 deny**：CAD agent 场景无 shell 需求；deny-by-default 消除清单式绕过风险（`cmd /c`、`cscript`、`wscript`、`mshta` 均能起 COM，逐条 deny 防不胜防）
- 规则顺序：`"*"` 全局默认永远放**第一条**，具体放行/收紧规则放后（最后匹配生效）

### 2.2 按 agent 覆写

- `cad-inspector`（只读测量 subagent）：仅 `sw_get_state` / `sw_measure` / `sw_validate` / `sw_assert` / `sw_job` allow，其余 deny
- `drafting`（工程图 subagent）：放开 `sw_drawing`，落盘类仍 ask

### 2.3 审批 UX（主程序原生对话框）

- 审批框必须给足上下文：**工具名 + 参数 diff**（如 `sw_save_doc → D:\proj\part1.SLDPRT`）+ **操作前截图**
- 三选：allow（本次）/ deny / always allow（**仅本会话内按 pattern 放行**——opencode 的 always 不落盘；持久化由主程序收到应答后写回 workspace 的 opencode.json）
- 数据流：opencode permission 事件 →（通道A）→ 主程序弹原生对话框 → 应答回传

---

## 3. 其他安全考量

- **宏工具（sw_macro）默认 ask**：VBA 宏可执行任意代码，等同 bash 权限等级
- **sidecar 不提供任意 COM 透传端点**：只暴露白名单复合操作，无 `executeRaw` 之类的兜底通道（防止越权调用破坏性 API）
- **日志脱敏**：token 不入日志；截图/模型路径按用户隐私策略处理
