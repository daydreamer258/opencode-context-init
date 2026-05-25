# 项目上下文初始化器（Project Context Bootstrap）

一套用户级的 opencode 资产，用于在任意项目首次进入时，通过交互式访谈自动生成项目专属的 Agent 文档体系（`AGENTS.md` + `docs/agent/`）。

文档全中文输出，安装一次，所有项目可用。

---

## 1. 解决的问题

通用 AI 编程 Agent 在面对具体项目时无法从代码自动推断的信息：

- 业务意图（项目是干什么的、给谁用的）
- 非标准构建流程（如 yaml → `mvn compile` → 接口代码生成）
- 团队约定（包结构、命名、注入方式）
- 历史坑点（新人最常踩的陷阱）
- AI 不该碰的边界（敏感目录、只读区、生成代码）

这些信息一旦显式写下来，Agent 在后续会话能直接读取，避免反复犯同样的错。

---

## 2. 整体架构

```
用户级安装                       项目级产出                  运行时效果
───────────                     ───────────                ────────────
Skill（哨兵）         ──触发──►   /init-project        ──产出──►   AGENTS.md
Command（工作流）                                                  + docs/agent/
                                                                  + 元规则自演化
```

### 2.1 用户级文件布局

```
~/.config/opencode/
├── commands/
│   └── init-project.md                    # /init-project 命令本体
└── skills/
    └── project-context-bootstrap/
        ├── SKILL.md                       # 哨兵：检测缺失并提示
        └── templates/
            ├── AGENTS.md.tmpl             # 根文档模板
            ├── overview.md.tmpl           # 业务概述
            ├── build-and-run.md.tmpl      # 构建与运行
            ├── conventions.md.tmpl        # 约定与风格
            ├── gotchas.md.tmpl            # 陷阱清单
            └── boundaries.md.tmpl         # AI 边界
```

### 2.2 职责分工

| 组件 | 类型 | 触发方式 | 职责 |
|---|---|---|---|
| `project-context-bootstrap` | Skill | 模型按 description 自动判断 | 进入新项目时检测缺失 `AGENTS.md`，建议用户运行 `/init-project`。不直接生成。 |
| `init-project` | Command | 用户显式输入 `/init-project` | 执行 5 阶段工作流，生成全部产物。 |
| `templates/` | 资源文件 | 被 Command 读取 | 提供产物的中文骨架。 |

**为什么 Skill 不直接生成？**
Skill 由模型自动触发，存在误触发风险。文件生成是写盘操作，应该由用户显式发起。哨兵 + 命令的双层设计让"提醒"与"执行"解耦。

---

## 3. 安装方式

本仓库的目录结构**完全对齐** `~/.config/opencode/` 的标准布局，安装即"复制"。

### 3.1 仓库结构

```
opencode-context-init/        # 本仓库根
├── README.md                 # 本文档
├── commands/
│   └── init-project.md
└── skills/
    └── project-context-bootstrap/
        ├── SKILL.md
        └── templates/
            ├── AGENTS.md.tmpl
            ├── overview.md.tmpl
            ├── build-and-run.md.tmpl
            ├── conventions.md.tmpl
            ├── gotchas.md.tmpl
            └── boundaries.md.tmpl
```

### 3.2 安装步骤

**方案 A：直接复制**（最简单，适合不打算更新）

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

**方案 B：符号链接**（推荐，仓库更新后立即生效）

```powershell
# Windows PowerShell（需要管理员或开启开发者模式）
$repo = "$PWD"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\commands\init-project.md" -Target "$repo\commands\init-project.md"
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\skills\project-context-bootstrap" -Target "$repo\skills\project-context-bootstrap"
```

```bash
# macOS / Linux
ln -s "$PWD/commands/init-project.md" ~/.config/opencode/commands/init-project.md
ln -s "$PWD/skills/project-context-bootstrap" ~/.config/opencode/skills/project-context-bootstrap
```

### 3.3 验证安装

启动 opencode，输入 `/init-project`，如果命令被识别即安装成功。

`opencode.json` 不需要改动——opencode 自动扫描这些标准路径。

---

## 4. 使用流程（用户视角）

### 首次进入一个新项目

```
1. cd 到项目根目录，启动 opencode
2. 简单问一句话（例如"帮我看下这个项目"）
3. 哨兵 Skill 检测到无 AGENTS.md，提示：
   "这个项目还没有 AGENTS.md，建议运行 /init-project 生成项目知识库。"
4. 用户输入 /init-project
5. Agent 走完 5 阶段工作流（详见第 5 节）
6. 产物写入项目根：AGENTS.md + docs/agent/
```

### 之后的日常使用

- `AGENTS.md` 被 opencode 自动加载到每次会话上下文
- Agent 在工作中发现未文档化的约定时，主动建议追加（元规则驱动）
- 用户可以随时手动编辑 `AGENTS.md` 或 `docs/agent/` 下任意文件

### 重新初始化

如果想从头来过，删掉 `AGENTS.md` 再跑 `/init-project` 即可。Command 不会覆盖已有文件，会先警告。

---

## 5. `/init-project` 工作流详解

### Phase 1：Discover（静默扫描）

Agent 不打扰用户，自动收集：

- **项目类型识别**：检查 `pom.xml` / `package.json` / `Cargo.toml` / `requirements.txt` / `go.mod` / `Gemfile` 等标志文件
- **构建脚本**：从 `scripts` 字段 / `Makefile` / `pom.xml` 的 plugin 段抓取
- **目录结构**：列顶层目录，标记非标准布局
- **代码生成信号**：检测 `*-generated/`、`build/generated/`、自定义 maven/gradle plugin、`*.proto`、yaml 驱动的代码生成器等
- **README / 已有文档**：抓取已有的项目说明作为参考

输出："我对这个项目的初步画像"——一份内部草稿，作为 Phase 2 提问的依据。

### Phase 2：Interview（交互访谈）

**A 组：验证猜测**（基于 Phase 1 的发现）

Agent 把每个不确定的猜测变成"是否+追问"的形式：

> 我看到 `pom.xml` 里有一个 `xxx-codegen-plugin`，触发于 `mvn compile`。是用来从 `src/main/resources/api/*.yaml` 生成接口代码吗？
> - 是 / 否
> - 如果是，生成的代码在哪个包下？AI 是否应该手动改这些生成代码？

**B 组：模型无法推断的事**（按主题问）

```
业务：这个项目主要解决什么业务问题？目标用户是谁？
约定：有没有团队特别坚持的写法（比如"业务逻辑只能写在 Service 层"）？
陷阱：新人加入后最常踩的 3 个坑是什么？
边界：哪些目录或文件，AI 绝对不应该改动？
```

**交互节奏**：
- 每组问完等用户回应，不堆问题
- 允许用户在任何时候"打断"补充未问到的内容
- 用户回答"不知道/不重要"也接受，对应字段留空或简写

### Phase 3：Draft

套用 `templates/` 下的模板，填入访谈结果，生成：

- `AGENTS.md`（50–100 行）：根入口，含项目一句话概述、关键命令、链接到 `docs/agent/` 各文件、元规则
- `docs/agent/overview.md`：业务意图
- `docs/agent/build-and-run.md`：构建/运行/测试 + 代码生成流程
- `docs/agent/conventions.md`：约定与风格
- `docs/agent/gotchas.md`：陷阱清单
- `docs/agent/boundaries.md`：AI 边界

全中文输出。

### Phase 4：Review

Agent 列出生成清单：

```
将创建：
  AGENTS.md
  docs/agent/overview.md
  docs/agent/build-and-run.md
  ...

确认（y）/ 修改某项（编号）/ 取消（n）？
```

用户可以指定"第 3 项里这条删掉"或"补一条 xxx 到陷阱清单"。

### Phase 5：Finalize

- 写盘
- 提示用户："建议把这些文件一并提交到 git，让团队共享"
- 给出后续使用建议

---

## 6. 生成产物的内容骨架

### 6.1 `AGENTS.md`（根入口）

```markdown
# 项目名

> 一句话描述：这个项目是干什么的、给谁用的

## 关键命令

构建：xxx
运行：xxx
测试：xxx

## 详细文档

- 业务概述：docs/agent/overview.md
- 构建与运行：docs/agent/build-and-run.md
- 约定与风格：docs/agent/conventions.md
- 陷阱清单：docs/agent/gotchas.md
- AI 边界：docs/agent/boundaries.md

## 给 Agent 的元规则

1. 修改代码前，先查 docs/agent/boundaries.md 确认不在禁区
2. 遇到未文档化的项目约定时，主动建议用户追加到对应文档
3. （其他项目特有的元规则）
```

### 6.2 `docs/agent/*.md`

每个细节文档独立 50–150 行，主题单一。

**`overview.md`**：业务问题、用户群、关键术语、与其他系统的关系
**`build-and-run.md`**：完整构建/运行/测试流程，特别强调代码生成步骤
**`conventions.md`**：包结构、命名规则、注入方式、错误处理、日志风格
**`gotchas.md`**：新人坑、历史 bug 教训、容易误改的地方
**`boundaries.md`**：只读目录、自动生成代码区、敏感配置、不要重构的遗留模块

---

## 7. 自我演化机制

`AGENTS.md` 内嵌一条元规则：

> **元规则 2**：当你（Agent）在工作中发现项目存在未被文档化的约定、陷阱或习惯做法时，主动提议追加到 `AGENTS.md` 或 `docs/agent/` 对应文件。在用户确认后再写入。

效果：

- 不需要专门的 `/update-project` 命令
- 文档随项目演化自然增长
- 用户始终掌握"是否写入"的最终决定权

---

## 8. 设计取舍说明

| 抉择 | 选定方案 | 理由 |
|---|---|---|
| 跨工具策略 | 只生成 `AGENTS.md`（不复制为 CLAUDE.md 等） | opencode 原生读 AGENTS.md；AGENTS.md 已是跨工具事实标准 |
| 触发方式 | Skill（哨兵）+ Command（工作流） | 提醒与执行解耦，避免误触发写盘 |
| 问题清单 | 通用一套 | 起步简单，未来按项目类型扩展 |
| 维护机制 | 内嵌元规则自动演化 | 不引入额外命令，长期自然增长 |
| 文档目录 | `docs/agent/`（单数） | 工具中立；与 opencode 自身的 `agents/`（复数）区分 |
| 哨兵积极度 | 仅在代码项目无 AGENTS.md 时提示一次 | 不烦人 |
| 模板示例 | 占位符 + 通用说明，不绑定具体语言 | 适应多技术栈，避免无关示例污染 |

---

## 9. 扩展与定制

未来如果有需要：

- **加新问题**：编辑 `commands/init-project.md` 的访谈脚本段
- **加新模板**：在 `templates/` 加文件 + 在 `init-project.md` 引用
- **按项目类型分支**：在 Phase 1 识别后跳转到对应问题清单（需要时再做）
- **生成多语言版本**：复制模板出 `templates-en/`，命令加参数选择

---

## 10. 验收标准

写完之后能做到：

1. 在任意空项目目录运行 `/init-project`，10 分钟内通过对话生成出合理的 `AGENTS.md` + `docs/agent/`
2. 哨兵 Skill 在无 AGENTS.md 时提示一次，有 AGENTS.md 时完全静默
3. 生成的 AGENTS.md 在新会话被 opencode 自动加载
4. 后续会话中，Agent 发现新约定时主动建议追加（元规则生效）
5. 全部产物为中文

---

## 11. 待你确认

- 这份设计是否覆盖了你预期的能力？
- 有没有想加 / 想砍的部分？
- Phase 2 的问题清单（业务 / 约定 / 陷阱 / 边界）够不够？是否需要补充维度？

确认后我会按上述结构产出 `SKILL.md`、`init-project.md` 和 6 个模板文件。
