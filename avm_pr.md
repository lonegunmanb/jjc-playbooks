# AVM PR 流程（共享部分 + 阶段路由）

> 你正在处理 **Azure Verified Module (AVM)** 仓库的 GitHub Pull Request。
>
> 本文件只包含**两阶段共享**的内容（PR 定位、核心原则、工具基线）。**阶段专属内容已拆分到独立文件**，必须按下方"阶段路由"按需加载。

---

## 0. 阶段路由（强制：进入工作前必须先判断卡片所在列）

第一件事：调用 gateway 工具 `trello_card_list` 读取卡片当前所在列名，按下表加载对应阶段文件。

```json
{"tool": "trello_card_list", "args": {"card_id": "<card_id>"}}
```

返回 `{id, name}`，`name` 即列名。

| 卡片所在列 | 阶段 | 必须加载的文件（绝对路径） | Agent 可做的事 |
|---|---|---|---|
| `{{kanban.plan.name}}` | `{{kanban.plan.name}}` 阶段 | `{{avm_pr_plan.md}}` | 读 PR / 评论 / Issue / diff、本地只读检查（外部 PR 才跑 porch）、写简报、移卡到 `{{kanban.wait.plan_review.name}}` |
| `{{kanban.action.name}}` | `{{kanban.action.name}}` 阶段 | `{{avm_pr_action.md}}` | approve deployment、push、force push、调用 Azure 资源、CI 修复循环、移卡到 `{{kanban.wait.action_review.name}}` 或 `{{kanban.wait.exception.name}}` |
| `{{kanban.wait.plan_review.name}}` | 等待人类批准计划 | —（不主动操作） | 仅在卡片有人类新评论时回应问题；不得移卡 |
| `{{kanban.wait.action_review.name}}` | 等待人类合并 PR | —（不主动操作） | 仅在卡片有人类新评论时回应问题；不得移卡、不得 `gh pr merge` |
| `{{kanban.wait.exception.name}}` | 等待人类介入 | —（不主动操作） | 仅在卡片有人类新评论时回应问题 |

> **⚠️ 加载错文件 = 走错阶段**。例如卡片在 `{{kanban.plan.name}}` 但加载了 `{{avm_pr_action.md}}`，agent 会以为自己有 push / approve 权限，从而越权操作。**必须先读卡片列、再加载对应文件**。
>
> **⚠️ 卡片不在 `{{kanban.action.name}}` 列时，禁止执行任何变更操作**（push / force push / approve deployment / 创建分支 / 触发 CI / `gh pr merge`）。这是核心原则 #7（见下文），**与卡片当前所处阶段无关，永远适用**。

---

## 1. 目标

PR 与 Issue 的区别：Issue 是"报告问题"，PR 是"提交方案"。

本流程目标：

1. 准确理解 PR 意图并评估合理性。
2. 按来源区分策略（Fork vs 本仓库分支）。
3. 输出可直接用于维护者决策的结构化结论。
4. 在维护者批准后，把 PR 的 CI 跑绿（或干净地交回人类）。

---

## 2. PR 定位（共享）

PR 信息已由 gateway 预填到 CARD CONTEXT（`card_id` / `github_repo` / `github_number` / `github_url` / 等），优先使用那里的值。下面的工具调用只在你需要原始 `name` / `desc` / `idList` 时才走。

### 2.1 读取卡片 description

如需原始描述（含截图、扩展链接等），调用 gateway 注册的 `trello_card_get` 工具：

```json
{"tool": "trello_card_get", "args": {"card_id": "<card_id>"}}
```

返回 `{id, name, desc, firstLine, idList, idBoard}`。**禁止**自己拼 `https://api.trello.com/1/...` 的 `Invoke-RestMethod` 调用——Trello 凭据由 Go 端持有，工具调用是唯一受支持的访问路径。

### 2.2 从第一行提取 repo / PR number

第一行必须是：`https://github.com/{owner}/{repo}/pull/{number}`

示例：`https://github.com/Azure/terraform-azurerm-avm-res-app-containerapp/pull/117`

- 仓库：`Azure/terraform-azurerm-avm-res-app-containerapp`
- PR 号：`117`

### 2.3 读取 GitHub PR 详情

```powershell
$prUrl = "https://api.github.com/repos/{owner}/{repo}/pulls/{number}"
$headers = @{ "Accept" = "application/vnd.github.v3+json"; "User-Agent" = "copilot-agent" }
if ($env:GITHUB_TOKEN) { $headers["Authorization"] = "token $env:GITHUB_TOKEN" }
$pr = Invoke-RestMethod -Uri $prUrl -Headers $headers
```

后续分析基于 `$pr.body`、`$pr.title`、`$pr.head`、`$pr.base`、`$pr.user`。

> **来源判定（关键）**：`$isExternal = $pr.head.repo.full_name -ne $pr.base.repo.full_name`
>
> - 相同 → 内部 PR（本仓库分支）
> - 不同 → 外部 PR（Fork 分支）
>
> 来源不同决定 `{{kanban.plan.name}}` 与 `{{kanban.action.name}}` 两阶段的具体路径。

---

## 3. 核心原则（共享，两阶段都必须遵守）

> **⚠️ 继承 `{{avm_issue.md}}` 第 2 节全部原则（判断必须有外部依据）。PR 场景同样强制执行。**

1. **任何决策、判断、行动计划，必须第一时间在对应的 Trello 卡片上评论汇报。** 人类不喜欢黑盒——Agent 必须通过卡片评论让人类实时看到：当前在做什么、为什么这么做、下一步打算做什么、遇到了什么问题。禁止"闷头干完再汇报"。最低频率：
   - 开始分析 PR 时：评论"开始分析，初步判断 …"
   - 做出关键决策时（例如：判定为安全/不安全、决定运行哪些测试、决定是否 push 修改）：评论决策内容和依据
   - 遇到阻塞或异常时：立即评论说明
   - 阶段性完成时（例如：安全审查完成、测试完成、修复 push 完成）：评论结果摘要
   - 卡片移动列之前：评论说明为何移动以及结论
2. 先理解意图，再审实现。
3. 严格区分来源，采用差异化验证。
4. 外部 PR 必须先做安全审查，再做测试。
5. 修复建议必须是"已验证可行"，禁止未验证建议。
6. **Agent 不得执行 `gh pr merge` 或任何形式的 PR 合并操作。** 合并决策由人类维护者做出，Agent 的终态最远只能是把卡片移到 `{{kanban.wait.action_review.name}}`。
7. **卡片不在 `{{kanban.action.name}}` 列时，禁止执行任何变更操作**（修改代码、push、force push、创建分支、触发 CI、approve deployment 等）。卡片评论中的讨论不等于批准。Agent 可以分析、调查、制定方案，但变更动作必须等卡片进入 `{{kanban.action.name}}`。

---

## 4. 工具基线（共享）

### 4.1 AVM 工具执行环境（强制：容器内运行）

> **⚠️ `terraform`、`tflint`、`conftest`、`./avm` 等 AVM 工具必须在容器内运行，不得在宿主机直接运行。**

- 主镜像：`mcr.microsoft.com/azterraform:avm-latest`
- 备用镜像：`mcr.azure.cn/azterraform:avm-latest`（主镜像超时可用）
- Pull 策略：`always`
- **容器运行时：`podman` 和 `docker` 均可**。下方示例统一写作 `podman`，但 CLI 参数 100% 兼容——你可以把所有 `podman` 直接替换成 `docker`（或反之），其余命令一字不改。请按宿主机实际安装的运行时选用；两者都装时优先 `podman`（与 CI 一致）。
- **路径占位符 `<work_dir>`**：指目标仓库在宿主机上的绝对路径（由 Gateway 在 CARD CONTEXT 中预填，Windows 形如 `C:\path\to\repo`，macOS/Linux 形如 `/path/to/repo`）。下方示例统一写作 `<work_dir>`，执行时请按宿主机实际路径替换；不要把它当成字面字符串。

#### 基本命令

```powershell
# 交互式
podman run --pull always -it --rm `
  -v "<work_dir>:/src" `
  -w /src `
  mcr.microsoft.com/azterraform:avm-latest `
  bash

# 单命令执行
podman run --pull always --rm `
  -v "<work_dir>:/src" `
  -w /src `
  mcr.microsoft.com/azterraform:avm-latest `
  <command>
```

#### 需要 Azure 认证时

```powershell
podman run --pull always --rm `
  -v "<work_dir>:/src" `
  -v "$HOME\.linux-az-credential:/home/runtimeuser/.azure" `
  -w /src `
  mcr.microsoft.com/azterraform:avm-latest `
  <command>
```

> 与 CI 使用同一镜像 + `--pull always`，可避免工具链版本偏差导致误判。

### 4.2 GitHub 交互方式（强制：仅限 gh CLI / GITHUB_TOKEN）

> **⚠️ 所有与 GitHub API 的交互必须通过 `gh` 命令行工具或环境变量 `GITHUB_TOKEN` 完成，不得使用其他认证方式。**

- **优先使用 `gh` CLI**：查询 PR、Issue、Review Comments、approve deployment、创建评论等操作统一使用 `gh api`、`gh pr`、`gh issue` 等子命令。
- **直接调用 REST API 时**：使用环境变量 `$env:GITHUB_TOKEN` 作为认证令牌，通过 `Authorization: token $env:GITHUB_TOKEN` 请求头传递。

---

## 5. 加载阶段文件并开始工作

读完上面 1–4 节后：

1. 调 Trello API 拿到卡片当前列名；
2. 按 §0 路由表加载对应阶段文件：
   - 卡片在 `{{kanban.plan.name}}` → 读 `{{avm_pr_plan.md}}`
   - 卡片在 `{{kanban.action.name}}` → 读 `{{avm_pr_action.md}}`
   - 卡片在其他列 → 不主动操作；只在人类评论时回应；不得移卡；
3. 严格按所加载阶段文件的章节顺序执行；
4. 每次卡片被移到新列，**重新走本节流程**（即重新判断、重新加载对应阶段文件）。
