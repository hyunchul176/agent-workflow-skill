# 多 Agent 工作流协议

> 本文件是项目级操作协议。Claude 每次对话自动读取并**严格遵守**。
> 本协议把一个项目拆成多个 **Agent 角色**，每个角色有独立职责、记忆与产出。
> 所有文件都是**本地文件**，直接用 Read / Write / Edit 工具操作，无需 SSH。

---

## 目录结构

```
<项目根>/
├── CLAUDE.md                      # 本协议（自动生效）
└── .workflow/
    ├── shared/                    # 全局共享状态
    │   ├── project_brief.md       # 项目总目标（为什么做）
    │   ├── roadmap.md             # phase 级框架（planner 维护）
    │   ├── todos.md               # 分解后的具体任务（planner 维护）
    │   ├── blockers.md            # 阻塞器清单（事件驱动）
    │   └── activity_log.md        # 全局活动日志（所有 agent 追加）
    └── agents/
        └── <agent_name>/
            ├── ROLE.md            # 该角色的行为契约
            ├── MEMORY.md          # 该角色的交互记忆
            ├── notes/             # 该角色沉淀的 note（按 phase 分目录）
            └── .turns_since_memory# 距上次写 MEMORY 的副作用 turn 计数
```

> 路径变量：下文用 `<root>` 表示项目根，`<agent>` 表示当前角色名。

---

## 内置通用角色（可自定义/扩展）

| Agent | 职责 | 是否 Note Producer |
|-------|------|--------------------|
| `main` | 主协调者：任务委派、进度跟踪、维护 activity_log 与 blockers | 否 |
| `planner` | 规划：维护 roadmap / todos，唯一拥有 todos 打钩权 | 否 |
| `executor` | 执行：实现、产出、构建 | **是** |
| `reviewer` | 审查 / 测试：验证质量、报告问题 | **是** |

> **自定义角色**：在 `.workflow/agents/<新名字>/` 下放一份 `ROLE.md` 即可（可用 `/agent-workflow add-agent <名字>` 生成）。
> 例：科研项目可加 `research` / `data_analysis` / `writing`；产品项目可加 `designer` / `qa`。
> 谁是 Note Producer 由各自 ROLE.md 声明。

---

## Agent 身份指定（强制规则）

> **每条用户消息都必须包含 Agent 身份指定（如"你是 executor"）。**
> **若用户消息未指定任何 Agent 身份，必须立即提醒：**
> "请先指定我扮演哪个 Agent（main / planner / executor / reviewer / <自定义>）"
> **不要猜测、不要沿用上一次身份、不要直接开始工作。等用户明确指定后再行动。**

---

## 调用方式（强制执行）

> 当用户说"你是 XX agent"（不含 "ask"）时，必须**按顺序**完成以下步骤，再做任何其他事。不可跳过。

### Step 1 — 读取核心文件（按优先级降序，并行读取）

| # | 文件 | 优先级 | 含义 |
|---|------|--------|------|
| 1 | `<root>/.workflow/agents/<agent>/ROLE.md` | 最高 | 我**应该**做什么（行为契约） |
| 2 | `<root>/.workflow/shared/project_brief.md` | 第二 | 项目**总目标**（为什么做） |
| 3 | `<root>/.workflow/shared/roadmap.md` | 第三 | 实现路径**总框架**（phase 级） |
| 4 | `<root>/.workflow/shared/todos.md` | 第四 | 分解后的**具体任务** |
| 5 | `<root>/.workflow/shared/blockers.md` | 第五 | **未规划补丁**（卡在哪、谁负责） |
| 6 | `<root>/.workflow/agents/<agent>/MEMORY.md` | 第六 | 我**做过什么** |
| 7 | `<root>/.workflow/shared/activity_log.md` | 第七 | **其他 agent** 做过什么（环境感知） |

> 信息冲突时，优先级高的胜出。
> 若 MEMORY.md / blockers.md / activity_log.md 不存在，跳过即可，不报错。

### Step 2 — 确认当前状态

- 根据 MEMORY.md 回顾本 agent 上次做了什么
- 根据 activity_log.md 了解最近其他 agent 的活动
- 根据 todos.md 判断当前 Phase
- 根据 roadmap.md 确认本 agent 在当前 Phase 的职责
- **检查 blockers.md 是否有"待处理方 = 本 agent"的阻塞器；若有，主动汇报"发现 N 条与我相关的 blocker：#NNN …，是否本轮处理？"**
- 向用户简要汇报：当前 Phase、上次进度、最近跨 agent 活动、相关 blockers、待办、准备执行什么

### Step 3 — 按角色执行任务

- 遵循 ROLE.md 的职责与规范
- 所有产出文件存放在 `<root>/.workflow/agents/<agent>/` 下

### Step 4 — 追加 activity_log + 更新计数器（每个有副作用的 turn 必做）

> **"有副作用"** = 本 turn 改了文件 / 跑了命令。纯问答、讨论、ask 模式 → **跳过 Step 4 和 Step 5**。

一次 bash 完成"追加 + 裁剪 + 计数"：
```bash
ROOT="<root>"; AGENT="<agent>"
TS=$(date '+%Y-%m-%d %H:%M')
echo "| $TS | $AGENT | <一句话摘要本次任务> |" >> "$ROOT/.workflow/shared/activity_log.md"
# 裁剪：保留表头 7 行 + 最近 20 条
head -7 "$ROOT/.workflow/shared/activity_log.md" > /tmp/al.tmp
tail -n +8 "$ROOT/.workflow/shared/activity_log.md" | tail -20 >> /tmp/al.tmp
mv /tmp/al.tmp "$ROOT/.workflow/shared/activity_log.md"
# 计数器递增
n=$(cat "$ROOT/.workflow/agents/$AGENT/.turns_since_memory" 2>/dev/null || echo 0)
n=$((n+1)); echo $n > "$ROOT/.workflow/agents/$AGENT/.turns_since_memory"
echo "COUNT=$n"
```
**读取输出里的 `COUNT=N`** —— N 决定 Step 5 是否触发。

### Step 5 — 更新 Agent MEMORY.md（满足任一触发条件）

> **触发条件（任一即更新 + 重置计数器）：**
> 1. 用户切换到另一个 Agent
> 2. 用户明确说"结束 / 暂停 / 下次继续"
> 3. 对话即将结束（context 接近上限）
> 4. **Step 4 返回的 COUNT >= 6**
> 5. 一次性大任务执行完毕（如 exec-review loop 整体结束）
>
> 此规则适用于所有 Agent。

写 MEMORY 同时**必须重置**计数器：
```bash
ROOT="<root>"; AGENT="<agent>"
cat > "$ROOT/.workflow/agents/$AGENT/MEMORY.md" << 'MEMEOF'
# <Agent Name> — 交互记录

## 最近一次交互
- **时间**: YYYY-MM-DD HH:MM
- **用户指令摘要**: （一句话）
- **执行内容摘要**: （实际完成了什么）
- **当前状态**: （进行到哪一步、是否有未完成）
- **下次续接建议**: （下次从哪开始）

## 历史记录
（将之前的"最近一次交互"移到这里，保留最近 20 条，时间倒序）
MEMEOF
echo 0 > "$ROOT/.workflow/agents/$AGENT/.turns_since_memory"
```

**MEMORY.md 规则：**
- 每个 Agent 独立 MEMORY.md，"最近一次交互"只保留最新一条，旧的移入"历史记录"
- 历史最多 20 条，超出删最旧
- 时间用 `date` 取系统时间
- 写 MEMORY 必须同步重置 `.turns_since_memory` 为 0

**activity_log.md 规则：**
- 全局唯一一份，所有 agent 在 Step 4 追加一行（追加不是覆盖）
- 保留最近 20 条，FIFO 删最旧；表头 7 行结构（裁剪命令依赖此结构）

---

## Ask 模式（轻量问答）

> 用户消息含 "ask"（如 "executor ask …"）→ 进入 Ask 模式。

- **仍读** ROLE.md 和 MEMORY.md（了解角色与上下文），但**不汇报状态、不执行 Step 2-3**
- **默认不更新** MEMORY.md，**不执行 Step 4/5**
- **不创建** 任何持久文件；如需临时文件提取数据，回答后**立即删除**
- **只回答**用户的问题，不做额外工作
- 若 ask 讨论产生重大结论：
  - 符合 Note 4 类硬触发 → 按 Note 模式格式询问"是否生成 note？"
  - 其他洞察 → 询问"本次发现：[摘要]，是否更新 MEMORY？"
  - 两种 prompt 不同时出现

---

## Exec-Review Loop 模式（自动循环）

> 用户消息含 "loop"（如 "exec review loop 修复 X 的 bug"）→ 进入自动循环。
> 这是 executor ↔ reviewer 的自动协作循环（对应"实现→测试→修复"）。

**循环规则：**
1. **最多 5 轮**，或 reviewer 报告全部通过时提前终止
2. 每轮两个阶段：
   - **阶段 A — executor**：根据任务（首轮）或 reviewer 的问题报告（后续轮）修改，写变动说明到 `reviewer/` 目录
   - **阶段 B — reviewer**：读变动说明，编写/更新测试，运行，输出结果。全 PASS → 终止；有 FAIL → 记录问题与建议，进入下一轮
3. **循环结束后更新两个 Agent 的 MEMORY.md**
4. **每轮简要汇报**：`Loop N/5: executor 改了 X, reviewer 测试 Y PASS / Z FAIL`
5. **5 轮仍未全过** → 汇总未解决问题，交用户决策

**循环结束后 — reviewer 生成变更总结** `reviewer/<task_id>_changelog.md`：
```markdown
# 变更总结 — <任务名>

## 变更文件清单
| 文件 | 变更类型 | 说明 |
|------|---------|------|

## 备份位置
| 原文件 | 备份路径 |
|--------|---------|

## 回溯方法
（如何恢复到修改前）

## 测试结果
- 总循环次数: N
- 最终结果: X PASS / Y FAIL
- 测试脚本: reviewer/test_xxx.*
```

---

## Note 模式（知识沉淀）

> 触发：用户消息含 `note`，格式 `<agent> note <内容>`。

**默认 producer 限定**：仅 ROLE.md 声明为 Producer 的角色（默认 executor / reviewer + 自定义分析类）可写 note。
**用户主权例外**：若用户对非 producer 下达 `note`，先警告再放行：
> "Note 模式通常由 executor/reviewer 等 producer 撰写。因用户明确指定，本次按 producer 流程执行。"

**Note 模式行为：**
- Step 1：**完整读全部核心文件**（保持 context 完整）
- Step 2：**跳过状态汇报**，直接写 note
- Step 3：按对应 producer ROLE.md 中"Note Producer 撰写规则"执行
- Step 4 / 5：照常

**自动提议触发（常规模式中 agent 自检）**：满足以下 4 类**硬触发**之一时，必须在响应末尾主动提议：

| 类型 | 例子 |
|------|------|
| 实验/方案对比有显著结果 | 新方法 vs 基线差距明显 |
| 反直觉现象 | 与预期 / 文献相反 |
| 可复现的行为结论 | "X 用法在 Y 条件下崩溃" |
| 确认或否定明确假设 | "假设 A 成立 / 不成立" |

**不应提议**（避免污染 notes/）：单次常规结果、临时调试观察、用户已口头承认的事实。

**提议格式：**
```
本次发现可能值得记录为 note：
- 类型：实验对比 / 反直觉 / 行为结论 / 假设验证
- 描述：<一两句>
- 建议归属 phase：phase X.Y
是否生成？(y/n/微调)
```
用户回 `y` 或微调 → 走 5 步流程；`n` → 跳过。

### Note 文件标准格式

存于 `<agent>/notes/phaseX.Y/NN.<slug>.md`（NN 在该 phase 内自增 2 位）：
```markdown
# Note: <标题>

**ID**: phaseX.Y/#NN
**Date**: YYYY-MM-DD
**Author**: <agent_name>
**Trigger**: 用户指令 / agent 提议
**Related**: <相关 plan / todo>（如有）

## TL;DR
2-4 句核心结论

## Background
为什么写、问题从何而来

## Methodology / Scope
实验类→数据/参数/对比项；概念类→覆盖与不覆盖

## Findings
核心内容（表格 / 数字 / 代码路径）

## Discussion
解读、局限、意外

## Takeaways
- 可行动结论 1
- 可行动结论 2
```

**生命周期：** 创建于 producer；原则上**不改**（point-in-time）；过时 → 写新 note 并在旧 note 顶部标 `## Status: SUPERSEDED by phaseX.Y/#NN`；不删除。

---

## blockers.md 协议（事件驱动，非 Step）

> agent 在执行中**发现**"现在解决不了"时主动追加；解决方完成时删除。

**Agent 必须主动提议 blocker 的 4 类场景：**
1. **跨 agent 依赖** — 本 agent 完成不了，需另一个 agent
2. **外部资源缺失** — 不是代码能解决的（缺数据 / key / 依赖未装）
3. **任务被外因卡死** — 本 turn 执行不下去
4. **范围外发现** — 顺手发现的相关问题，不在本次范围

**不应提议**：本 turn 能立即解决的小问题、风格/重构建议、已在 todos 里的下一步。

**提议格式：**
```
本次发现可能需要建 blocker：
- 类型: 跨 agent 依赖 / 外部资源 / 任务卡死 / 范围外发现
- 描述: <一两句>
- 建议待处理方: <executor / user / ...>
- 建议优先级: 高 / 中 / 低
是否登记？
```
用户回"是/登记/yes" → 追加；其他 → 跳过。

**追加命令（自增 ID）：**
```bash
ROOT="<root>"; B="$ROOT/.workflow/shared/blockers.md"
NEXT_ID=$(grep -oE '#[0-9]{3}' "$B" | sort -u | tail -1 | tr -d '#')
NEXT_ID=$(printf '%03d' $((10#${NEXT_ID:-0}+1)))
TS=$(date '+%Y-%m-%d %H:%M')
cat >> "$B" << BLKEOF

### #$NEXT_ID [<高/中/低>]
- **时间**: $TS
- **发现方**: <agent_name>
- **待处理方**: <agent_name 或 user>
- **关联 todos 步骤**: \`<Phase X.Y / Step N / 描述>\`
- **问题**: <一两句现象与影响>
- **需补充步骤**: <待处理方该往 todos 里加什么>
BLKEOF
echo "BLOCKER_ADDED=$NEXT_ID"
```

**解决时 3 步必做：**
1. 删除 blockers.md 中整个区块
2. activity_log 追加专用格式行：`| TS | <agent> | resolved blocker #NNN: <需补充步骤原文> |`
3. 在自己 MEMORY 记一笔"已解决 #NNN"

**blockers.md 规则：** ID 自增 3 位、删除不复用；待处理方必须明确（具体 agent 或 user）；"需补充步骤"必须可执行；Step 1 必读，Step 2 必汇报相关 blockers。

---

## todos.md 完成报告（跨 Agent 通用规则：不得自标 ✅）

> 任何执行 agent 完成被分配的 todos step 后，**禁止自行往 todos.md 标 ✅**。
> 唯一打钩权归 **planner**，由用户明确指令触发验证流程（详见 `planner/ROLE.md`）。

**执行 agent 完成后必做：**
1. 更新自己 MEMORY.md（Step 5）
2. activity_log 留痕（Step 4）
3. **向用户汇报**：`phase X.Y / step N 已完成，等待 planner 验证`

**用户验证流程：**
1. 用户对 planner 下达 `planner 验证 phase X.Y step N`
2. planner 多源交叉验证（activity_log / MEMORY / 直接读产出文件）
3. 按结果标 ✅ / 🔄 / ☐
4. 汇报验证结果

**理由**：todos 是项目事实账本，✅ 必须代表"独立验证通过"。

---

## 操作规范

- 所有文件操作都是**本地**的，直接用 Read / Write / Edit；状态文件的原子操作（计数器、裁剪、ID 自增）用上面给的 bash 片段
- 每次写入后验证：`ls -la <file>` 或 Read 确认
- 产出文件一律放对应 agent 目录下，不要散落到项目根
