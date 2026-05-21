# AzureRM Provider Issue 分类流程编排器（Orchestrator）

> **本文件是分类流程的"指挥棒"——只规定 Step A → E 的编排、subagent 调度、人审门、看板汇报、最终路由。**
>
> **分类的具体规则**（状态机、证据层级、试验设计、同行审查的判断标准等）位于
> `{{azurerm_provider_issue_classification.md}}`
> （以下简称"**分类内核**"），由每个 Step 调度的 subagent 读取并执行。
>
> **主 agent 不需要、也不应该阅读分类内核。** 主 agent 的职责仅限于：
> 1. 按本文件定义的顺序调度 subagent
> 2. 把控人审门（等待人类显式审批）
> 3. 把每个 Step 的产出原样汇报到 Trello 卡片
> 4. 校验 subagent 的输出是否符合本文件定义的输出契约
> 5. 在 Step D 按路由表加载下游专属文件

---

# 前置：从 Trello 卡片获取 Issue 信息（主 agent 自己执行）

你正在处理 **HashiCorp AzureRM Provider** 仓库 (`hashicorp/terraform-provider-azurerm`) 的 GitHub Issue。

### 第一步：读取卡片 description

Gateway 已经在 CARD CONTEXT 中预填了 `card_id`、`work_type`、`github_repo`、`github_number`、`github_url` 等信息——**优先使用 CARD CONTEXT 的值**。

如果你需要原始 `name` / `desc` / `firstLine`（例如扫描描述里的额外链接、截图、上下文），调用 gateway 注册的 `trello_card_get` 工具：

```json
{"tool": "trello_card_get", "args": {"card_id": "<card_id>"}}
```

返回 `{id, name, desc, firstLine, idList, idBoard}`。**禁止**自己拼 `https://api.trello.com/1/...` 的 `Invoke-RestMethod` 调用——Trello 凭据由 Go 端持有，工具调用是唯一受支持的访问路径。

### 第二步：从第一行 URL 提取仓库名和 Issue 号

第一行是一个 GitHub URL，格式为：`https://github.com/{owner}/{repo}/issues/{number}`

例如：`https://github.com/hashicorp/terraform-provider-azurerm/issues/28601`
- **仓库名**：`hashicorp/terraform-provider-azurerm`
- **Issue 号**：`28601`

### 第三步：读取 GitHub Issue 详情

```powershell
$issueUrl = "https://api.github.com/repos/{owner}/{repo}/issues/{number}"
$headers = @{ "Accept" = "application/vnd.github.v3+json"; "User-Agent" = "copilot-agent" }
if ($env:GITHUB_TOKEN) { $headers["Authorization"] = "token $env:GITHUB_TOKEN" }
$issue = Invoke-RestMethod -Uri $issueUrl -Headers $headers
```

### 第四步：(已由 gateway 自动完成,无需主 agent 操作)

Gateway 在创建本 worker session 之前已经做了下面两件事:

1. **`git clone --depth 1`** `<github_url>` 进 `<work_dir>`(由 Go 端 `WorkDirPreparer` 完成)。
2. **检测 `<work_dir>/go.mod`**,若第一行是 `module github.com/hashicorp/terraform-provider-azurerm`,**Go 端 `aiassistedrefresh` 包会同步**克隆 `WodansSon/terraform-azurerm-ai-assisted-development` 到一个临时目录,根据当前 OS 选择对应的安装器（Windows 走 `pwsh + install-copilot-setup.ps1`,macOS/Linux 走 `bash + install-copilot-setup.sh`),依次执行 `-Bootstrap`、`-Clean`、安装,从而把上游最新版的 `.github/instructions/`、`.github/prompts/`、`.github/skills/` 铺到 `<work_dir>`,并在当前还在 `main` 分支时创建 `issue-<n>` 工作分支。整个过程**不再 spawn 任何 Copilot 子 session**,也不再依赖 `refresh-copilot-setup.ps1` 这个本地脚本——它在你被 spawn 之前已经全部完成。

> **⚠️ 主 agent 不要再去手动执行任何 AI 辅助文件刷新脚本,也不要重复 `git clone`。** 这两件事在你被 spawn 时就已经由 gateway 同步完成（成功或失败都已经发生）,再跑一次只会浪费 turn 或制造矛盾。
>
> 你只需要从 CARD CONTEXT 段记住 `<work_dir>` 的绝对路径,后续每次调度 subagent(Step A、B、C)都把它原样作为输入传入——subagent 在全新会话中不知道主 agent 的当前工作目录,也不能自己推断仓库位置。

---

# 全局临时约束（最高优先级，向所有下游 Step / subagent / playbook 透传）

> **⚠️ 临时约束（禁止在 PR body 和 PR 评论中引用上游 Issue）**
>
> 当前流程仍在测试阶段，为避免给 `hashicorp/terraform-provider-azurerm` 项目维护团队造成困扰，**禁止在 PR body 或任何 PR 评论中以任何形式链接、引用或提及上游 Issue 编号**。本约束同时适用于：
>
> 1. **PR body 本身**（首次创建时的 body、后续 `gh pr edit` 修改的 body）。
> 2. **PR 内发表的任何评论**——包括但不限于：
>    - PR 对话评论（`gh pr comment`、网页 "Add a comment"）
>    - 整体 review 回复（`gh pr review --comment` / `--approve` / `--request-changes` 的 body）
>    - 行级 review comment 回复（`gh api .../pulls/<n>/comments` 创建的回复、网页 inline reply）
>    - 修复反馈时引用 commit SHA 的回复正文
>
> 原因：即使只是"在回复里顺手引用一下"上游 Issue 编号，GitHub 的反向链接（mentioned / cross-referenced）机制也会在上游 Issue 时间线里产生一条通知，造成维护团队困扰；引用本身就是噪声源，没有"轻量引用"的安全形式。
>
> 具体禁止形式（PR body 与 PR 评论一视同仁）：
>
> - 禁止 GitHub 自动关联关键字：`Fixes #<n>`、`Closes #<n>`、`Resolves #<n>`、`Fix #<n>`、`Close #<n>`、`Resolve #<n>` 等（含跨仓库形式 `owner/repo#<n>`、完整 URL 形式 `https://github.com/.../issues/<n>`）。
> - 禁止裸 `#<n>` 引用上游 Issue。
> - 禁止在 PR body 模板的 "Description"、"Related Issue"、"Testing"、checklist 注释或任何其它章节粘贴上游 Issue 链接。
> - 禁止在 PR 评论正文里粘贴上游 Issue 链接，或写 "see #<n>"、"per the original issue #<n>"、"as discussed in <upstream-issue-url>" 之类的引用。
> - 模板中若存在引导填写 Issue 链接的章节（如 "Related Issue"），保留章节标题与 HTML 注释原样，章节正文留空或写 "N/A (testing workflow, intentionally unlinked)"。
>
> 如果需要让人类维护者知道本 PR 对应哪个 Issue，**只在 Trello 看板评论里说明**，绝不写进 PR body，也绝不写进 PR 评论。
>
> 本约束解除前，**任何即将发出的 PR body 或 PR 评论文本**（无论是子 agent 写的、主 agent 写的、还是模板拼接出来的），在调用 `gh pr create` / `gh pr edit` / `gh pr comment` / `gh pr review` 等命令之前必须先做一次自检：若命中上述任一形式的引用，必须删除后再发出。**主 agent 调度任何会创建 PR 或在 PR 中发评论的下游 subagent 时，必须把本条约束原样透传到子 agent 的 prompt 中，不得依赖子 agent 自己去重新阅读其它 playbook。**

---

# 1. 总览

## 1.1 文件分工

| 文件 | 角色 | 阅读者 |
|---|---|---|
| 本文件 [azurerm_provider_issue.md]({{azurerm_provider_issue.md}}) | 流程编排器：Step A → E 顺序、subagent 调度、人审门、看板汇报、路由 | **主 agent**（你） |
| 分类内核 [azurerm_provider_issue_classification.md]({{azurerm_provider_issue_classification.md}})（绝对路径：`{{azurerm_provider_issue_classification.md}}`） | 分类规则与方法论：状态机、证据层级、试验框架、输出格式 | 每个 Step 调度的 **subagent** |
| 下游专属文件（`*_bug.md` / `*_feature_request.md` / `*_question.md` / `*_security.md`） | 分类完成后的具体处理流程 | 主 agent 在 Step D 加载 |

## 1.2 五步流程一图概览

```
[前置：1–3 步主 agent 拉取 Issue;第 4 步已由 gateway 自动完成]
        │
        ▼
┌──────────────────────────────────────────────┐
│ Step A — 初次分类 + 试验设计                  │
│   主 agent 调度 Step A subagent              │
│   subagent 读分类内核 → 输出初分类简报        │
│        │                                      │
│        ▼                                      │
│   主 agent 把简报贴到看板                     │
└──────────────────────────────────────────────┘
        │
        ▼  [人审门 A→B（条件触发）：仅当 Step A 提出了需审批试验时才是人审门；
        │   无试验时 → 跳过 Step B，直接进入 Step C]
        │
┌────────────────────────────────────────────┐
│ Step B — 执行试验 + 重新评估（可跳过）         │
│   主 agent 调度 Step B subagent（全新会话）   │
│   subagent 执行试验 → 输出修订简报            │
│        │                                      │
│        ▼                                      │
│   主 agent 把简报贴到看板                     │
└──────────────────────────────────────────────┘
        │
        ▼  [无人审门：直接进入 C]
        │
┌──────────────────────────────────────────────┐
│ Step C — 独立同行审查                         │
│   主 agent 调度 Step C reviewer subagent     │
│       （全新会话，无 A/B 的上下文）            │
│   subagent 输出 PASS / OVERRIDE              │
│        │                                      │
│        ▼                                      │
│   主 agent 把审查记录贴到看板                 │
└──────────────────────────────────────────────┘
        │
        ▼  [人审门 C→D：人类审批最终分类]
        │
┌──────────────────────────────────────────────┐
│ Step D — 加载类型专属指南（主 agent 执行）    │
│   按路由表加载下游文件，从此不再使用本文件    │
│   或分类内核                                   │
└──────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────┐
│ Step E — 简报归档与看板状态更新               │
│   主 agent 把 A→D 全部产出归档到卡片，更新     │
│   卡片状态/标签                               │
└──────────────────────────────────────────────┘
```

---

# 2. 主 agent 的通用职责（贯穿所有 Step）

## 2.1 进度汇报（看板评论，强制）

> **⚠️ 每一个 Step（A、B、C、D、E）的开始时和结束时，主 agent 必须在 Trello 卡片上发表评论。** 这是看板协同模型的硬要求——人类只看卡片来跟踪状态，主 agent 沉默推进等于让人类失明。

**汇报方式**：调用 gateway 注册的 `trello_card_comment` 工具（`{card_id, text}`）。**禁止**手写 `Invoke-RestMethod` 去击 api.trello.com。

**每个 Step 至少两条评论**：

1. **开始时**：声明即将进入的 Step、本步将执行的关键动作（如"Step B 开始：执行已批准的 3 个试验"）、即将调度哪个 subagent。
2. **结束时**：本步产出的关键结论 + 进入下一步所需的人类决策（若有）。

**关键中间事件也要汇报**：如 Step B 中某个试验失败、Step C 中 reviewer 给出 OVERRIDE 等，应即时评论，不要攒到 Step 结束才一并汇报。

**Five Whys 链条必须落卡**：Step A subagent 如果在其简报中返回了 Five Whys 因果链追踪记录，主 agent **必须**把完整的因果链以 `{{kanban.agent_comment_prefix}} ` 开头的评论原样贴到卡片上，不得截断或概括。这条链是后续 reviewer（Step C）和重建 worker（session reap 后恢复上下文）唯一可用的复盘依据。

**等待人类审查时**：Step C 结束评论必须以"⚠️ 需要人类确认 — 流程已暂停"开头，第一段就告诉人类：(a) reviewer 结论与建议分类；(b) 必须做出哪几种显式决策之一；(c) 在收到决策前主 agent 不会推进。完整模板见 §5.1 第 6 步。**目的是让人类一眼就明白这条评论需要他动作，而不是把它当成一条普通的进度汇报忽略掉。**

**加载下一阶段文件时**：Step D 评论必须列出实际加载的文件名（含完整路径），便于事后追溯。

## 2.2 Subagent 调度的通用约束

每次调度 subagent 时，主 agent 必须：

1. **使用全新会话**：每个 Step 都是独立 subagent，**不能复用前序 Step 的会话**。这是为了避免上下文偏见污染审查。
2. **完整传入分类内核文件路径**：subagent 不知道当前工作目录，必须用绝对路径
   `{{azurerm_provider_issue_classification.md}}`
3. **完整传入目标 Provider 仓库的本地工作目录路径**（前置·第四步中的 `<work_dir>`）：Step A/B/C 均需在该目录下阅读源代码、运行测试或 `terraform plan`。subagent 不知道主 agent 当前位置，必须显式传入绝对路径。
4. **完整传入卡片快照与 Issue 内容**：包括 Trello 卡片 ID、当前已知的卡片评论、GitHub Issue 的 URL；要求 subagent 通过 `gh` CLI 重新拉取最新内容（特别是 Step C reviewer，必须重新拉取以避免缓存污染）。
5. **传入前序 Step 的简报**（仅 Step B、C 需要）：把 Step A（和 Step B）subagent 返回的完整简报作为 subagent 的输入。
6. **明确声明 subagent 的角色**：在 prompt 开头写明"你是 Step A subagent" / "你是 Step B subagent" / "你是 Step C reviewer subagent"，让 subagent 找到分类内核中对应的角色导航段。

## 2.3 Subagent 输出的合规性校验

主 agent 收到 subagent 返回的简报后，必须校验是否符合本文件定义的**输出契约**（见各 Step 章节末的"输出契约"小节）。如果 subagent 输出缺字段、自相矛盾或越界（如 Step A subagent 擅自执行了需审批试验），主 agent 应：

1. 在卡片上记录该违规
2. 重新调度该 Step subagent，并在 prompt 中明确指出违规之处要求修正
3. 不得"擅自补全"subagent 的产出——补全等于绕过隔离
### 2.3.1 试验方法对照检查（仅 Step B 适用，强制）

> **⚠️ 主 agent 收到 Step B 简报后，必须逐个对照“人类批准的试验方法” vs “subagent 实际执行的方法”。人类批准的就是要执行的，没有商量。**

**判定不一致的典型例子**（均视为违规“降级”）：

| 人类批准的方法 | 被降级后的方法 | 违规原因 |
|---|---|---|
| `terraform apply` 真实部署 | `terraform plan` 不 apply | 避开了真实 Azure 响应 |
| REST API PUT 写操作 | REST API GET 只读 + 推断 | 没验证写路径 |
| 真实 Azure 资源复现 | Go 单元测试 + mock data | mock 不代表真实系统行为 |
| `TF_ACC=1` 验收测试 | 非 ACC 单元测试 | 验收测试才会连 Azure |
| 任何 Tier 1 试验 | 静态分析 / 源代码阅读 / 文档查阅 | 证据层级被虚假拔高 |

**发现不一致时主 agent 必须**：

1. 即使 Step B subagent 声称试验“成功”，仍不接受该结果
2. 在卡片上明确记录违规：“Step B 将试验从 X 降级为 Y，违反§4.2 的 no-excuse 原则”
3. 重新调度 Step B subagent，prompt 中明确禁止任何替代方案，要求原样执行人类批准的方法
4. 如果 Step B subagent 报告“环境/凭据不足”，主 agent **不得自作主张换方法**，必须中止试验、回到人审门 A→B，请人类决定是：(a) 提供凭据/环境；(b) 修改试验方案；(c) 放弃该试验## 2.4 人审门看板状态管理（强制）

> **⚠️ 凡是遇到需要人类显式决策才能推进的门（当前包括：人审门 A→B «条件触发» 和人审门 C→D «永远触发»），主 agent 在发出“等待人类审查”评论之前，必须检查卡片是否在 `{{kanban.wait.plan_review.name}}` 列。如果不在，必须先将卡片移动过去，再发评论。**
>
> **⚠️ 人审门 A→B 是条件触发的：仅当 Step A subagent 提出了需审批的试验时才生效。如果 Step A 未提出试验，主 agent 不要误以为存在人审门——不移动卡片、不发“等待审查”评论、直接进入 Step C。人审门 C→D 是永远触发的，无论 reviewer 输出 PASS 还是 OVERRIDE 都需人类确认。**

**为什么**：仅靠评论不够——人类默认只看看板视图上的“列”来达需要其决策的卡片。卡片如果不移动，人类不会收到任何视觉信号，agent 会陷入“沉默等待”状态。

**检查与移动（全部走 gateway 工具，不允许 `Invoke-RestMethod`）**：

1. 读卡片当前列名：`{"tool": "trello_card_list", "args": {"card_id": "<card_id>"}}` → `{id, name}`。
2. 如果 `name` 不是 `{{kanban.wait.plan_review.name}}`，调用 `trello_card_move` 带 `target_list_name="{{kanban.wait.plan_review.name}}"`：

```json
{"tool": "trello_card_move", "args": {"card_id": "<card_id>", "target_list_name": "{{kanban.wait.plan_review.name}}"}}
```

工具会自己查看板上同名列 → 赋 `idList` → PUT 卡片，返回 `{from:{id,name}, to:{id,name}}` 作为审计证据。

**顺序**：必须先移动卡片 → 再发“等待人类审查”评论。如果顺序反了，可能出现评论发了但卡片还没动的瞬间窗口，人类看不到。
---

# 3. Step A — 初次分类 + 试验设计

## 3.1 主 agent 动作

1. **看板评论（开始）**：`Step A 开始：调度 subagent 完成阶段 1 信息获取、阶段 2 上下文构建、初次分类与试验设计`
2. **调度 Step A subagent**，输入包括：
   - 角色声明："你是 Step A subagent"
   - 分类内核完整路径：`{{azurerm_provider_issue_classification.md}}`
   - Trello 卡片 ID 和已知的卡片信息
   - GitHub Issue 的 owner / repo / number / URL
   - 目标 Provider 仓库的本地工作目录路径
3. **等待 subagent 返回简报**
4. **校验输出契约**（见 §3.3）
5. **看板评论（结束）**：把简报全文贴上去，包括 Five Whys 链条（如有）。
   - 若简报中包含需审批试验方案 → 依 §3.4 走人审门 A→B，标注“待人类审批试验计划”
   - 若简报中无试验方案（PRIMARY 置信度 ≥ 90 且无歧义）→ **不走人审门**，仅标注“无需试验，自动进入 Step C”，不移动卡片

## 3.2 Step A subagent 应执行的事项（仅供主 agent 参考，由 subagent 在分类内核中查找细节）

- 阶段 1：信息获取（Ingest）— 拉取 Issue、处理截图 OCR
- 阶段 2：上下文构建（Context）— 代码定位、版本检查、API 验证、层级判定
- 走分类内核 §6.1.1 状态机的静态可达路径
- 输出两个候选分类（PRIMARY / SECONDARY）+ 各自置信度
- 若需要，按分类内核 §6.3.2 设计 1–3 个需审批试验
- 若状态机触发 Five Whys（S2_WHYS 节点），完整记录因果链

## 3.3 Step A subagent 的输出契约

主 agent 校验 subagent 返回的简报必须包含以下字段：

| 字段 | 要求 |
|---|---|
| **PRIMARY 候选** | 分类（Bug / FR / Question / Security）+ 0–100 置信度分数 |
| **SECONDARY 候选** | 与 PRIMARY 不同的分类 + 0–100 置信度分数；`PRIMARY_CONFIDENCE >= SECONDARY_CONFIDENCE` |
| **状态机路径** | 走过的所有决策节点 + 每个节点选择该分支的具体证据 |
| **置信度评分依据** | 列出 Tier 1/2/3 证据来源，符合分类内核 §6.2 规则 |
| **试验方案** | 若 PRIMARY 置信度 < 90，必须包含 1–3 个试验方案；每个方案有"预期结果 A/B → 类别映射" |
| **Five Whys 链条**（如有触发） | 每层 Why 的提问、全部假设、验证证据、采纳/否决理由、最终落点 |
| **问题层级判定** | Provider 层 / Azure API 层 / Terraform Core 层 |
| **受影响资源** | `azurerm_xxx` 资源名 |

## 3.4 人审门 A → B（条件触发）

> **⚠️ 人审门 A→B 仅在 Step A subagent 提出了需审批试验时才生效。**
>
> **情况一：Step A 提出了需审批试验**（人审门生效）
>
> - **首先**：依 §2.4 检查卡片是否在 `{{kanban.wait.plan_review.name}}` 列，不在则先移动过去，再发“等待人类审查”评论
> - 批准后，主 agent 进入 Step B，可在真实 Azure 环境上执行已批准的需审批试验
> - 若人类要求修改试验方案，主 agent 重新调度 Step A subagent（带上人类的反馈）
>
> **情况二：Step A 未提出任何需审批试验**（即 PRIMARY 置信度 ≥ 90 且无歧义）
>
> - **不打扰人类**：不移动卡片、不发“等待审查”评论
> - 仅在卡片发一条评论明确记录“Step A 完成，无需试验，自动进入 Step C”（以便人类事后追溯）
> - 主 agent **直接跳过 Step B**，进入 Step C

---

# 4. Step B — 执行试验 + 重新评估

## 4.1 主 agent 动作

1. **看板评论（开始）**：`Step B 开始：调度新的 subagent 执行已批准的 N 个试验`
2. **调度 Step B subagent**（**全新会话**，与 Step A subagent 隔离），输入包括：
   - 角色声明："你是 Step B subagent"
   - 分类内核完整路径：`{{azurerm_provider_issue_classification.md}}`
   - **目标 Provider 仓库的本地工作目录路径**（`<work_dir>`）— 在该目录下运行试验、`go test`、`terraform plan` 等
   - **Step A subagent 返回的完整简报**
   - **人类批准的试验方案清单（原样传入，主 agent 不得以任何形式改写、降级、或附加“如果没有凭据可以用 X 代替”“可以用单元测试代替”等替代提示）**
   - 当前 Trello 卡片的最新状态、最新评论
   - GitHub Issue 的最新内容

   > **⚠️ 调度 Step B subagent 的 prompt 中，严禁出现“你不需要真的跑 terraform apply”、“可以用 Go unit test + mock data 代替”、“如果缺 Azure 凭据可以...”这类语句。人类批准的方案是合同，主 agent 是传话人，不是仲裁人。**
3. **等待 subagent 返回修订简报**
4. **校验输出契约**（见 §4.3）
5. **看板评论（结束）**：把修订简报全文贴上去，明确标注"无需人类审批，自动进入 Step C"

## 4.2 Step B subagent 应执行的事项

> **⚠️ 人类批准的试验方案 = 必须执行的方案。任何“借口”都不能被接受：包括但不限于“没有 Azure 凭据”、“不想产生费用”、“资源创建慢”、“单元测试足够证明”、“Tier 1 证据可以用静态分析代替”等。**

- **严格按已批准的设计执行试验。以下行为均视为非法降级**：
  - 把“`terraform apply` 真实部署”改成“`terraform plan` 不 apply”
  - 把“REST API PUT 写操作”改成“REST API GET 只读 + 推断”
  - 把“真实 Azure 资源复现”改成“Go 单元测试 + mock data”
  - 把“`TF_ACC=1` 验收测试”改成“非 ACC 单元测试”
  - 任何形式的“用静态分析 / 源代码阅读 / 文档查阅替代真实环境试验”
- Tier 1 试验若需要创建/修改 Azure 资源，**必须真做**。如果缺凭据/环境，**必须中止试验、回报主 agent**，由主 agent 与人类协商；**不得自作主张换方法，不得以任何实用主义理由（“这样更快”、“这样也能说明问题”、“不需要凭据也能验证”）跳过或替代原始设计**
- **每个试验执行完毕立即清理资源（强制）**：
  - **试验前推荐**：把该试验创建的所有真实 Azure 资源集中到一个独立 Resource Group（命名约定如 `tf-trial-issue-<N>-<timestamp>`），便于事后一键销毁
  - **试验后必须执行**：`terraform destroy -auto-approve` 或 `az group delete --name <rg> --yes --no-wait`
  - **必须验证清理生效**：等待删除完成后用 `az group exists --name <rg>` 或 `terraform state list`（应为空）验证，把验证输出作为“清理证明”记录到简报
  - **清理失败时**：**不得隐瞒**。必须在简报中明确报告“清理未完成：<具体原因，如资源被锁、依赖关系未解除等>”，并主动提示主 agent 通知人类手工介入。为了交差而伪造清理成功证明是严重违规
  - **禁止跨试验复用资源**：防止上一个试验的脏状态污染下一个试验
- 按分类内核 §6.2 重新计算两个候选的置信度
- 应用分类内核 §6.3 / §6.1.1 的"试验结果与状态机节点的强绑定"规则
- 如果 PRIMARY/SECONDARY 排名翻转，更新 REVISED_PRIMARY

## 4.3 Step B subagent 的输出契约

| 字段 | 要求 |
|---|---|
| **试验执行记录** | 每个试验的命令、原始输出、与"预期结果 A/B"的匹配判定、得到的 Tier 1 证据 |
| **执行方法与批准方法的对照**（必填） | 对每个试验，明确写出：“批准的方法 = X / 实际执行的方法 = Y / 是否一致 = 是/否”。**不一致时必须说明原因，并明确标注该试验结果不计入 Tier 1，等待主 agent 与人类协商重做。试验未能完成时也必须填写该字段说明原因** |
| **修订后 PRIMARY** | 分类 + 0–100 置信度（基于试验结果重算）|
| **修订后 SECONDARY** | 分类 + 0–100 置信度 |
| **REVISED_PRIMARY 是否翻转** | 若 Step A 中的 PRIMARY 与 SECONDARY 在置信度上发生翻转，明确声明 |
| **资源清理证明** | 对每个创建了真实 Azure 资源的试验，必须包含：(a) 清理命令（`terraform destroy` / `az group delete`）；(b) 命令输出；(c) 验证输出（如 `az group exists --name <rg>` 返回 `false` 或 `terraform state list` 为空）。**清理失败必须如实报告**，不得隐瞒；主 agent 收到后需告知人类手工介入 |
| **预承诺映射执行情况** | 若试验结果触发了 Step A 中的"结果 X → 类别 Y"映射，必须明确执行该映射 |

## 4.4 门 B → C

> **无单独人类审批。** 主 agent 收到 Step B 修订简报并校验合规后，立即进入 Step C。

---

# 5. Step C — 独立同行审查（Reviewer）

## 5.1 主 agent 动作

1. **看板评论（开始）**：`Step C 开始：调度独立 reviewer subagent 审查 Step A + B 的分类过程`
2. **调度 Step C reviewer subagent**（**全新会话**，与 Step A、B subagent 完全隔离），输入包括：
   - 角色声明："你是 Step C reviewer subagent"
   - 分类内核完整路径（reviewer 必读 §6.6）：`{{azurerm_provider_issue_classification.md}}`
   - **目标 Provider 仓库的本地工作目录路径**（`<work_dir>`）— reviewer 可能需要打开源代码独立核对状态机路径上引用的具体行
   - **Step A subagent 的完整简报**
   - **Step B subagent 的完整修订简报**
   - Trello 卡片 ID 和 GitHub Issue 的 URL — **告知 reviewer 必须通过 `gh` CLI 重新拉取最新内容，不能信赖前序 Step 的缓存**
3. **等待 subagent 返回审查记录**
4. **校验输出契约**（见 §5.3）
5. **看板状态调整**：依§2.4 检查卡片是否在 `{{kanban.wait.plan_review.name}}` 列，不在则先移动过去
6. **看板评论（结束）**：评论必须严格按以下格式撰写，**第一段就明确告诉人类需要做什么**，避免人类把这条评论误以为只是进度汇报而忽略：

   ```
   ## ⚠️ 需要人类确认 — 流程已暂停

   本次 Issue 分类已完成 reviewer 审查，**流程现已暂停在人审门 C→D，需要你显式回复确认才能继续**。
   - reviewer 结论：<PASS / OVERRIDE>
   - 主 agent 采纳的最终分类建议：<类别> / 置信度 <分数>
   - 你需要做的：阅读下方完整审查记录后，在卡片回复中明确表态：
     1. 「批准分类为 X / 置信度 Y」 → 主 agent 进入 Step D
     2. 「推翻 reviewer 的 OVERRIDE，回到 Step B 修订分类」 → 见 §5.4 选项 2
     3. 「推翻整个分类，回到 Step A 重新设计试验」 → 见 §5.4 选项 3
   - 在收到你的明确回复之前，主 agent 不会推进到 Step D，也不会调度任何新的 subagent。

   ---

   ### 【reviewer 原话】
   <将 Step C subagent 返回的审查记录全文原样贴上 — 不改写、不截断>

   ### 【主 agent 编排说明】
   - reviewer 结论：<PASS / OVERRIDE>
   - 主 agent 采纳的最终分类建议：<类别> / 置信度 <分数>
   - 当前流程状态：暂停于人审门 C→D，等待人类显式确认
   ```

7. **进入人审门 C→D**（见 §5.4）：主 agent 在此停下。**收到人类在卡片上的显式决策（"批准分类为 X / 置信度 Y" 或对应推翻指令）之前，主 agent 不得自行进入 Step D，也不得调度任何新的 subagent。**

## 5.2 Step C reviewer subagent 应执行的事项

参见分类内核 §6.6（"同行审查 Peer Review：Step C reviewer subagent 的角色定义"）。

## 5.3 Step C reviewer subagent 的输出契约

返回必须是 PASS 或 OVERRIDE 之一：

### PASS

| 字段 | 要求 |
|---|---|
| **明确声明** | "PASS：确认 Step B 修订后的分类为 `<类别>` / 置信度 `<分数>`" |
| **审查覆盖** | 简要说明审查覆盖了哪些检查项、为何认为没有违规 |
| **试验方法合规性检查**（仅当 Step B 执行了试验时必填） | 独立核对“人类批准的试验方法 = ?” vs “Step B 实际执行的方法 = ?”。如发现 Step B 偊偷降级（如把真实 apply 换成单元测试 + mock），**不得给 PASS**，必须转为 OVERRIDE 并要求重做试验 |

### OVERRIDE

| 字段 | 要求 |
|---|---|
| **明确声明** | "OVERRIDE：修正分类为 `<新类别>` / 置信度 `<新分数>` / 状态机路径 `<节点列表>`" |
| **违反规则列表** | 列出违反的具体规则（如"违反 §2.3 第 X 条"）|
| **修正后的状态机分支证据链** | 完整的修正后路径与每个节点的证据 |
| **是否需要回到 Step A** | 若有未消除的不确定性（包括 Step B 偊偷降级导致原试验作废），标注"建议主 agent 回到 Step A 重新设计试验” |

## 5.4 人审门 C → D

> **⚠️ 等待人类审批最终分类。** 人类的选项：
>
> 1. **接受 reviewer 的结论**（PASS 或 OVERRIDE 都可被接受）
> 2. **推翻 reviewer 的 OVERRIDE**，回到 Step B 修订后的分类
> 3. **推翻整个分类**，要求主 agent 设计新一轮试验回到 Step A
>
>
> - PASS 时：最终分类 = reviewer 确认的 Step B 修订分类
> - OVERRIDE 时：最终分类 = reviewer 修正后的分类与置信度
>
> 人类显式决策时，决策必须在看板上显式记录（"人类批准分类为 X / 置信度 Y"）。

---

# 6. Step D — 加载类型专属指南（主 agent 自己执行）

## 6.1 主 agent 动作

1. **看板评论（开始）**：`Step D 开始：根据最终分类 <X> 加载下游专属文件`
2. **按下面的路由表加载对应的下游文件**
3. **看板评论（结束）**：明确列出实际加载的文件名（含完整路径），便于事后追溯

> **⚠️ 进入 Step D 后，主 agent 不再使用本文件或分类内核文件作为分析依据。** 此后所有"行动"均由下游专属文件指引。

## 6.2 路由表

| 最终分类 | 必须加载的下一阶段文件 |
|---|---|
| **Bug** | [azurerm_provider_issue_bug.md]({{azurerm_provider_issue_bug.md}})（保留为参考），主推进路径 = [azurerm_provider_issue_bug_plan.md]({{azurerm_provider_issue_bug_plan.md}})（Plan 阶段）→ 人类批准 Plan 后 → [azurerm_provider_issue_bug_action.md]({{azurerm_provider_issue_bug_action.md}})（Action 阶段） |
| **Feature Request** | [azurerm_provider_issue_feature_request.md]({{azurerm_provider_issue_feature_request.md}}) |
| **Question / 反馈** | [azurerm_provider_issue_question.md]({{azurerm_provider_issue_question.md}}) |
| **Security** | [azurerm_provider_issue_security.md]({{azurerm_provider_issue_security.md}}) |

加载对应文件后，主 agent 按该文件描述的子流程继续，本文件不再约束后续分析（除非该文件显式回引本文件的某条规则）。

---

# 7. Step E — 简报归档与看板状态更新（主 agent 自己执行）

## 7.1 主 agent 动作

1. **归档**：把 Step A → D 的全部产出归档到看板卡片，至少包括：
   - Step A subagent 的初次分类简报
   - 人类对试验计划的批准/修改决策
   - Step B subagent 的修订简报
   - Step C reviewer subagent 的审查记录
   - 人类对最终分类的决策
   - Step D 实际加载的下游文件名（含完整路径）
2. **更新卡片状态/标签**：根据最终分类，更新卡片到下一阶段对应值（如 `bug:plan-pending`、`feature-request:design-pending` 等，具体依看板约定）
3. **看板评论（结束）**：声明"分类阶段完成，进入下游 `<X>` 流程"

此后所有"行动"由主 agent 在对应阶段文件的指引下执行（包括 PR、回复用户、调整标签），仍遵循"主 agent 执行 + 人类审批"的协同模型。
