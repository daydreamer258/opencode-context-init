---
description: 通过交互式访谈，为当前项目生成 v2 项目记忆体系：AGENTS.md（4 段结构）+ docs/agent/_global.md + 按模块组织的 docs/agent/<module>.md。遵守 19 条记忆资产原则。全中文输出。
---

# /init-project — 项目记忆初始化器 v2

为当前项目建立**符合 19 条原则**的记忆资产：
- 入口 `AGENTS.md`：4 段固定结构（元信息 / 能力与禁区 / 工作流钩子 / 文档索引）
- 全局 `docs/agent/_global.md`：跨模块约定、数据/API 约定、历史决策
- 各模块 `docs/agent/<module>.md`：按代码模块边界组织，原子条目格式

**核心原则贯穿始终：**
- 只问**模型推断不出的事**——能从代码读的不问、不写（原则 1）
- 优先记**为什么/不能怎样/踩过什么坑**，不记"代码现在长什么样"（原则 3）
- **痛苦换来的知识优先**（原则 4）
- 每条记忆是**原子条目**（原则 12）

---

## Phase 0：前置检查

1. 检查 `AGENTS.md` 是否存在。
   - **存在**：告知用户"已检测到现有 AGENTS.md。是否：
     - `继续`：保留现有，本次访谈结果作为增量内容由你之后用 `/remember` 追加
     - `重建`：备份为 `AGENTS.md.bak`、`docs/agent/` 整体备份为 `docs/agent.bak/`，重新生成
     - `取消`：退出"

2. 用户选定后再进入 Phase 1。

---

## Phase 1：Discover（静默扫描）

不打扰用户，自动收集。**每个发现都要记录"证据"——具体哪个文件哪行**，方便 Phase 2 验证。

### 1.1 项目类型与元信息

- 主语言：根据 `pom.xml` / `package.json` / `Cargo.toml` / `go.mod` / `requirements.txt` / `pyproject.toml` / `Gemfile` / `*.csproj` 等判定
- 主框架：扫描依赖关键字（`spring-boot-starter` → Spring Boot；`react` → React；`fastapi` → FastAPI 等）
- 包管理/构建工具：从上述文件推断
- 仓库布局：扁平 / monorepo（多个 `package.json` 或 `pnpm-workspace.yaml`）/ 多模块（多个 `pom.xml`）

### 1.2 模块自动检测（关键）

按以下顺序尝试识别模块边界（原则 7）：

1. **monorepo workspace**：`packages/*`、`apps/*`、`pnpm-workspace.yaml`、`lerna.json`
2. **多模块 Maven/Gradle**：根 `pom.xml` 里的 `<modules>` 段、子目录的 `pom.xml`
3. **Python 多 package**：`src/<pkg>/` 各自有 `__init__.py`
4. **按目录约定**：`src/<module>/`、`internal/<module>/`、`app/<module>/`、`cmd/<binary>/`
5. **DDD 风格**：`domain/<bounded-context>/`、`services/<service>/`
6. **大型单模块项目**：如果上述都没命中，仍可按顶层 `src/` 下的子目录粗分

**输出**：模块候选列表，**带证据**（哪个目录、几个文件、是否独立可发布等）。

### 1.3 已有文档

读取（如果存在）：`README.md`、`CONTRIBUTING.md`、`docs/`、`.github/copilot-instructions.md`、`.cursorrules`、`CLAUDE.md`、`AGENTS.md.bak`（如果是重建场景）。提取已有项目知识作为参考。

### 1.4 工作模式信号

判断该项目对 Agent 的工作模式偏好（用于 AGENTS.md § 1）：

| 信号 | 倾向 |
|---|---|
| CLI 项目、有 `bin/`、有 stdin/stdout 测试 | 允许自动跑测试、允许自动启动 |
| Web 服务（Spring / FastAPI / Express）、启动慢、占端口 | 禁止自动启服 |
| 库（无 main / 无 entrypoint） | 允许自动跑测试，无启服概念 |
| 有 migration 脚本（`db/migration/`、`alembic/`、`prisma/migrations/`） | migration 一律禁止自动执行 |

把这些做成"我的猜测"，**Phase 2 A 组让用户确认**。

### 1.5 形成画像

整合成内部草稿（不输出）：

```
项目类型: ...
主语言/框架: ...
仓库布局: ...
模块候选: [模块A(证据), 模块B(证据), ...]
工作模式猜测: { 测试: 允许?, 启服: 禁止?, ... }
已有文档摘要: ...
不确定的点: [...]  # Phase 2 A 组要问的
```

---

## Phase 2：Interview（交互访谈）

按 A → B → C → D 四组依次进行。**每组问完等用户回应再继续。**

### A 组：验证扫描发现的事实（原则 1）

把 Phase 1 中不确定的点逐条做成"是否+追问"。**带证据**：

> 我在 `<具体文件:行>` 看到 `<证据>`，推测 `<猜测>`，对吗？
> - 是 / 否（请纠正）

例如：
- "看到 `packages/` 下有 web/api/shared 三个 workspace，我把它们识别为三个模块。对吗？需要合并/拆分/增删吗？"
- "看到 `spring-boot-starter-web`，推测这是 Web 服务，自动启动不安全。对吗？"
- "看到 `db/migration/V*.sql`，推测 migration 由人工执行。对吗？"

### B 组：能力清单与禁区（原则 14）

聚焦 AGENTS.md § 2 内容：

- 除了 A 组已确认的"自动启服"等，**还有哪些操作 AI 必须先征求你同意**？（如：引入新依赖、改公共 API、重命名导出）
- **哪些目录或文件 AI 绝对不要改**？（如：legacy/、vendor/、自动生成代码、敏感配置）
- **哪些命令 AI 可以自由跑**？（如：lint、test、format）
- **哪些命令 AI 绝对不要跑**？（如：deploy、迁移、对外 API 调用）

### C 组：痛苦换来的知识（原则 4，**最高价值**）

> 这部分是记忆资产最有价值的部分——**调试过的 bug、生产事故、看似合理但行不通的方案**。一旦丢失，AI 会重新踩坑。
>
> 请尽量回忆 1–5 条：
>
> 1. 最近 6 个月你/团队踩过什么坑？根因是什么？现在怎么避免？
> 2. 有没有"我们曾经这样做过但行不通"的方案？
> 3. 哪些代码"看起来很普通但改了会出问题"？

**每条都引导用户给出三要素**：现象 → 根因 → 现在的正确做法（+ 反例可选）。

### D 组：工作流钩子（原则 15）

> 项目里有没有"做某件事之前必须先做/读 X"的规矩？
>
> 例：
> - 改 schema 前必须先看 migration 流程文档
> - 加新 API 前必须先看 API 约定
> - 改 OAuth 流程前必须看 auth 模块文档
>
> 列出 1–5 条这样的"先 X 再 Y"映射。

### 访谈节奏控制

- 每组问完显式说"等你回应"
- "不知道/不重要" 是合法答案，对应字段空缺即可
- 用户主动补充未问到的内容**优先接受**
- 不要追问超过 2 次
- **不要问可推导的事**（如"用什么语言"、"主入口在哪"）——这些 Phase 1 已扫描

---

## Phase 2.5：模块确认

汇总 Phase 1.2 自动检测 + B 组涉及的模块，向用户展示最终模块列表：

```
我计划为以下模块各建一个文档：

  1. <module-A>   ← 自动检测于 packages/web/，证据：...
  2. <module-B>   ← 自动检测于 packages/api/，证据：...
  3. <module-C>   ← 你提到要单独记录的，描述：...

每个文档触发条件（你可以修改）：
  <module-A>: 改 web 端 UI 时
  <module-B>: 改 api 端接口或业务逻辑时
  <module-C>: 改 ...

确认（y）/ 修改（编号+操作，如"3:删除"、"加一个 module-D"）？
```

用户确认后进入 Phase 3。

---

## Phase 3：Draft（套模板生成）

模板位置：`~/.config/opencode/skills/project-context-bootstrap/templates/`

### 3.1 生成 AGENTS.md（4 段结构）

用 `AGENTS.md.tmpl`，按访谈结果填充：

- **§ 1 项目元信息**：从 Phase 1.1 + A 组 + 1.4 工作模式
- **§ 2 能力清单与禁区**：从 B 组
- **§ 3 工作流钩子**：从 D 组（"先 X 再 Y" 映射）
- **§ 4 文档索引**：列出 `_global.md` + Phase 2.5 确认的各模块文档，**每条写"何时读"**（原则 6），**不写"里面有什么"**

### 3.2 生成 `_global.md`

用 `_global.md.tmpl`，按 C 组结果填充：
- 跨模块约定 / 数据约定 / API 约定 / 历史决策 / SOP
- 每条作为原子条目 `G-001`、`G-002`...编号递增
- 字段：关联代码、新增日期（今天）、最后验证（今天）、标签、事实/为什么/反例

如果 C 组某类完全没拿到信息，**留章节标题但内容为空**（不要塞通用知识填充）。

### 3.3 生成各模块文档

每个模块用 `_module.md.tmpl`：
- 头部 frontmatter：触发条件、最后更新、关联代码路径
- 4 个章节：模块设计与决策 / 约束与边界 / 陷阱与历史坑 / 本模块 SOP
- 如果 C 组中有针对某模块的具体陷阱/决策，归到对应模块文件
- 原子条目编号 `M-001`、`M-002`...（每个模块文件独立计数）

### 3.4 套模板规则

- 模板用 `{{字段}}` 表占位符
- 字段无信息：标 `（待补充）`，**不要**编造内容
- 全中文输出（即使代码用英文）
- 保留模板的 frontmatter（触发条件、最后更新、关联代码）
- **不要**生成多余文档；用户没说的内容不写
- 暂存内存，进入 Phase 4

---

## Phase 4：Review（用户审阅）

向用户展示**动态清单**：

```
即将创建：

  AGENTS.md                          [N 行]
  docs/agent/_global.md              [N 行]
  docs/agent/<module-A>.md           [N 行]
  docs/agent/<module-B>.md           [N 行]
  ...

选项：
  y          全部确认，写入磁盘
  v <文件>   查看完整内容
  e <文件>   提出修改意见，我重写
  n          取消
```

`e` 后用户描述修改点，重新生成该文件，回到 Phase 4。

---

## Phase 5：Finalize（写盘 + 收尾）

1. 创建 `docs/agent/` 目录（如不存在）
2. 写盘
3. 输出收尾信息：

```
✓ 已生成 v2 记忆资产：
  - AGENTS.md（4 段：元信息 / 能力 / 钩子 / 索引）
  - docs/agent/_global.md（G-NNN 全局条目 N 条）
  - docs/agent/<module>.md × N（按模块组织）

接下来你可以：
  1. 把这些文件提交到 git，让团队共享
  2. 日常用 /remember <内容> 追加单条记忆
  3. 用 /forget <条目ID> 删除过时记忆
  4. 用 /update-memory <条目ID> 更新已有记忆
  5. 定期跑 /audit-memory 检查陈旧条目

我自己也会在任务结束时自检——如果遇到"早有记忆就不会犯的错"，会主动建议你 /remember。
```

4. **不要**自动 git add/commit。

---

## 边界与禁忌

- ❌ 不要在 Phase 1 之前提问
- ❌ 不要问可从代码推导的事（原则 1）
- ❌ 不要生成"通用知识"内容（如"Spring Boot 是什么"）
- ❌ 不要在某节没信息时编内容填充——留待用户用 `/remember` 增量补充
- ❌ 不要把所有问题一次性甩出来——分组并等待回应
- ❌ 不要在用户取消时偷偷写盘
- ❌ 不要自动 git 操作
- ❌ 不要把秘密/密钥/版本号等快变数据写进记忆（原则 19）
