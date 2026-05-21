# AzureRM Provider Bug 类 Issue 处理指南 — Action 阶段

> 本文件覆盖 **实施与审查阶段**：在 [azurerm_provider_issue_bug_plan.md]({{azurerm_provider_issue_bug_plan.md}}) 第 11 节"计划审批门槛"通过后，由 Agent 在目标 Provider 仓库内实际写代码、跑测试、做自审查、提交 PR。
> 本文件假定目标仓库已通过 gateway 内置的 `aiassistedrefresh` 包（在 worker session 被 spawn 之前同步执行，Windows 走 `pwsh + install-copilot-setup.ps1`、macOS/Linux 走 `bash + install-copilot-setup.sh`）安装了 [WodansSon/terraform-azurerm-ai-assisted-development](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development) 的指令文件、prompts 和 skills。

## 1. 适用范围与目标

本阶段开始的前提：

1. Plan 阶段已产出唯一默认修复方案。
2. 人类维护者已 **明确批准** 该方案，并确认了批准范围（拟改文件清单、测试清单、是否破坏性、是否需要文档更新）作为 **预期范围**。
3. 目标仓库（`hashicorp/terraform-provider-azurerm` 或其 fork）已克隆到本地，且配置好 `origin` remote。
4. AI 辅助文件已经物理铺设到目标仓库的 `.github/`、`.vscode/`（如未铺设，先执行 §3 的 setup）。

### 1.1 Action 阶段的最高目标

> **本阶段目标 = 让本次 Pull Request 既能修复问题、又能通过所有代码审查反馈与 CI 测试、最终被人类维护者合并。**

所有规则、门禁、范围讨论都服务于这一目标。Plan 批准范围是 **起点和默认期望**，不是不可突破的天花板：当扩大范围是"让 PR 被合并"的必要条件时，agent 应在记录差异并自检后继续推进；当扩大范围会偏离 Issue 主旨或引入争议性变更时，必须回到 Plan 阶段补充审批。

### 1.2 PR 最小化原则（强制）

Agent 的每一次代码改动都必须满足：

1. **最小差异**：只做"为了修复问题 + 通过审查 + 通过测试"所必需的改动，不做与本次 Issue 无关的优化。**是否需要在修复路径上做局部重构、风格调整、命名/注释规范化，由目标仓库 `.github/instructions/` 下的 AI assisted development 指令文件（特别是 `code-clarity-enforcement.instructions.md`、`schema-patterns.instructions.md`、`code-review-compliance-contract.instructions.md` 等）明确要求来决定**：指令要求做的，必须做且视为白名单内；指令未要求的顺手重构与风格清理，一律不做。
2. **聚焦于 Issue**：每一个被修改的文件都必须能回答"它为什么必须改"——理由属于以下白名单之一才允许出现在本次 PR：
   - 直接修复 Issue 报告的 bug（Plan 批准的核心改动）
   - Plan 没批准但属于同一根因链的必要修改（例如修了 Flatten 必须同步调 Expand 才能消除 drift）
   - 自审查或人类审查反馈明确要求的修改（包括 lint、注释、命名等）
   - 为了让 CI 通过而必须修复的"被本次改动牵连失败的现有测试"（必要时新增/调整测试夹具）
   - 让验收测试覆盖到本次 bug 触发条件的新增/修改测试用例
   - 文档同步（仅当 Plan 已批或审查反馈要求）
3. **可解释性**：PR 描述里能为每一类扩展改动给出清晰理由，并在自审查中主动说明"为何这一改动属于上述白名单"。

### 1.3 硬边界（不可越过）

以下规则任何情况下都不得突破，必要时必须中止 Action 阶段并回到 Plan：

- 不得修改 AI 辅助工具的指令、prompt、skill 文件，或目标仓库下的 `.github/instructions/`、`.github/prompts/`、`.github/skills/`、`.vscode/` 等基础设施文件。
- 提交前必须通过 §7 的清理与门禁检查，确认 AI 基础设施文件没有被一起提交。
- 任何路径上的失败均不得通过 `--no-verify`、跳过测试、删除 lint 规则、`-skip` 真实测试用例等方式绕过。
- 不得擅自引入 **破坏性变更**（schema 改动触发 ForceNew、状态迁移、删除/重命名公开属性、改变默认值语义等）。Plan 没批准的破坏性变更必须回 Plan 阶段。
- 不得擅自把 Issue 的根因结论翻转（如发现实际属于 Azure API 层），必须回 Plan 阶段重新分类。
- 不得擅自做 **AI assisted development 指令文件未要求** 的机会性重构、文件搬移、大规模格式化、跨 service 的清理；是否需要重构与风格调整一律以加载的 AI assisted development 指令文件为准。
- 未获人类授权前，agent 不得用人类账号在 PR/Issue 中代发 review 回复（与 Plan §1 的协同模型一致）。

### 1.4 范围扩展的判定（软边界，agent 可自决）

当出现 Plan 没明确列出的改动需求时，agent 按下列流程自决：

1. **是否落在 §1.2 白名单内？**
   - 否 → 不做。
   - 是 → 进入第 2 步。
2. **是否触碰 §1.3 任意一条硬边界？**
   - 是 → 中止并回 Plan（参见 §6）。
   - 否 → 进入第 3 步。
3. **改动是否仍保持 PR 最小化？** 用"如果不做这项改动，PR 是否还能被合并/通过 CI/通过 review"自检。
   - 不能不做 → 允许做，并在 §5.6 的 PR 描述中显式列出"超出 Plan 原始范围的扩展改动 + 理由"。
   - 可以不做 → 不做。

#### 关于"被本次改动牵连失败的测试"和"修复前就失败的测试"

- **被本次改动牵连失败的测试**：必须修复，属于 §1.2 白名单。修复手段优先级：调整测试夹具/期望值 > 调整本次 fix 实现 > 新增小范围 helper。禁止 `t.Skip` 或删除断言绕过。
- **本次改动之前就已失败的测试**：默认 **不修**，在 PR 描述中标注"pre-existing failure, out of scope"。例外情况：审查反馈明确要求修复，或该失败与本次 bug 同源（修了本次 bug 顺带修复）——此时按 §1.4 第 3 步评估后纳入。
- **缺失的覆盖度测试**：若 Plan §7.2 的覆盖度分析指出存在缺口，必须新增测试覆盖本次 bug 的触发条件。

## 3. 环境准备（首次或刷新时执行）

### 3.1 安装/刷新 AI 辅助文件到目标仓库

Gateway 在你被 spawn 之前已经通过内置的 `aiassistedrefresh` Go 包同步完成了安装/刷新（会根据当前 OS 自动选 ps1 或 sh 版本的上游安装器）。**你不要再手动调任何 `refresh-copilot-setup.ps1` 或 `install-copilot-setup.{ps1,sh}`** ——重跑只会费资源。

如果出现指令文件看起来过期，联系运维者让 gateway 重启一下（hook 会在下一张发入同仓库的卡片重新拉取上游最新内容），不要自己在 worker 里重跑脚本。

过去的效果（仅供参考）：

1. 在目标仓库铺设 `.github/instructions/`、`.github/prompts/`、`.github/skills/` 等。
2. 若当前还在 `main` 分支且 gateway 获取到 issue 号，会自动创建 `issue-<id>` 工作分支。

### 3.2 Acceptance 测试凭据

跑 `TestAcc*` 需要以下环境变量（缺失时只能跑 unit 测试，acceptance 必须明确跳过并在 PR 描述中说明原因）：

`ARM_SUBSCRIPTION_ID`、`ARM_TENANT_ID`、`ARM_CLIENT_ID`、`ARM_CLIENT_SECRET`、`ARM_TEST_LOCATION`、`ARM_TEST_LOCATION_ALT`

## 5. 实施流程（核心循环）

> **⚠️ 子 agent 委派总览（强制）**
>
> §5 的工作必须按下列分组，**每一组由一个全新启动的独立子 agent 完整执行后退出**，主 agent 不得自己执行组内步骤。每组结束后由主 agent 接收子 agent 的结构化交付物，验证后再启动下一组。
>
> | 组别 | 覆盖章节 | 职责简述 |
> |---|---|---|
> | **G1** | §5.1 | 工作目录与分支 setup、复述批准范围 |
> | **G2** | §5.2 + §5.3 | 写代码 + 修复验证（含 §5.3.1 失败重试循环） |
> | **G3** | §5.4 | 独立审查（本身已要求子 agent，零问题循环） |
> | **G4** | §5.5 | 清理 AI 基础设施文件 + 提交 + 推送 |
> | **G5** | §5.6 | 创建 PR + PR 范围审查 |
>
> §5.7（人类审查反馈处理）不在 G1–G5 内，由主 agent 视反馈类型按需新启子 agent。
>
> **启动子 agent 时主 agent 仅需在 prompt 中传入：本组覆盖的章节链接（让子 agent 自行加载完整规则）+ 必要的运行时上下文（issue id、目标仓库绝对路径、批准的工作分支、Plan 批准范围摘要、上一组交付物）+ 本组"必要命令包"。** 不需要把本文件全部规则塞进 prompt。

> **⚠️ 进度汇报（看板评论，强制）**
>
> 主 agent **每次启动 G1–G5 任一子 agent 时，都必须在 Trello 卡片上发表两条评论**：
>
> 1. **启动评论（子 agent 启动前）**：说明本次启动的组别、覆盖章节、传给子 agent 的关键上下文摘要（issue id、分支、上一组交付物要点）。
> 2. **结果评论（子 agent 退出后）**：贴出该子 agent 的结构化交付物 / 结论 / 失败原因，必要时附关键命令输出片段。G3 多轮审查时，**每一轮**都要发一对启动+结果评论。
>
> **调用方式**（与本工作流既有约定一致，评论必须以 `{{kanban.agent_comment_prefix}} ` 开头）：
>
> 沉默推进 = 让人类失明，等同于违反协同模型。任何一组只发了启动评论但没补结果评论 → 视为该组未完成，下一组禁止启动。

### 5.1 Phase 0 — Setup（G1：独立子 agent 执行）

**主 agent 启动 G1 时传入的必要命令包：**

```bash
cd <目标 provider 克隆绝对路径>
git status
git rev-parse --abbrev-ref HEAD       # 必须等于批准的工作分支名
cat .github/copilot-instructions.md   # 作为会话总入口阅读
```

**G1 子 agent 步骤：**

1. 执行上述命令，确认工作目录与分支。
2. 若分支不是批准的工作分支（如 `issue-<id>` 或 `fix/<issue-id>`），中止并把当前分支报回主 agent。
3. 读取 `.github/copilot-instructions.md`。
4. 对照 Plan 批准范围，输出"本次将要改动的文件 / 函数 / 测试 / 文档清单"作为交付物返回主 agent。

**G1 交付物**：当前分支名、工作目录、Plan 批准范围复述、拟改动清单。

**G1 看板评论**：按总览"进度汇报"硬要求，启动 G1 前发 `{{kanban.agent_comment_prefix}} G1 启动...` 评论；G1 子 agent 退出后发 `{{kanban.agent_comment_prefix}} G1 结果...` 评论，附交付物全文。

### 5.2 Phase 1 — 写代码（G2 第一阶段，与 §5.3 合并为同一个子 agent）

> **G2 = §5.2 + §5.3**：写代码与修复验证由 **同一个独立子 agent** 完成（这样它在写完代码后立即接着跑 §5.3 验证与失败重试循环，不丢上下文）。

**主 agent 启动 G2 时传入的必要命令包：**

```bash
# 写代码阶段不需要预先指定命令，按下方步骤执行；以下为本组允许使用的"参考读取"清单
ls .github/instructions/
ls .github/skills/
```

**G2 写代码步骤：**

1. 按需读取目标仓库 `.github/instructions/` 下与本次改动类型相关的 instructions（schema / error / azure / clarity 等）。
2. 如果是新增资源/重写资源，读取 `.github/skills/resource-implementation/SKILL.md` 作为工作流模板。
3. **默认在 Plan 批准范围内**修改 Go 文件与函数；遇到必须扩展的情形（同根因连带、CI 修复、审查反馈），按 §1.4 自决流程评估后再动手，且全程对照 §1.2 的白名单与 §1.3 的硬边界。
4. 满足非破坏性约束（见 [Plan §8]({{azurerm_provider_issue_bug_plan.md}})）：不引入无关 drift、不改变既有语义、不要求用户手改 state。
5. 重构与风格清理以 AI 辅助开发指令文件为准：指令要求的（如 `code-clarity-enforcement` 对注释/命名的硬要求、`schema-patterns` 对 schema 定义顺序的要求等）必须达成；指令未要求的顺手重构一律不做。每提交一行新代码前自问："这一行是否服务于让本 PR 被合并？是否是加载的指令文件明确要求的？"——两者都不是则删掉。

### 5.3 Phase 2 — 修复验证（G2 第二阶段，含失败重试循环）

> 仍由 §5.2 同一个 G2 子 agent 继续执行；不要换 agent。

**主 agent 启动 G2 时传入的"验证阶段必要命令包"：**

```bash
# 最小验证（每次代码改动后必跑）
go build ./...
go vet ./...
go test ./internal/services/<service>/...

# Acceptance（凭据可用时；变量见 §3.2）
TF_ACC=1 go test ./internal/services/<service>/ \
  -run TestAccXxx_yyy -v -timeout 180m

# 创建/恢复首次失败暂存点（按需选其一）
git stash push -u -m "bug-fix-checkpoint-<issue>"
git stash list
git stash apply <stash-ref>
# 或使用 tag
git tag bug-fix-checkpoint-<issue>
git reset --hard bug-fix-checkpoint-<issue>
```

**G2 验证步骤：**

1. 读取 `.github/skills/acceptance-testing/SKILL.md` 与 `testing-guidelines.instructions.md`。
2. 跑最小验证。
3. 凭据可用时跑 acceptance（命令见上方包）。
4. 同时跑 `_basic` 与 `_complete` 测试，验证未引入回归。
5. 如适用，使用报告者的 HCL 配置做端到端验证。
6. 失败 → 进入 §5.3.1 失败重试循环；成功 → 输出 §5.3.3 交付物给主 agent，结束 G2。

**G2 交付物**：通过的测试用例列表、关键测试输出片段、最终代码 diff、超出 Plan 原始范围的扩展改动清单（含归属类别）。

**G2 看板评论**：启动 G2 前发 `{{kanban.agent_comment_prefix}} G2 启动...` 评论；G2 子 agent 退出后发 `{{kanban.agent_comment_prefix}} G2 结果...` 评论，附交付物。若 G2 在 §5.3.1 失败重试循环中走完了多轮但最终成功，**结果评论中必须列出每个方向 × 每次尝试的摘要**（参见 §5.3.2）。若进入 §5.3.4 修复方案未知流程，结果评论中必须明确写"修复方案未知，回 Plan 阶段"。

#### 5.3.1 失败重试机制（方向探索式修复循环）

> **核心思路**：首次验证失败时创建暂存点，制定至多 3 种修复方向，每个方向允许 3 次尝试。方向失败后回退到暂存点再试下一方向。所有方向耗尽则回退到暂存点进入"修复方案未知"流程。
>
> **⚠️ 每个修复方向必须有外部依据支撑**：通过搜索 Azure REST API 规范、Provider 源码惯例、相关 Issue/PR 讨论等找到明确的证据后才可确定该方向。禁止凭猜测制定修复方向。

```
首次验证失败
│
├─ 分析失败原因
├─ 制定至多 3 种修复方向（D1, D2, D3），记录分析结论与方向列表
├─ 创建暂存点：git stash 或 git tag bug-fix-checkpoint-<issue>
│
▼ 方向 D1
├─ 尝试 1/3 → 修改 Go 代码 → 编译 → 运行验收测试
│   ├─ 成功 → ✅ 清理暂存点，记录方案与证据
│   └─ 失败 → 分析原因，同方向改进，尝试 2/3
├─ 尝试 2/3 → 调整修复 → 验证
│   └─ 失败 → 同方向改进，尝试 3/3
├─ 尝试 3/3 → 调整修复 → 验证
│   └─ 失败 → ❌ 方向 D1 失败
│
├─ 回退到暂存点
│
▼ 方向 D2（流程同上）
▼ 方向 D3（流程同上）
│
▼ 所有方向耗尽（最多 3×3 = 9 次尝试）
├─ 回退到暂存点
├─ 汇总每个方向 × 每次尝试的失败原因
└─ 进入"修复方案未知"流程（回到 Plan 阶段，提交补充简报供维护者重新决策）
```

硬性规则：

- **暂存点 = 首次验证失败时的代码状态**，不是修复过程中的中间态。
- **每换方向前**，必须回退到暂存点，确保每个方向从相同起点开始。
- **每次重试必须基于上次失败原因做出有意义的方案调整**，禁止无变化重复尝试。
- **修复方向必须有依据**：每个方向必须附带参考链接（Azure REST API 规范、Provider 源码惯例、GitHub Issue/PR 等）。
- **方向触及 §1.3 硬边界**（如必须引入破坏性变更、必须翻转根因结论、必须改动 AI 基础设施文件等）：立即中止 Action 阶段，回到 Plan 阶段补充审批。
- **方向仅仅是"超出 Plan 原始批准的文件清单" 但仍落在 §1.2 白名单内**（例如同根因牵连的相邻函数、被本次改动牵连失败的测试夹具）：按 §1.4 自决流程评估后可以继续，无需回 Plan，只需在 §5.6 PR 描述中如实记录。
- **最多 3 个方向 × 每方向 3 次 = 9 次尝试**。
- 任一尝试验证成功 → 清理暂存点，进入 Phase 3。
- 全部耗尽 → 回退到暂存点 → 进入"修复方案未知"流程（标记 `bug` + `needs-investigation`，回 Plan 阶段）。

#### 5.3.2 验证记录（每次尝试必须记录）

| 字段 | 说明 |
|---|---|
| 方向编号 | D1 / D2 / D3 |
| 方向描述 | 该方向的修复思路 |
| 参考依据 | 该方向的外部依据链接（Azure REST API 规范、Provider 源码模式、GitHub Issue/PR 等） |
| 尝试编号 | 该方向内从 1 开始递增（1/3、2/3、3/3） |
| 修改的 Go 文件与函数 | 本次尝试具体改动了哪些文件和函数 |
| 方案摘要 | 本次尝试的具体改动（与上次的差异） |
| 验证结果 | 成功/失败 |
| 运行的测试用例 | 执行了哪些验收测试 |
| 失败原因 | 失败时填写：现象、报错、与预期的差距 |
| 下一步调整方向 | 失败时填写：同方向改进思路，或标记方向失败准备切换 |

汇报示例：

```text
方向 D1 尝试 #2/3 — Bug 修复验证

方向 D1：修改 Flatten 函数，在 API 返回 nil 时设置空列表而非跳过，消除 drift
参考依据：Provider 源码中 `flattenNetworkSecurityGroupRules()` 的处理模式
         Azure REST API 规范 `Microsoft.Network/networkSecurityGroups 2023-09-01`

修改文件：internal/services/network/network_security_rule_resource.go
  - flattenNetworkSecurityRuleConfig(): 添加 nil check，当 API 返回 nil 时写入空列表

运行测试：TestAccNetworkSecurityRule_basic, TestAccNetworkSecurityRule_update

失败原因：Flatten 修复了 drift，但 Expand 函数在空列表时发送了空数组给 API，
         API 返回 400 错误"至少需要一个规则"

下一步：同时修改 Expand 函数，空列表时不发送该字段（omitempty）
```

#### 5.3.3 验证成功时的处理

1. 记录最终成功方案与验证证据（通过的测试用例列表）。
2. 收集 PR 描述需要的证据片段（关键测试输出、Plan diff 等）。
3. 进入 Phase 3 自审查。

#### 5.3.4 修复方案未知时的处理

1. 回退到暂存点。
2. 汇总每个方向 × 每次尝试的方案摘要与失败原因。
3. **退出 Action 阶段**，回到 Plan 阶段，将"修复方案未知"作为补充简报提交，标记 `bug` + `needs-investigation`。
4. 执行环境清理（见 [Plan §7.5]({{azurerm_provider_issue_bug_plan.md}})）。

### 5.4 Phase 3 — 自审查（G3：硬门禁，独立子 agent + 零问题循环）

> **G3 = 本节**。本组本身就是子 agent 流程，但每一"轮"都必须 **重新启动新的独立子 agent**，不得复用前一轮上下文。

**主 agent 启动 G3 每一轮时传入的必要命令包：**

```bash
# 仅供子 agent 使用的本地审查工具
azurerm-linter --mode local-diff --format json | jq .
git diff origin/main...HEAD
```

不通过本阶段，禁止进入 Phase 4 提交。

> **⚠️ 强制要求**：本阶段的代码审查 **必须** 由一个 **全新启动的独立子 agent** 执行（无主 agent 写代码时积累的上下文/动机偏差）。主 agent 不得自审、不得"我自己再 diff 一遍就当审过"。
>
> **⚠️ 零问题循环**：本阶段是一个循环——必须存在 **某一轮子 agent 审查返回 0 条审查意见**（无 error/blocker、无 warning 待处理、无 nit 待处理），才允许推进到 Phase 4。任何一轮存在哪怕 1 条未解决的审查意见，都必须回到 Phase 2 修复后重启新一轮独立子 agent 审查。

#### 5.4.1 启动独立子 agent 审查

每一轮按下列步骤执行：

1. **启动一个全新子 agent**，在其 prompt 中显式要求它依次加载并完整读取以下文件后再开始审查（这些文件路径相对于目标 Provider 仓库 `<work_dir>`）：
   - `.github/prompts/code-review-local-changes.prompt.md`
   - `.github/prompts/code-review-docs.prompt.md`
   - `.github/instructions/code-review-compliance-contract.instructions.md`
   - 本文件 §1（适用范围与目标，含 §1.2 PR 最小化白名单与 §1.3 硬边界）作为审查时的项目特定约束
2. 子 agent 输入还必须包含：
   - 当前 `git diff`（相对 `origin/main` 的全量差异）
   - Plan 阶段批准范围摘要（拟改文件清单、是否破坏性、文档要求）
   - 主 agent 已知的"超出 Plan 原始范围的扩展改动清单 + 归属类别"
3. 子 agent 必须按其加载的 prompt 步骤逐步执行，并 **额外** 执行 azurerm-linter 的 local-diff 模式：
   ```bash
   azurerm-linter --mode local-diff --format json | jq .
   ```
4. 子 agent 必须输出结构化审查报告，至少包含：
   - **阻断级问题（error/blocker）** 列表（来自 prompt + linter）
   - **警告级问题（warning）** 列表
   - **nit / suggestion** 列表
   - **PR 最小化自检结果（§1.2）**：逐文件、逐 hunk 给出"为什么必须改这一处？理由是否在白名单内？"——任何找不到清晰白名单理由的改动列为审查意见
   - **范围漂移自检结果（§1.4）**：列出本轮实际改动相对 Plan 批准范围的差异，对每一项扩展改动给出归属（同根因 / CI 修复 / 审查反馈 / 测试覆盖 / 文档同步），并确认任何一项都没踩 §1.3 硬边界
   - **本轮总审查意见数**（一个整数，等于上述各类问题条数之和，nit 也算）

#### 5.4.2 处理审查结果

按子 agent 报告的"本轮总审查意见数"分流：

- **= 0**：本轮通过，**且本轮通过的代码状态必须是即将进入 Phase 4 的状态**（不得再做任何修改，否则视为新一轮，需重新启动子 agent 审查）。允许推进到 Phase 4。
- **> 0**：
  1. 阻断级与 PR 最小化 / 范围漂移问题：必须修复或回退相应改动。
  2. 警告级：默认必须修；若有正当不修理由，必须能在 PR 描述中以书面理由列出（理由本身需在下一轮子 agent 审查中再次接受检验，仍被标为 warning 则视为未解决）。
  3. nit / suggestion：默认采纳；若选择不采纳，必须在 PR 描述中给出理由，规则同上。
  4. 修复完成后回到 Phase 2 重新跑最小验证 + 相关 acceptance 测试，再回到本节启动 **新一轮全新子 agent** 审查。**禁止复用前一轮子 agent 的审查上下文。**

#### 5.4.3 循环上限与异常处理

- **同一轮审查结果中，相同问题重复出现 ≥ 3 轮仍未消除**：视为修复方向陷入震荡，按 §5.3.1 失败重试机制升级处理（必要时回退到首次失败暂存点 + 切换修复方向，或回 Plan 阶段补充审批）。
- **子 agent 报告的问题与 §1.3 硬边界冲突**（例如要求修改 AI 基础设施文件、要求引入破坏性变更等）：立即按 §6 中止 Action 阶段、回 Plan 阶段。
- **每一轮审查的子 agent prompt、加载的文件清单、原始报告、本轮总审查意见数** 都必须留档，PR 描述需说明"本 PR 经过 N 轮独立子 agent 审查，最后一轮 0 问题"。

**G3 看板评论**：G3 是多轮循环——**每一轮**都要发一对评论：启动评论写明"第 N 轮 G3 启动"；结果评论贴出本轮总审查意见数与各类问题概要，最终通过的那一轮必须明确写"第 N 轮 G3 通过，0 问题，准备进入 G4"。

### 5.5 Phase 4 — 清理与提交（G4：独立子 agent 执行，硬门禁）

**主 agent 启动 G4 时传入的必要命令包**（仅作为 G4 子 agent 的"参考命令清单"——**所有命令一律由 G4 子 agent 自己执行**，主 agent 不得代跑；占位符 `<...>` 也必须由 G4 子 agent 在执行时自行决定真实值，主 agent 不得预先填好）：

> **⚠️ commit message 由 G4 子 agent 自主决定**
>
> 第 5 步的 `git commit -m "..."` 内的 commit message 文本 **必须由 G4 子 agent 自己根据实际 diff、Plan 批准范围和目标仓库 commit message 规范现场撰写**。主 agent **禁止**在 G4 prompt 里直接给出成型的 commit message 文本，**禁止**让子 agent"照抄"任何预设字符串；主 agent 只能传入"目标仓库 commit message 规范在哪、Plan 批准范围摘要、issue id"等上下文，由 G4 子 agent 自决。同理，第 3 步 `git add` 的具体文件清单、第 6 步推送的 `<分支名>`，也都由 G4 子 agent 在执行时按实际情况确定。

```bash
# 1. 清理 AI 基础设施文件（按平台二选一；Gateway 已经会在下一轮 hook 中同步跑 install-copilot-setup.{ps1,sh} -Clean）
~/.terraform-azurerm-ai-installer/install-copilot-setup.sh -clean -repo-directory $(pwd)
# 或 PowerShell：
# pwsh -NoProfile -File "$env:USERPROFILE\.terraform-azurerm-ai-installer\install-copilot-setup.ps1" -RepoDirectory <目标仓库绝对路径> -Clean

# 2. 确认无 AI 残留
git status
git status --porcelain | grep -E '^(\s*[AM?]+\s+)\.github/(instructions|prompts|skills)/' && echo "BLOCK: AI files leaked" && exit 1

# 3. 逐文件 add（禁止 git add -A / git add .）；<file1> <file2> 由 G4 子 agent 依据实际 diff 决定
git add <file1> <file2> ...

# 4. 复核暂存内容
git diff --cached

# 5. 提交（commit message 必须由 G4 子 agent 自己撰写，遵循目标仓库 commit message 规范；不得使用主 agent 给定的成型文本）
git commit -m "<G4 子 agent 自撰的规范化 commit message>"

# 6. 远端配置 + 推送（必须推到 <fork_owner> fork；<分支名> 由 G4 子 agent 用 git rev-parse --abbrev-ref HEAD 取当前分支）
git remote -v
# 缺失时：git remote add <fork_owner> https://github.com/<fork_owner>/terraform-provider-azurerm.git
git push -u <fork_owner> <分支名>
```

**G4 子 agent 步骤：**

1. 如果之前在目标仓库铺设过 AI 文件，必须先用上方"步骤 1"命令清理。
2. 运行 `git status`，**必须确认** `.github/instructions/`、`.github/prompts/`、`.github/skills/` 下没有 AI 文件残留进入暂存区或工作区。
3. 仅 `git add` 业务代码、测试、文档；逐文件 add，**禁止 `git add -A` 或 `git add .`**；要 add 的文件清单由 G4 子 agent 自己根据 `git status` / `git diff` 现场决定。
4. `git diff --cached` 复核一遍待提交内容，与批准范围对照。
5. 提交：**G4 子 agent 自己撰写 commit message**，遵循目标仓库 commit message 规范；禁止照抄主 agent prompt 里的任何预设文本；commit message 必须能客观反映本次实际暂存内容（不是 Plan 原始批准的范围拷贝）。
6. 推送到远端（见下方临时限制），`<分支名>` 由 G4 子 agent 自取当前分支名。

**G4 交付物**：commit SHA、G4 子 agent 自撰的 commit message 全文、推送目标 remote 与分支、`git diff --cached` 概要。

**G4 看板评论**：启动 G4 前发 `{{kanban.agent_comment_prefix}} G4 启动...` 评论；G4 子 agent 退出后发 `{{kanban.agent_comment_prefix}} G4 结果...` 评论，附 commit SHA、**G4 子 agent 自撰的 commit message 全文**、推送的 remote/分支、`git diff --cached` 概要。

> **⚠️ 临时限制（推送与 PR 目标仓库）**
>
> 占位符 `<fork_owner>` 指操作者个人 fork 的 GitHub 登录名（例如仓库 URL `https://github.com/<your-handle>/terraform-provider-azurerm` 中的 `<your-handle>`）。执行前必须按宿主机实际使用的 fork 账号替换所有 `<fork_owner>` 出现。
>
> 在本限制解除之前，**所有分支一律推送到 `<fork_owner>/terraform-provider-azurerm`**，不得直接推到 `hashicorp/terraform-provider-azurerm` 或其它 fork。
>
> 1. 推送前确认 remote 配置：本地必须存在指向 `https://github.com/<fork_owner>/terraform-provider-azurerm.git`（或其 SSH 等价 URL）的 remote（建议命名为 `<fork_owner>`，若仅有一个 remote 也可使用 `origin`，前提是其 URL 指向该仓库）。
> 2. 推送命令固定为 `git push -u <fork_owner> <分支名>`（或对应的 remote 名）。禁止使用 `--force` 推送已发布分支。
> 3. 若本地 remote 缺失或 URL 不符，先 `git remote add <fork_owner> https://github.com/<fork_owner>/terraform-provider-azurerm.git`（或 `git remote set-url` 修正），再推送。

### 5.6 Phase 5 — 提交 PR 与 PR 范围审查（G5：独立子 agent 执行）

> **⚠️ 临时限制（PR 目标仓库与 base 分支）**
>
> 在本限制解除之前，**PR 必须开在 `<fork_owner>/terraform-provider-azurerm` 仓库内**，base 分支为 **该仓库的 `main`**（即 `<fork_owner>:main`），head 是上一步推送到 `<fork_owner>` 的分支。**禁止**把 PR 开向 `hashicorp/terraform-provider-azurerm` 或任何其它上游仓库。`<fork_owner>` 定义同上节。

> **⚠️ PR 模板硬约束（强制，不可改写）**
>
> G5 子 agent **必须严格按照** 以下 URL 提供的 hashicorp 上游官方 PR 模板创建 PR，**严禁自己发挥**——不得新增、删除、合并、改写模板内的章节标题、复选框、HTML 注释；只能在模板已有的章节内填入内容。
>
> 模板源：`https://raw.githubusercontent.com/hashicorp/terraform-provider-azurerm/refs/heads/main/.github/pull_request_template.md`
>
> 子 agent 第一件事必须是把该 URL 的原文 fetch 下来落到本地临时文件 `pr-body.md`，然后 **仅在该文件内** 填写内容；落地的最终 body 必须保留模板的全部原始结构。

> **⚠️ PR Title 规范（强制）**
>
> G5 子 agent 在执行 `gh pr create --title "..."` 之前，**务必完整阅读** 以下绝对路径的本指南文件，并 **严格按照其中关于 PR title 命名的规定** 来构造标题，**禁止自由发挥**：
>
> ```
> {{azurerm_provider_issue_bug_action.md}}
> ```
>
> 主 agent 启动 G5 时必须把上述路径作为强制阅读项写进子 agent 的 prompt；子 agent 在没读完该文件之前不得调用 `gh pr create`。

**主 agent 启动 G5 时传入的必要命令包：**

```bash
# 1. 拉取上游官方 PR 模板（严禁自己拼写或本地拼凑）
curl -fsSL \
  https://raw.githubusercontent.com/hashicorp/terraform-provider-azurerm/refs/heads/main/.github/pull_request_template.md \
  -o pr-body.md

# 2. 在 pr-body.md 内"原位"填写各章节，禁止删改章节标题与 HTML 注释；
#    所有事实性补充（Issue 编号、验证证据、扩展改动清单、测试状态等）
#    必须放进模板已有的章节，找不到对应章节则放进模板预留的 "Description" 段。

# 3. 用填好的模板创建 PR（仓库与 base 锁定到 <fork_owner> fork）
gh pr create \
  --repo <fork_owner>/terraform-provider-azurerm \
  --base main \
  --head <fork_owner>:<分支名> \
  --title "<符合上游 commit message 规范的标题>" \
  --body-file pr-body.md

# 4. PR 范围审查
cat .github/prompts/code-review-committed-changes.prompt.md
git fetch origin main
git diff origin/main..HEAD
```

**G5 子 agent 步骤：**

1. 用上方命令包的"步骤 1" curl 官方 PR 模板到 `pr-body.md`，**禁止**本地编造或从其它分支/仓库复制模板。
2. 在 `pr-body.md` 内填写内容，遵守"PR 模板硬约束"——保留模板原结构，只填空。
3. 用"步骤 3"的 `gh pr create` 提交 PR，仓库锁 `<fork_owner>/terraform-provider-azurerm`，base 锁 `main`。
4. 读取 `.github/prompts/code-review-committed-changes.prompt.md`，对 `origin/main..HEAD` 或 PR diff 再做一次 PR 范围审查。
5. 修复审查发现的问题；任何代码修改都必须回到 G2 → G3 → G4 重新走流程，再回到 G5 更新 PR。

**G5 交付物**：PR URL、PR number、最终 `pr-body.md` 内容、PR 范围审查报告。

**G5 看板评论**：启动 G5 前发 `{{kanban.agent_comment_prefix}} G5 启动...` 评论；G5 子 agent 退出后发 `{{kanban.agent_comment_prefix}} G5 结果...` 评论，必须附 PR URL、PR number、PR 范围审查结论（无阻断 / 有阻断回 G2）。

### 5.7 Phase 6 — 处理人类维护者的审查反馈

PR 提交后进入人类审查阶段。审查反馈是 Action 阶段的延续，不是新一轮 Plan：agent 负责把反馈转化为最小、可解释的代码变更，并代发回复（须人类授权后用人类账号），目标始终是把 PR 推到可合并状态。

1. **拉取并清单化反馈**：用 `gh pr view <PR-id> --comments` 获取 review comments 与 line comments，逐条建立 todo（编号、引用位置、要求、归类）。
2. **逐条分类**：
   - **must-fix**（请求改动 / 阻断 approve / CI 失败相关）：必须解决。
   - **nit / style / suggestion**：默认采纳；若有正当理由不改，须在回复中给出理由。
   - **scope-change request**（要求加新 feature、改动其他 service、引入破坏性变更）：按 §6 评估，必要时回 Plan。
3. **must-fix 默认允许扩展范围**：审查反馈引发的代码改动天然属于 §1.2 白名单第 3 项，agent 可直接动手，无需回 Plan，但仍须遵守 §1.3 硬边界与 §1.4 第 2 步硬边界检查。
4. **修复后的回归**：每轮反馈处理完成都要重新跑 §5.3 的最小验证 + 相关 acceptance 用例 + §5.4 的自审查（azurerm-linter local-diff），确保修反馈没引入新问题。
5. **GitHub 回复礼仪**：每条 must-fix 修完后用 `gh pr review --comment` 或在对应 line comment 下回复，引用 commit SHA 说明改在哪里；nit 未采纳时给出简短理由；不要在未修复时口头承诺。
6. **代发授权**：与 Plan §1 的协同模型一致——所有以人类账号发出的 review 回复，必须先把回复正文挂到看板待审，得到人类批准后再用人类账号发出。

## 6. 范围漂移与边界破坏的处理

本节只列出**必须中止 Action 阶段、回到 Plan 阶段重新审批**的情况——它们都是 §1.3 硬边界被触发的体现。**仅仅是"超出 Plan 原始批准的文件清单"** 不构成中止理由，按 §1.4 自决流程处理即可。

触发回 Plan 的硬边界事件：

1. 实测发现修复必须引入 **破坏性变更**（schema 改动触发 ForceNew、状态迁移、删除/重命名公开属性、改默认值语义等），而 Plan 未批准。
2. 根因实际不在 Provider 层（例如确认是 Azure API 层、上游 SDK 层），需走 [Plan §9]({{azurerm_provider_issue_bug_plan.md}}) 的 B6/B7 路由重新分类。
3. 失败重试机制 9 次尝试全部耗尽，仍未找到可行修复方向。
4. 审查反馈要求扩展到 **跨 Issue / 跨 service 的重构、新功能、文档大改** 等明显超出本 Issue 主旨的工作。
5. 审查反馈与 Plan 已批准的兼容性结论冲突（例如要求改成破坏性方案）。
6. "修复前就失败的测试" 经评估后必须修复，但修复手段会触碰 §1.3 硬边界（如要求大规模重构或破坏性变更）。

中止时必须：

- 回退到能让 PR 处于干净状态的暂存点（`git stash` / `git restore .` / `git reset` 视情况而定，禁止 `--force` 推送已发布分支）。
- 在 Plan 阶段提交补充简报，说明 Action 阶段发现的事实、已尝试的方向、建议的下一步决策选项。
- 若 PR 已开启，在 PR 中代发说明（须人类授权后用人类账号），告知维护者本 PR 暂停推进、原因与下一步。

## 7. 完成标准（可对外宣布修复完成）

1. PR 已创建，CI 全部通过。
2. 所有 Plan 测试计划要求的用例已运行并通过，证据已附在 PR 描述中。
3. azurerm-linter local-diff 与 PR 范围审查均无阻断问题。
4. 目标仓库 `git status` 干净，无 AI 基础设施文件残留。
5. 复现环境已清理（`terraform destroy` 或 `az group delete --no-wait`）。
6. PR 进入维护者 review 队列。
