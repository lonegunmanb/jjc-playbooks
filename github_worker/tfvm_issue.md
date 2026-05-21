# 前置：从 Trello 卡片获取 Issue 信息

你正在处理一个 **Legacy Terraform Module** 仓库的 GitHub Issue。

### 第一步：读取卡片 description

Gateway 已经在 CARD CONTEXT 中预填了 `card_id`、`work_type`、`github_repo`、`github_number`、`github_url` 等信息——**优先使用 CARD CONTEXT 的值，不要再去拉一次 Trello**。

如果你确实需要原始的 `name` / `desc` / `firstLine`，调用 gateway 注册的 `trello_card_get` 工具：

```json
{"tool": "trello_card_get", "args": {"card_id": "<card_id>"}}
```

返回 `{id, name, desc, firstLine, idList, idBoard}`。**禁止**自己拼 `https://api.trello.com/1/...` 的 `Invoke-RestMethod` 调用——Trello 凭据由 Go 端持有，工具调用是唯一受支持的访问路径。

### 第二步：从第一行 URL 提取仓库名和 Issue 号

第一行是一个 GitHub URL，格式为：`https://github.com/{owner}/{repo}/issues/{number}`

例如：`https://github.com/Azure/terraform-azurerm-aks/issues/118`
- **仓库名**：`Azure/terraform-azurerm-aks`
- **Issue 号**：`118`

（CARD CONTEXT 已经把 `github_repo` / `github_number` / `github_url` 拆好了，直接用即可。）

### 第三步：读取 GitHub Issue 详情

```powershell
$issueUrl = "https://api.github.com/repos/{owner}/{repo}/issues/{number}"
$headers = @{ "Accept" = "application/vnd.github.v3+json"; "User-Agent" = "copilot-agent" }
if ($env:GITHUB_TOKEN) { $headers["Authorization"] = "token $env:GITHUB_TOKEN" }
$issue = Invoke-RestMethod -Uri $issueUrl -Headers $headers
```

读取到 Issue 后，根据 `$issue.body`、`$issue.title`、`$issue.labels` 等信息，按下面的流程开始分析。

### 第四步：处理正文中的截图/图片附件

扫描 Issue 正文（`$issue.body`），查找图片 URL（通常是 `https://github.com/user-attachments/assets/...` 或其他图片链接，格式为 `![...](url)` 或 `<img src="url" ...>`）。

对于每个找到的图片 URL，**下载图片并使用 `markitdown` 进行 OCR 提取文字内容**：

```powershell
# 下载图片
Invoke-WebRequest -Uri "<image_url>" -OutFile "image_temp.png"
# 使用 markitdown OCR 提取内容
markitdown image_temp.png
```

将 OCR 提取的文字内容作为 Issue 正文的补充信息，纳入后续分析。截图中可能包含关键的错误日志、Terraform plan 输出、配置片段或 Azure Portal 界面信息，这些是分类和根因分析的重要证据。如果 OCR 无法提取有意义的内容（如纯图表或模糊截图），记录该事实并在分析中注明。

---

# Issue 分类与处理流程（面向维护者）

## 1. 目的

本文档定义了 Agent 帮助维护者分类和处理仓库 Issue 的标准流程。

目标：

- 对每个 Issue 进行一致的分类。
- 为维护者提供可直接决策的信息。
- 为每个 Issue 提出一个清晰的下一步行动方案。

## 2. 核心原则：决策必须有据可依

> **⚠️ 强制要求：Agent 提出的每一个观点、判断和建议都必须有对应的外部信源作为证据支撑。禁止基于推测或"常识"做出结论。**

### 2.1 可接受的证据来源

Agent 在分析 Issue、提出建议或做出判断时，**必须**引用以下至少一种外部信源：

| 证据类型 | 说明 | 示例 |
|---|---|---|
| **测试运行结果** | 通过 `terraform plan`、`terraform apply`、`terraform test` 等命令实际运行获得的输出 | 测试报告、错误日志、plan 输出 |
| **AzureRM Provider 源代码** | HashiCorp `terraform-provider-azurerm` 仓库中的源代码 | `github.com/hashicorp/terraform-provider-azurerm/internal/services/...` |
| **资源的官方文档** | Terraform Registry 上对应 resource/data source 的官方文档 | `registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/...` |
| **Terraform 官方文档** | HashiCorp Terraform 核心功能的官方文档 | `developer.hashicorp.com/terraform/docs/...` |
| **Azure REST API 定义** | Azure REST API 规范，用于确认 Azure 资源的实际行为和属性 | `learn.microsoft.com/en-us/rest/api/...` 或 `github.com/Azure/azure-rest-api-specs` |
| **Azure 官方文档** | Microsoft 官方的 Azure 服务文档 | `learn.microsoft.com/en-us/azure/...` |

### 2.2 证据引用格式

Agent 在输出中引用证据时，应遵循以下格式：

- **引用源代码**：给出文件路径和关键代码片段，例如：  
  > 根据 `terraform-provider-azurerm` 源代码 (`internal/services/containers/container_app_resource.go#L245`)，该属性为 Optional。

- **引用文档**：给出文档 URL 和相关段落，例如：  
  > 根据 [AzureRM 文档](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/container_app#ingress)，`ingress` 块中的 `target_port` 为必填字段。

- **引用测试结果**：给出命令和输出摘要，例如：  
  > 执行 `terraform plan` 后确认，移除 `revision_suffix` 不会触发资源重建（plan 输出显示 0 to destroy）。

- **引用 REST API 定义**：给出 API 路径和属性说明，例如：  
  > 根据 Azure REST API 规范 (`Microsoft.App/containerApps 2024-03-01`)，`properties.configuration.ingress.targetPort` 为必需属性（required: true）。

### 2.3 禁止事项

- ❌ **禁止**在没有证据的情况下断言某个属性是必填/选填。
- ❌ **禁止**在没有运行测试的情况下声称"修改不会导致破坏性变更"。
- ❌ **禁止**基于模型训练数据中的"记忆"来回答技术问题——必须实时查阅源代码或文档。
- ❌ **禁止**在建议中使用"应该是"、"通常是"、"一般来说"等模糊表述替代实际验证。
- ❌ **禁止**想当然地假设 Terraform 或 Provider 不支持某种功能。Terraform 和 AzureRM Provider 持续演进，新版本会引入重要的新特性（例如 `ephemeral` 资源从 Terraform 1.10 开始支持，cross-variable reference validation 从 Terraform 1.9 开始支持）。**在断言"某功能不支持"或"某功能做不到"之前，必须先查阅：**
  1. [Terraform 官方文档](https://developer.hashicorp.com/terraform/language)确认当前版本支持的语法和功能。
  2. [Terraform CHANGELOG](https://github.com/hashicorp/terraform/blob/main/CHANGELOG.md) 确认功能引入的版本。
  3. [AzureRM Provider CHANGELOG](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/CHANGELOG.md) 确认 Provider 侧的功能支持情况。
  4. 检查模块的 `terraform` required_version 和 required_providers 约束，确认目标版本范围。
- ❌ **禁止**在 {{kanban.plan.name}} 阶段主动提议关闭 Issue。除非人类在卡片评论中明确命令关闭，否则 Agent 的职责是分析问题、提出方案，而不是建议放弃。即使 Issue 看起来无效、重复或已过时，也应在简报中如实呈现判断依据，由人类维护者决定是否关闭。

### 2.4 无法获取证据时的处理

如果 Agent 无法获取足够的外部证据来支持某个判断：

1. **明确标注**该判断为"未验证"。
2. **列出**已尝试但未成功的验证方法。
3. **建议**维护者需要自行验证的具体步骤。
4. **不要**伪造或编造证据来源。

### 2.5 GitHub 交互方式（强制）

> **⚠️ 所有与 GitHub API 的交互必须通过 `gh` 命令行工具或环境变量 `GITHUB_TOKEN` 完成，不得使用其他认证方式。**

- **优先使用 `gh` CLI**：查询 PR、Issue、Review Comments、approve deployment、创建评论等操作统一使用 `gh api`、`gh pr`、`gh issue` 等子命令。
- **直接调用 REST API 时**：使用环境变量 `$env:GITHUB_TOKEN` 作为认证令牌，通过 `Authorization: token $env:GITHUB_TOKEN` 请求头传递。

## 3. Issue 分类

Agent 必须将每个 Issue 归入一个主要类别。

| 类别 | 定义 | 典型信号 | 参见
|---|---|---|---|
| Bug | 预期行为未被满足且可以复现。 | "不工作"、回归、错误输出 | `{{tfvm_issue_bug.md}}` |
| 功能请求 | 请求新功能或增强现有功能。 | "请支持"、"添加选项" | `{{tfvm_issue_feature_request.md}}` |
| 破坏性变更 / 行为变更 | 请求的变更可能影响现有用户或显著改变行为。 | 需要迁移、兼容性风险 | |
| 文档 | 文档/示例/README 有误、缺失或不清晰。 | "文档不清楚"、示例不匹配 | |
| 问题 / 反馈 | 使用疑问、设计澄清或一般性反馈，没有直接的缺陷证据。 | "我应该怎么"、"这是预期行为吗" | `{{tfvm_issue_question.md}}` |
| 安全漏洞 | 安全漏洞、敏感信息泄露或策略风险行为。 | 密钥泄露、权限提升、安全问题 | `{{tfvm_issue_security.md}}` |

### 3.1 Agent 分类输出（必须）

对每个 Issue，Agent 应提供：

1. 建议的类别。
2. 置信度：高、中、低。
3. 来自 Issue 内容的证据摘录。
4. 如果认为分类不当，说明建议的目标类别及理由。

## 4. Agent 评估清单

在维护者做出决策之前，Agent 应评估：

1. 这是否与已有的 open/closed Issue 重复？
2. Issue 类型是否被正确选择？
3. 是否有足够的信息供维护者决策？
4. 是否存在需要私下处理的安全隐患？
5. 是否在等待作者补充信息？
6. 是否应该转换为 Bug/功能请求/文档类 Issue？

## 5. Terraform 测试常见问题

### 5.1 Azure Feature 未注册

在运行 `terraform plan` 或 `terraform apply` 测试时，有时会遇到类似以下错误：

> The subscription is not registered to use namespace 'Microsoft.App'

或：

> Feature 'Microsoft.XXX' is not registered for subscription ...

遇到这类问题，说明当前订阅尚未注册该 Azure Resource Provider。可以在 Terraform 代码中添加如下 `azapi` 资源块来自动注册：

```hcl
data "azurerm_client_config" "current" {}

resource "azapi_resource_action" "register_microsoft_app" {
  action      = "/providers/Microsoft.App/register"
  method      = "POST"
  resource_id = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  type        = "Microsoft.Resources/subscriptions@2021-04-01"
}
```

> **说明**：将 `Microsoft.App` 替换为实际需要注册的 Resource Provider 名称（如 `Microsoft.ContainerService`、`Microsoft.Network` 等），并相应修改资源名称以保持语义清晰。

### 5.2 Quota 不足

运行 `terraform apply` 时，如果遇到类似以下错误：

> QuotaExceeded: Operation could not be completed as it results in exceeding approved ...

或：

> The requested VM size is not available in the current region ...

说明当前 location 的资源配额不足。可以尝试更换 `location` 参数为其他区域。优先选择以下区域：

* `eastus`
* `eastus2`
* `westeurope`
* `westus`
* `japaneast`
* `japanwest`

通常在模块的 `variables.tf` 或测试用例中找到 `location` 变量并修改即可。