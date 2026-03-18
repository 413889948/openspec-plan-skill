# OpenSpec Planner

将大型需求拆成多个可独立推进的 change，并用 `PLAN.md` 管理依赖、状态和执行顺序。

说明：这里提到的委托是当前运行环境提供的 `task` 工具接口，不是 OpenCode 官方 agents 文档里的 `@agent` 调用方式。

## 适用场景

- 需求跨多个模块或阶段
- 单个 change 预计超过 8 个 tasks
- 需要明确 hard/soft 依赖
- 需要并行创建多个 change 产出物

## 核心边界

- `openspec-plan` 只负责规划、委托、验证
- `openspec-ff-change` 负责单个 change 的产出物生成
- `openspec-plan` 不直接写 `openspec/changes/*/proposal.md` 或 `specs/*.md`
- 缺少关键上下文、关键文件或校验失败时，必须失败，不做 best effort 补写

## 标准流程

1. 收集需求
2. 拆分 change，标明 hard/soft 依赖
3. 让用户确认拆分方案
4. 写入 `openspec/plans/<plan-id>/PLAN.md`
5. 委托 `openspec-ff-change` 创建每个 change 的 `applyRequires` 产出物
6. 验证文件和状态
7. 输出可执行顺序与下一步命令

说明：如果某个 change 生成 `tasks.md`，其最后一个任务应回写来源 `PLAN.md`，把该 change 标记为“已实现”。

## 状态模型

每个 change 使用以下状态：

`planned -> creating -> created -> verified -> ready`

失败时标记为 `failed`。

## 委托协议

调用 `task()` 时必须包含合法签名：至少提供 `category` 或 `subagent_type`，并提供 `description`。这是当前环境的任务委托接口要求，不是 OpenCode 官方 `@agent` 语法。

每个 task 委托负载至少包含以下字段：

- `TASK`
- `ROLE`
- `PLAN CONTEXT`
- `PLAN PATH`
- `CHANGE ID`
- `CHANGE DETAIL`
- `DEPENDENCIES`
- `RELEVANT REQUIREMENTS`
- `CONSTRAINTS`
- `SUCCESS CRITERIA`
- `MUST NOT`

缺少任一字段时，直接视为 `invalid delegation payload`。

## 调度规则

- 无 hard 依赖的 change 可并行运行
- 有 hard 依赖的 change 只能在上游 `verified` 或 `ready` 后启动
- 后台任务按 30-60 秒轮询
- 单任务最长等待 10 分钟
- 超时或临时失败最多重试 1 次
- 重试后仍失败则标记 `failed`

## 验证规则

主 agent 必须同时验证：

- `CHANGE.md` 存在且非空
- `proposal.md` 存在且非空
- 所需 `specs/**/*.md` 存在且非空
- `openspec-cn status --change "<name>" --json` 中 `applyRequires` 全部为 `done`
- 任务代理返回的文件列表与实际文件一致
- 如果存在 `tasks.md`，最后一个任务是否要求回写来源 `PLAN.md` 并将当前 change 标记为“已实现”

任一项失败都不能标记为 `ready`。

## PLAN.md 最小结构

- 总览：日期、需求摘要、change 数量
- 表格：编号、名称、目标、hard、soft、状态
- 依赖图
- 建议执行顺序

## 示例

输入：

```text
我需要一个完整的用户认证系统，包括注册、登录、JWT、OAuth 和权限管理。
```

可能拆分为：

```text
001-user-schema
002-auth-api
003-oauth-integration
004-permission-system
005-auth-ui
```

依赖示意：

```text
[001-user-schema] --hard--> [002-auth-api]
[002-auth-api] --hard--> [003-oauth-integration]
[002-auth-api] --hard--> [004-permission-system]
[002-auth-api] --hard--> [005-auth-ui]
```

## 输出

完成后应给出：

- `plan-id`
- `PLAN.md` 路径
- 每个 change 的目标、依赖、状态
- 失败项
- 下一步命令：`/opsx-apply <change-name>`

## 相关 skill

- `openspec-ff-change`: 生成单个 change 的产出物
- `openspec-apply-change`: 实现 change 中的任务
- `openspec-verify-change`: 验证实现是否匹配产出物
- `openspec-archive-change`: 实现完成后归档
