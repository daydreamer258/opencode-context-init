---
name: project-context-bootstrap
description: 当用户在某个代码项目中开始对话，且项目根目录缺少 AGENTS.md 文件时，提示用户运行 /init-project 来生成项目专属的 Agent 知识库。只在项目存在 .git 目录或常见构建文件（pom.xml、package.json、Cargo.toml、go.mod、requirements.txt、pyproject.toml、Gemfile、build.gradle 等）时触发。本 Skill 只负责提示，不直接生成文件。
---

# Project Context Bootstrap（哨兵）

## 你的职责

在用户进入一个**代码项目**且**缺少 `AGENTS.md`** 时，提醒用户初始化项目知识库。

## 触发条件（同时满足）

1. 当前工作目录或其根目录是一个代码项目，判定标准是存在以下任一文件：
   - `.git/`（git 仓库）
   - `pom.xml` / `build.gradle` / `build.gradle.kts`（Java）
   - `package.json`（Node.js）
   - `Cargo.toml`（Rust）
   - `go.mod`（Go）
   - `requirements.txt` / `pyproject.toml` / `setup.py`（Python）
   - `Gemfile`（Ruby）
   - `composer.json`（PHP）
   - `*.csproj` / `*.sln`（.NET）
   - `Makefile` / `CMakeLists.txt`（C/C++）

2. 项目根目录**没有**以下任一文件：
   - `AGENTS.md`
   - `.opencode/AGENTS.md`

## 你要做的事

向用户输出**一次性提示**，**不要重复**：

> 检测到这个项目还没有 `AGENTS.md`，建议运行 `/init-project` 来生成项目专属的 Agent 知识库。
> 
> 这会通过几轮对话，把项目的业务意图、构建流程、团队约定和已知陷阱整理成结构化文档，让我（和其他 Agent）以后能更准确地理解你的项目。
> 
> 现在是否要运行？（你也可以稍后手动输入 `/init-project`）

## 你不要做的事

- ❌ 不要自己生成 `AGENTS.md` 或任何文档文件，那是 `/init-project` 命令的职责
- ❌ 不要在同一会话中反复提示
- ❌ 不要在用户明确表示"暂不需要"后继续催促
- ❌ 不要在非代码项目（如纯文档目录、空目录）中触发
- ❌ 不要在已有 `AGENTS.md` 的项目中触发——此时静默即可
