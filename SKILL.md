---
name: openspec-plan
description: 将大型需求分解为多个小型 change，防止单个 change 上下文溢出。当用户需求过大（需要 8+ 个 tasks）、需要依赖管理、或避免执行时幻觉问题时使用。自动创建带编号的 changes（001-xxx, 002-xxx），生成依赖图和 PLAN.md 总览文档。
license: MIT
compatibility: 需要 openspec CLI (openspec-cn)。
metadata:
  author: openspec
  version: "1.1"
  generatedBy: "1.2.0"
---

将大型需求拆成多个可独立推进的 changes。此 skill 只做规划、委托、验证；不直接生成 change 产出物。

说明：本文中的委托方式指当前运行环境提供的 `task` 工具接口，不是 OpenCode 官方 agents 文档中的 `@agent` 调用语法。

## 何时使用

- 需求涉及多个模块或阶段
- 单个 change 预计会超过 8 个 tasks
- 需要明确 hard/soft 依赖和执行顺序

## 核心规则

### 角色边界

- 你是协调者，不是产出物执行者
- 产出物必须由当前环境的 `task(..., load_skills=["openspec-ff-change"], ...)` 委托任务代理直接生成
- 禁止在当前上下文写入 `openspec/changes/*/proposal.md`、`specs/*.md` 等文件
- 禁止使用 `openspec-new-change` 代替 `openspec-ff-change`

### 拆分规则

- 最多 8 个 changes
- 每个 change 单一职责、命名为 `<三位编号>-<kebab-case>`
- 始终区分 `hard` 和 `soft` 依赖
- 如果出现同名 change，绝不静默覆盖；建议复用或换编号

### 状态规则

每个 change 使用以下状态：

`planned -> creating -> created -> verified -> ready`

失败时标记为 `failed`，不得假装成功。

## 执行流程

### 1. 收集需求

如果用户没有给出明确需求，使用 AskUserQuestion 开放式提问获取完整背景后再继续。

### 2. 生成拆分方案

输出：

- change 列表
- 每个 change 的目标
- hard/soft 依赖
- ASCII DAG
- 建议执行顺序

使用 TodoWrite 跟踪拆分过程。

### 3. 用户确认

在创建任何 change 之前，必须让用户确认拆分方案。

### 4. 写入 `openspec/plans/<plan-id>/PLAN.md`

PLAN.md 只保留必要信息：

- 总览：日期、需求摘要、change 数量
- 表格：编号、名称、目标、hard、soft、状态
- 依赖图
- 建议执行顺序

并在该步骤记录 `PLAN.md` 的完整路径，后续通过 `task` 委托时必须传给对应 change 的任务代理。

### 5. 委托创建 change 产出物

对每个已确认的 change，必须通过当前环境的 `task` 工具启动独立任务代理。

#### task 委托负载必填块

调用 `task()` 时必须提供合法签名：至少包含 `category` 或 `subagent_type`，并包含 `description`。这是当前环境的任务委托接口要求，不是 OpenCode 官方 agents 页面定义的 `@agent` 语法。

- `TASK`: 创建该 change 并完成所有 `applyRequires`
- `ROLE`: 你是该 change 的唯一执行者，必须亲自生成文件
- `PLAN CONTEXT`: 整体需求与本次拆分背景
- `PLAN PATH`: 来源 `PLAN.md` 的完整路径
- `CHANGE ID`: 该 change 的完整名称
- `CHANGE DETAIL`: 名称、编号、目标、范围
- `DEPENDENCIES`: hard/soft 依赖，以及需要读取的依赖文件路径
- `RELEVANT REQUIREMENTS`: 与该 change 直接相关的需求片段
- `CONSTRAINTS`: 技术栈、命名、边界条件
- `SUCCESS CRITERIA`: 需要生成的文件与完成条件
- `MUST NOT`: 禁止只建目录、禁止实现代码、禁止再委托

缺少任一必填块时，不得委托；直接返回 `invalid delegation payload`。

#### task 委托模板

```typescript
task(
  category="writing",
  load_skills=["openspec-ff-change"],
  description="Create change artifacts",
  run_in_background=true,
  prompt="""
## TASK
创建 change <change-name> 并完成所有 applyRequires 产出物。

## ROLE
你是该 change 的唯一执行者。所有产出物必须由你直接生成，不要等待主 agent，也不要继续委托给其他 agent。

## PLAN CONTEXT
<整体需求与本次拆分背景>

## PLAN PATH
openspec/plans/<plan-id>/PLAN.md

## CHANGE ID
<change-name>

## CHANGE DETAIL
- 名称: <change-name>
- 编号: <001>
- 目标: <1-2 句>
- 范围: <边界与关键内容>

## DEPENDENCIES
- hard: <依赖列表与必须读取的文件路径>
- soft: <可参考内容>

## RELEVANT REQUIREMENTS
<与本 change 直接相关的需求片段>

如果该 change 会生成 `tasks.md`，最后一个任务必须是：找到 `PLAN PATH` 指向的计划文档，把当前 change 的状态或清单项更新为“已实现”。

## CONSTRAINTS
<技术约束、命名约定、接口约束>

## SUCCESS CRITERIA
- `openspec/changes/<change-name>/CHANGE.md` 存在且非空
- `openspec/changes/<change-name>/proposal.md` 存在且非空
- 需要的 `specs/**/*.md` 存在且非空
- `applyRequires` 中所有产出物状态为 `done`
- 如果存在 `tasks.md`，其最后一个任务必须要求回写 `PLAN PATH`，将当前 change 标记为“已实现”
- 返回已生成文件列表

## MUST NOT
- 不要只创建目录
- 不要忽略 hard 依赖
- 不要实现代码
- 不要修改主规范
- 不要再次委托
"""
)
```

#### 调度规则

- 无 hard 依赖的 changes：可 `run_in_background=true` 并行创建
- 有 hard 依赖的 changes：仅在上游状态为 `verified` 或 `ready` 后再启动
- 每个后台任务收集结果时必须设置 30-60 秒轮询，单任务最长等待 10 分钟
- 超时、临时错误或返回不完整结果时最多重试 1 次；再次失败则标记 `failed`
- 后台任务卡住超过超时后必须取消并报告，不得无限等待

### 6. 强制验证

任务代理返回后，主 agent 必须逐个检查：

1. `CHANGE.md` 是否存在且非空
2. `proposal.md` 是否存在且非空
3. `specs/**/*.md` 是否存在且非空
4. 任务代理返回的文件列表是否与实际文件一致
5. `openspec-cn status --change "<name>" --json` 中 `applyRequires` 是否全部为 `done`

验证失败时：

- 禁止主 agent 自己补文件
- 将状态标记为 `failed`
- 重试一次任务代理；仍失败则明确报告错误

验证通过后，更新状态为 `verified`，全部完成后标记 `ready`。

## 护栏

- 不直接调用 `openspec-cn new` 生成目标 change 的产出物
- 不直接写 change 产出物文件
- 不接受只有“目录已创建”的结果
- 不接受缺少 `ROLE`、`CHANGE DETAIL`、`DEPENDENCIES` 的简略 prompt
- 不接受依赖文件缺失、`instructions` 缺失或 `template` 缺失时的“最佳努力”生成
- `context` 和 `rules` 属于约束，不是文件正文

## 输出

完成后输出：

- plan-id 与 `PLAN.md` 路径
- 每个 change 的目标、依赖、状态
- 失败项（如有）
- 下一步：按顺序执行 `/opsx-apply <change-name>`
