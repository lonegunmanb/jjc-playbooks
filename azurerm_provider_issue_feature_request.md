# AzureRM Provider 功能请求类 Issue 处理指南

## 1. 适用范围

当 Issue 被归类为功能请求（请求 Provider 新增资源、数据源、属性暴露或行为增强）时，Agent 必须使用本指南协助维护者完成预处理。

Provider 功能请求的典型类型：

- **新增资源**：请求支持一个全新的 `azurerm_xxx` resource。
- **新增数据源**：请求支持一个全新的 `data.azurerm_xxx` data source。
- **新增属性**：请求在已有资源中暴露新的属性（Azure API 已支持但 Provider 未暴露）。
- **升级 API 版本**：请求将资源使用的 Azure API 版本升级到更新版本以获取新功能。
- **行为增强**：请求改善现有行为（如更好的错误信息、更精确的 diff、更合理的默认值）。

本阶段目标是：

1. 判断请求的功能在 Azure API 层面是否可用且在 Provider 层面是否可实现。
2. 评估实现该功能对现有用户的兼容性影响。
3. 产出高质量的可行性分析与 Go 实现计划，供维护者决策。

## 2. 处理原则（强制）

1. 先验证可行性，后承诺实现
   - 在确认 Azure API 支持和 Provider 架构兼容之前，不做实现承诺。
2. 向后兼容优先
   - 新增功能必须以可选方式引入，不改变现有属性的语义或默认行为。
   - 新增属性应设为 `Optional` 或 `Computed`，确保现有用户升级 Provider 后 `terraform plan` 无 drift。
3. 遵循 Provider 现有模式
   - 新增代码必须遵循 Provider 仓库的编码规范和架构模式。
   - Schema 定义、Expand/Flatten 函数、验收测试的风格必须与同一服务目录下的现有资源一致。
4. 计划先行，审批后执行
   - Agent 在该阶段主要产出可行性分析与实现计划；涉及代码实现和 PR 提交的动作需维护者审批。
5. 安全优先
   - 若功能请求涉及安全敏感操作（如密钥管理、权限变更），需额外评估安全影响。
   - 涉及密钥/密码的属性必须标记为 `Sensitive: true`。
6. Terraform 管理资源需要完整 CRUD 权限
   - Provider 假设调用方对被管理资源拥有完整读写权限。请求为部分权限路径（如仅 Tag Contributor）做专门优化的 Feature Request 应予以拒绝（参考 Issue #26498）。

## 3. 准入决策树（新增属性类请求）

> 本决策树适用于「在已有资源中添加/暴露新字段」类功能请求。新增资源、新增数据源等其他类型请求可参考本决策树的通用判定逻辑，但不必严格逐步执行。

### 3.1 参考文件对照表

决策树中的判定步骤引用了以下仓库指令文件（来自 [terraform-azurerm-ai-assisted-development](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development)）：

| 决策树中的引用 | 仓库文件完整路径 |
|---|---|
| Feature Flag Management | [`.github/instructions/api-evolution-patterns.instructions.md`](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development/blob/main/.github/instructions/api-evolution-patterns.instructions.md) |
| Breaking Change Patterns | [`.github/instructions/migration-guide.instructions.md`](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development/blob/main/.github/instructions/migration-guide.instructions.md) |
| FivePointOh Feature Flag Patterns | [`.github/instructions/schema-patterns.instructions.md`](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development/blob/main/.github/instructions/schema-patterns.instructions.md) |
| Schema Type Patterns / Validation Patterns | [`.github/instructions/schema-patterns.instructions.md`](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development/blob/main/.github/instructions/schema-patterns.instructions.md) |
| Schema Flattening | [`.github/instructions/provider-guidelines.instructions.md`](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development/blob/main/.github/instructions/provider-guidelines.instructions.md) |
| CRUD Patterns | [`.github/instructions/implementation-guide.instructions.md`](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development/blob/main/.github/instructions/implementation-guide.instructions.md) |
| PATCH/PUT 处理 | [`.github/instructions/azure-patterns.instructions.md`](https://github.com/WodansSon/terraform-azurerm-ai-assisted-development/blob/main/.github/instructions/azure-patterns.instructions.md) |

### 3.2 决策树

```
用户请求在某个 resource 中添加/暴露一个新字段
│
├─ 1. Azure API 是否支持该字段？
│   │
│   ├─ 否 → ❌ 拒绝
│   │       （Provider 不能实现 Azure API 不支持的功能）
│   │
│   └─ 是 → 继续判断
│       │
│       ├─ 1a. 该字段在哪个 API 版本中可用？
│       │   ├─ 仅在 Preview API 中 → ⚠️ 暂缓
│       │   │   （等待 GA，除非 provider 的 feature flag 支持 preview）
│       │   │   参照: .github/instructions/api-evolution-patterns.instructions.md
│       │   │          → Feature Flag Management
│       │   │
│       │   └─ 在 GA API 中可用 → 继续
│       │
│       ├─ 2. 该字段是否属于 AzureRM Provider 的管辖范围？
│       │   │
│       │   │   判定标准：该字段是否属于
│       │   │   当前 resource 在 ARM 层面的配置属性？
│       │   │
│       │   ├─ 否 → ❌ 拒绝（应在其他 resource 中实现）
│       │   │   │
│       │   │   常见情况:
│       │   │   - 该字段属于子资源（应作为独立 resource）
│       │   │   - 该字段属于 data plane 操作（不是 control plane）
│       │   │   - 该字段属于另一个关联 resource 的属性
│       │   │   - 该功能应通过 azurerm_xxx_association resource 实现
│       │   │
│       │   └─ 是 → 继续
│       │
│       ├─ 3. 该字段是否会对已有用户造成 breaking change？
│       │   │   参照: .github/instructions/migration-guide.instructions.md
│       │   │          → 💔 Breaking Change Patterns
│       │   │   参照: .github/instructions/schema-patterns.instructions.md
│       │   │          → 🚀 FivePointOh Feature Flag Patterns
│       │   │
│       │   ├─ 是（新 Required 字段、改变已有字段语义等）
│       │   │   ├─ 可以通过 FivePointOh feature flag 管理？
│       │   │   │   ├─ 是 → ✅ 接受，但必须走 breaking change 流程
│       │   │   │   └─ 否 → ⚠️ 暂缓到下一个 major version
│       │   │   └─ 可以作为 Optional 字段无破坏地添加？
│       │   │       └─ 是 → ✅ 接受
│       │   │
│       │   └─ 否 → 继续
│       │
│       ├─ 4. Schema 设计是否符合 Provider 规范？
│       │   │   参照: .github/instructions/schema-patterns.instructions.md
│       │   │          → 📋 Schema Type Patterns
│       │   │          → ✅ Validation Patterns
│       │   │          → ⚙️ Azure-Specific Schema Patterns
│       │   │   参照: .github/instructions/provider-guidelines.instructions.md
│       │   │          → Schema Flattening
│       │   │
│       │   │   检查项:
│       │   │   - Enabled/Disabled 枚举应建模为 boolean
│       │   │   - "None" 值应在 expand/flatten 中内部处理
│       │   │   - 不必要的 wrapper 结构应扁平化
│       │   │   - 验证值应使用 SDK PossibleValues 函数
│       │   │   - 字段命名遵循 snake_case + 语义化
│       │   │
│       │   ├─ 用户提议的 schema 设计不合规
│       │   │   └─ ✅ 接受需求，但 schema 设计由 maintainer 决定
│       │   │
│       │   └─ 合规 → 继续
│       │
│       └─ 5. 实现可行性评估
│           │   参照: .github/instructions/implementation-guide.instructions.md
│           │          → CRUD Patterns
│           │   参照: .github/instructions/azure-patterns.instructions.md
│           │          → PATCH/PUT 处理
│           │
│           ├─ 5a. 该字段的读写是否在现有 CRUD 生命周期内？
│           │   ├─ 是 → 低成本，可在现有 Create/Update 中添加
│           │   └─ 否 → 需要额外 API 调用，评估复杂度
│           │
│           ├─ 5b. PATCH vs PUT 影响
│           │   参照: .github/instructions/azure-patterns.instructions.md
│           │          → PATCH 操作处理
│           │   ├─ 如果资源用 PATCH：新 Optional 字段可安全添加
│           │   └─ 如果资源用 PUT：需确保 Read→Update 回传所有字段
│           │
│           └─ 最终判定
│               ├─ 低成本 + 低风险 → ✅ 接受，标记为 Feature Request
│               ├─ 高成本但需求合理 → ✅ 接受，标记需要 design review
│               └─ 不符合 Provider 设计理念 → ❌ 拒绝并说明原因
```

### 3.3 决策树使用说明

- Agent 在第 4 节的快速处理流程中遇到「新增属性」类请求时，**必须按本决策树的 5 个步骤逐一判定**，在维护者简报中记录每步的结论与证据。
- 决策树中任何步骤产生「❌ 拒绝」或「⚠️ 暂缓」结论时，**立即停止后续步骤**，直接路由到对应的功能请求路由（F4/F7 等）。
- 「参照」所列文件是 schema 设计与实现的权威参考，Agent 在产出 Go 实现计划时**应查阅这些文件**以确保方案符合 Provider 规范。

## 4. 快速处理流程

对每个功能请求 Issue，按顺序执行：

1. 需求归一化
   - 提炼一句话需求定义：用户想要什么能力、解决什么场景、当前的阻塞点是什么。
   - 确定功能请求类型：新增资源 / 新增数据源 / 新增属性 / 升级 API 版本 / 行为增强。
2. 属性定位与跨体系映射
   - 建立用户术语 ↔ `azurerm` Schema 属性名 ↔ Azure REST API 属性路径的映射关系。

     | 维度 | 内容 |
     |------|------|
     | 用户原文 | 用户在 Issue 中使用的原始表述 |
     | Azure REST API 资源 | 对应的 REST API 资源类型和 API 版本 |
     | REST API 属性路径 | 属性在 REST API JSON body 中的完整路径 |
     | AzureRM 资源 | 对应的 `azurerm_xxx` resource 类型 |
     | AzureRM Schema 属性名 | 属性在 Provider Schema 中的名称（如 `template.0.revision_suffix`） |
     | 功能名/产品术语 | Azure 文档/Portal 中的叫法 |

3. 现有支持检查
   - 在进入可行性分析之前，先检查 Provider 是否**已经支持**请求的功能：
     - ① **Schema 层**：检查资源的 Schema 定义中是否已存在对应属性。
     - ② **Expand/Flatten 层**：检查属性是否已在 Expand 和 Flatten 函数中处理（可能 Schema 中存在但实际未传递给 API）。
     - ③ **文档层**：检查 `website/docs/r/` 下的文档是否已记录该属性。
     - ④ **CHANGELOG / PR 层**：检查是否在近期版本中已实现但用户使用的是旧版本。
   - **判定结论**：

     | 发现 | 结论 | 路由 |
     |------|------|------|
     | Schema 中已暴露且 Expand/Flatten 已实现 | 已支持 | → F8（已支持） |
     | Schema 中存在但 Expand/Flatten 未处理 | 部分实现（可能是遗漏） | → 可能转为 Bug |
     | Schema 中不存在 | 未支持 | → 继续后续流程 |

4. Azure API 可行性验证
   - 检查 Azure REST API 规范，确认请求的功能在 API 层面是否可用。
   - **检查功能的 GA 状态**：确认该功能在 Azure REST API stable 版本中是否存在。若仅在 preview API 中存在，直接路由到 F7（Preview 功能）。
   - 确认 API 属性的类型、是否必填、是否只读、默认值行为。
5. Go SDK 支持检查
   - 检查 `hashicorp/go-azure-sdk` 中是否已包含对应 API 版本的 SDK 结构体。
   - 如果 SDK 尚未更新，评估是否需要先提交 SDK PR。
6. 兼容性影响评估
   - 评估新增 Schema 属性对现有用户的影响。
   - 确认新属性可以以 `Optional` + `Computed` 或纯 `Computed` 方式引入。
   - 若不可避免改变现有行为，按破坏性变更流程处理。
7. Go 实现方案设计
   - 如果经过评估只有一种可行方案，给出该方案的完整描述。
   - 如果有多种不同方案都可以实现，给出**至多三种**方案的描述。每个方案的描述应简单易懂。
   - 方案排序原则（优先级从高到低）：
     1. **简单优于复杂**：代码改动小、逻辑直观的方案排在前面。
     2. **低风险优于高风险**：不引入 breaking change、不需要复杂前置条件的方案排在前面。
     3. **治本优于治标**：从根本上解决问题的方案优于临时绕过的方案。
   - 每个方案需明确：
     - 拟新增/修改的 Schema 属性定义（属性名、类型、Required/Optional/Computed、Default、ForceNew、ValidateFunc）。
     - 拟修改的 Expand 函数（HCL → API 请求的转换逻辑）。
     - 拟修改的 Flatten 函数（API 响应 → state 的转换逻辑）。
     - 拟修改的 CRUD 函数（如需要）。
     - 验收测试计划（新增测试用例、修改现有测试用例）。
     - 文档更新计划（`website/docs/r/` 下的文档）。
     - **⚠️ 风险提示**（如有）：每个方案如果存在风险，必须在此处特别说明，包括但不限于：
       - 是否涉及 breaking change
       - 是否需要复杂的前置条件（如等待上游 SDK 更新、API 版本升级等）
       - 是否可能导致长时间运行的操作
       - 是否需要对某个字段做特殊处理（如 hardcode、条件分支等增加维护负担的设计）
       - 是否依赖未公开的 API 行为或内部机制
8. 重复与关联检查
   - 检查是否与 open/closed Issue 或已有 PR 重复。
   - 检查 Azure SDK for Go 仓库是否有相关 Issue。
9. 维护者简报输出
   - 使用第 4 节模板输出可直接决策的信息。

## 5. Agent 输出模板（给维护者）

对每个功能请求 Issue，Agent 输出必须包含：

1. 分类摘要
   - 建议类别：功能请求
   - 功能请求类型：新增资源 / 新增数据源 / 新增属性 / 升级 API 版本 / 行为增强
   - 置信度：高/中/低
   - 受影响资源：`azurerm_xxx`（已有资源）或新资源名
   - 证据摘录：来自 Issue 的关键原文
2. 需求分析
   - 一句话需求定义
   - 用户场景描述
   - 当前阻塞点
   - 属性映射表：（用户原文 → REST API 资源/属性路径 → AzureRM Schema 属性名）
3. 可行性结论
   - Azure 功能 GA 状态：GA / Preview（证据：stable API 版本号或文档链接）
   - Azure API 支持状态：已暴露 / 未暴露
   - Go SDK 支持状态：已包含 / 未包含（SDK 版本和结构体路径）
   - Provider 当前使用的 API 版本
   - 证据：引用 API 规范、SDK 源代码或 Provider 源代码
4. 兼容性评估
   - 新增属性可否以 Optional/Computed 方式引入：是/否
   - 对现有用户 `terraform plan` 的影响：无 drift / 可能 drift（说明原因）
   - 是否涉及破坏性变更：否 / 是（若是，标注 `[BREAKING-CHANGE]`）
5. Go 实现计划
   - 可行性状态：可实现 / 需等待 SDK 更新 / 需等待 API GA / 不可实现
   - （可实现时）方案列表（至多三种，按优先级排列：简单 > 低风险 > 治本）：
     - **方案 N**：（方案名称——一句话概括）
       - 实现目标：
       - 拟修改的 Go 文件列表：
       - Schema 变更：（属性名、类型、Required/Optional/Computed、Default、ForceNew、ValidateFunc）
       - Expand 函数变更：（映射逻辑摘要）
       - Flatten 函数变更：（映射逻辑摘要）
       - CRUD 函数变更（如需要）：
       - 验收测试计划：新增/修改的测试用例
       - 文档更新：`website/docs/r/` 下需更新的文件
       - **⚠️ 风险提示**：（如无风险写"无"；如有风险必须逐条列明：breaking change、复杂前置条件、长时间运行、字段特殊处理、依赖未公开 API 等）
   - （需等待上游时）：
     - 阻塞原因：SDK Issue 链接 / API preview 版本号
     - 临时替代方案（如有）
   - （不可实现时）：
     - 技术约束说明
     - 替代方案建议（如有）
6. 维护者行动建议（审批项）
   - 建议决策：批准实现 / 等待 SDK 更新 / 等待 API GA / 要求补充信息
   - 负责人：
   - 时间框：
   - 完成标准：
7. 可直接发送的回复草稿
   - 面向 Issue 提交者的简洁回复，说明当前结论与下一步。

## 6. 可行性验证指南（Agent 执行）

### 5.1 Azure 功能 GA 状态检查（强制，前置阻断检查）

> **⚠️ 此检查为前置阻断检查。若功能为 Preview，直接路由到 F7，不继续后续可行性验证。**

Agent 必须按以下步骤判定功能的 GA 状态：

1. **检查 Azure REST API 规范**（`github.com/Azure/azure-rest-api-specs`）：
   - 找到对应服务的 API 定义目录（如 `specification/app/resource-manager/Microsoft.App/`）。
   - 检查该功能对应的属性/操作是否存在于 `stable` 目录下的 API 版本中。
   - 若仅存在于 `preview` 目录 → **判定为 Preview**。
   - 若存在于 `stable` 目录 → **判定为 GA**。

2. **检查 Azure 官方文档**（`learn.microsoft.com/en-us/azure/`）：
   - 若页面标注 `Preview`、`Public Preview`、`(preview)` → **判定为 Preview**。
   - 若无 Preview 标记 → 视为 GA（需与步骤 1 一致）。

3. **记录判定结论**：GA 状态、证据（stable/preview API 版本号、文档链接）。

### 5.2 Azure API 属性详情检查

1. 查阅 Azure REST API 规范中对应资源的属性定义：
   - 属性是否为 `required` / `readOnly` / `x-ms-mutability`。
   - 属性的类型（string、integer、boolean、object、array）。
   - 属性的枚举值（如有）。
   - 属性的默认值（如有）。
2. 记录这些信息，它们将直接影响 Provider Schema 的设计：
   - API `required` → Schema `Required: true`
   - API `readOnly` → Schema `Computed: true`
   - API `x-ms-mutability: ["create"]` → Schema `ForceNew: true`
   - API 枚举值 → Schema `ValidateFunc: validation.StringInSlice(...)` 或 `ValidateFunc: validation.StringIsNotEmpty`

### 5.3 Go SDK 支持检查

1. 检查 `hashicorp/go-azure-sdk` 是否已包含目标 API 版本：
   - 在 `resource-manager/<service>/<api-version>/` 目录下查找。
   - 确认请求的属性是否存在于 Go 结构体中。
2. 如果 SDK 未更新：
   - 检查是否有 open PR 正在更新。
   - 评估是否可以先在 Provider 中使用旧版 SDK + 手动扩展。
   - 记录 SDK 更新阻塞。

### 5.4 Provider 架构兼容性检查

1. 检查目标资源的现有 Schema 结构，确认新属性是否可以自然融入。
2. 检查现有的 Expand/Flatten 函数，确认新增映射逻辑的复杂度。
3. 如果是新增资源，检查同一服务目录下的现有资源，了解代码风格和架构模式。
4. 确认是否需要新增 client 初始化逻辑（`internal/clients/`）。

## 7. 向后兼容性约束（核心）

### 6.1 默认验收标准

新增功能默认必须满足以下标准（除非维护者批准例外）：

1. 纯可选引入
   - 新增 Schema 属性必须为 `Optional`、`Computed` 或 `Optional + Computed`，不引入新的 `Required` 属性（除非是全新资源）。
2. 计划稳定
   - 现有用户升级 Provider 后，对已有配置执行 `terraform plan` 应无 drift。
3. 测试覆盖
   - 新增功能必须有对应的验收测试。
   - 现有测试（`_basic`、`_complete`）仍须通过。
4. 文档同步
   - 新增属性必须在 `website/docs/r/` 下的文档中添加说明。

### 6.2 若功能请求隐含破坏性变更（必须高亮）

若请求的功能在技术上无法以纯可选方式引入（如需要将 `Optional` 改为 `Required`、或改变 `Default` 值），Agent 必须在维护者简报中使用醒目标记：

`[BREAKING-CHANGE][REQUIRES-MAINTAINER-APPROVAL]`

并且必须额外提供：

1. 为什么无法以可选方式引入（技术约束说明）。
2. 受影响的现有 Schema 属性/行为。
3. 迁移指南草案（用户需要执行的步骤）。
4. 版本策略建议（是否需要在 major 版本中实现）。
5. 是否需要 state migration 函数（`StateUpgraders`）。

## 8. 功能请求路由表

| 路由 | 触发条件 | Agent 提供内容 | 建议标签/状态 | 默认行动方案 |
|---|---|---|---|---|
| F1: 可实现且向后兼容 | API 已 GA，SDK 已支持，可以可选方式引入 | 可行性证据 + Go 实现计划 | `enhancement`，移除 `needs-triage` | 负责人：资源维护者。期限：3 个工作日内批准实现计划。完成标准：PR 已提交并进入 review。 |
| F2: 可实现但涉及破坏性变更 | 功能可实现但无法以纯可选方式引入 | 破坏性影响评估 + 迁移草案 + 版本建议 | `enhancement` + `breaking-change` | 负责人：资源维护者/版本负责人。期限：3 个工作日内完成破坏性审查。完成标准：明确版本策略。 |
| F3: 需等待 SDK 更新 | API 已 GA 但 Go SDK 尚未包含对应版本 | SDK 缺口说明 + SDK Issue/PR 链接（如有） | `enhancement` + `upstream` | 负责人：值班维护者。期限：1 个工作日内回复用户。完成标准：用户已知晓阻塞原因。 |
| F4: 超出 Provider 职责范围 | 请求的功能不属于 Provider 应承担的职责（如 Terraform Core 层面的限制） | 范围判定理由 + 替代方案指引 | 保留 `enhancement` | 负责人：值班维护者。期限：1 个工作日内回复。完成标准：已解释范围边界。 |
| F5: 证据不足待补充 | 缺少关键上下文（使用场景、期望 Schema 结构、版本信息等） | 缺失信息清单 + 追问模板 | `waiting-response` | 负责人：值班维护者。期限：1 个工作日内追问。完成标准：作者补齐信息或进入超时策略。 |
| F6: 实为其他类型 | 分析后发现不属于功能请求（实际是 Bug/安全等） | 重分类证据与目标类别理由 | 调整为对应类型标签 | 负责人：值班维护者。期限：同日。完成标准：类别完成重定向。 |
| F7: Preview 功能，暂不支持 | 功能仅存在于 Azure REST API Preview 版本中 | GA 状态判定证据（API 版本号 + 文档链接） | `enhancement` + `upstream` | 负责人：值班维护者。期限：1 个工作日内回复。完成标准：已告知用户功能处于 Preview 阶段。 |
| F8: 功能已支持 | Provider 已在现有 Schema 中支持该功能 | 已支持的属性名 + 文档链接 + 所需最低 Provider 版本 | 可直接关闭 | 负责人：值班维护者。期限：1 个工作日内回复。完成标准：用户已收到使用指引。 |
| F9: 违反完整 CRUD 权限原则 | 请求为部分权限路径做专门优化 | 原则说明 + 变通方案（Azure CLI + `ignore_changes`） | 可直接关闭 | 负责人：值班维护者。期限：1 个工作日内回复。完成标准：已解释设计原则并提供变通方案。 |

## 9. 回复草稿模板（可直接发 Issue）

### 8.1 可实现且向后兼容

Thank you for this feature request! We've completed an initial feasibility analysis and confirmed that this can be implemented.

Summary:
1. Requested feature: <one-line description>
2. Azure API support: The feature is available in the stable API version `<api-version>`
3. Compatibility: This will be introduced as an optional attribute with no impact on existing configurations

We've prepared an implementation plan and will submit a PR after maintainer review. Progress will be tracked in this issue.

### 8.2 需等待 SDK 更新

Thank you for this feature request! We've completed a feasibility analysis.

Summary:
1. Requested feature: <one-line description>
2. Azure API support: Available in stable API version `<api-version>`
3. Blocker: The Go SDK (`hashicorp/go-azure-sdk`) has not yet been updated to include the `<api-version>` structures

We'll track the SDK update and prioritize this feature once the SDK is available.

### 8.3 超出 Provider 职责范围

Thank you for your feedback! After analysis, we've determined that this falls outside the scope of the AzureRM Provider.

Reason: <brief explanation>

Alternative approaches:
- <guidance on how to achieve the goal outside the Provider>

### 8.4 需要补充信息

Thank you for your feature request! To better evaluate feasibility, we need some additional context:

1. Use case: What scenario would this feature enable?
2. Expected schema: What would the Terraform configuration look like? (pseudocode is fine)
3. Current blocker: What is the impact of not having this feature?
4. Version info: Your Terraform and AzureRM Provider versions

We'll continue evaluation once we have this information.

### 8.5 涉及破坏性变更

Thank you for this feature request! We've confirmed this is technically feasible.

However, implementing this feature may require changes to existing attribute behavior, which could affect current configurations. We're evaluating the compatibility impact and will determine the best approach (potentially targeting the next major version).

Progress will be tracked in this issue.

### 8.6 功能处于 Preview 阶段

Thank you for this feature request! We've completed a feasibility analysis.

Summary:
1. Requested feature: <one-line description>
2. Status: This feature is currently in Azure Preview (evidence: only available in API version `<preview-api-version>`, not in stable version `<stable-api-version>`)

The AzureRM Provider generally does not add support for Preview features, as their API contracts and behavior may change before GA. We'll re-evaluate once the feature reaches General Availability.

In the meantime, you can use the [AzAPI Provider](https://registry.terraform.io/providers/Azure/azapi/latest) to access Preview API features directly.

### 8.7 功能已支持

Thank you for your feedback! This feature is already supported in the AzureRM Provider.

You can use it as follows:
- Resource: `<resource_name>`
- Attribute: `<attribute_name>`
- Documentation: [link to Terraform Registry docs]

This has been available since Provider version `<version>`. If you're on an older version, please upgrade.

If this doesn't address your specific need, please provide more details about the exact behavior you're looking for.

## 10. 计划审批门槛（Gate）

在进入代码实现（提 PR）前，Agent 提交的计划至少要满足：

1. 有可行性验证证据（GA 状态确认 + API 属性详情 + SDK 支持检查）。
2. 有明确的 Go 实现方案，包含 Schema 定义、Expand/Flatten 变更和 CRUD 变更。
3. 有兼容性评估，确认新增功能对现有用户的影响。
4. 有测试计划，覆盖新增功能验证和现有测试回归。
5. 有文档更新计划。
6. 若涉及破坏性变更，已提供第 6.2 节全部材料并明确高亮。
