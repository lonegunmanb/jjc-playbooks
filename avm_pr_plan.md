# AVM PR 流程 — {{kanban.plan.name}} 阶段（卡片在 `{{kanban.plan.name}}` 列时加载）

> **前置阅读（强制）**：本文件只覆盖 {{kanban.plan.name}} 阶段专属内容。继承 `{{avm_pr.md}}` 的全部共享内容（PR 定位、核心原则、工具基线）。如未读取共享文件，禁止开始本阶段工作。

## 0. 阶段定义

- **目标**：理解 PR、发现问题、提出方案。**不执行任何修复**（不 push、不 force push、不 approve deployment、不创建分支）。
- **触发**：卡片刚进入 `{{kanban.plan.name}}` 列，或卡片被人类从其他列退回 `{{kanban.plan.name}}`。
- **终态**：在卡片发布结构化简报（按第 6 节模板），将卡片移动到 `{{kanban.wait.plan_review.name}}`。

## 1. PR 基础信息采集

### 1.1 必采集元数据

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
| 已有 Review Comments | `pulls/{n}/comments` + `pulls/{n}/reviews` + `issues/{n}/comments` | 维护者/贡献者的讨论记录 |

### 1.2 读取已有 Code Review Comments（强制）

> 读取 PR 详情时，必须同时读取已有的 Code Review Comments。这些评论是维护者和贡献者之间的讨论记录，对理解 PR 上下文至关重要。

```powershell
# 行级 review comments（代码行上的评论）
gh api "repos/{owner}/{repo}/pulls/{number}/comments" --paginate --jq '.[] | {user: .user.login, path: .path, line: .line, body: .body, created_at: .created_at}'

# 整体 reviews（approve / request changes / comment）
gh api "repos/{owner}/{repo}/pulls/{number}/reviews" --paginate --jq '.[] | {user: .user.login, state: .state, body: .body, submitted_at: .submitted_at}'

# PR 对话评论（非行级的普通评论）
gh api "repos/{owner}/{repo}/issues/{number}/comments" --paginate
```

### 1.3 处理 PR 描述中的截图/图片附件

扫描 PR 描述（`$pr.body`），查找图片 URL（`https://github.com/user-attachments/assets/...` 或 `![...](url)` / `<img src="url" ...>`）。

对每个图片 URL，下载并用 `markitdown` 做 OCR：

```powershell
Invoke-WebRequest -Uri "<image_url>" -OutFile "image_temp.png"
markitdown image_temp.png
```

OCR 提取的文字纳入后续分析。截图常含 CI 输出、plan diff、错误日志或 Azure Portal 信息——是 review 的重要上下文。OCR 无意义时记录该事实并在分析中注明。

### 1.4 来源判定（关键）

```powershell
$isExternal = $pr.head.repo.full_name -ne $pr.base.repo.full_name
```

- 相同 → 本仓库分支 PR（**内部 PR**）
- 不同 → Fork 分支 PR（**外部 PR**）

来源不同决定整个验证策略和 {{kanban.action.name}} 阶段路径，必须在简报第一段明确标注。

### 1.5 关联 Issue 判定

检查 PR 描述是否含 `Fixes #N` / `Closes #N` / `Resolves #N` 或 Issue URL：

- 有关联 Issue → 走第 2.1 节
- 无关联 Issue → 走第 2.2 节

### 1.6 已有 Review Comments 分析（强制）

> 若 PR 已有 review comments，必须在意图审查（第 2 节）之前完成本节分析。已有评论反映了维护者的关注点和贡献者的回应，直接影响后续审查方向。

#### 1.6.1 评论分类

| 类别 | 识别方式 | 说明 |
|------|----------|------|
| 维护者反馈 | 来自 repo collaborator/member 的评论 | 高优先级，代表官方意见 |
| 贡献者回复 | PR 作者的回复 | 理解贡献者意图与解释 |
| 社区讨论 | 其他参与者的评论 | 补充视角 |
| Bot/CI 评论 | 来自 bot 或 CI 系统 | 自动化检查结果 |

#### 1.6.2 合理性与正确性审查

对每条**非 bot 的实质性评论**评估：

1. **评论指出的问题是否成立**
   - 审查评论引用的代码位置，验证问题是否真实存在；
   - 若评论指出 bug / 设计缺陷 / 不合规，独立验证该判断是否正确；
   - 若评论提出的修改建议可能引入新问题，记录潜在风险。
2. **贡献者的回应是否充分**
   - 是否已解决维护者提出的问题；
   - 回应是否合理（已修改 / 给出充分理由拒绝修改）；
   - 是否有未回应的维护者评论（可能阻塞合并）。
3. **讨论是否达成共识**
   - 识别仍在争议中的设计决策；
   - 识别已 resolved 与未 resolved 的讨论线程；
   - 记录维护者明确要求但尚未实现的变更。

#### 1.6.3 纳入简报

简报中（第 6 节）必须包含：

- 维护者已提出的关键问题（避免重复提出相同问题）；
- 尚未解决的讨论点；
- 贡献者回应中不正确或不充分的部分；
- Agent 对争议点的独立判断（附依据）。

## 2. 意图与合理性审查

### 2.1 有关联 Issue

1. 读取关联 Issue 全量信息（标题、描述、标签、评论）。
2. 交叉验证：
   - PR 是否对准 Issue 问题；
   - 修复路径是否合理（参照 `{{avm_issue_bug.md}}` / `{{avm_issue_feature_request.md}}`）；
   - Issue 中提到但 PR 未覆盖的内容是否存在；
   - PR 是否包含 Issue 未提到的额外改动（文档更新、lint 修复、中央同步类改动可视情况接受）。
3. 写入交叉验证结论并纳入最终简报。

### 2.2 无关联 Issue（构造"虚拟 Issue"）

1. 从标题、描述、commit、diff 推理 PR 意图。
2. 一句话定义"要解决的问题"（含预期行为 vs 当前行为）。
3. 按 `{{avm_issue.md}}` 第 3 节分类（Bug / 功能 / 文档 / 破坏性等）。
4. 按对应类型指南评估合理性：
   - Bug → `{{avm_issue_bug.md}}`
   - 功能请求 → `{{avm_issue_feature_request.md}}`
   - 文档 → 准确性与完整性
   - 其他 → 对应指南
5. 输出分类与合理性结论。

## 3. 安全审查（外部 PR 必须优先）

任何测试前，必须完成：

| 检查项 | 说明 | 发现问题时行动 |
|--------|------|----------------|
| Workflow 文件 | `.github/workflows/` 是否修改 | 🔴 安全红线，建议关闭 PR |
| CI 配置 | `.github/`、`.azure-pipelines/` 等是否修改 | 🔴 安全红线，建议关闭 PR |
| 外部脚本执行 | 是否引入下载/执行外部脚本 | 🟡 深入审查意图 |
| 环境变量泄露 | 是否读取/输出敏感变量 | 🔴 安全红线 |
| Provider 配置 | 是否指向非官方 registry/backend | 🔴 安全红线 |

结论：

- 通过 → 继续后续流程。
- 不通过 → 简报中明确风险，建议关闭 PR；停止测试运行。

## 4. 代码审查

### 4.1 变更范围

- 阅读完整 diff，确认改动意图与范围。
- 归类：新功能 / Bug 修复 / 文档 / 重构 / 依赖升级 / 混合。
- 列出关键文件与关键改动点。

### 4.2 AVM 合规检查

| 维度 | 要求 |
|------|------|
| Provider 模式一致性 | AzureRM / AzAPI / AzAPI-模拟-AzureRM 不混用 |
| 变量设计 | 默认值、类型约束、validation 合理 |
| 非破坏性 | 升级后 `terraform plan` 不应出现意外 drift（结合 examples 验证） |
| 文档同步 | `README.md`、`_header.md`、`examples/` 同步更新 |
| 常见错误 | 无遗留 `TODO`，不误提交 `terraform.lock.hcl` / `terraform.tfvars` |

### 4.3 兼容性 / 破坏性评估

重点检查：

- 新增无默认值 required variable；
- 修改现有变量语义或类型；
- 删除/重命名既有变量或输出；
- 稳定配置下 `terraform plan` 是否出现意外 update / add / delete。

存在破坏性变更时，简报必须包含：

```
[BREAKING-CHANGE][REQUIRES-MAINTAINER-APPROVAL]
```

### 4.4 配置漂移（Configuration Drift）规则

> ❗ 配置漂移是一种错误，不可接受，必须尽全力修复。

配置漂移：`terraform apply` 后再次执行 `terraform plan` 出现非空变更（资源属性与声明状态不一致）。通常是 Provider 行为、API 返回值或模块逻辑的 bug。

处理约束：

- 测试中发现漂移时，必须作为待修复错误记录并尝试解决。
- **禁止在根模块中将漂移字段添加到 `lifecycle { ignore_changes }`** 来规避（example 代码中可以），除非得到人类维护者明确许可。
- 应从根因解决：修复属性映射、调整默认值、或在模块逻辑中正确处理 API 返回值差异。
- 若确实无法在模块层面解决（如 Provider bug），简报中详细说明原因，请求维护者决策是否例外允许 `ignore_changes`。

**参考**：[AVM Replicator Executor Guide](https://raw.githubusercontent.com/lonegunmanb/avm-replicator-2-azapi/refs/heads/main/replicator/executor.md) 中的漂移处理指导。

### 4.5 AzAPI-模拟-AzureRM 额外检查

- 是否遵循"精确复制"原则；
- 新增 validation 是否完整复制 AzureRM Provider 验证；
- 默认值是否与 AzureRM 一致；
- body 结构分层是否正确（根级 vs body 内）。

## 5. 验证策略（按来源分支）

### 5.1 内部 PR（本仓库分支）

**{{kanban.plan.name}} 阶段不做本地测试运行**，只做第 1~4 节的分析。CI 验证与修复在 {{kanban.action.name}} 阶段做（参见 `{{avm_pr_action.md}}`）。

可引用已有 CI 结果做参考，但**不主动触发新 CI**。

### 5.2 外部 PR（Fork 分支）

不能直接在 Fork PR 上触发 CI，需本地验证。

#### 5.2.1 本地检查策略：分离"同步问题"与"代码质量问题"

`pr-check` 包含两类检查：

1. **中央代码同步检查**：grept、mapotf/avmfix、docs 生成（这些会真实改文件）。
2. **代码质量检查**：TFLint、Well-Architected/Conftest。

关键事实：

- `grept` 是"执行同步"的工具，不是只读验证；会增删改文件。
- `mapotf` 是"执行变换"的工具，不是只读验证；会修改 `.tf`。
- `pr-check` 的第一步会检查 uncommitted changes；有改动则直接失败。

关键诊断：

> 若 `pr-check` 报"uncommitted changes/文件有改动"，且此前确实已运行并提交过 `pre-commit`，则根因通常是 `Azure/avm-terraform-governance` 在之后又更新；重新运行 `pre-commit` 并提交新 diff 即可。

#### 5.2.2 阶段一：先做 pre-commit 同步

> 下方示例里的 `<work_dir>` 指目标仓库在宿主机上的绝对路径（由 Gateway 在 CARD CONTEXT 中预填）；`podman` 与 `docker` 命令行参数等价，可按宿主机实际安装的运行时替换。

```powershell
# 在容器内运行 pre-commit 同步中央治理内容
podman run --pull always --rm `
  -v "<work_dir>:/src" `
  -w /src `
  -e PORCH_NO_TUI=1 `
  mcr.microsoft.com/azterraform:avm-latest `
  ./avm pre-commit

# 记录重置点
cd <work_dir>
$resetTarget = git rev-parse HEAD

# 临时提交（仅为消除 uncommitted changes 干扰）
git add .
git commit -m "chore: pre-commit sync"
```

> 该 commit 为临时提交，**绝不推送**。检查结束后必须回滚。

#### 5.2.3 阶段二：只跑代码质量检查

```powershell
podman run --pull always --rm `
  -v "<work_dir>:/src" `
  -v "$HOME/.azure:/root/.azure:ro" `
  -w /src `
  mcr.microsoft.com/azterraform:avm-latest `
  ./avm pr-check
```

说明：

- `external-pr-check.porch.yaml` 挂载到 `/src/.porch.yaml`；`porch run` 默认读取该文件。
- Conftest 依赖 `terraform plan`，通常需要 `.azure` 挂载。
- 若仅跑 TFLint，可不挂载 `.azure`。

#### 5.2.4 代码实现审查

阅读 diff，检查逻辑正确性（不需要容器）。

#### 5.2.5 检查完成后的清理（强制）

```powershell
cd <work_dir>
git reset --hard $resetTarget
```

必须恢复初始状态，确保不残留临时同步提交。

#### 5.2.6 问题分类记录

| 类别 | 示例 | 说明 |
|------|------|------|
| Lint 错误 | TFLint warning/error | 格式、命名、AVM 规范 |
| Conftest 失败 | Well-Architected 不合规 | 安全/弹性策略 |
| 代码实现问题 | 逻辑错误、类型不匹配、缺 validation | 功能正确性 |
| 兼容性问题 | breaking change、missing defaults | 升级影响 |

**限制**：Agent 不能直接修改 Fork 分支代码，只能给出处理方案。

#### 5.2.7 向维护者给出两种方案

**方案 A：退回贡献者修复**

- 在 PR 评论列出问题；
- 每个问题给出"已验证有效"的修复方法；
- 等贡献者修复后再进入 {{kanban.plan.name}}。

适用：问题多/复杂；贡献者响应快；需贡献者自行判断设计取舍。
优点：保留贡献者 authorship、维护者成本低。
缺点：依赖贡献者响应与能力。

**方案 B：转 release 分支由 Agent/维护者修复**

- 在 Trello 卡片记录问题与修复方式；
- 建议流程：
  1. 从最新 `main` 创建 `release/<description>`；
  2. 将外部 PR 的 base 改到 release 分支并合并；
  3. 在 release 分支修复 lint/conftest/代码问题；
  4. 从 release 分支向 `main` 开新 PR（触发 CI e2e）；
  5. CI 通过后等待维护者合并。

适用：问题少且可快速修；贡献者不响应或很慢；变更价值高不宜长期搁置。
优点：速度快，可跑完整 CI。
缺点：维护者/Agent 承担修复工作；贡献者部分失去 authorship（原 PR 仍保留记录）。

## 6. 对维护者的输出模板（必含）

### 6.1 PR 摘要

- PR 标题与编号
- 作者与来源类型（外部 / 内部）
- 一句话说明 PR 在做什么

### 6.2 意图审查

- 关联 Issue：有 / 无
- 有关联时：交叉验证结论（是否真正解决）
- 无关联时：虚拟 Issue 分类与合理性
- 总体结论：合理 / 部分合理 / 不合理

### 6.3 安全审查

- 结论：通过 / 发现问题
- 若有问题：明确风险点

### 6.4 代码审查发现

- 变更范围摘要
- AVM 合规性结论
- 兼容性评估：非破坏性 / 可能破坏性
- 按严重程度列问题清单

### 6.5 测试结果

- 内部 PR：引用已有 CI 状态（不主动触发；{{kanban.action.name}} 阶段才会执行 approve + 等待）
- 外部 PR：本地 lint / conftest / 代码审查结论

### 6.6 唯一默认行动建议（P1–P6）

| 路由 | 触发条件 | 建议行动 |
|------|----------|----------|
| P1: 直接可合并 | 安全通过 + 代码无问题 + 测试通过 | Approve & Merge |
| P2: 需要小修改 | 有小问题但整体合理 | Request Changes（附修改清单） |
| P3: 外部 PR 需修复（方案 A） | Fork PR + 有问题 + 建议退回贡献者 | 在 PR 评论给问题与已验证修复方法 |
| P4: 外部 PR 需修复（方案 B） | Fork PR + 有问题 + 建议 Agent 修复 | 合并到 release 分支后修复并开新 PR |
| P5: 意图不合理 | 虚拟 Issue 审查不通过 / 修改方向错误 | Close with comment（附理由） |
| P6: 安全问题 | 安全审查不通过 | Close PR & report contributor |

### 6.7 可直接发送的 Review Comment 草稿

- 可直接贴到 GitHub PR；
- 包含问题与改进建议；
- 语气友好、建设性。

## 7. {{kanban.plan.name}} 阶段终态

完成上述步骤后：

1. 在卡片发布结构化简报（按第 6 节模板）；
2. 将卡片移动到 `{{kanban.wait.plan_review.name}}`。

> ⚠️ {{kanban.plan.name}} 阶段**不得**移动卡片到 `{{kanban.wait.action_review.name}}`、`{{kanban.action.name}}` 或 `{{kanban.wait.exception.name}}`。这些列由人类维护者或 {{kanban.action.name}} 阶段的 Agent 控制。

> ⚠️ {{kanban.plan.name}} 阶段**不得**执行任何变更操作（push、force push、approve deployment、创建分支、触发 CI、改卡片状态除 → `{{kanban.wait.plan_review.name}}` 外）。变更操作只在 {{kanban.action.name}} 阶段执行，且必须读取 `{{avm_pr_action.md}}`。
