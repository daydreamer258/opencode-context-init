---
description: 通过交互式访谈，为当前项目生成专属的 Agent 知识库（AGENTS.md + docs/agent/）。全中文输出。
---

# /init-project — 项目上下文初始化器

你的任务：通过 5 阶段工作流，为当前项目生成一份完整的、中文的 Agent 知识库。

**核心原则：**
- 全程中文与用户交互
- 不假设，不猜测——猜测必须先跟用户确认
- 用户的回答永远优先于你的推断
- 一次只问一组问题，等待回应再继续
- 用户可以随时插入补充内容，你要接受并整合

---

## Phase 0：前置检查

1. 检查当前目录是否已存在 `AGENTS.md`。如果存在，告知用户：
   > 检测到 `AGENTS.md` 已存在。继续会**不会覆盖**它，但可以追加新发现的内容。
   > 选择：
   > - `继续`：基于现状追加
   > - `重建`：先备份旧文件为 `AGENTS.md.bak`，然后重新生成
   > - `取消`：退出

2. 确认用户意图后再进入 Phase 1。

---

## Phase 1：Discover（静默扫描）

**不打扰用户**，自动收集项目信息。使用 Read/Glob/Grep 工具完成：

### 1.1 项目类型识别

检查以下文件是否存在，记录所有命中的：
- `pom.xml`、`build.gradle`、`build.gradle.kts` → Java/JVM
- `package.json` → Node.js / TypeScript
- `Cargo.toml` → Rust
- `go.mod` → Go
- `requirements.txt`、`pyproject.toml`、`setup.py` → Python
- `Gemfile` → Ruby
- `composer.json` → PHP
- `*.csproj`、`*.sln` → .NET
- `Makefile`、`CMakeLists.txt` → C/C++

### 1.2 抓取关键命令

按项目类型从对应文件提取：
- `package.json` → `scripts` 字段
- `Makefile` → 顶层 target
- `pom.xml` → 关键 plugin、profile
- `pyproject.toml` → `[tool.poetry.scripts]` 或 `[project.scripts]`
- 其他：查找 README、CONTRIBUTING 等已有文档中的命令

### 1.3 目录结构扫描

列出顶层目录（深度 1–2），标记：
- 源码目录（src/、app/、lib/）
- 测试目录（test/、tests/、spec/）
- 配置目录（config/、conf/、resources/）
- 生成代码目录（build/、generated/、target/、dist/）
- 文档目录（docs/）
- 非标准目录（不属于以上任何类别的）

### 1.4 代码生成信号

特别留意以下模式（这是模型最容易猜错的部分）：
- Maven/Gradle 的 codegen plugin
- `.proto` 文件 + protoc
- yaml/json 驱动的接口/代码生成（OpenAPI、Swagger、自定义 DSL）
- 注解处理器（Lombok 之外的）
- 构建期模板渲染
- ORM 模型生成（Prisma、SQLAlchemy migrations、etc）

记录每个发现的生成器：触发命令、输入路径、输出路径。

### 1.5 已有文档

读取（如果存在）：
- `README.md` / `README.zh.md`
- `CONTRIBUTING.md`
- `.github/copilot-instructions.md`
- `.cursorrules` / `.cursor/rules/*.mdc`
- `CLAUDE.md`

把里面已有的项目知识提取出来作为参考，避免在访谈中重复问。

### 1.6 形成画像

把以上信息整合成一份**内部草稿**（不输出给用户），结构：

```
项目类型: [识别结果]
关键命令: [构建/运行/测试]
顶层结构: [关键目录]
代码生成: [是否有，如何工作]
已有文档: [已提取的项目知识]
不确定的点: [需要用户确认的猜测]
```

---

## Phase 2：Interview（交互访谈）

按 A、B 两组依次进行。**每组问完等用户回应再继续下一组。**

### A 组：验证你的猜测

把 Phase 1 中"不确定的点"逐条转成问题。格式：

> 我从项目里看到 `<具体证据>`，推测 `<猜测>`，对吗？
> - 是 / 否
> - 如果是，请补充：`<追问>`

**示例**（仅作格式参考，实际问题取决于扫描结果）：

> 我看到 `pom.xml` 里有 `openapi-generator-maven-plugin`，触发于 `mvn compile`，输入路径是 `src/main/resources/openapi/*.yaml`，输出到 `target/generated-sources/openapi/`。我推测这是用来从 YAML 规格生成接口代码的，对吗？
> - 是/否
> - 如果是：生成的代码 AI 是否应该手动改？还是只能改 YAML？

一次性列出所有 A 组问题（可以编号），等用户逐条回答或批量回答。

### B 组：模型推断不出的事

按以下 4 个维度提问，每个维度 1–2 个问题：

**业务（Business）：**
- 这个项目主要解决什么业务问题？目标用户是谁？
- （如果适用）和团队的其他系统是什么关系？

**约定（Conventions）：**
- 团队有没有特别坚持的写法或风格？（举例：业务逻辑只能在 Service 层、必须用构造器注入、错误必须用统一异常类等）
- 新写一段代码时，AI 最容易选错的"风格"是什么？

**陷阱（Gotchas）：**
- 新人加入后最常踩的 3 个坑是什么？
- 有没有"看起来能改但改了会出问题"的地方？

**边界（Boundaries）：**
- 哪些目录或文件，AI **绝对**不应该改动？（举例：自动生成代码、第三方 vendor、敏感配置、即将废弃的遗留模块）
- 有没有"只能读，不能写"的区域？

### 访谈节奏控制

- 每组问完，**显式说"等你回应"**，不要急着进入下一阶段
- 用户回答"不知道"或"不重要"是合法答案——记录为空或简写
- 用户主动补充任何内容（未在你问题列表里的），**优先接受并整合**
- 如果用户回答模糊，可以追问 1 次，不要反复追问

---

## Phase 3：Draft（套模板生成）

模板位置：`~/.config/opencode/skills/project-context-bootstrap/templates/`

需要生成的文件：

| 模板 | 产出位置 |
|---|---|
| `AGENTS.md.tmpl` | `AGENTS.md`（项目根） |
| `overview.md.tmpl` | `docs/agent/overview.md` |
| `build-and-run.md.tmpl` | `docs/agent/build-and-run.md` |
| `conventions.md.tmpl` | `docs/agent/conventions.md` |
| `gotchas.md.tmpl` | `docs/agent/gotchas.md` |
| `boundaries.md.tmpl` | `docs/agent/boundaries.md` |

### 套模板规则

1. 模板用 `{{字段名}}` 表示占位符，用访谈结果填充
2. 如果某个字段访谈中没拿到信息，用以下规则处理：
   - 关键字段（如项目类型、构建命令）：标 `（待补充）`
   - 次要字段（如某条陷阱）：跳过该条
3. 全部输出中文（即使原项目代码用英文命名）
4. 保持模板的结构和锚点不变，方便后续 Agent 引用

### 暂不写盘

生成后**暂存在内存中**，进入 Phase 4 给用户审阅。

---

## Phase 4：Review（用户审阅）

向用户展示生成清单：

```
即将创建以下文件：

  AGENTS.md                                     [N 行]
  docs/agent/overview.md                        [N 行]
  docs/agent/build-and-run.md                   [N 行]
  docs/agent/conventions.md                     [N 行]
  docs/agent/gotchas.md                         [N 行]
  docs/agent/boundaries.md                      [N 行]

选项：
  y         全部确认，写入磁盘
  v <文件>  查看某个文件的完整内容
  e <文件>  对某个文件提出修改意见，我会重写
  n         取消，不写入
```

根据用户选择处理：
- `v`：完整打印该文件内容
- `e`：让用户描述修改点，重新生成该文件，再次进入 Review
- `y`：进入 Phase 5
- `n`：友好退出，告知用户访谈结果已丢弃

---

## Phase 5：Finalize（写盘 + 收尾）

1. 写入所有文件（注意：先确认 `docs/agent/` 目录存在，不存在则创建）
2. 输出收尾信息：

```
✓ 已生成 AGENTS.md 和 docs/agent/ 下 5 个文档

建议你：
  1. 把这些文件提交到 git，让团队共享
  2. 在新的 opencode 会话中验证 AGENTS.md 是否被自动加载
  3. 后续我（Agent）在工作中发现新的项目约定时，会主动建议追加到对应文档

任何时候你也可以手动编辑 AGENTS.md 或 docs/agent/ 下的文件。
```

3. 结束命令。**不要**自动 git add / commit——那是用户的决定。

---

## 边界与禁忌

- ❌ 不要在 Phase 1 之前就提问——先扫描完，提问才有针对性
- ❌ 不要把所有问题一次性甩给用户——分 A/B 两组依次问
- ❌ 不要假装"已经懂了"——任何不确定的事都要在 A 组确认
- ❌ 不要在用户取消时偷偷保留草稿
- ❌ 不要在生成的文档中堆砌通用知识（如"Spring Boot 是什么"）——只写**这个项目专属**的内容
- ❌ 不要覆盖已有的 AGENTS.md（Phase 0 已处理）
- ❌ 不要自动 git commit，让用户自己决定
