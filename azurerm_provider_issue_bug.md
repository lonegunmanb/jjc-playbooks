# AzureRM Provider Bug 类 Issue — 阶段路由

> 当 agent 读到本文件时，说明 [azurerm_provider_issue.md]({{azurerm_provider_issue.md}}) §6.0 编排流程已经把 Issue 的最终分类锁定为 **Bug**。本文件只负责**按看板列状态把 agent 路由到对应的阶段指南**，不再承载分析规则。

## 路由判定

读取 Trello 卡片当前所在的列名（调用 gateway 工具 `{"tool": "trello_card_list", "args": {"card_id": "<card_id>"}}` → `{id, name}`。禁止自己拼 `https://api.trello.com/1/...` 的 `Invoke-RestMethod` 调用。）

| 卡片当前列 | 阶段 | 必须加载并严格遵循的下一阶段文件 |
|---|---|---|
| **不在** "{{kanban.action.name}}" 列（如 "Triaged"、"Plan"、"Awaiting Approval" 等任何非 {{kanban.action.name}} 列） | Plan 阶段 | [azurerm_provider_issue_bug_plan.md]({{azurerm_provider_issue_bug_plan.md}}) |
| **在** "{{kanban.action.name}}" 列 | Action 阶段 | [azurerm_provider_issue_bug_action.md]({{azurerm_provider_issue_bug_action.md}}) |

## 路由规则

1. **以最新列状态为准**：在做出路由决策前，必须重新拉取卡片的当前列名，不得依赖任何缓存或更早步骤里的快照（人类可能在分类完成后把卡片移到了不同列）。
2. **二选一，无第三路径**：本文件只承认上表两条路径。若卡片处于其他无法确定的状态（如已归档、已删除、列名缺失），停止推进并在卡片上挂"路由阻塞"说明等待人类处置。
3. **加载即移交**：选定下一阶段文件后，agent 必须完整阅读该文件并按其规则继续工作；本文件不再提供任何额外的 Bug 处理指引。
4. **越界禁止**：未读取并采纳目标阶段文件之前，agent 不得动手做该阶段的代码修改、PR 提交、对外回复等任何"行动"。
