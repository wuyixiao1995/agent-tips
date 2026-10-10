---
name: session-continuity
title: 会话连续性（收尾 / 继续）
display_name: 长任务接力
display_name_en: Session Continuity
description: |
  长任务在一个 session 里会变蠢（context rot），跨 session 又容易失忆。本 skill 提供一套
  「收尾 / 继续」会话连续性机制：收尾把当前进度压成可交接文档（5段式简报 + 交接件 + 日志 + 记忆），
  继续把状态接力到下一 session。逻辑层架构通用（任何带持久注入记忆 + 长 session 的 agent 都适用），
  介质层以 WorkBuddy 为示例、可改。触发词可自定义。
description_zh: |
  一个 session 聊太长会变蠢？遇到大活一个session干不完怎么办？长任务接力让你把长任务切成一段段干净的对话来做：每次收工留下清楚的交接单（做了啥、定了啥、下一步干啥），下次开工照着接上就能干，不用从头讲、不丢上下文、不重来。用它就把它切成几段，每段都清爽。
description_en: |
  Long tasks go dumb inside one session (context rot) and lose memory across sessions. This skill provides
  a "closeout / continue" session-continuity mechanism: closeout compresses progress into a handoff doc
  (5-section briefing + handoff file + daily log + memory update); continue relays state to the next session.
  The logic layer is architecture-agnostic; the medium layer uses WorkBuddy as an example and is editable.
  Trigger words (closeout / continue) are customizable.
version: 1.0.0
author: 甘晓东
trigger:
  - 收尾
  - 交接
  - handoff
  - 继续
  - continue
  - 收尾完成
  - 会话交接
agent_created: true
---

# 会话连续性（收尾 / 继续）

> 一句话：长任务在一个 session 里待久了会「变蠢」（上下文腐烂），跨 session 又容易「失忆」（记忆没接上）。本 skill 用「收尾 + 继续」把长任务切成多个干净 session，进度不丢、状态可接力。

> ▶ **加载即播报（主动报）**：本 skill 每次被触发加载，agent 在回复开头**主动用一句话告知用户**——
> 「『收尾 / 继续』的触发词可自定义，你只需让我改本 skill 顶部变量即可，无需你手动编辑文件。」
> 目的：让用户明确知道，换词 / 换介质都是「驱动 agent 自己改」的事，不是用户的技术负担。（详见 §3）

---

## 与「注入层治理」的关系（一套栈，不是两个工具）

- **地基**：`injection-layer-governance`（已发布：https://skillhub.cn/skills/user_f6d0b8c5/injection-layer-governance）保住**持久记忆层不溢出**——MEMORY / USER / rules 不被静默截断。
- **上层建筑**：本 skill 靠读 MEMORY + 交接件把状态接力到下一 session。
- **关键**：地基烂了，继续接起来也是接进一个**失忆的 agent**——因为继续读的 MEMORY 本身已被截秃。所以注入层治理是「继续」可靠的**前置条件**。两者是**垂直栈**，不是并列工具。

## 介质层 vs 逻辑层（给换介质的用户）

- **逻辑层（本 skill §1）= 架构通用**：任何带「持久注入记忆 + 长 session」结构的 agent 都适用（OpenClaw 架构是其中一家，不是唯一）。不含任何介质专属路径。
- **介质层（本 skill §2）= WorkBuddy 专用示例**：只放 WB 的文件路径 / 截断阈值 / 脚本调用。**换介质时，只改 §2，不动 §1。**
- 为什么这么分：逻辑不会过期，介质会。OpenClaw 等同类用户自己驱动 agent 改 §2 即可，本 skill 不替你绑死 WB。

---

## 0. 何时用 / 何时不用

**用**：
- 一段长任务要结束当前 session、留给未来的自己 / 另一个 agent 接续 → **收尾**
- 新 session 开始、要接回上次进度 → **继续**
- 用户主动说「收尾 / 继续」（或自定义词）

**不用**：
- 只是单轮问答、无跨 session 状态 → 不需要
- 记忆本身要治理（截断 / 瘦身）→ 那是 `injection-layer-governance`，不是本 skill

---

## 1. 逻辑层（架构通用 · 不依赖任何介质）

### 1.1 收尾协议（Closeout）

目标：把当前 session 的进度压成**自包含交接件**，未来只读这一份就能接续，不需回读对话、不需重跑脚本。

**六步（顺序不可反）**：
1. **追加当日日志**：在当日日志文件（daily log）append 一行本次做了什么、关键决策、未决项。
2. **写交接件【本session】段**：每次收尾都更新。结构见 §1.3。
3. **如是主线任务 → 同步更新交接件【主线状态】段**：让「继续」有准星。
4. **按需更新记忆文件**（项目级或用户级）：只沉淀跨任务可复用的条目，不搬过程数据。
5. **回复中输出 5 段式简报（不可省略）**：目标 / 决定 / 规则（引用用户原话）/ 状态 / 下一步。
6. **告知用户「收尾完成，session 可结束」**。

**收尾硬检查点**：收尾完成前，运行一个检查脚本，各项全绿才可宣布「收尾完成」。检查项含：交接件是否更新、日志是否追加、记忆是否在阈值内、有无新挂起项未双写。

### 1.2 继续协议（Continue）—— 从零定义

目标：新 session 无缝接回上次进度，用户不需重新交代背景。

**四步**：
1. **识别模式**：裸「继续」= 回主线；「继续 [任务名]」= 接特定任务。
2. **取权威源**：主线状态在交接件【主线状态】；特定任务扫描日志找该任务最近记录。
   - ⚠️ 若系统已在开场注入记忆 + 交接件【主线状态】，**不重复 Read**（冗余 re-Read 属浪费、且易越界）。
3. **上下文纪律**：以注入/读取到的【主线状态】为节点准星，找「下一步」动作；**不拉跨空间 / 跨线的记忆与定时任务**（防止越出当前容器边界）。
4. **输出就绪摘要**：先给「已做 / 卡点 + 待厘清问题（开放式）」，**不高频抛选择题**；仅当用户明确要选项才列。

### 1.3 交接件结构（Handoff doc）

| 段 | 含义 | 何时写 |
|---|---|---|
| 【主线状态】 | 主线的当前节点 + 下一步 | 仅主线收尾时更新 |
| 【本session】 | 本次 session 做了什么、卡点、未决 | 每次收尾都更新 |

**5 段式简报字段**：目标 / 决定 / 规则（引用原话）/ 状态 / 下一步。

**引用不重复**：交接件只放「上下文浓缩 + 指路（路径/URL）」，不抄 specs / 计划树 / 日志原文。

**脱敏**：API key、密码、PII 一律 `<REDACTED>` 占位。

---

## 2. 介质适配层（WorkBuddy 专用示例 · 换介质改这里）

> ⚠️ 以下全部是 WB 专属。换介质（如 OpenClaw）时，**只改本节**，把路径 / 阈值 / 脚本换成你的介质等效物。

| 项 | WB 示例 | 换介质改哪 |
|---|---|---|
| 项目级记忆 | `<workspace>/.workbuddy/memory/MEMORY.md` | 你的项目记忆路径 |
| 项目级日志 | `<workspace>/.workbuddy/memory/YYYY-MM-DD.md`（append-only） | 你的日志路径 |
| 用户级记忆 | `~/.workbuddy/MEMORY.md` | 你的用户记忆路径 |
| 交接件 | `<workspace>/HANDOFF.md`（含【主线状态】/【本session】） | 你的交接件路径/名称 |
| 截断阈值 | 项目级记忆 8000 / 用户级记忆 4000 / 身份三件套 10000 / 规则 40000 / 个人指令 1500（字符数） | 你的介质上限 |
| 收尾硬检查点脚本 | `C:\Users\admini\.workbuddy\skills\check_closeout.sh`（7 项检查） | 你的等效检查 / 或省略 |
| 路径分隔符 | Windows 反斜杠（Git Bash 下用正斜杠） | 你的 OS 分隔符 |

**调用示例（WB）**：
```bash
bash C:/Users/admini/.workbuddy/skills/check_closeout.sh
```

**注意**：WB 的记忆文件每轮开局自动注入——这是「收尾/继续」在 WB 上「每轮都在场」强制力的来源。换介质若没有自动注入机制，需自行保证记忆/交接件在 session 开头被读取。

---

## 3. 自定义触发词（主动报）

**默认触发词**（本 skill frontmatter `trigger`）：收尾 / 交接 / handoff / 继续 / continue。

**可自定义**：这两个动作的词不是写死的。用户可用自己习惯的词（如「交班」「接着干」）。

**主动报动作（agent 必做）**：
> 本 skill 每次被触发加载，agent 在回复开头**主动用一句话告知用户**：
> 「『收尾 / 继续』的触发词可自定义——你只需让我改本 skill 顶部变量即可，无需你手动编辑文件。」
> 目的：让用户明确知道，换词 / 换介质都是「驱动 agent 自己改」的事，不是用户的技术负担。

**怎么改**：让 agent 编辑本 skill SKILL.md 顶部 `trigger` 列表，换成你的词即可。WB 上建议**保留默认词作路由 hint**，自定义词靠「告诉 agent 映射关系」兜底（平台路由靠 frontmatter trigger 点火，生造词不会自动触发）。

---

## 4. 红线（来自协同纪律，仅指针）

- **说停即冻结**（停止纪律 §1）：用户说停，立即冻结全部工具动作。
- **禁止伪造用户输入**（停止纪律 §10）：「用户说 X」必须能指认原话。
- **声称即负债**：宣布「已扫描 / 完整」前必须有命令证据。
- **选项生成三闸门**：抛选项前过范围 / 铁律 / 单一性闸。

> 以上为跨项目硬纪律，完整定义见用户级 `rules/` 与 `MEMORY.md`，本 skill 不重述。

---

## 5. 版本记录

| 版本 | 变更 |
|---|---|
| 1.0.0 | 初版。从 handoff 薄壳 + 记忆协议蒸馏为自包含 skill；收尾逻辑层与介质层分离；继续协议从零定义；触发词可自定义 + 主动报。与 injection-layer-governance 互指为栈。原 handoff skill 转为弃用重定向桩。 |
