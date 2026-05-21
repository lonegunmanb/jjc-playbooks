# AzureRM Provider 问题/反馈类 Issue 处理指南

## 1. 适用范围

本指南定义了当 Issue 被归类为「问题 / 反馈」时，Agent 应如何协助维护者处理。

Provider 的问题/反馈类 Issue 通常包括：

- 对资源属性行为的疑问（"这个属性是做什么的？"、"这是预期行为吗？"）
- 对 `terraform plan` diff 的困惑（"为什么每次都显示变更？"）
- 对 Provider 配置或认证的疑问（"如何配置 Managed Identity 认证？"）
- 对版本升级影响的疑问（"升级到 4.x 需要改什么？"）
- 设计澄清或一般性反馈

## 2. 简要处理流程

对每个问题/反馈类 Issue，按顺序执行以下步骤：

1. 归纳问题
   用一句话总结维护者需要关注的核心问题。
2. 收集决策上下文
   - 搜索已有 Issue/Discussion，看是否已有类似讨论。
   - 检查 Terraform Registry 文档，看是否已有答案。
   - 检查 Provider 源代码（Schema 定义），确认实际行为。
   - 检查 Provider CHANGELOG，确认是否为已知变更。   
3. 确定路由
   从 [路由表](#6-问题反馈路由表) 中的 Q1 到 Q6 选择一个路由。
4. 构建维护者简报
   提供包含风险和工作量的紧凑决策报告。
5. 制定下一步行动方案
   明确负责人、时间窗口和完成标志。

### 2.1 特殊路径："如何实现 xxx"

如果提交者在询问实现指导（例如："如何使用 azurerm_xxx 实现某功能"），使用以下默认行动路径：

1. 构建最小可运行配置
   基于 Terraform Registry 文档和 Provider 源代码中的测试用例，构造最小化 HCL 配置。
2. 验证配置可用性
   参考测试文件（`*_test.go`）中的测试配置模板，确认参数名、类型和嵌套结构。
3. 记录约束
   记录 Provider 版本要求、Azure API 限制、前提假设和必要的前置条件。
4. 回复可执行的指导
   将验证过的配置方案、关键参数和预期结果回复到 Issue 中。

当用户意图是面向实现的时候，应优先采用此路径，而非纯概念性的回答。

## 3. 决策报告模板（Agent → 维护者）

对每个问题/反馈类 Issue，Agent 的输出应包含：

1. 分类摘要
   - 建议类别：问题/反馈
   - 置信度：高/中/低
   - 受影响资源：`azurerm_xxx`（如适用）
   - 理由：
2. 维护者决策信息
   - 用户意图：
   - 相关文档/源代码证据：
   - 重复检查：
   - 不处理的风险：
   - 工作量估计：低/中/高
3. 建议路由
   - 路由 ID：Q1/Q2/Q3/Q4/Q5/Q6
   - 选择理由：
4. 可直接发送的维护者回复草稿
5. **代码示例（如涉及 HCL 配置，必须提供）**
   - 如果建议用户修改配置，**必须**在回复草稿中包含具体的 HCL 代码示例。
   - 代码示例应基于 Provider 测试文件（`*_test.go`）中的测试配置模板或 Terraform Registry 文档改写，确保参数名、类型、嵌套结构真实可用。
   - 不得凭空编造属性名或虚构配置。
6. 下一步行动方案（必须）
   - 负责人：
   - 期限：
   - 完成标准：

## 4. 行动方案规则

Agent 必须指定恰好一个默认行动方案，不能给出多个模糊选项。

方案格式：

- 行动：一个明确的步骤。
- 负责人：具体角色。
- 时间框：以工作日为单位的目标。
- 完成标准：客观的完成条件。

示例：

- 行动：回复答案并关闭 Issue。
- 负责人：值班维护者。
- 时间框：1 个工作日。
- 完成标准：维护者已回复，标签已更新，Issue 已关闭。

## 5. 回复草稿模板

### 5.1 直接回答（Q1）

Thank you for your question!

<直接回答问题，引用 Terraform Registry 文档或 Provider 源代码作为依据>

Reference:
- [Terraform Registry documentation](<link>)

If this answers your question, we'll close this issue. Feel free to reopen if you have further questions.

### 5.2 回答并建议改善文档（Q2）

Thank you for your question!

<回答问题>

We've noted that this area could benefit from improved documentation. We'll track a documentation update to make this clearer for future users.

### 5.3 需要补充信息（Q6）

Thank you for reaching out! To help you effectively, we need some additional information:

1. Terraform version (`terraform version`)
2. AzureRM Provider version
3. The relevant Terraform configuration (redacted of sensitive values)
4. The specific behavior or output you're seeing
5. What you expected to happen instead

## 6. 问题/反馈路由表

使用下表进行路由和维护者支持。

| 路由 | 触发条件 | Agent 提供给维护者的信息 | 建议的标签/状态 | 行动方案 |
|---|---|---|---|---|
| Q1: 回答并关闭 | 问题明确，文档/源代码中已有答案，无需产品变更。 | 1) 直接回复草稿，2) 文档/源代码证据链接，3) 关闭理由 | 移除 `needs-triage` | 负责人：值班维护者。期限：1 个工作日内。完成标准：回复已发布且 Issue 已关闭。 |
| Q2: 回答并补充文档 | 相同问题可能重复出现，或文档虽存在但不够清晰。 | 1) 回复草稿，2) 具体的文档缺口，3) 建议的文档修补范围 | 添加 `documentation` | 负责人：维护者 + 文档贡献者。期限：1 天内回复，3 天内提交文档 PR。完成标准：文档更新已合并。 |
| Q3: 转为 Bug | 分析过程中发现可复现的错误行为。 | 1) 最小复现摘要，2) 预期行为 vs 实际行为，3) 影响说明 | 重新标记为 `bug` | 负责人：资源维护者。期限：1 天内完成分类决策。完成标准：Bug 已确认并进入 Bug 处理流程。 |
| Q4: 转为功能请求 | 用户请求尚不支持的功能，且有明确价值。 | 1) 功能请求摘要，2) API 支持状态检查，3) 兼容性初评 | 重新标记为 `enhancement` | 负责人：资源维护者。期限：3 天内完成功能分类。完成标准：功能请求已确认并进入功能请求处理流程。 |
| Q5: 安全升级 | 出现任何漏洞、密钥泄露或安全风险的迹象。 | 1) 风险摘要（不含敏感信息），2) 即时遏制建议，3) 安全报告指引 | 重新标记为 `security` | 负责人：安全响应人员。期限：立即确认。完成标准：已转入安全处理通道。 |
| Q6: 需要作者反馈 | 因提交者缺少上下文信息，决策被阻塞。 | 1) 缺少的具体字段清单，2) 回复模板，3) 超时建议 | 添加 `waiting-response` | 负责人：值班维护者。期限：1 天内追问。完成标准：作者已回复或进入过期策略。 |
