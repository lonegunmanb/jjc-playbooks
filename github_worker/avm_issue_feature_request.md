# 功能请求类 Issue 处理指南（面向维护者）

## 1. 适用范围

当 Issue 被归类为 `功能请求`（请求新功能或增强现有功能）时，Agent 必须使用本指南协助维护者完成预处理。

本阶段目标是：

1. 判断请求的功能是否在模块职责范围内且技术上可行。
2. 评估实现该功能对现有用户的兼容性影响。
3. 产出高质量的可行性分析与实现计划，供维护者决策。

## 2. 处理原则（强制）

1. 先验证可行性，后承诺实现
   - 在确认 Provider 支持、API 支持和模块架构兼容之前，不做实现承诺。
2. 向后兼容优先
   - 新功能必须以可选方式引入，不改变现有输入的语义或默认行为。
   - 新增变量必须有合理默认值，确保现有用户升级时 terraform plan 无 drift。
3. 范围边界清晰
   - 明确区分"模块应做"和"用户应在模块外自行处理"的边界。
   - 不把 Provider 层面的限制或 Azure 平台层面的限制误判为模块缺失功能。
4. 计划先行，审批后执行
   - Agent 在该阶段主要产出可行性分析与实现计划；涉及代码实现、合并、发布的动作需维护者审批。
5. 安全优先
   - 若功能请求涉及安全敏感操作（如密钥管理、权限变更），需额外评估安全影响。
6. 仅支持 GA（正式发布）功能
   - 我们不支持 Azure Preview 功能。如果请求的功能仅存在于 Azure REST API 的 Preview 版本中，而 GA（正式）版本中不包含，则该功能属于 Preview，不予实现。
   - 判定方法：查阅 Azure REST API 规范（`github.com/Azure/azure-rest-api-specs`），确认该功能对应的属性/操作是否存在于 **stable** 目录下的 API 版本中（如 `2024-03-01`），而非仅存在于 **preview** 目录下的版本（如 `2024-03-01-preview`）。或者，如果该功能的 Azure 官方文档页面明确标注为 Preview / Public Preview，也视为未 GA。
   - 如果文档中没有提到 Preview 标记，且功能存在于 stable API 版本中，则视为 GA。
7. 遵循模块现有 Provider 模式
   - AVM 模块在 Provider 使用上存在三种模式，新功能实现必须与模块现有模式保持一致：
     - **AzureRM 模式**：模块主要使用 `azurerm` provider 资源，变量按 `azurerm` resource schema 暴露。
     - **AzAPI 模式**：模块主要使用 `azapi` provider 资源，变量按 Azure REST API schema 暴露。
     - **AzAPI-模拟-AzureRM 模式**：模块使用 `azapi` provider 作为底层实现，但变量的命名、结构和语义刻意模拟 `azurerm` resource schema，使用户感知上与 `azurerm` 模块一致。
   - Agent 必须在分析阶段识别模块属于哪种模式，并在实现方案中严格遵循。混用模式会导致用户困惑和维护困难。

## 3. 快速处理流程

对每个功能请求 Issue，按顺序执行：

1. 需求归一化
   - 提炼一句话需求定义：用户想要什么能力、解决什么场景、当前的阻塞点是什么。
2. 属性定位与跨体系映射
   - 在进行任何代码检查之前，**先明确用户请求的功能在不同技术体系中的对应关系**。用户可能使用 Azure Portal 功能名、`azurerm` 属性名、REST API 属性名，甚至 Azure 文档中的产品术语来描述需求，Agent 必须先将其翻译为可搜索的具体标识。
   - **① 确定用户使用的术语体系**：
     - 用户使用的是 `azurerm` 资源属性名（如 `target_port`、`sku_name`）？
     - 用户使用的是 Azure REST API 属性名（如 `targetPort`、`sku.name`）？
     - 用户使用的是 Azure Portal / 文档中的功能名（如 "Ingress"、"Zone Redundancy"）？
     - 用户使用的是模糊描述（如 "支持自定义域名"、"配置多个副本"）？
   - **② 建立跨体系映射表**：

     | 维度 | 内容 | 示例 |
     |------|------|------|
     | 用户原文 | 用户在 Issue 中使用的原始表述 | "support `revision_suffix`" |
     | Azure REST API 资源 | 对应的 REST API 资源类型和 API 版本 | `Microsoft.App/containerApps` (stable: `2024-03-01`) |
     | REST API 属性路径 | 属性在 REST API JSON body 中的完整路径 | `properties.template.revisionSuffix` |
     | AzureRM 资源 | 对应的 `azurerm` resource 类型 | `azurerm_container_app` |
     | AzureRM 属性名 | 属性在 `azurerm` resource schema 中的名称 | `template.0.revision_suffix` |
     | 功能名/产品术语 | Azure 文档/Portal 中的叫法 | "Revision suffix" |

   - **③ 映射方法**：
     - 若用户给出的是 `azurerm` 属性名 → 查阅 `terraform-provider-azurerm` 源代码或 Terraform Registry 文档，找到对应的 REST API 属性路径。
     - 若用户给出的是 REST API 属性名 → 查阅 `terraform-provider-azurerm` 源代码，确认 `azurerm` 侧是否有对应属性及其命名。
     - 若用户给出的是功能名或模糊描述 → 先查阅 Azure 官方文档确定 REST API 资源和属性路径，再反查 `azurerm` 侧的映射。
     - 若 `azurerm` Provider 尚未支持该属性，映射表中 AzureRM 属性名填写"不存在"。
   - **④ 记录映射结论**：将填好的映射表纳入后续所有步骤的工作上下文。后续搜索模块代码时，应同时使用映射表中的 `azurerm` 属性名和 REST API 属性路径进行搜索，不遗漏任何一种实现方式。
3. 现有支持检查
   - 在进入可行性分析之前，先检查模块是否**已经支持**请求的功能。该检查必须基于步骤 2 建立的映射表，同时使用 `azurerm` 属性名和 REST API 属性名进行搜索。
   - **检查层次**（按从浅到深的顺序）：
     - ① **变量层**：检查 `variables.tf` 中是否已存在对应的变量，用户可以直接配置。
     - ② **资源层（硬编码/默认值）**：检查主资源文件（`main.tf` 等）中，该属性是否已在资源块中以硬编码值或 `locals` 计算值的方式设置。若是，功能已实现但未暴露为用户可配置的变量。
     - ③ **资源层（dynamic 块）**：检查是否通过 `dynamic` 块条件性地包含了该属性，可能受其他变量的间接控制。
     - ④ **补丁层（azapi 补丁）**：在 AzureRM 模式的模块中，检查是否通过 `azapi_update_resource` 或 `azapi_resource_action` 作为补丁设置了该属性。
     - ⑤ **JSON body 层（AzAPI 模块）**：在 AzAPI 模式或 AzAPI-模拟-AzureRM 模式的模块中，检查 `azapi_resource` 的 `body` 参数（JSON/HCL 对象）中是否已包含该 REST API 属性。
   - **检查辅助信源**：
     - 检查 `examples/`：是否已有示例演示了该功能的用法。
     - 检查 `README.md`：是否已有文档说明该功能。
     - 检查最近的 Release/CHANGELOG：是否在近期版本中已实现但用户使用的是旧版本。
   - **判定结论**：

     | 发现 | 结论 | 路由 |
     |------|------|------|
     | 变量已暴露，用户可直接配置 | 完全支持 | → F8（已支持） |
     | 属性已在资源块中硬编码/设置，但未暴露为变量 | 已实现但不可配置 | → 继续评估：是否应暴露为变量（可能是功能增强请求） |
     | 属性存在于资源块中但缺少用户请求的某些子属性 | 部分支持 | → 记录已支持部分，继续评估缺失部分 |
     | 模块中完全未涉及该属性 | 未支持 | → 继续后续流程 |

4. 范围判定
   - 判断该功能属于模块职责还是应由用户在模块外处理。
   - 判断是否为 Provider/Azure 平台层面的限制（不属于模块可解决的范畴）。
5. 模块 Provider 模式识别
   - 分析模块的 `main.tf`（或主资源文件），确认模块主要使用 `azurerm` 还是 `azapi` 资源。
   - 分析 `variables.tf` 中暴露的变量结构，判断变量 schema 是按 `azurerm` resource schema 还是 Azure REST API schema 设计的。
   - 确定模块属于三种模式中的哪一种：AzureRM 模式 / AzAPI 模式 / AzAPI-模拟-AzureRM 模式。
   - 该结论将约束后续实现方案的 Provider 选择和变量设计。
6. 可行性验证
   - 检查 AzureRM Provider 是否已支持对应的资源属性或数据源。
   - 检查 Azure REST API 是否已暴露对应的能力。
   - **检查功能的 GA 状态**：确认该功能在 Azure REST API stable 版本中是否存在（非仅 preview 版本）。若仅在 preview API 中存在，直接路由到 F7（Preview 功能），不继续后续评估。
   - 检查 Terraform 语言层面是否有语法或功能限制。
   - 检查模块现有架构是否能自然扩展以容纳此功能。
7. 重复与关联检查
   - 检查是否与 open/closed Issue 或已有 PR 重复。
   - 检查是否有相关的上游 Issue（Provider 或 Terraform Core）。
8. 兼容性影响评估
   - 评估新增变量/输出对现有用户的影响。
   - 确认新功能是否可以以纯可选方式引入（默认值 = 当前行为）。
   - 若不可避免改变现有行为，按破坏性变更流程处理。
9. 实现方案设计
   - 给出一个唯一的默认实现方案。
   - 明确拟新增/修改的变量、资源块、输出。
   - **新增变量/资源块必须遵循步骤 5 识别的 Provider 模式**（详见第 5.5 节）。
   - 明确测试策略。
10. 原型验证（建议）
    - 在本地环境中基于 examples/ 构造原型配置，验证核心路径可行。
    - 验证新增变量 + 默认值不影响已有示例的 plan 结果。
    - 验证后执行环境清理。
11. 维护者简报输出
    - 使用第 4 节模板输出可直接决策的信息。

## 4. Agent 输出模板（给维护者）

对每个功能请求 Issue，Agent 输出必须包含：

1. 分类摘要
   - 建议类别：功能请求
   - 置信度：高/中/低
   - 证据摘录：来自 Issue 的关键原文
2. 需求分析
   - 一句话需求定义
   - 用户场景描述
   - 当前阻塞点
   - 属性映射表：（用户原文 → REST API 资源/属性路径 → AzureRM 资源/属性名）
3. 可行性结论
   - Azure 功能 GA 状态：GA/Preview（证据：stable API 版本号或文档链接）
   - 模块 Provider 模式：AzureRM / AzAPI / AzAPI-模拟-AzureRM（证据：主资源文件中的 resource 类型 + 变量 schema 风格）
   - Provider 支持状态：已支持/部分支持/未支持
   - API 支持状态：已暴露/未暴露
   - Terraform 语言限制：无/有（说明具体限制）
   - 模块架构兼容性：可自然扩展/需重构/不兼容
   - 证据：引用源代码、文档或 API 定义
4. 兼容性评估
   - 是否可以纯可选方式引入：是/否
   - 对现有用户 terraform plan 的影响：无 drift/可能 drift（说明原因）
   - 是否涉及破坏性变更：否/是（若是，标注 `[BREAKING-CHANGE]`）
5. 实现计划
   - 可行性状态：可实现/需等待上游/不可实现
   - （可实现时）默认方案（必须唯一）：
     - 实现目标：
     - 使用的 Provider 模式：（必须与模块现有模式一致）
     - 拟新增/修改的变量：（名称、类型、默认值、说明；schema 风格须与现有变量一致）
     - 拟新增/修改的资源块：（resource 类型须与模块现有模式一致）
     - 拟新增/修改的输出：
     - 测试计划：新增测试用例、现有示例回归验证
     - 风险与缓解：
   - （需等待上游时）：
     - 阻塞原因：Provider Issue 链接 / Terraform Core Issue 链接
     - 临时替代方案（如有）
   - （不可实现时）：
     - 技术约束说明
     - 替代方案建议（如有）
6. 维护者行动建议（审批项）
   - 建议决策：批准实现/等待上游/拒绝并说明/要求补充信息
   - 负责人：
   - 时间框：
   - 完成标准：
7. 可直接发送的回复草稿
   - 面向 Issue 提交者的简洁回复，说明当前结论与下一步。

## 5. 可行性验证指南（Agent 执行）

### 5.1 Provider 支持检查

1. 在 `terraform-provider-azurerm` 源代码中查找对应资源的 Schema 定义。
2. 确认请求的属性是否已存在于 Provider 的 Schema 中。
3. 若不存在，检查 Provider 的 open Issue 和 PR，确认是否有计划支持。
4. 记录 Provider 对应资源文件路径和关键 Schema 定义。

### 5.2 Azure 功能 GA 状态检查（强制，必须先于其他检查）

> **⚠️ 此检查为前置阻断检查。若功能为 Preview，直接路由到 F7，不继续后续可行性验证。**

Agent 必须按以下步骤判定功能的 GA 状态：

1. **检查 Azure REST API 规范**（`github.com/Azure/azure-rest-api-specs`）：
   - 找到对应服务的 API 定义目录（如 `specification/app/resource-manager/Microsoft.App/`）。
   - 检查该功能对应的属性/操作是否存在于 `stable` 目录下的 API 版本中。
   - 若该属性/操作 **仅** 存在于 `preview` 目录下的 API 版本中（如 `2024-03-01-preview`），而 `stable` 目录下的最新版本中不包含 → **判定为 Preview**。
   - 若该属性/操作存在于 `stable` 目录下的 API 版本中（如 `2024-03-01`）→ **判定为 GA**。

2. **检查 Azure 官方文档**（`learn.microsoft.com/en-us/azure/`）：
   - 查找该功能的文档页面。
   - 若页面标题或正文中包含 `Preview`、`Public Preview`、`(preview)` 等标记 → **判定为 Preview**。
   - 若无任何 Preview 标记 → 视为 GA（需与步骤 1 结论一致）。

3. **记录判定结论**：
   - GA 状态：GA / Preview
   - 证据：stable API 版本号、preview API 版本号、文档链接
   - 若两个来源结论矛盾，以 Azure REST API 规范（`azure-rest-api-specs`）为准

### 5.3 Azure API 支持检查

1. 查阅 Azure REST API 规范，确认请求的能力是否在 API 层面已暴露。
2. 若 API 已支持但 Provider 未支持，这是一个 Provider 层面的缺口，模块无法直接解决（可考虑 `azapi` 作为临时方案）。
3. 若 API 也未支持，则属于 Azure 平台层面的限制。

### 5.4 模块架构兼容性检查

1. 检查模块现有的变量结构（`variables.tf`），确认新功能是否可以自然地融入现有变量层级。
2. 检查现有的 `dynamic` 块和条件逻辑，确认新增分支是否会导致过度复杂化。
3. 检查 `outputs.tf`，确认是否需要暴露新的输出值。
4. 检查 `locals.tf`，确认数据转换逻辑的影响。

### 5.5 模块 Provider 模式识别（强制）

Agent 必须在分析阶段识别模块的 Provider 使用模式，并在实现方案中严格遵循。

**识别步骤：**

1. **扫描主资源文件**（`main.tf` 或拆分后的资源文件）：
   - 统计 `azurerm_*` 和 `azapi_resource` / `azapi_update_resource` / `azapi_resource_action` 资源的数量和比例。
   - 如果核心资源（非辅助性的 role assignment、diagnostic setting 等）主要使用 `azurerm_*` → 初步判定为 **AzureRM 模式**。
   - 如果核心资源主要使用 `azapi_resource` → 初步判定为 **AzAPI 模式**或 **AzAPI-模拟-AzureRM 模式**，需进一步检查变量。

2. **分析变量 schema 风格**（`variables.tf`）：
   - **AzureRM schema 风格**：变量名和结构与 `azurerm` resource 的参数一致（如使用 `sku_name` 而非 `sku.name`，使用 `identity` 块而非 `properties.identity`，使用下划线命名如 `container_port` 而非驼峰命名 `containerPort`）。
   - **AzAPI / REST API schema 风格**：变量结构直接反映 Azure REST API 的 JSON 结构（如使用嵌套 `properties` 对象，驼峰命名如 `targetPort`，结构与 API spec 一一对应）。

3. **判定模式**：

| 核心资源类型 | 变量 schema 风格 | 模式 |
|---|---|---|
| `azurerm_*` | AzureRM 风格 | **AzureRM 模式** |
| `azapi_resource` | REST API 风格 | **AzAPI 模式** |
| `azapi_resource` | AzureRM 风格 | **AzAPI-模拟-AzureRM 模式** |

4. **记录结论**，包括：
   - 判定的模式
   - 证据：主资源 resource 类型列表、变量命名/结构示例
   - 对新功能实现的约束：应使用什么 resource 类型、变量应按什么风格设计

**实现约束：**

- **AzureRM 模式**的模块：新功能应使用 `azurerm_*` 资源实现，变量按 `azurerm` resource schema 设计。若 `azurerm` 尚不支持所需属性，可考虑用 `azapi_update_resource` 或 `azapi_resource_action` 作为补丁，但变量层面仍应保持 AzureRM 风格。
- **AzAPI 模式**的模块：新功能应使用 `azapi_resource` / `azapi_update_resource` / `azapi_resource_action` 实现，变量按 REST API schema 设计。
- **AzAPI-模拟-AzureRM 模式**的模块：新功能应使用 `azapi_resource` / `azapi_update_resource` / `azapi_resource_action` 实现，但变量命名和结构必须模拟 `azurerm` 对应资源的参数风格（下划线命名、扁平化结构等），使用户无需了解底层 REST API 结构。

> **⚠️ AzAPI-模拟-AzureRM 模式的特殊要求：** 当模块被识别为此模式时，Agent 必须先阅读参考文档并遵循其规范。详见 {{avm_issue.md}} §2.5「AzAPI-模拟-AzureRM 模式模块的参考文档」。

### 5.6 原型验证（建议）

1. 基于最接近的现有 example 创建原型配置，加入新功能的变量设置。
2. 执行 `terraform plan`，验证：
   - 新配置可以正常 plan 通过。
   - 使用默认值的原有配置 plan 无 drift。
3. 如条件允许，执行 `terraform apply` 验证端到端可行性。
4. 验证完成后执行环境清理（优先 `terraform destroy`）。

## 6. 向后兼容性约束（核心）

### 6.1 默认验收标准

新增功能默认必须满足以下标准（除非维护者批准例外）：

1. 纯可选引入
   - 所有新增变量必须有默认值，且默认值等价于"功能未启用"（即现有行为不变）。
2. 计划稳定
   - 对既有稳定配置执行 `terraform plan`，应无 drift。
3. 类型安全
   - 新增变量的类型约束应尽量精确（使用 `object({...})` 而非 `any`），并附带 validation 块。
4. 文档同步
   - 新增变量/输出必须有清晰的 description。
   - README 和 examples 需同步更新。

### 6.2 若功能请求隐含破坏性变更（必须高亮）

若请求的功能在技术上无法以纯可选方式引入，Agent 必须在维护者简报中使用醒目标记：

[BREAKING-CHANGE][REQUIRES-MAINTAINER-APPROVAL]

并且必须额外提供：

1. 为什么无法以可选方式引入（技术约束说明）。
2. 受影响的现有变量/行为。
3. 迁移指南草案。
4. 版本策略建议（是否需要 major 版本变更）。

## 7. 功能请求路由表

| 路由 | 触发条件 | Agent 提供内容 | 建议标签/状态 | 默认行动方案 |
|---|---|---|---|---|
| F1: 可实现且向后兼容 | Provider 已支持，模块可自然扩展，无破坏性影响 | 可行性证据 + 实现计划 + 原型验证结果（如有） | 保留 `Type: Feature Request`，移除 `Needs: Triage` | 负责人：模块维护者。期限：3 个工作日内批准实现计划。完成标准：实现计划获批并进入开发。 |
| F2: 可实现但涉及破坏性变更 | 功能可实现但无法以纯可选方式引入 | 破坏性影响评估 + 迁移草案 + 版本建议 | `Type: Feature Request` + `Needs: Attention` | 负责人：模块维护者/版本负责人。期限：3 个工作日内完成破坏性审查。完成标准：明确是否接受破坏性方案或选择替代路径。 |
| F3: 需等待上游 | Provider 或 Terraform Core 尚未支持所需能力 | 上游 Issue 链接 + 临时替代方案（如有） + `azapi` 可行性 | `Type: Feature Request` + `Status: Blocked` | 负责人：值班维护者。期限：1 个工作日内回复用户。完成标准：用户已知晓阻塞原因和替代方案。 |
| F4: 超出模块职责范围 | 请求的功能不属于模块应承担的职责 | 范围判定理由 + 替代实现指引 | 保留 `Type: Feature Request` | 负责人：值班维护者。期限：1 个工作日内回复。完成标准：已向用户解释范围边界并提供替代方案。 |
| F5: 证据不足待补充 | 缺少关键上下文（使用场景、期望行为、版本信息等） | 缺失信息清单 + 追问模板 | `Needs: Author Feedback` | 负责人：值班维护者。期限：1 个工作日内追问。完成标准：作者补齐信息或进入超时策略。 |
| F6: 实为其他类型 | 分析后发现不属于功能请求（实际是 Bug/文档/安全等） | 重分类证据与目标类别理由 | 调整为对应类型标签 | 负责人：值班维护者。期限：同日。完成标准：类别完成重定向并给出解释回复。 |
| F7: Preview 功能，不予支持 | 功能仅存在于 Azure REST API Preview 版本中，stable 版本不包含 | GA 状态判定证据（API 版本号 + 文档链接）+ Preview → GA 的追踪方式 | 保留 `Type: Feature Request` + `Status: Blocked` | 负责人：值班维护者。期限：1 个工作日内回复用户。完成标准：已告知用户功能处于 Preview 阶段，待 GA 后重新评估。 |
| F8: 功能已支持 | 模块已通过现有变量/配置支持请求的功能，用户可能不知情 | 已支持的变量名/路径 + 用法示例（基于 examples/）+ 所需最低模块版本 | 保留 `Type: Question/Feedback`，移除 `Needs: Triage` | 负责人：值班维护者。期限：1 个工作日内回复。完成标准：用户已收到使用指引，Issue 关闭。 |

## 8. 回复草稿模板（可直接发 Issue）

### 8.1 可实现且向后兼容

感谢您的功能请求！我们已完成初步可行性分析，确认该功能可以实现。

当前结论：
1. 请求的功能：<一句话描述>
2. Provider 支持：<已支持/通过 azapi 可实现>
3. 兼容性：该功能将以可选方式引入，不会影响现有用户的配置

我们已制定实现计划，待维护者审批后将进入开发阶段。进展会在本 Issue 中同步。

### 8.2 需等待上游支持

感谢您的功能请求！我们已完成可行性分析。

当前结论：
1. 请求的功能：<一句话描述>
2. 阻塞原因：AzureRM Provider 尚未支持 `<属性/资源>`（关联 Issue：<链接>）
3. 临时替代方案：<如有，简述；如无，说明暂无可用方案>

我们将持续关注上游进展，Provider 支持后会优先评估在模块中集成。

### 8.3 超出模块职责范围

感谢您的反馈！经过分析，我们认为该功能目前不在本模块的职责范围内。

原因：<简述范围边界判断>

替代方案建议：
<提供用户可在模块外自行实现的具体指引，附代码示例>

如果社区有较多类似需求，我们会重新评估是否将其纳入模块范围。

### 8.4 需要补充信息

感谢您的功能请求！为了更好地评估可行性，我们需要一些补充信息：

1. 使用场景：您计划在什么场景下使用该功能？
2. 期望行为：该功能启用后，您期望的 Terraform 配置是什么样的？（可提供伪代码）
3. 当前阻塞：目前缺少该功能对您的影响是什么？是否有临时解决方案？
4. 版本信息：您使用的 Terraform 版本、Provider 版本和模块版本

收到后我们会继续评估并给出明确结论。

### 8.5 涉及破坏性变更

感谢您的功能请求！我们已完成初步可行性分析，确认该功能在技术上可以实现。

需要注意的是，该功能的引入可能涉及对现有行为的变更，存在对现有用户配置造成影响的风险。我们正在向维护者提交兼容性评估与迁移方案，待审批后再确定最终实现路径。

进展会在本 Issue 中同步。

### 8.6 功能处于 Preview 阶段

感谢您的功能请求！我们已完成可行性分析。

当前结论：
1. 请求的功能：<一句话描述>
2. 状态：该功能目前仍处于 Azure Preview 阶段（证据：仅存在于 API 版本 `<preview-api-version>` 中，stable API `<stable-api-version>` 中不包含）

根据本模块的策略，我们仅支持 GA（正式发布）的 Azure 功能。Preview 功能的 API 和行为可能在正式发布时发生变化，提前集成会增加维护负担和破坏性变更风险。

该功能 GA 后，我们会优先评估集成。您也可以在功能正式发布后重新提交请求，或关注相关的 Azure 更新公告。

如需在 Preview 阶段使用该功能，建议使用 `azapi` provider 直接调用 Preview API。

### 8.7 功能已在模块中支持

感谢您的反馈！经过检查，该功能已在本模块中支持。

您可以通过以下方式使用：
- 变量：`<变量名>`
- 用法示例：

```hcl
<基于 examples/ 的具体配置片段>
```

该功能自模块版本 `<版本号>` 起可用。如果您使用的是较早版本，请升级到最新版本。

如果上述方案未能满足您的需求，或者您需要的是该功能的某个特定子能力，请补充说明具体的使用场景和期望行为，我们会进一步评估。

## 9. 计划审批门槛（Gate）

在进入代码实现前，Agent 提交的计划至少要满足：

1. 有可行性验证证据（GA 状态确认 + Provider 支持检查 + API 支持检查），且功能已确认为 GA。
2. 有明确的实现方案，包含变量定义、资源修改和输出变更。
3. 有兼容性评估，确认新增功能对现有用户的影响。
4. 有测试计划，覆盖新功能验证和现有示例回归。
5. 若涉及破坏性变更，已提供第 6.2 节全部材料并明确高亮。

## 10. 维护者 KPI（可选但推荐）

按月跟踪：

1. 功能请求 Issue 首次响应时长。
2. 可在 5 个工作日内完成可行性结论的比例。
3. 功能请求转化为实际实现的比例。
4. 功能实现后引入的回归 Issue 数量。
5. 因破坏性影响导致的升级阻塞 Issue 数量。
