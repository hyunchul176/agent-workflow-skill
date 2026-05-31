---
name: agent-workflow
description: 在本地项目里搭建并运行一套"多 Agent 角色"工作流。把项目拆成 main/planner/executor/reviewer 等角色，每个角色有独立职责、记忆(MEMORY)、活动日志(activity_log)、阻塞器(blockers)、知识沉淀(notes)。支持 Ask 模式、Exec-Review 自动循环、Note 知识沉淀、blockers 协议。当用户想初始化该工作流、新增 agent 角色、或询问工作流用法时使用本 skill。纯本地文件，无需 SSH。
---

# Agent Workflow Skill

一套可移植的**多 Agent 角色工作流**。把单人项目变成一支"虚拟团队"：不同角色分工、各自记忆、互相留痕、知识沉淀。所有状态是项目内的本地 Markdown 文件，无需任何服务器。

## 子命令

根据用户传入的参数（`$ARGUMENTS`）判断要做什么：

### `init` — 初始化工作流（首次使用）

在**当前项目根目录**搭建工作流骨架：

1. 确认项目根（当前工作目录）。设 `ROOT` = 项目根绝对路径。
2. 创建目录与状态文件：
   ```bash
   ROOT="$(pwd)"
   mkdir -p "$ROOT/.workflow/shared"
   mkdir -p "$ROOT/.workflow/agents"/{main,planner,executor,reviewer}
   ```
3. 把本 skill 的 `templates/` 内容复制进去：
   - `templates/CLAUDE.md` → `<ROOT>/CLAUDE.md`
     **若 `<ROOT>/CLAUDE.md` 已存在**：不要覆盖！改为把协议写到 `<ROOT>/.workflow/WORKFLOW.md`，并提示用户在自己的 CLAUDE.md 顶部加一行 `@.workflow/WORKFLOW.md` 引入，或手动合并。
   - `templates/shared/*.md` → `<ROOT>/.workflow/shared/`
   - `templates/agents/<role>/ROLE.md` → `<ROOT>/.workflow/agents/<role>/ROLE.md`（main/planner/executor/reviewer 各一份）
   - 每个 agent 目录建空 `MEMORY.md` 占位与 `notes/` 目录
4. 把 templates 里的占位符让用户填：询问用户**项目一句话目标**，写进 `shared/project_brief.md` 顶部。
5. 验证：列出 `.workflow/` 树，确认文件齐全。
6. 告诉用户下一步：
   - 编辑 `.workflow/shared/project_brief.md`（总目标）、`roadmap.md`（phase 框架）、`todos.md`（任务）
   - 之后每条消息以"你是 <agent>"开头即可触发协议
   - 需要新角色用 `/agent-workflow add-agent <名字>`

### `add-agent <name>` — 新增自定义角色

1. `mkdir -p "<ROOT>/.workflow/agents/<name>/notes"`
2. 复制 `templates/ROLE.template.md` → `<ROOT>/.workflow/agents/<name>/ROLE.md`，把占位符 `<AGENT_NAME>` 替换为 `<name>`
3. 建空 `MEMORY.md`
4. 询问用户该角色的**一句话职责**和**是否为 Note Producer**，填进 ROLE.md
5. 提醒：若该角色参与 phase 分工，让 planner 在 `roadmap.md` 里登记其职责

### `status` — 查看工作流当前状态

读取并汇总：当前 phase（todos.md）、各 agent 上次活动（activity_log.md）、未解决 blockers、各 agent MEMORY 摘要。只读，不改文件。

### 无参数 / 其他 — 解释用法

简要说明这套工作流是什么、有哪些角色和模式、如何 `init`。

## 重要说明

- 本 skill 只负责**搭建脚手架和管理结构**。日常的 Step 1-5 执行协议由 `init` 安装到项目的 `CLAUDE.md`（或 `.workflow/WORKFLOW.md`）后**自动生效**，不需要每次调用本 skill。
- 协议全文见 `templates/CLAUDE.md`，是这套工作流的权威定义。
- 一切本地操作，绝不引入 SSH / 远程依赖。
