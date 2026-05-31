# <AGENT_NAME> — 角色契约 (ROLE.md)

> 本文件定义 `<AGENT_NAME>` 的职责边界与行为规范。Step 1 最高优先级读取。

## 一句话职责
<在这里写这个角色的核心职责，例如："负责把 planner 的 todos 转化为具体实现产物"。>

## 是否 Note Producer
<是 / 否>。
（Note Producer 才可在 Note 模式写 note；通常为 executor / reviewer / 分析类角色。）

## 我负责什么
- <职责 1>
- <职责 2>

## 我不负责什么（交给谁）
- <非职责 1> → 交给 <某 agent>
- <非职责 2> → 交给 <某 agent>

## 产出存放位置
- 一律放在 `.workflow/agents/<AGENT_NAME>/` 下

## 工作规范
- 完成被分配的 todos step 后**不自标 ✅**，向用户汇报"等待 planner 验证"
- 发现卡点按 blockers 协议提议登记
- 有副作用的 turn 走 Step 4/5

## Note Producer 撰写规则（仅 Producer 角色填写）
若本角色是 Producer，撰写 note 时按 5 步：
1. 确认触发类型（实验对比 / 反直觉 / 行为结论 / 假设验证）与归属 phase
2. 在 `notes/phaseX.Y/` 下确定自增编号 NN
3. 按 CLAUDE.md 中"Note 文件标准格式"撰写
4. 引用具体数字 / 代码路径 / 对比项，避免空泛
5. 写完在 activity_log 追加一行留痕（Step 4）
