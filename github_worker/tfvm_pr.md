# TFVM PR 流程（精简版，保留全部约束）

## 0. 前置：从 Trello 卡片定位 PR

你正在处理 **Terraform Verified Module (TFVM)** 仓库的 GitHub Pull Request。PR 信息已由 gateway 预填到 CARD CONTEXT（`card_id` / `github_repo` / `github_number` / `github_url`）——**优先使用 CARD CONTEXT 的值**，下面的工具调用只在你需要原始 `name` / `desc` 时才走。

### 0.1 读取卡片 description

如需原始描述，调用 gateway 注册的 `trello_card_get` 工具：

```json
{"tool": "trello_card_get", "args": {"card_id": "<card_id>"}}
```

返回 `{id, name, desc, firstLine, idList, idBoard}`。**禁止**自己拼 `https://api.trello.com/1/...` 的 `Invoke-RestMethod` 调用——Trello 凭据由 Go 端持有，工具调用是唯一受支持的访问路径。

### 0.2 从第一行提取 repo / PR number

第一行必须是：`https://github.com/{owner}/{repo}/pull/{number}`

示例：`https://github.com/Azure/terraform-azurerm-aks/pull/123`

- 仓库：`Azure/terraform-azurerm-aks`
- PR 号：`123`

### 0.3 读取 GitHub PR 详情

```powershell
$prUrl = "https://api.github.com/repos/{owner}/{repo}/pulls/{number}"
$headers = @{ "Accept" = "application/vnd.github.v3+json"; "User-Agent" = "copilot-agent" }
if ($env:GITHUB_TOKEN) { $headers["Authorization"] = "token $env:GITHUB_TOKEN" }
$pr = Invoke-RestMethod -Uri $prUrl -Headers $headers
```

后续分析基于 `$pr.body`、`$pr.title`、`$pr.head`、`$pr.base`、`$pr.user`。

### 0.4 读取已有 Code Review Comments

> **⚠️ 读取 PR 详情时，必须同时读取已有的 Code Review Comments。这些评论是维护者和贡献者之间的讨论记录，对理解 PR 上下文至关重要。**

```powershell
# 行级 review comments（代码行上的评论）
$reviewComments = gh api "repos/{owner}/{repo}/pulls/{number}/comments" --paginate

# 整体 reviews（approve / request changes / comment）
$reviews = gh api "repos/{owner}/{repo}/pulls/{number}/reviews" --paginate

# PR 对话评论（非行级的普通评论）
$issueComments = gh api "repos/{owner}/{repo}/issues/{number}/comments" --paginate
```

如需筛选关键字段（减少输出量）：

```powershell
# 行级 review comments：作者、文件、行号、内容、创建时间
gh api "repos/{owner}/{repo}/pulls/{number}/comments" --paginate --jq '.[] | {user: .user.login, path: .path, line: .line, body: .body, created_at: .created_at}'

# 整体 reviews：作者、状态、内容
gh api "repos/{owner}/{repo}/pulls/{number}/reviews" --paginate --jq '.[] | {user: .user.login, state: .state, body: .body, submitted_at: .submitted_at}'
```

---

## 1. 目标

PR 与 Issue 的区别：Issue 是“报告问题”，PR 是“提交方案”。

本流程目标：

1. 准确理解 PR 意图并评估合理性。
2. 按来源区分策略（Fork vs 本仓库分支）。
3. 输出可直接用于维护者决策的结构化结论。

## 2. 核心原则

> **⚠️ 继承 `{{tfvm_issue.md}}` 第 2 节全部原则（判断必须有外部依据）。PR 场景同样强制执行。**

额外原则（必须全部满足）：

1. 先理解意图，再审实现。
2. 严格区分来源，采用差异化验证。
3. 外部 PR 必须先做安全审查，再做测试。
4. 修复建议必须是"已验证可行"，禁止未验证建议。
5. **Agent 不得执行 `gh pr merge` 或任何形式的 PR 合并操作。** 合并决策由人类维护者做出，Agent 的终态是将卡片移到 `{{kanban.wait.action_review.name}}`。
6. **卡片不在 `{{kanban.action.name}}` 列时，禁止执行任何行动计划。** 卡片评论中的讨论不等于批准。Agent 可以分析、调查、制定方案，但在卡片进入 `{{kanban.action.name}}` 之前，不得执行修改代码、push、创建分支、触发 CI 等任何变更操作。

### 2.1 TFVM 工具执行环境（强制）

> **⚠️ `terraform`、`tflint`、`make pre-commit` 等工具必须在容器内运行，不得在宿主机直接运行。**

- 主镜像：`mcr.microsoft.com/azterraform:latest`
- 备用镜像：`mcr.azure.cn/azterraform:latest`（主镜像超时可用）
- Pull 策略：`always`

#### 基本命令

```powershell
# 交互式
podman run --pull=always -it --rm `
	-v "${pwd}:/src" `
	-w /src `
	mcr.microsoft.com/azterraform:latest `
	bash

# 单命令执行
podman run --pull=always --rm `
	-v "${pwd}:/src" `
	-w /src `
	mcr.microsoft.com/azterraform:latest `
	<command>
```

#### 常用示例

```powershell
# pre-commit（自动生成文档 + 格式化）
podman run --pull=always --rm -v "${pwd}:/src" -w /src mcr.microsoft.com/azterraform:latest make pre-commit
```

> 说明：本机只执行 `make pre-commit`（格式化与文档生成）。lint、单元测试、e2e 等其他检查统一交给 GitHub Actions 运行，不在本机执行 `make pr-check`。

### 2.2 CI 状态监控工具：prblocker（强制用于 CI 等待）

> **⚠️ 等待 CI 完成时，必须使用 `prblocker` 工具轮询，禁止使用 `Start-Sleep` 固定等待。**
>
> **禁止行为（强制）**：
> - 禁止在 approve deployment 后自行执行 `Start-Sleep` 等待。
> - 禁止在调用 `prblocker` 前手动运行 `gh run view`、`gh pr checks` 等命令反复检查 CI 状态。
> - approve deployment 完成后，必须**立即**运行 `prblocker`，由它负责所有轮询和状态检测。

[prblocker](https://github.com/lonegunmanb/prblocker) 是一个 Go CLI 工具，持续轮询 GitHub Check Runs 和 Commit Status API，直到 PR 的 CI 检查到达终态。它会自动处理所有轮询和状态检测，Agent 不应自行实现任何形式的等待或状态检查逻辑。

#### 退出码含义

| 退出码 | 含义 | 后续行动 |
|--------|------|----------|
| 0 | 所有检查通过（success/skipped/neutral） | CI 通过，继续流程 |
| 1 | 至少一个检查失败/超时/取消 | 进入修复循环 |
| 2 | 有检查需要人工操作（环境审批等） | 用 `gh` CLI approve deployment 后重新运行 prblocker |

#### 基本用法

```powershell
# 获取 PR 最新 commit SHA
$sha = (gh pr view <number> --repo <owner>/<repo> --json headRefOid -q .headRefOid)

# 轮询等待 CI 完成（每 30 秒检查一次）
prblocker --owner <owner> --repo <repo> --pr <number> --commit-sha $sha --interval 30
```

#### 典型轮询流程

```
approve deployment → 运行 prblocker
  → 退出码 0：CI 通过，继续
  → 退出码 1：CI 失败，进入修复循环
  → 退出码 2：有新的 pending approval → approve 后重新运行 prblocker
```

### 2.3 GitHub 交互方式（强制）

> **⚠️ 所有与 GitHub API 的交互必须通过 `gh` 命令行工具或环境变量 `GITHUB_TOKEN` 完成，不得使用其他认证方式。**

- **优先使用 `gh` CLI**：查询 PR、Issue、Review Comments、approve deployment、创建评论等操作统一使用 `gh api`、`gh pr`、`gh issue` 等子命令。
- **直接调用 REST API 时**：使用环境变量 `$env:GITHUB_TOKEN` 作为认证令牌，通过 `Authorization: token $env:GITHUB_TOKEN` 请求头传递。

## 3. PR 基础信息采集

### 3.1 必采集元数据

| 字段 | 来源 | 说明 |
|------|------|------|
| PR 标题 | `$pr.title` | 一句话描述变更 |
| PR 描述 | `$pr.body` | 详细上下文 |
| 作者 | `$pr.user.login` | GitHub 用户名 |
| 来源分支 | `$pr.head.label` | `user:branch` / `org:branch` |
| 目标分支 | `$pr.base.ref` | 通常 `main` |
| 关联 Issue | `Fixes #N` / `Closes #N` 等 | 可为空 |
| 当前状态 | `$pr.state`、`$pr.draft` | open/closed、draft |
| CI 状态 | GitHub Checks API | 是否已有结果 |
| 已有 Review Comments | `pulls/{n}/comments` + `pulls/{n}/reviews` + `issues/{n}/comments` | 维护者/贡献者的讨论记录，见 0.4 节 |

### 3.1.1 处理 PR 描述中的截图/图片附件

扫描 PR 描述（`$pr.body`），查找图片 URL（通常是 `https://github.com/user-attachments/assets/...` 或其他图片链接，格式为 `![...](url)` 或 `<img src="url" ...>`）。

对于每个找到的图片 URL，**下载图片并使用 `markitdown` 进行 OCR 提取文字内容**：

```powershell
# 下载图片
Invoke-WebRequest -Uri "<image_url>" -OutFile "image_temp.png"
# 使用 markitdown OCR 提取内容
markitdown image_temp.png
```

将 OCR 提取的文字内容作为 PR 描述的补充信息，纳入后续分析。截图中可能包含 CI 输出、plan diff、错误日志或 Azure Portal 界面信息，这些是 review 的重要上下文。如果 OCR 无法提取有意义的内容（如纯图表或模糊截图），记录该事实并在分析中注明。

### 3.2 来源判定（关键）

```powershell
$isExternal = $pr.head.repo.full_name -ne $pr.base.repo.full_name
```

- 相同：本仓库分支 PR（内部 PR）
- 不同：Fork 分支 PR（外部 PR）

### 3.3 关联 Issue 判定

检查 PR 描述是否含 `Fixes #N` / `Closes #N` / `Resolves #N` 或 Issue URL。

- 有关联 Issue → 走 4.1
- 无关联 Issue → 走 4.2

### 3.4 已有 Code Review Comments 分析（强制）

> **⚠️ 若 PR 已有 review comments，必须在意图审查（第 4 节）之前完成本节分析。已有评论反映了维护者的关注点和贡献者的回应，直接影响后续审查方向。**

采集到 review comments 后，按以下维度分析：

#### 3.4.1 评论分类

| 类别 | 识别方式 | 说明 |
|------|----------|------|
| 维护者反馈 | 来自 repo collaborator/member 的评论 | 高优先级，代表官方意见 |
| 贡献者回复 | PR 作者的回复 | 理解贡献者意图与解释 |
| 社区讨论 | 其他参与者的评论 | 补充视角 |
| Bot/CI 评论 | 来自 bot 或 CI 系统 | 自动化检查结果 |

#### 3.4.2 合理性与正确性审查

对每条**非 bot 的实质性评论**，评估：

1. **评论指出的问题是否成立**：
   - 审查评论引用的代码位置，验证问题是否真实存在；
   - 若评论指出 bug / 设计缺陷 / 不合规，独立验证该判断是否正确；
   - 若评论提出的修改建议可能引入新问题，记录潜在风险。

2. **贡献者的回应是否充分**：
   - 贡献者是否已解决维护者提出的问题；
   - 回应是否合理（代码已修改 / 给出了充分理由拒绝修改）；
   - 是否有未回应的维护者评论（可能是阻塞合并的原因）。

3. **讨论是否达成共识**：
   - 识别仍在争议中的设计决策；
   - 识别已解决（resolved）与未解决的讨论线程；
   - 记录维护者明确要求但尚未实现的变更。

#### 3.4.3 纳入简报

在第 8 节输出模板中，已有 review comments 的分析结论必须包含：

- 维护者已提出的关键问题（避免重复提出相同问题）；
- 尚未解决的讨论点（需要关注）；
- 贡献者回应中不正确或不充分的部分（需要指出）；
- Agent 对评论中争议点的独立判断（附依据）。

## 4. 意图与合理性审查

### 4.1 有关联 Issue

1. 读取关联 Issue 全量信息（标题、描述、标签、评论）。
2. 交叉验证：
	 - PR 是否对准 Issue 问题；
	 - 修复路径是否合理（参照 `{{tfvm_issue_bug.md}}` / `{{tfvm_issue_feature_request.md}}`）；
	 - Issue 中提到但 PR 未覆盖的内容是否存在；
	 - PR 是否包含 Issue 未提到的额外改动（文档更新、lint 修复可视情况接受）。
3. 写入交叉验证结论并纳入最终简报。

### 4.2 无关联 Issue（构造“虚拟 Issue”）

1. 从标题、描述、commit、diff 推理 PR 意图。
2. 一句话定义“要解决的问题”（含预期行为 vs 当前行为）。
3. 按 `{{tfvm_issue.md}}` 第 3 节分类（Bug/功能/文档/破坏性等）。
4. 按对应类型指南评估合理性：
	 - Bug → `{{tfvm_issue_bug.md}}`
	 - 功能请求 → `{{tfvm_issue_feature_request.md}}`
	 - 文档 → 准确性与完整性
	 - 其他 → 对应指南
5. 输出分类与合理性结论。

## 5. 安全审查（外部 PR 必须优先）

任何测试前，必须完成：

| 检查项 | 说明 | 发现问题时行动 |
|--------|------|----------------|
| Workflow 文件 | `.github/workflows/` 是否修改 | 🔴 安全红线，建议关闭 PR |
| CI 配置 | `.github/`、`.azure-pipelines/` 等是否修改 | 🔴 安全红线，建议关闭 PR |
| 外部脚本执行 | 是否引入下载/执行外部脚本 | 🟡 深入审查意图 |
| 环境变量泄露 | 是否读取/输出敏感变量 | 🔴 安全红线 |
| Provider / Backend 配置 | 是否指向非官方 registry/backend | 🔴 安全红线 |

结论：

- 通过 → 继续后续流程。
- 不通过 → 在简报中明确风险，建议关闭 PR；停止测试运行。

## 6. 代码审查

### 6.1 变更范围

- 阅读完整 diff，确认改动意图与范围。
- 归类：新功能 / Bug 修复 / 文档 / 重构 / 依赖升级 / 混合。
- 列出关键文件与关键改动点。

### 6.2 TFVM 合规检查

> **TFVM 仅支持 AzureRM 实现，不存在 AzAPI / Porch 路径。**

| 维度 | 要求 |
|------|------|
| Provider 一致性 | 仅允许 AzureRM，不引入其他实现路径 |
| 变量设计 | 默认值、类型约束、validation 合理 |
| 非破坏性 | 升级后 `terraform plan` 不应出现意外 drift（结合 examples 验证） |
| 文档同步 | `README.md`、`examples/` 与变量/输出保持一致 |
| 常见错误 | 无遗留 `TODO`，不误提交敏感或临时文件 |

### 6.3 兼容性 / 破坏性评估

重点检查：

- 新增无默认值 required variable；
- 修改现有变量语义或类型；
- 删除/重命名既有变量或输出；
- 稳定配置下 `terraform plan` 是否出现意外 update/add/delete。

存在破坏性变更时，简报必须包含：

```
[BREAKING-CHANGE][REQUIRES-MAINTAINER-APPROVAL]
```

### 6.3.1 配置漂移（Configuration Drift）处理规则

> **❗ 配置漂移是一种错误，不可接受，必须尽全力修复。**

配置漂移是指 `terraform apply` 后再次执行 `terraform plan` 出现非空变更（即资源属性与声明状态不一致）。这通常是 Provider 行为、API 返回值或模块逻辑的 bug。

处理约束：

- 在测试中发现配置漂移时，必须将其作为待修复的错误记录并尝试解决。
- **禁止在根模块中将漂移字段添加到 `lifecycle { ignore_changes }` 来规避问题**，除非得到人类维护者的明确许可。
- 应从根因上解决漂移：修复属性映射、调整默认值、或在模块逻辑中正确处理 API 返回值差异。
- 若确实无法从模块层面解决（如 Provider bug），应在简报中详细说明原因，并请求维护者决策是否例外允许 `ignore_changes`。

## 7. 验证策略（按来源分支）

### 7.1 本仓库分支 PR（内部 PR）

**{{kanban.plan.name}} 阶段不做 CI 触发和修复**，只做 3~6 节分析。若需要本地辅助处理（如生成文档与格式化），可在容器内执行：

```powershell
podman run --pull=always --rm -v "${pwd}:/src" -w /src mcr.microsoft.com/azterraform:latest make pre-commit
```

本机不执行 `make pr-check`，lint、测试等检查交给 GitHub Actions。CI 验证与修复放到 {{kanban.action.name}} 阶段（见第 9 节）。以下为 CI 操作的参考说明：

CI 注意事项：

- **必须先用 `gh` CLI approve deployment**——CI 测试需要环境审批才会开始运行，不 approve 则测试永远不会启动。CI 流水线可能包含多个阶段/环境的审批，需要**逐个 approve 直到真正的 e2e 测试开始运行**。TFVM 的 e2e 测试 job 名称通常包含 `E2E Test`、`end to end`、`acc-tests` 等关键词，必须在 CI 中看到至少一个此类测试 job 已开始运行，才算完成了本次所有 approve。
- **approve 完成后立即运行 `prblocker`**，不得先执行 `Start-Sleep`、`gh run view`、`gh pr checks` 等任何形式的手动等待或状态检查：
  ```powershell
  $sha = (gh pr view <number> --repo <owner>/<repo> --json headRefOid -q .headRefOid)
  prblocker --owner <owner> --repo <repo> --pr <number> --commit-sha $sha --interval 30
  ```
  根据退出码决定下一步：
  - **退出码 0**（所有检查通过）→ CI 通过，卡片汇报并移到 `{{kanban.wait.action_review.name}}`；
  - **退出码 1**（检查失败）→ 进入修复循环；
  - **退出码 2**（需要审批）→ 用 `gh` CLI approve 新的 pending deployment，然后重新运行 `prblocker`。

#### CI 失败修复循环（方向探索式）

> **核心思路**：CI 首次失败时创建暂存点，制定至多 3 种修复方向，每个方向允许 3 次尝试。方向失败后回退到暂存点再试下一方向。所有方向耗尽则回退到暂存点让人类介入。
>
> **⚠️ 每个修复方向必须有外部依据支撑**：通过搜索官方文档、Provider 源码、Issue 讨论等找到明确的证据后才可确定该方向。写下修复方向时必须附带参考信息的链接（如 Terraform/Provider 文档 URL、GitHub Issue/PR 链接、源码文件路径等）。禁止凭猜测制定修复方向。

```
CI 首次失败
│
├─ 分析失败原因
├─ 制定至多 3 种修复方向（D1, D2, D3），在卡片汇报分析结论与方向列表
├─ 创建暂存点：git tag ci-fix-checkpoint-<pr>
│
▼ 方向 D1
├─ ⚠️ 检查卡片位置：不在 {{kanban.action.name}} → 立即停止，卡片评论汇报当前状态并终止
├─ 尝试 1/3 → 实施修复 → 运行 make pre-commit → push → approve deployment → prblocker
│   ├─ 退出码 0 → ✅ 成功，删除 tag，移到 {{kanban.wait.action_review.name}}
│   ├─ 退出码 2 → approve 后重跑 prblocker
│   └─ 退出码 1 → 检查卡片位置 → 检查人类新评论 → 仍在 {{kanban.action.name}} 且无人类干预 → 同方向改进，尝试 2/3
├─ 尝试 2/3 → 运行 make pre-commit → push → approve → prblocker
│   └─ 退出码 1 → 检查卡片位置 → 检查人类新评论 → 同方向改进，尝试 3/3
├─ 尝试 3/3 → 运行 make pre-commit → push → approve → prblocker
│   └─ 退出码 1 → ❌ 方向 D1 失败
│
├─ git reset --hard ci-fix-checkpoint-<pr>
├─ git push --force
│
▼ 方向 D2
├─ ⚠️ 检查卡片位置：不在 {{kanban.action.name}} → 立即停止
├─（3 次尝试，每次失败后检查卡片位置）
├─ 成功 → ✅ 删除 tag，流程结束
├─ 3 次全失败 → reset + force push 到暂存点
│
▼ 方向 D3
├─ ⚠️ 检查卡片位置：不在 {{kanban.action.name}} → 立即停止
├─（3 次尝试，每次失败后检查卡片位置）
├─ 成功 → ✅ 删除 tag，流程结束
├─ 3 次全失败 → reset + force push 到暂存点
│
▼ 所有方向耗尽（最多 3×3 = 9 次 CI 尝试）
├─ git reset --hard ci-fix-checkpoint-<pr>
├─ git push --force  ← 人类看到的是 CI 首次报错时的代码状态
├─ git tag -d ci-fix-checkpoint-<pr>
├─ 卡片写详细总结（每个方向 × 每次尝试的失败原因）
└─ 移到 {{kanban.wait.exception.name}}，人类介入
```

硬性规则：

- **暂存点 = CI 首次失败时的代码状态**，不是修复过程中的中间态。
- **卡片位置检查（强制）**：每次尝试修复前和每次换方向前，必须调用 Trello API 检查卡片是否仍在 `{{kanban.action.name}}` 列。如果卡片已被移出 `{{kanban.action.name}}`，说明人类希望 Agent 停下来，必须**立即停止所有操作**，在卡片评论中汇报当前进度（正在尝试哪个方向、第几次尝试、当前状态），然后终止。不得继续 push、不得 force push、不得移动卡片。
- **人类评论检查（强制）**：每次尝试失败后（prblocker 退出码 1），在检查卡片位置的同时，必须读取卡片最新评论，检查是否有不以 `{{kanban.agent_comment_prefix}}` 开头的新评论（即人类评论）。若发现人类新评论：
  - **人类给出指令**（如"换个方向"、"停止测试"、"先回滚"等）→ 回复 `{{kanban.agent_comment_prefix}} 了解`，然后按人类指令执行。
  - **人类提出问题**（如"当前是什么状态？"、"为什么选这个方向？"等）→ 以 `{{kanban.agent_comment_prefix}}` 开头的评论回答人类的问题，然后继续当前修复流程。
  - **人类命令停止** → 回复 `{{kanban.agent_comment_prefix}} 了解，正在清理测试环境`，执行环境清理（优先在测试目录执行 `terraform destroy`；若 destroy 失败或超时，使用 `az group delete --name <resource-group-name> --no-wait --yes` 异步删除资源组），清理完成后在卡片汇报清理结果和当前进度，然后终止。不得继续 push、不得 force push、不得移动卡片。
- **每换方向前**，必须 `git reset --hard` + `git push --force` 回到暂存点，确保每个方向从相同起点开始。
- **每次 push 前必须运行 `make pre-commit`**（在容器内执行），确保格式化和文档生成均已完成。未运行 pre-commit 直接 push 是被禁止的。
- 每次 push 后必须在卡片汇报：当前方向（D?）、尝试次数、修改文件、修改内容、原因。
- **修复方向必须有依据**：每个方向必须附带参考链接（官方文档、Provider 源码、GitHub Issue/PR、Stack Overflow 等）。无依据的方向不得执行。
- **最多 3 个方向 × 每方向 3 次 = 9 次尝试**。
- 任一尝试 CI 通过（退出码 0）→ 删除 checkpoint tag，移到 `{{kanban.wait.action_review.name}}`，流程结束。
- 全部耗尽 → reset + force push 回暂存点 → 删除 tag → 写总结 → 移到 `{{kanban.wait.exception.name}}`。

汇报示例：

```text
{{kanban.agent_comment_prefix}} 方向 D2 尝试 #2/3 — CI 失败修复

方向 D2：通过 dynamic block 替代 count 来避免 known-after-apply 问题
参考依据：https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks
         https://github.com/hashicorp/terraform-provider-azurerm/issues/XXXXX

失败原因：tflint 报错 `terraform_naming_convention` on variable `MyVar` in variables.tf:L42

修复：将变量名 `MyVar` 改为 `my_var`（snake_case），同步更新 main.tf 和 examples/ 中的引用。
参考依据：https://www.terraform.io/language/values/variables#naming-conventions

已推送 commit: chore: fix variable naming convention (D2 attempt 2)

等待 CI 重跑中。
```

### 7.2 Fork 分支 PR（外部 PR）

不能直接在 Fork PR 上触发受限 CI，需本地验证。

#### 7.2.1 本地检查顺序

```powershell
# 生成文档并格式化
podman run --pull=always --rm -v "${pwd}:/src" -w /src mcr.microsoft.com/azterraform:latest make pre-commit
```

说明：

- TFVM **不使用 Porch**，不需要 `.porch.yaml` 或 `porch run`。
- TFVM **不需要中央仓库同步**，无需“临时同步提交 + 回滚”流程。
- 本机只执行 `make pre-commit`，不执行 `make pr-check`。lint、测试等检查统一在 GitHub Actions 中运行（外部 PR 的 CI 由维护者授权后触发）。

#### 7.2.2 问题分类记录

| 类别 | 示例 | 说明 |
|------|------|------|
| Lint/格式错误 | fmt、命名、风格问题 | 规范一致性 |
| 文档生成差异 | README/examples 自动更新 | 可追溯变更 |
| 测试/检查失败 | GitHub Actions CI 失败 | 功能正确性与质量门禁 |
| 兼容性问题 | breaking change、missing defaults | 升级影响 |

**限制**：Agent 不能直接修改 Fork 分支代码，只能给出处理方案。

#### 7.2.3 向维护者给出两种方案

**方案 A：退回贡献者修复**

- 在 PR 评论列出问题；
- 每个问题给出“已验证有效”的修复方法；
- 等贡献者修复后再进入 {{kanban.plan.name}}。

适用：问题多/复杂；贡献者响应快；需贡献者自行判断设计取舍。

**方案 B：转 release 分支由 Agent/维护者修复**

- 在 Trello 卡片记录问题与修复方式；
- 建议流程：
	1. 从最新 `main` 创建 `release/<description>`；
	2. 将外部 PR 的 base 改到 release 分支并合并；
	3. 在 release 分支修复格式、文档、测试问题；
	4. 从 release 分支向 `main` 开新 PR；
	5. **必须先用 `gh` CLI 逐个 approve deployment**——不 approve 则 CI 测试不会启动。需持续 approve 直到看到至少一个名称包含 `E2E Test`、`end to end`、`acc-tests` 的测试 job 已开始运行，才算完成所有 approve；
	6. 使用 `prblocker` 轮询等待 CI 完成（退出码 2 表示有新的 pending approve，需 approve 后重新运行 prblocker）；
	7. CI 失败时，按内部 PR 同样的**方向探索式修复循环**执行（至多 3 方向 × 每方向 3 次，创建暂存点，每换方向回退到暂存点 force push）；
	8. CI 通过 → 删除 checkpoint tag，移到 `{{kanban.wait.action_review.name}}`；
	9. 所有方向耗尽 → reset + force push 回暂存点 → 写总结 → 移到 `{{kanban.wait.exception.name}}` 并终止。

适用：问题少且可快速修；贡献者不响应或很慢；变更价值高不宜长期搁置。

## 8. 对维护者的输出模板（必含）

### 8.1 PR 摘要

- PR 标题与编号
- 作者与来源类型（外部 / 内部）
- 一句话说明 PR 在做什么

### 8.2 意图审查

- 关联 Issue：有 / 无
- 有关联时：交叉验证结论（是否真正解决）
- 无关联时：虚拟 Issue 分类与合理性
- 总体结论：合理 / 部分合理 / 不合理

### 8.3 安全审查

- 结论：通过 / 发现问题
- 若有问题：明确风险点

### 8.4 代码审查发现

- 变更范围摘要
- TFVM 合规性结论
- 兼容性评估：非破坏性 / 可能破坏性
- 按严重程度列问题清单

### 8.5 测试结果

- 内部 PR：CI 通过 / 失败（附失败详情；如有环境门禁，注明是否已 approve deployment）
- 外部 PR：本地 `make pre-commit` 结论 + GitHub Actions CI 结论（如已触发）

### 8.6 唯一默认行动建议（P1-P6）

| 路由 | 触发条件 | 建议行动 |
|------|----------|----------|
| P1: 直接可合并 | 安全通过 + 代码无问题 + 测试通过 | Approve & Merge |
| P2: 需要小修改 | 有小问题但整体合理 | Request Changes（附修改清单） |
| P3: 外部 PR 需修复（方案 A） | Fork PR + 有问题 + 建议退回贡献者 | 在 PR 评论给问题与已验证修复方法 |
| P4: 外部 PR 需修复（方案 B） | Fork PR + 有问题 + 建议 Agent 修复 | 合并到 release 分支后修复并开新 PR |
| P5: 意图不合理 | 虚拟 Issue 审查不通过 / 修改方向错误 | Close with comment（附理由） |
| P6: 安全问题 | 安全审查不通过 | Close PR & report contributor |

### 8.7 可直接发送的 Review Comment 草稿

- 可直接贴到 GitHub PR；
- 包含问题与改进建议；
- 语气友好、建设性。

## 9. 生命周期与看板流转

### 9.1 {{kanban.plan.name}} 阶段

目标：**理解 PR、发现问题、提出方案**，不执行修复，不触发 CI。

通用步骤：

1. 做第 3 节信息采集；
2. 做第 4 节意图审查；
3. 做第 5 节安全审查（外部 PR 优先）；
4. 做第 6 节代码审查。

外部 PR 额外步骤：

1. 执行 7.2.1 本地检查（仅 `make pre-commit`，其他检查交给 GitHub Actions）；
2. 记录问题（7.2.2）；
3. 简报中提出方案 A 或 B（7.2.3）。

本仓库分支 PR（内部 PR）额外说明：

- {{kanban.plan.name}} 阶段仅做意图与合理性分析，可选在本机运行 `make pre-commit` 辅助处理格式化与文档；
- 可引用已有 CI 结果，但不主动触发新 CI，不 approve deployment。

{{kanban.plan.name}} 阶段产出：

- 在卡片发布结构化简报（按第 8 节）；
- 将卡片移动到 `{{kanban.wait.plan_review.name}}`。

### 9.2 {{kanban.action.name}} 阶段（维护者批准后）

#### 9.2.1 本仓库分支 PR

进入 {{kanban.action.name}} 表示：已批准 Agent 持续修复直到 CI 通过。

流程：

1. Approve CI run（如需）；
2. **必须先用 `gh` CLI approve deployment**——CI 测试（尤其 e2e）需要环境审批才会开始运行，不 approve 则测试永远不会启动。CI 流水线可能包含多个阶段/环境的审批，需要**逐个 approve 直到真正的 e2e 测试开始运行**。TFVM 的 e2e 测试 job 名称通常包含 `E2E Test`、`end to end`、`acc-tests` 等关键词，必须在 CI 中看到至少一个此类测试 job 已开始运行，才算完成了本次所有 approve；
3. **approve 完成后立即运行 `prblocker`**，不得先执行 `Start-Sleep`、`gh run view`、`gh pr checks` 等任何形式的手动等待或状态检查：
   ```powershell
   $sha = (gh pr view <number> --repo <owner>/<repo> --json headRefOid -q .headRefOid)
   prblocker --owner <owner> --repo <repo> --pr <number> --commit-sha $sha --interval 30
   ```
   根据退出码决定下一步：
   - **退出码 0**（所有检查通过）→ CI 通过，卡片汇报并移到 `{{kanban.wait.action_review.name}}`；
   - **退出码 1**（检查失败）→ 进入修复循环（见第 7.1 节 CI 失败修复循环）；
   - **退出码 2**（需要审批）→ 用 `gh` CLI approve 新的 pending deployment，然后重新运行 `prblocker`。

#### 9.2.2 外部 PR：方案 A 被批准

- 在 GitHub PR 评论中回复问题清单与已验证修复建议；
- 在卡片汇报已回复；
- 卡片移到 `{{kanban.wait.action_review.name}}`（等待贡献者响应）；
- 贡献者后续 push 会触发新 webhook，再进入 {{kanban.plan.name}}。

#### 9.2.3 外部 PR：方案 B 被批准

1. 从最新 `main` 创建 `release/<description>`；
2. 将外部 PR 的 base 改为 release 并合并；
3. 在 release 分支执行修复：
   - 运行 `make pre-commit`（在容器内执行）；
   - 修复 {{kanban.plan.name}} 阶段记录的 lint/代码问题；
   - 提交并推送；
4. 从 release 分支向 `main` 创建新 PR（触发 CI）；
5. **必须先用 `gh` CLI 逐个 approve deployment**——不 approve 则 CI 测试不会启动。需持续 approve 直到看到至少一个名称包含 `E2E Test`、`end to end`、`acc-tests` 的测试 job 已开始运行，才算完成所有 approve；
6. 使用 `prblocker` 轮询等待 CI 完成（退出码 2 表示有新的 pending approve，需 approve 后重新运行 prblocker）；
7. CI 失败时，按内部 PR 同样的**方向探索式修复循环**执行（见第 7.1 节，至多 3 方向 × 每方向 3 次，创建暂存点，每换方向回退到暂存点 force push）；
8. CI 通过 → 删除 checkpoint tag，移到 `{{kanban.wait.action_review.name}}`；
9. 所有方向耗尽 → reset + force push 回暂存点 → 写总结 → 移到 `{{kanban.wait.exception.name}}` 并终止。

### 9.3 遇到问题时

若出现无法自行解决的阻塞，按 Issue 流程处理：

- 在卡片留下详细说明；
- 将卡片移动到 `{{kanban.wait.exception.name}}`。
