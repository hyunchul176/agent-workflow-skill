# Agent Workflow — 多 Agent 角色工作流 Skill

一套可移植、纯本地的 **Claude Code 工作流**。把一个项目拆成多个 **Agent 角色**（main / planner / executor / reviewer + 你自定义的），每个角色有独立职责、记忆、活动留痕和知识沉淀。适合一个人想用"虚拟团队"方式、有纪律地推进复杂项目。

> 纯本地文件，无需任何服务器 / SSH。

> **两种使用深度：**
> - 只想要工作流功能 → 看下面"安装（2 步）"，把 skill 丢进现有项目即可。
> - 想从零搭一台干净的终端环境（Starship + WSL + Zellij + Claude Code + 本 skill）→ 看 **[SETUP.md](SETUP.md)**。

---

## 这套工作流能给你什么

- **角色分工**：每条消息以"你是 executor / planner / …"开头，Claude 自动切换到对应角色的职责与记忆。
- **持久记忆**：每个角色有自己的 `MEMORY.md`，下次对话自动续接上次进度。
- **环境感知**：全局 `activity_log.md` 让每个角色知道别人最近做了什么。
- **阻塞器协议（blockers）**：卡住的事登记下来，明确谁负责、怎么补。
- **知识沉淀（Note）**：重要发现按标准格式归档，不再丢失。
- **Exec-Review 自动循环**：执行↔审查最多 5 轮自动迭代修 bug。
- **任务账本（todos）**：只有 planner 能打 ✅，保证"完成"代表真的验证过。

---

## 安装（2 步）

1. 把本仓库里的 `.claude/skills/agent-workflow/` 文件夹整个复制到你的**目标项目根目录**下（与你的代码放一起）。
   ```
   你的项目/
   └── .claude/skills/agent-workflow/   ← 复制进来
   ```
2. 在该项目里打开 Claude Code，运行：
   ```
   /agent-workflow init
   ```
   它会自动：
   - 在项目根创建 `.workflow/`（共享状态 + 各角色目录）
   - 安装工作流协议到项目的 `CLAUDE.md`（若已有 CLAUDE.md，会改放到 `.workflow/WORKFLOW.md` 并提示你如何引入，不覆盖你的文件）
   - 让你填一句话项目目标

完成后，工作流即"常驻生效"。

---

## 日常用法

```
你是 planner，把项目目标拆成 phase 和 todos
你是 executor，实现 phase 1 step 1
你是 reviewer，测试 executor 刚才的实现
你是 executor ask，这个函数当前用了哪种缓存策略？      # ask = 轻量问答，不留痕
exec review loop 修复登录模块的并发 bug                  # 自动循环
你是 reviewer note 并发下连接池未加锁会偶发 500          # 沉淀知识
你是 planner 验证 phase 1 step 1                          # 打 ✅
```

## 角色与模式速查

| 内置角色 | 职责 |
|---------|------|
| `main` | 协调、委派、跟踪 |
| `planner` | 规划 roadmap/todos，唯一打 ✅ 权 |
| `executor` | 实现、产出（Note Producer） |
| `reviewer` | 测试、审查（Note Producer） |

新增自定义角色：`/agent-workflow add-agent <名字>`（如 `designer` / `data_analysis` / `writing`）。

| 模式 | 触发 |
|------|------|
| 标准执行 | `你是 <agent>` |
| Ask 轻量问答 | `<agent> ask …` |
| Exec-Review 循环 | `... loop ...` |
| Note 知识沉淀 | `<agent> note …` |
| blockers 登记/解决 | 对话中按提议格式确认 |

完整协议见 `.claude/skills/agent-workflow/templates/CLAUDE.md`。

---

## 目录结构（安装后）

```
你的项目/
├── CLAUDE.md                    # 工作流协议（init 安装）
├── .claude/skills/agent-workflow/   # 本 skill
└── .workflow/
    ├── shared/
    │   ├── project_brief.md     # 总目标
    │   ├── roadmap.md           # phase 框架
    │   ├── todos.md             # 具体任务
    │   ├── blockers.md          # 阻塞器
    │   └── activity_log.md      # 全局日志
    └── agents/<role>/
        ├── ROLE.md              # 角色契约
        ├── MEMORY.md            # 角色记忆
        └── notes/               # 知识沉淀
```

---

## 自定义建议

- **改角色集**：删掉用不上的角色目录，或 `add-agent` 加新角色；同时更新 `roadmap.md` 里的职责。
- **改触发词**：协议里的 `ask` / `loop` / `note` 关键词可在 `CLAUDE.md` 里改成你习惯的词。
- **接版本控制**：本 skill 不含 git 同步逻辑，你用自己的 git 流程即可；`.workflow/` 建议纳入版本管理以便团队/多机同步。

---

## 交付包内容

| 路径 | 用途 |
|------|------|
| `README.md` | 本文件 — skill 功能说明与用法 |
| `SETUP.md` | 从零搭建终端环境指南（Starship + WSL + Zellij + Claude Code） |
| `.claude/skills/agent-workflow/` | 工作流 skill 本体（含协议与模板） |
| `dotfiles/starship.toml` | Starship 提示符配置（Pastel Powerline） |
| `dotfiles/zellij/` | Zellij 多窗格配置与布局 |
