# 项目记忆资产体系（v2）

一套用户级的 opencode 资产，按 **19 条项目记忆资产设计原则**为任意项目建立、维护、审计专属的记忆库。

---

## 1. 解决什么问题

通用 AI Agent 在面对具体项目时反复犯同样的错——**因为没有把项目专属的"代码推不出来的知识"沉淀下来**：

- 团队为什么这样设计 / 不能怎样做 / 踩过什么坑
- 跨多个文件才能拼出的业务逻辑
- 看似合理但行不通的方案
- AI 在本项目里的能力边界与禁区

本仓库的资产**强制**以原则约束记忆库的"准入、结构、迭代"，避免记忆库变成垃圾堆。

---

## 2. 19 条原则速览

| # | 原则 | 落地点 |
|---|---|---|
| 1 | 不可推导性原则 | `/remember` 过滤 1 + `init-project` 不问可推导事 |
| 2 | 探索成本阈值原则 | `failure-loop` 触发条件之一 |
| 3 | 不变性优先原则 | 条目模板强制"为什么"字段 |
| 4 | 痛苦换来的知识优先 | `init-project` Phase 2 C 组 + `gotcha` 类型 |
| 5 | AGENTS.md 作为唯一入口 | AGENTS.md § 4 文档索引 |
| 6 | 摘要-详情分层 | 索引写"何时读"而非"写了什么" |
| 7 | 模块边界对齐 | `docs/agent/<module>.md` 按代码模块组织 |
| 8 | 可发现性原则 | 每个 doc 头部 frontmatter（触发条件/最后更新/关联代码） |
| 9 | 用户驱动写入 | 所有 Skill 只建议、不写盘；写盘必须经 `/remember` 等命令 |
| 10 | 写入前 dedupe | `/remember` 过滤 3、4 |
| 11 | 衰减与审计 | 时间戳字段 + `/audit-memory` 命令 + `memory-auditor` Skill |
| 12 | 原子化条目 | 条目模板（ID + 字段 + 内容） |
| 13 | 项目类型识别 | AGENTS.md § 1 项目元信息 |
| 14 | 能力清单与禁区 | AGENTS.md § 2 |
| 15 | 工作流钩子 | AGENTS.md § 3 |
| 16 | 优先 Skill 而非 Command | `memory-entry-writer` / `memory-auditor` Skill 共享逻辑 |
| 17 | 写入有模板，读取无模板 | 强制条目结构（写入时） |
| 18 | 失败回路 | `failure-loop` Skill |
| 19 | 不放秘密、不放快变数据 | `/remember` 过滤 2 |

---

## 3. 整体架构

```
用户级安装（~/.config/opencode/）          项目级产出（项目根/）
─────────────────────────────              ──────────────────────
commands/                                  AGENTS.md（4 段固定结构）
  init-project                                § 1 项目元信息
  remember                                    § 2 能力清单与禁区
  forget                                      § 3 工作流钩子
  update-memory                               § 4 文档索引（何时读）
  audit-memory                                § 5 给 Agent 的元规则
                                           
skills/                                    docs/agent/
  project-context-bootstrap (哨兵)            _global.md（G-NNN 全局条目）
  project-memory-assistant (建议)             <module-A>.md（M-NNN 模块条目）
  failure-loop (建议)                         <module-B>.md
  memory-entry-writer (共享逻辑)              ...
  memory-auditor (共享逻辑)
```

### 3.1 用户级文件布局

```
~/.config/opencode/
├── commands/
│   ├── init-project.md       # /init-project — 首次生成记忆库
│   ├── remember.md           # /remember <内容> — 追加单条原子条目
│   ├── forget.md             # /forget <ID> — 删除条目
│   ├── update-memory.md      # /update-memory <ID> — 更新条目
│   └── audit-memory.md       # /audit-memory — 审计陈旧条目
└── skills/
    ├── project-context-bootstrap/
    │   ├── SKILL.md          # 哨兵：检测缺失 AGENTS.md 时建议跑 /init-project
    │   └── templates/
    │       ├── AGENTS.md.tmpl      # 4 段结构根模板
    │       ├── _global.md.tmpl     # 全局条目模板
    │       └── _module.md.tmpl     # 模块条目模板
    ├── project-memory-assistant/
    │   └── SKILL.md          # 对话进行中建议跑 /remember（仅建议、不写盘）
    ├── failure-loop/
    │   └── SKILL.md          # 任务结束时自检"早知道就好"的教训
    ├── memory-entry-writer/
    │   └── SKILL.md          # 原子条目的"如何写"共享规范
    └── memory-auditor/
        └── SKILL.md          # 审计陈旧条目的共享逻辑
```

### 3.2 职责分工

| 组件 | 类型 | 触发 | 职责 |
|---|---|---|---|
| `init-project` | Command | 用户显式 `/init-project` | 5 阶段访谈 + 模块识别 → 生成 AGENTS.md（4 段）+ `_global.md` + 各模块 doc |
| `remember` | Command | 用户显式 `/remember <内容>` | 过滤（推导/秘密/重复/冲突）→ 原子条目 → 用户确认 → 追加 |
| `forget` | Command | 用户显式 `/forget <ID>` | 二次确认 → 删除条目（保留 git 痕迹） |
| `update-memory` | Command | 用户显式 `/update-memory <ID>` | 展示当前 → 应用修改 → 自动刷新"最后验证" |
| `audit-memory` | Command | 用户显式 `/audit-memory` | 扫描 5 类问题 → 报告 → 用户处置 |
| `project-context-bootstrap` | Skill | 自动 | 无 AGENTS.md 时提示用户跑 `/init-project` |
| `project-memory-assistant` | Skill | 自动 | 对话中用户陈述项目级事实时**建议** `/remember`，不写盘 |
| `failure-loop` | Skill | 自动 | 任务结束时自检"早知道就好"的教训，**建议** `/remember` |
| `memory-entry-writer` | Skill | 被命令调用 | 原子条目格式规范 + 过滤逻辑（共享） |
| `memory-auditor` | Skill | 被命令调用 | 审计扫描的 5 类问题判定（共享） |

**为什么 Skill 与 Command 分工？**（原则 16）

- **Command** = 轻量入口、用户触发、做一件具体的事。`/remember` 用户每次记一条用。
- **Skill** = 共享逻辑、可被多入口复用、避免格式漂移。`memory-entry-writer` 定义"如何写一条合规条目"，被 `/remember` 和 `/update-memory` 都引用。

---

## 4. 安装方式

仓库结构对齐 `~/.config/opencode/`。

### 4.1 仓库结构

```
opencode-context-init/
├── README.md
├── commands/
│   ├── init-project.md
│   ├── remember.md
│   ├── forget.md
│   ├── update-memory.md
│   └── audit-memory.md
└── skills/
    ├── project-context-bootstrap/
    │   ├── SKILL.md
    │   └── templates/
    │       ├── AGENTS.md.tmpl
    │       ├── _global.md.tmpl
    │       └── _module.md.tmpl
    ├── project-memory-assistant/SKILL.md
    ├── failure-loop/SKILL.md
    ├── memory-entry-writer/SKILL.md
    └── memory-auditor/SKILL.md
```

### 4.2 安装步骤

**方案 A：直接复制**

```powershell
# Windows PowerShell
git clone https://github.com/daydreamer258/opencode-context-init.git
cd opencode-context-init
Copy-Item -Recurse commands "$env:USERPROFILE\.config\opencode\"
Copy-Item -Recurse skills   "$env:USERPROFILE\.config\opencode\"
```

```bash
# macOS / Linux
git clone https://github.com/daydreamer258/opencode-context-init.git
cd opencode-context-init
cp -r commands ~/.config/opencode/
cp -r skills   ~/.config/opencode/
```

**方案 B：符号链接**（仓库更新自动生效）

```powershell
# Windows PowerShell（需管理员或开发者模式）
$repo = "$PWD"
foreach ($cmd in @("init-project","remember","forget","update-memory","audit-memory")) {
    New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\commands\$cmd.md" -Target "$repo\commands\$cmd.md"
}
foreach ($skill in @("project-context-bootstrap","project-memory-assistant","failure-loop","memory-entry-writer","memory-auditor")) {
    New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\skills\$skill" -Target "$repo\skills\$skill"
}
```

```bash
# macOS / Linux
for cmd in init-project remember forget update-memory audit-memory; do
    ln -s "$PWD/commands/$cmd.md" ~/.config/opencode/commands/$cmd.md
done
for skill in project-context-bootstrap project-memory-assistant failure-loop memory-entry-writer memory-auditor; do
    ln -s "$PWD/skills/$skill" ~/.config/opencode/skills/$skill
done
```

### 4.3 验证安装

启动 opencode，输入 `/init-project` 看是否识别。

---

## 5. 使用流程

### 5.1 新项目首次接入

```
1. cd 到项目根，启动 opencode
2. 哨兵 Skill 自动提示："这个项目没有 AGENTS.md，建议跑 /init-project"
3. 跑 /init-project
4. 走完 5 阶段访谈（Discover → Interview → 模块确认 → Draft → Review → Finalize）
5. 产物落盘：AGENTS.md + docs/agent/_global.md + docs/agent/<module>.md × N
```

访谈中重点：
- A 组：**验证**自动扫描的发现（不问可推导事）
- B 组：能力清单 + 禁区
- C 组：**痛苦换来的知识**（最高价值）
- D 组：工作流钩子（"做 X 前先读 Y"）

### 5.2 日常增量记忆

四种途径，**全部用户驱动**（原则 9）：

| 场景 | 命令 |
|---|---|
| 你想记一条事实/决策/陷阱 | `/remember <内容>` |
| 你想删一条过时记忆 | `/forget <ID>` 或 `/forget <关键词>` |
| 你想更新一条记忆并刷新验证时间 | `/update-memory <ID>` |
| 定期检查记忆库健康度 | `/audit-memory` |

Agent 自动行为（**仅建议**）：

- 对话中你陈述项目级事实 → `project-memory-assistant` 建议你跑 `/remember`
- 任务结束 Agent 自检 → `failure-loop` 建议你把教训记下来
- 任何写盘**必须**由用户显式跑命令

---

## 6. 产物结构详解

### 6.1 AGENTS.md（4 段固定结构）

```markdown
# 项目名

> 一句话描述

## § 1 项目元信息
（语言、框架、布局、Agent 工作模式偏好）

## § 2 能力清单与禁区
（✅ 可做 / ❌ 不可做 / ⚠️ 需确认）

## § 3 工作流钩子
（识别意图 → 必读文档表）

## § 4 文档索引
（每个 docs/agent/*.md 的"何时读"摘要）

## § 5 给 Agent 的元规则
（不可推导优先、不擅自写记忆、任务结束自查等）
```

### 6.2 `docs/agent/_global.md`

跨模块的全局约定、数据/API 约定、历史决策、SOP。原子条目编号 `G-NNN`。

### 6.3 `docs/agent/<module>.md`

按代码模块边界组织（原则 7）。每个模块文件包含：
- 模块设计与决策（`decision`）
- 约束与边界（`constraint`）
- 陷阱与历史坑（`gotcha`，**最有价值**）
- 本模块 SOP（`sop`）

原子条目编号 `M-NNN`（每个模块独立计数）。

### 6.4 原子条目格式（原则 12、17）

每条记忆是独立的、可单独增删改的"事实"：

```markdown
### M-003: OAuth refresh 并发死锁  [gotcha]
- 关联代码: src/auth/oauth.ts:42-58
- 新增: 2026-05-26
- 最后验证: 2026-05-26
- 标签: oauth, race-condition, auth

**事实**: 同一 client_id 并发 token refresh 会触发 provider 端死锁
**为什么**: provider 对 client_id 加锁，并发请求互相等待
**反例**: 用 Promise.all 并发刷新——会死锁
```

字段说明见 `skills/memory-entry-writer/SKILL.md`。

---

## 7. 写入闸门：4 道过滤（在 `/remember` 内）

任何写入都要过 4 道闸门。**任一报警都先告知用户，不默默 ban**。

| 过滤 | 拒绝什么 | 对应原则 |
|---|---|---|
| 1. 可推导性 | 能从代码/配置/目录直接读出来的事实 | 1 |
| 2. 秘密/快变 | 密钥、token、当前版本号、临时联系人 | 19 |
| 3. dedupe | 与现有条目语义相同 | 10 |
| 4. 冲突 | 与现有条目事实矛盾（先 supersede 或 coexist） | 10 |

---

## 8. 设计取舍

| 抉择 | 选择 | 原则 |
|---|---|---|
| 跨工具策略 | 只生成 `AGENTS.md`（opencode 原生读，跨工具事实标准） | — |
| 文档组织 | 按代码模块（`<module>.md`），不按类别（gotchas/conventions/...） | 7 |
| 写入触发 | 用户显式命令；Skill 只建议不写盘 | 9 |
| 条目格式 | 原子化、ID 编号、强制时间戳/标签/类型 | 12、17 |
| 时间戳维护 | 手动字段（`新增` + `最后验证`），由命令自动刷新 | 11 |
| 模块识别 | 自动检测 + 用户确认补充（hybrid） | 7 |
| 命令 vs Skill | 命令做入口；Skill 承载共享逻辑（写入规范、审计逻辑、失败回路） | 16 |
| 内容过滤 | 4 道闸门（不可推导/秘密快变/dedupe/冲突） | 1、10、19 |
| 自演化 | 三档建议（哨兵 / 对话中 / 任务结束），但**写盘永远显式** | 9、18 |

### 与 v1 的关键差异

| v1 | v2 |
|---|---|
| 5 个固定标准文档（overview/build-and-run/conventions/gotchas/boundaries） | 推翻——主题变成模块文档内的章节；跨模块的归 `_global.md` |
| 6 个可选专题（verification 等） | 推翻——按需在模块文档或 `_global.md` 内开章节 |
| AGENTS.md "元规则 4：主动追加" | 推翻——Skill 只建议，不写盘 |
| 记忆助手 Skill 询问"我帮你记到 X 吗" | 软化——只建议跑 `/remember`，不询问代写 |
| `/remember` 简单分类 + 写入 | 升级——4 道过滤 + 原子条目 + 共享逻辑 |
| 无 `/forget`、`/update-memory`、`/audit-memory` | 新增三命令支持原则 9、11 |
| 无任务结束自检 | 新增 `failure-loop` Skill 支持原则 18 |
| 散落的边界（boundaries.md） | 集中到 AGENTS.md § 2 能力与禁区 |
| 散落的 SOP（多文档） | 集中到 AGENTS.md § 3 工作流钩子（指引）+ 模块文档 § SOP（步骤） |

---

## 9. 扩展与定制

需要时：

- **加新命令**：在 `commands/` 加 `.md`
- **加新 Skill**：在 `skills/<name>/SKILL.md`
- **改条目格式**：编辑 `skills/memory-entry-writer/SKILL.md` 一处即可（被多命令复用）
- **改审计规则**：编辑 `skills/memory-auditor/SKILL.md`

---

## 10. 验收标准

1. 任意空项目跑 `/init-project`，能识别项目类型 + 模块，生成 AGENTS.md（4 段）+ `_global.md` + N 个模块 doc
2. `/remember` 在 4 道过滤上都能正确报警（试录入可推导事实、密钥、重复条目、冲突条目）
3. `/forget <ID>` 能定位并二次确认后删除
4. `/update-memory <ID>` 修改后"最后验证"自动刷新，"新增"日期不动
5. `/audit-memory` 能扫出 STALE_CODE / STALE_TIME / DERIVABLE / INCOMPLETE_WHY / DUPLICATE 5 类问题
6. 哨兵 Skill 无 AGENTS.md 时提示一次
7. `project-memory-assistant` 在对话中用户陈述项目级事实时建议 `/remember`，**绝不**自己询问代写
8. `failure-loop` 在任务结束时自检，无候选时静默
9. 全部产物中文
