# AVM PR 流程 — {{kanban.action.name}} 阶段（卡片在 `{{kanban.action.name}}` 列时加载）

> **前置阅读（强制）**：本文件只覆盖 {{kanban.action.name}} 阶段专属内容。继承 {{avm_pr.md}} 的全部共享内容（PR 定位、核心原则、工具基线）。如未读取共享文件，禁止开始本阶段工作。

---

## 0. ⚠️ 入口仪式：你已切换到执行模式

进入本文件意味着卡片已被人类维护者批准移到 `{{kanban.action.name}}` 列。此后所有操作都是**真实变更、可能不可逆**：会调用真实 GitHub API、推送 commit、调用 Azure 资源、可能产生费用、可能 force push 历史。

**进入执行模式前，先读完本节再开始任何动作。**

### 0.1 {{kanban.action.name}} 阶段的唯一终态条件

> **`prblocker` 退出码 = 0**（即 PR 上所有 GitHub Check Runs 与 Commit Status 都到达 success / skipped / neutral）。

只有满足这一条件，卡片才可移到 `{{kanban.wait.action_review.name}}`。其他任何"看起来完成了"的判据**都不算终态**——见下节反模式。

### 0.2 禁止的认知模式（强制阅读，每条都对应过真实事故）

| ❌ 错误认知 | ✅ 正确认知 |
|---|---|
| "WAITING 状态的 job 说明 CI 在跑" | WAITING = **没在跑**，等待环境审批。Agent 必须自己 approve，否则 job 永远不启动。 |
| "代码已经写完了 / CI 在自动跑了 → 我可以移到 {{kanban.wait.action_review.name}}" | 唯一终态是 prblocker 退出码 0。"代码完整"是 {{kanban.plan.name}} 阶段的判据，不是 {{kanban.action.name}} 的。 |
| "环境审批是维护者的事 / 等维护者 approve" | 环境审批（GitHub Actions environment deployment approval）是 **{{kanban.action.name}} Agent 自己的事**，不是看板审批，**两者完全不同**。 |
| "看到几个绿勾就是 verified" | 必须 12 个关键 job（lint + integration + 全部 examples e2e）至少 success/skipped/neutral，prblocker 退出码 0 才算 verified。 |
| "CI 失败 → 看一眼、改一改、push 再试试" | 必须按"3 方向 × 3 次"的方向探索式修复循环执行（见第 4 节），每个方向有外部依据、有反例预期、有暂存点。 |
| "approve 完先 sleep / 看看状态再说" | 严禁 `Start-Sleep`、严禁先跑 `gh run view` / `gh pr checks`。approve 完成必须**立即**调 `prblocker`。 |

**自检**：在执行任何操作前，先问自己——"我现在的判断是不是上面任意一条 ❌？" 是的话，**停下，重新读本节**。

### 0.3 {{kanban.action.name}} 阶段强制 checklist（任何一项不满足都不得移卡到 {{kanban.wait.action_review.name}}）

```
[ ] 已对该 PR 的所有 pending_deployments 用 gh CLI 显式 approve（不是等待维护者）
[ ] 已确认至少一个 examples/<name> 的 e2e job 进入 in_progress 或更新状态
[ ] 已运行 prblocker 并取得退出码
[ ] 退出码 = 0（所有检查 success/skipped/neutral）
    └ 满足才可移到 {{kanban.wait.action_review.name}}
[ ] 退出码 = 2 → 已再次 approve 并重跑 prblocker（循环至 ≠ 2）
[ ] 退出码 = 1 → 已按"3 方向 × 3 次"修复循环执行；通过则移 {{kanban.wait.action_review.name}}，耗尽则回滚至暂存点并移 {{kanban.wait.exception.name}}
```

---

## 1. prblocker 工具（CI 状态监控）

> **⚠️ 等待 CI 完成时，必须使用 `prblocker` 工具轮询，禁止使用 `Start-Sleep` 固定等待。**
>
> **禁止行为**：
> - approve deployment 后**禁止**先执行 `Start-Sleep` 等待。
> - 调用 `prblocker` 前**禁止**手动运行 `gh run view`、`gh pr checks` 等命令反复检查 CI 状态。
> - approve deployment 完成后必须**立即**运行 `prblocker`，由它负责所有轮询和状态检测。

[prblocker](https://github.com/lonegunmanb/prblocker) 是一个 Go CLI 工具，持续轮询 GitHub Check Runs 和 Commit Status API，直到 PR 的 CI 检查到达终态。它会自动处理所有轮询和状态检测，Agent **不应自行实现**任何形式的等待或状态检查逻辑。

### 1.1 退出码含义（核心决策表）

| 退出码 | 含义 | 后续行动 |
|--------|------|----------|
| **0** | 所有检查通过（success/skipped/neutral） | CI 通过，卡片汇报后移到 `{{kanban.wait.action_review.name}}` |
| **1** | 至少一个检查失败/超时/取消 | 进入第 4 节 CI 失败处理 |
| **2** | 有检查需要人工操作（环境审批等） | 用 `gh` CLI approve 新的 pending deployment，然后**立即**重新运行 prblocker |

### 1.2 基本用法

```powershell
# 获取 PR 最新 commit SHA
$sha = (gh pr view <number> --repo <owner>/<repo> --json headRefOid -q .headRefOid)

# 轮询等待 CI 完成（每 30 秒检查一次）
prblocker --owner <owner> --repo <repo> --pr <number> --commit-sha $sha --interval 30
```

---

## 2. 内部 PR（本仓库分支）执行流程

进入 {{kanban.action.name}} 表示：已批准 Agent 持续修复直到 CI 通过。

### 2.1 第一步：approve deployment（不是维护者的事，是 Agent 的事）

> **⚠️ 这是 PR 126 事故的根源——Agent 误以为环境审批是维护者的事，结果整个 {{kanban.action.name}} 阶段一行都没做。再次强调：环境审批是 {{kanban.action.name}} Agent 必须自己执行的第一步。**

CI 测试（尤其 e2e）需要环境审批才会开始运行，**不 approve 则测试永远不会启动**。CI 流水线可能包含多个阶段/环境的审批，需要**逐个 approve 直到真正的 e2e 测试开始运行**。

AVM 的 e2e 测试以 `examples/` 目录下的子目录名命名（如 `default`、`complete`、`vnet` 等），**必须在 CI 中看到至少一个以这些子目录名为名称的测试 job 已开始运行**，才算完成了本次所有 approve。

操作示例：

```powershell
# 1. 找到当前 PR 的最新 workflow run
$runId = (gh run list --repo <owner>/<repo> --branch <head-branch> --limit 1 --json databaseId -q '.[0].databaseId')

# 2. 查询 pending deployments
gh api "repos/<owner>/<repo>/actions/runs/$runId/pending_deployments"

# 3. 对每个 pending environment 调 approve
gh api "repos/<owner>/<repo>/actions/runs/$runId/pending_deployments" `
  -X POST `
  -f environment_ids='[<env_id>]' `
  -f state=approved `
  -f comment='approved by agent (in action)'

# 4. 验证：至少一个 examples/<name> 的 job 进入 in_progress
gh run view $runId --repo <owner>/<repo> --json jobs -q '.jobs[] | select(.name | contains("Test Example")) | {name, status}'
```

只要还能看到 `pending_deployments` 不为空，或 `Test Example - *` 中没有任何一个进入非 `queued`/`waiting` 状态，就**继续 approve**。

### 2.2 第二步：立即运行 prblocker

approve 完成后**立即**运行（不得先 sleep、不得先 `gh run view` / `gh pr checks` 检查状态）：

```powershell
$sha = (gh pr view <number> --repo <owner>/<repo> --json headRefOid -q .headRefOid)
prblocker --owner <owner> --repo <repo> --pr <number> --commit-sha $sha --interval 30
```

### 2.3 第三步：按退出码分支

- **退出码 0**（所有检查通过）→ 在卡片汇报"CI 全部通过 ✅"，移到 `{{kanban.wait.action_review.name}}`，流程结束。
- **退出码 2**（需要审批）→ 回到 2.1 再 approve，然后立即重跑 prblocker（循环直到 ≠ 2）。
- **退出码 1**（有失败）→ 进入第 4 节。

---

## 3. 外部 PR 的 {{kanban.action.name}}

### 3.1 方案 A 被批准

- 在 GitHub PR 评论中回复问题清单与已验证修复建议；
- 在卡片汇报已回复；
- 卡片移到 `{{kanban.wait.action_review.name}}`（等待贡献者响应）；
- 贡献者后续 push 会触发新 webhook，再进入 {{kanban.plan.name}}。

### 3.2 方案 B 被批准

1. 从最新 `main` 创建 `release/<description>`；
2. 将外部 PR 的 base 改为 release 并合并；
3. 在 release 分支执行修复：
   - 在容器内运行 `./avm pre-commit` 同步中央治理；
   - 修复 {{kanban.plan.name}} 阶段记录的 lint/conftest/代码问题；
   - 提交并推送；
4. 从 release 分支向 `main` 创建新 PR（触发 CI）；
5. 按第 2 节同样的流程：approve deployment（必须由 Agent 自己做）→ prblocker 轮询；
6. CI 失败时按第 4 节的方向探索式修复循环执行；
7. CI 通过 → 删除 checkpoint tag，移到 `{{kanban.wait.action_review.name}}`；
8. 所有方向耗尽 → reset + force push 回暂存点 → 写总结 → 移到 `{{kanban.wait.exception.name}}` 并终止。

---

## 4. CI 失败处理

### 4.1 第一步：判断是否需要修改代码

| 失败类型 | 示例 | 处理路径 |
|---|---|---|
| 瞬态/环境问题 | 资源配额、Azure API 抖动、超时、runner 故障 | 走 4.2 重跑 |
| 人类指示重跑 | 卡片评论"重跑 CI" | 走 4.2 重跑 |
| 代码问题 | lint、conftest、plan 漂移、e2e 实际逻辑错误 | 走 4.3 修复循环 |

### 4.2 CI 重跑（gh run rerun）

> **⚠️ 当分析失败原因后判断不需要修改代码时，必须使用 `gh run rerun` 重跑失败的 job，而不是进入修复循环。**

```powershell
# 1. 找到失败的 workflow run ID
$runId = (gh run list --repo <owner>/<repo> --branch <branch> --status failure --limit 1 --json databaseId -q '.[0].databaseId')

# 2. 只重跑失败的 job（不重跑已通过的）
gh run rerun $runId --repo <owner>/<repo> --failed

# 3. 如有新的 pending deployment，按 2.1 再 approve
# 4. 立即用 prblocker 等待结果
$sha = (gh pr view <number> --repo <owner>/<repo> --json headRefOid -q .headRefOid)
prblocker --owner <owner> --repo <repo> --pr <number> --commit-sha $sha --interval 30
```

**重跑次数限制：同一失败原因最多重跑 3 次**。3 次后仍失败：

- 失败原因不变（仍是环境/瞬态）→ 卡片汇报详情，移到 `{{kanban.wait.exception.name}}`。
- 失败原因变化（变为代码问题）→ 进入 4.3 修复循环。

### 4.3 修复循环（方向探索式）

> **核心思路**：CI 首次失败时创建暂存点，制定至多 3 种修复方向，每方向 3 次尝试。方向失败后回退到暂存点再试下一方向。所有方向耗尽则回退到暂存点让人类介入。
>
> **⚠️ 不同方向 = 不同的因果假设**。如果两个方向的前提假设相同（只是实现手段不同），算同一个方向。自检：三个方向是否在回答不同的「为什么会失败」。
>
> **⚠️ 每个方向必须附带反面预期**。写下方向时同时写明「如果假设错误，预期会观察到什么现象」。尝试后若观察到反面预期，果断放弃，不要在同一假设上微调。
>
> **⚠️ 每个修复方向必须有外部依据支撑**。通过搜索官方文档、Provider 源码、Issue 讨论等找到明确证据后才可确定该方向。写下方向时必须附带参考链接（Terraform/Provider 文档 URL、GitHub Issue/PR 链接、源码文件路径等）。**禁止凭猜测制定修复方向**。

#### 4.3.1 流程图

```
CI 首次失败
│
├─ 分析失败原因
├─ 判断：是否需要修改代码？
│   ├─ 不需要 → 4.2 重跑（最多 3 次）
│   └─ 需要 → 进入下方修复循环
│
├─ 做隔离实验缩小根因（改一个变量，观察结果变不变）
├─ 制定至多 3 种方向（D1/D2/D3），每方向不同因果假设，附反面预期 + 依据链接
├─ 卡片汇报：实验结论、方向列表
├─ 创建暂存点：git tag ci-fix-checkpoint-<pr>
│
▼ 方向 D1
├─ ⚠️ 检查卡片位置：不在 {{kanban.action.name}} → 立即停止，卡片汇报当前状态并终止
├─ 尝试 1/3 → 实施修复 → 容器内运行 pre-commit → push → approve deployment（按 2.1）→ prblocker
│   ├─ 退出码 0 → ✅ 删除 tag，移到 {{kanban.wait.action_review.name}}
│   ├─ 退出码 2 → approve 后重跑 prblocker
│   └─ 退出码 1 → 检查卡片位置 → 检查人类新评论 → 仍在 {{kanban.action.name}} 且无人类干预 → 同方向改进，尝试 2/3
├─ 尝试 2/3 → pre-commit → push → approve → prblocker
│   └─ 退出码 1 → 检查卡片 → 检查评论 → 同方向改进，尝试 3/3
├─ 尝试 3/3 → pre-commit → push → approve → prblocker
│   └─ 退出码 1 → ❌ 方向 D1 失败
│
├─ git reset --hard ci-fix-checkpoint-<pr>
├─ git push --force
│
▼ 方向 D2（同 D1 流程）
▼ 方向 D3（同 D1 流程）
│
▼ 所有方向耗尽（最多 3×3 = 9 次 CI 尝试）
├─ git reset --hard ci-fix-checkpoint-<pr>
├─ git push --force  ← 人类看到的是 CI 首次报错时的代码状态
├─ git tag -d ci-fix-checkpoint-<pr>
├─ 卡片写详细总结（每个方向 × 每次尝试的失败原因）
└─ 移到 {{kanban.wait.exception.name}}，人类介入
```

#### 4.3.2 硬性规则

- **暂存点 = CI 首次失败时的代码状态**，不是修复过程中的中间态。
- **每次 push 前必须运行 `./avm pre-commit`**（在容器内执行），确保格式化、文档生成、中央治理同步均已完成。**未运行 pre-commit 直接 push 是被禁止的**。
- **每次 push 后必须在卡片汇报**：当前方向（D?）、尝试次数、修改文件、修改内容、原因。
- **修复方向必须有依据**：每个方向必须附带参考链接（官方文档、Provider 源码、GitHub Issue/PR、Stack Overflow 等）。无依据的方向不得执行。
- **每换方向前**必须 `git reset --hard` + `git push --force` 回到暂存点，确保每个方向从相同起点开始。
- **最多 3 个方向 × 每方向 3 次 = 9 次尝试**。
- 任一尝试 CI 通过（退出码 0）→ 删除 checkpoint tag，移到 `{{kanban.wait.action_review.name}}`，流程结束。
- 全部耗尽 → reset + force push 回暂存点 → 删除 tag → 写总结 → 移到 `{{kanban.wait.exception.name}}`。

#### 4.3.3 卡片位置检查（强制，每次尝试前 + 每次换方向前）

每次尝试修复**前**和每次换方向**前**，必须调用 Trello API 检查卡片是否仍在 `{{kanban.action.name}}` 列。如果卡片已被移出：

- 说明人类希望 Agent 停下来；
- **立即停止所有操作**——不得继续 push、不得 force push、不得移动卡片；
- 在卡片评论中汇报当前进度（正在尝试哪个方向、第几次尝试、当前状态）；
- 然后终止。

#### 4.3.4 人类评论检查（强制，每次失败后）

每次尝试失败后（prblocker 退出码 1），在检查卡片位置的同时，必须读取卡片最新评论，检查是否有不以 `{{kanban.agent_comment_prefix}}` 开头的新评论（即人类评论）。若发现人类新评论：

- **人类给出指令**（如"换个方向"、"停止测试"、"先回滚"等）→ 回复 `{{kanban.agent_comment_prefix}} 了解`，然后按人类指令执行。
- **人类提出问题**（如"当前是什么状态？"、"为什么选这个方向？"等）→ 以 `{{kanban.agent_comment_prefix}}` 开头的评论回答，然后继续当前修复流程。
- **人类命令停止** → 回复 `{{kanban.agent_comment_prefix}} 了解，正在清理测试环境`，执行环境清理：
  1. 优先在测试目录执行 `terraform destroy`；
  2. 若 destroy 失败或超时，使用 `az group delete --name <resource-group-name> --no-wait --yes` 异步删除资源组；
  3. 清理完成后在卡片汇报清理结果和当前进度，然后终止。
  - 不得继续 push、不得 force push、不得移动卡片。

#### 4.3.5 汇报示例

```text
{{kanban.agent_comment_prefix}} 方向 D2 尝试 #2/3 — CI 失败修复

方向 D2：通过 dynamic block 替代 count 来避免 known-after-apply 问题
假设：count 依赖的值在 plan 阶段未知，导致 provider 无法确定资源数量
反面预期：如果假设错误，改为 dynamic block 后同一错误仍会出现
参考依据：https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks
         https://github.com/hashicorp/terraform-provider-azurerm/issues/XXXXX

失败原因：tflint 报错 `terraform_naming_convention` on variable `MyVar` in variables.tf:L42

修复：将变量名 `MyVar` 改为 `my_var`（snake_case），同步更新 main.tf 和 examples/ 中的引用。
参考依据：https://azure.github.io/Azure-Verified-Modules/specs/terraform/#id-tfnfr2---category-code-style---snake_case

已推送 commit: chore: fix variable naming convention (D2 attempt 2)

等待 CI 重跑中。
```

---

## 5. {{kanban.action.name}} 阶段终态

只有以下两种合法终态：

| 终态 | 触发条件 | 卡片去向 |
|---|---|---|
| ✅ 成功 | prblocker 退出码 0（所有检查通过） | `{{kanban.wait.action_review.name}}`（等待人类合并；Agent 不得 `gh pr merge`） |
| ❌ 耗尽 | 9 次方向尝试全部失败，或遇到不可解阻塞 | reset + force push 到暂存点 → 删除 tag → 写总结 → `{{kanban.wait.exception.name}}` |

**不允许的"终态"**（这些都不是终态，看到必须继续工作）：

- "代码已经写完了" → 不是终态。
- "CI 在跑了 / 我没事干了" → 不是终态。
- "几个绿勾出现了" → 不是终态。
- "等维护者 approve deployment" → 错误认知，approve 是 Agent 自己的事。
- "我累了 / context 太长了" → 不是终态。Agent 必须按流程走完。
