# AzureRM Provider Bug 类 Issue 处理指南 — Plan 阶段

> **本文件覆盖：** Issue 已被人类在分类阶段确认为 Bug 之后，由 agent 在 Trello 卡片上产出 **PR 概要** 供人类审查。
>
> **本文件不覆盖：** 实际写代码、跑测试、提 PR、处理 review 反馈——这些归 [azurerm_provider_issue_bug_action.md]({{azurerm_provider_issue_bug_action.md}})。

## 1. 前提与边界

### 1.1 启动前提

进入本阶段时，[azurerm_provider_issue.md]({{azurerm_provider_issue.md}}) 的 Step A→C 已经在卡片上产出并经人类确认：

- 分类结论 = Bug
- 受影响资源 + 属性
- 根因定位（Go 文件 + 函数 + 子类）
- 复现结论（Step B 试验记录或"未复现，原因 X"）
- 问题层级（Provider / Azure API / Terraform Core）

**这些都是 Plan 的输入事实，不要在 Plan 阶段重做或翻案。** 若发现与实际不符，在卡片上请求人类重启 Step B 或 Step A，不要就地修改。

### 1.2 协同模型

Agent + 人类通过看板协同：

- 所有"行动"都是 agent 执行，人类只做审批/否决/调整。
- 卡片当前应在 `{{kanban.wait.plan_review.name}}` 列；agent 在该列只能产出待审批内容，不得改代码、提 PR、代发 Issue 评论。
- 人类在卡片上明确批准 PR 概要后，卡片由人类移到 `{{kanban.action.name}}` 列，agent 才进入 Action 阶段。

### 1.3 Plan 阶段不做的事

- 不写代码、不跑 `go build` / `go test` / `terraform plan`
- 不创建 / 修改 / 删除任何 Azure 资源
- 不复现 Bug（复现是分类阶段的责任）
- 不预读 Action 阶段才需要的实施指令文件（`implementation-*`、`testing-*`、`code-review-*` 等）
- 不代发 GitHub Issue / PR 评论

## 2. Plan 阶段的产出：PR 概要

唯一交付物是一段写在 Trello 卡片上的 **PR 概要**，作为 `{{kanban.agent_comment_prefix}}` 评论发出。结构如下，长度控制在让人类一屏看完：

```
## PR 概要 — Issue #<number>

### 一句话问题
<受影响资源 + 用户实际遇到的现象，1 句>

### 根因（引用分类阶段证据）
- 文件：<repo-relative path>:<line>
- 函数：<func name>
- 缺陷类型：<Schema / CRUD / Expand-Flatten / API 版本 / 验证 / 竞态 / Crash 之一>
- 关键证据：<Step C reviewer 已确认的 1–3 行结论，引用卡片评论 #N>

### 修复方案（一句话 + 关键改动）
- 一句话：<例如"为 Create 函数中的 connection_policy PUT 调用包一层 pluginsdk.Retry，处理 server 异步创建后的 eventual consistency 404">
- 关键改动：
  1. <文件:函数：改成什么，参照仓内已有同类模式 X>
  2. <如有第二处必改的地方>
- 参照模式：<repo 内已有的同类实现，例如 mssql_database_resource.go:620>

### 兼容性
- 是否破坏性：否 / 是（"是"必须按 §3 高亮并提供迁移说明）
- `terraform plan` 对既有配置的预期影响：无 drift / 仅 update in-place / 其他

### 测试
- 新增 / 修改的 acceptance test：<TestAccXxx_yyy 名称或"无新增，复用 Z 测试">
- 已知覆盖度缺口：<参见 §4 覆盖度分析；无则写"无"

### Workaround
- 是否存在：是 / 否
- 若是：<操作步骤 1 行；并在"维护者决策"中提供"先回复 workaround"选项>

### 维护者决策（请勾选其一）
- [ ] 批准 PR 概要 → agent 进入 Action 阶段
- [ ] 仅代发 workaround 回复，暂不修代码
- [ ] 调整方案：<人类填写要求>
- [ ] 否决（理由：<人类填写>）
```

> **不要扩成长篇文档。** PR 概要的目的是让人类用 1–2 分钟决定 "可以动手 / 还要改 / 不要做"，不是预先写完整 PR 描述。完整 PR 描述在 Action 阶段提交 PR 时再写。

## 3. 默认非破坏性；破坏性必须高亮

修复方案默认满足：

1. 现有用户仅需升级 Provider 版本，无需改 HCL。
2. 既有稳定配置 `terraform plan` 无 drift。
3. 不改属性语义、不引入强制新参数、不要求手工 state 操作。

如果根因评估显示**无法**满足上述任一条，PR 概要标题必须加 `[BREAKING-CHANGE]`，并在"修复方案"段补 3 行：

- 为何无法非破坏性修复（被否决的非破坏性方案 + 否决理由）
- 用户迁移步骤简述
- 是否需要 `StateUpgraders`

破坏性方案的细节模板（state migration 代码骨架、changelog 写法、feature flag 模式）在 Action 阶段加载仓内 `migration-guide.instructions.md` / `schema-patterns.instructions.md` 时再处理；Plan 阶段只需说明决策与影响。

## 4. 何时需要做"覆盖度分析"

**默认不做。** 只在分类阶段证据显示问题**只在特定参数值下触发**（特定 SKU / region / API 版本 / 磁盘类型等）且 Step B 试验未覆盖该参数时，做一次纯静态分析（**不调 Azure**）：

1. 在 `*_test.go` 找到与用户场景最接近的现有测试。
2. 比对关键参数值与用户配置的差异。
3. 在 PR 概要的"已知覆盖度缺口"中写明：最相关测试名 + 关键差异 + 结论是"Provider 通用缺陷"还是"仅在某些参数值下触发"。

如果结论是"仅在某些参数值下触发"，PR 概要要在"修复方案"中加一行：是否考虑该问题可能源自 Azure API 端的局部行为差异，以及这影响修复优先级的方式。

## 5. 进入 Action 阶段的门槛

人类在卡片评论中明确写出"批准 PR 概要"或等价表述后，agent 才能：

1. 把卡片状态/列变更交由人类处理（agent 不自移卡片到 `{{kanban.action.name}}`）。
2. 加载 [azurerm_provider_issue_bug_action.md]({{azurerm_provider_issue_bug_action.md}}) 进入 Action 阶段。
3. Action 阶段以 PR 概要中"修复方案 + 测试 + 兼容性"三段为唯一输入，不得擅自扩大范围。

未获批准前，agent 在卡片上等待，可以回答人类追问，但不得开始写代码、不得代发 GitHub 评论、不得切分支推送。
