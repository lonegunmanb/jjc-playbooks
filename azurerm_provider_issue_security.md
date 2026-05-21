# AzureRM Provider 安全漏洞类 Issue 处理指南

## 1. 适用范围

当 Issue 被归类为安全漏洞、敏感信息泄露或潜在可被利用的安全风险时，Agent 必须使用本指南协助维护者处理。

Provider 特有的安全风险场景：

- **State 中的明文密码**：敏感属性未标记 `Sensitive: true`，导致值以明文存储在 state 文件中。
- **Plan/Apply 输出泄露**：敏感值在 `terraform plan` 或 `terraform apply` 输出中未被遮蔽。
- **认证凭据处理不当**：Provider 认证配置中的凭据处理逻辑存在泄露风险。
- **权限提升**：Provider 行为允许通过 Terraform 配置实现非预期的权限提升。
- **日志中的敏感信息**：Provider 的 debug 日志中输出了敏感信息（API 密钥、Bearer token 等）。
- **供应链风险**：Provider 依赖的第三方库存在已知安全漏洞。

## 2. 强制规则

1. **不得在公开 GitHub Issue 中处理漏洞细节。**
2. AzureRM Provider 安全问题应通过 HashiCorp 的安全报告流程提交：
   - 报告入口：<https://www.hashicorp.com/security>
   - 邮箱通道：<security@hashicorp.com>
3. 不在公开线程要求 PoC、利用代码、完整复现细节或敏感配置。
4. 遵循 Coordinated Vulnerability Disclosure (CVD) 原则。
5. Agent 的职责是完成技术预评估、修复建议和 handoff 材料，最终由维护者人工提交安全报告。

## 3. 快速分流流程

对每个疑似安全 Issue，按顺序执行：

1. 识别风险级别
   - 高风险信号：凭据泄露（state/log/output）、认证绕过、权限提升、远程代码执行、供应链漏洞。
   - 中风险信号：敏感属性未标记 `Sensitive`、日志级别过于详细、信息泄露（非凭据）。
   - 低风险信号：文档中的安全建议缺失、最佳实践未遵循。
2. 技术预评估（Agent 必做）
   - 研究 Provider Go 源代码，确认报告是否成立：
     - 检查受影响属性的 Schema 定义中 `Sensitive` 标记。
     - 检查 Expand/Flatten 函数中是否有敏感值的日志输出。
     - 检查 Provider 认证逻辑中凭据的处理方式。
   - 输出结论：`有效` / `部分有效` / `无效或证据不足`。
   - 给出最小修复方向（Schema 修复、日志清理、代码修复），并说明潜在行为影响。
3. 公开面遏制
   - 不在公开评论中复现漏洞。
   - 不扩散敏感细节；如已出现敏感信息，计划后续清理（见第 8 节）。
4. 构建维护者 handoff case
   - 由 Agent 生成可供维护者手工提交安全报告的 case（见第 6 节模板）。
5. 状态和标签处理
   - 添加 `security`（若仓库标签体系支持）。
   - 移除 `needs-triage`。
   - 标记为已转入安全通道后关闭公开 Issue。
6. 记录最小元数据
   - 仅记录是否已通知报告人、维护者是否已提交安全报告、是否需要后续公开修复公告。
   - 不记录漏洞可利用细节。

## 4. Agent 输出模板（给维护者）

Agent 对安全类 Issue 的输出必须包含：

1. 分类摘要
   - 建议类别：安全漏洞
   - 置信度：高/中/低
   - 受影响资源：`azurerm_xxx`（如适用）
   - 证据摘录：仅引用不敏感内容
2. 风险判断
   - 风险等级：高/中/低
   - 风险类型：凭据泄露 / 敏感属性暴露 / 认证缺陷 / 权限提升 / 供应链 / 其他
   - 是否存在敏感信息暴露：是/否
   - 是否需要立即遏制：是/否
3. 有效性评估与修复建议
   - 报告有效性：有效 / 部分有效 / 无效或证据不足
   - 证据与推理：引用非敏感的 Go 源代码证据（Schema 定义、日志调用等）
   - 建议修复路径：Schema 修复（添加 Sensitive 标记）/ 代码修复 / 日志清理 / 依赖升级
   - 修复优先级：P0 / P1 / P2
   - 修复的兼容性影响：添加 `Sensitive` 标记可能导致 plan diff 变化
4. 维护者执行建议（唯一默认方案）
   - 行动：维护者手工提交安全报告至 HashiCorp Security，并按第 8 节执行公开 Issue 清理计划
   - 负责人：值班维护者或安全响应负责人
   - 时间框：同日（24 小时内）完成首轮处置
   - 完成标准：安全报告已提交，公开 Issue 已去敏并回复政策说明
5. 可直接发送的回复草稿
   - 使用第 5 节模板，避免技术细节
6. 安全报告 handoff case
   - 使用第 6 节模板生成可直接复制的提交材料

## 5. 维护者回复模板（公开 Issue）

Thank you for your security report. Per our security policy, we are unable to discuss vulnerability details in a public GitHub issue.

Security issues in the AzureRM Provider should be reported through HashiCorp's security reporting process:
- Web: https://www.hashicorp.com/security
- Email: security@hashicorp.com

The maintainers have received your report and will ensure it is submitted through the appropriate security channel. It will be handled through the Coordinated Vulnerability Disclosure process.

Sensitive details in this issue will be removed, and the issue will then be closed. If a fix is released, it will be communicated through the Provider's CHANGELOG and release notes.

We appreciate your help in improving the security of the AzureRM Provider.

## 6. 安全报告 Handoff Case 模板（Agent 生成）

> 说明：Agent 生成此模板内容，维护者人工提交至 HashiCorp Security。

1. 标题（Title）
   - `[AzureRM Provider][Security] <一句话漏洞摘要，不含利用细节>`
2. 受影响的 Provider 版本
   - 版本范围：
   - 最新版本是否受影响：
3. 受影响的资源/组件
   - 资源名（`azurerm_xxx`）或 Provider 组件（认证、日志等）：
   - 受影响的 Go 源文件路径：
4. 漏洞类型（Issue Type）
   - 例如：敏感属性未标记 Sensitive / 日志泄露凭据 / 认证绕过 / 权限提升 / 供应链漏洞
5. 影响评估（Impact）
   - 攻击前提（需要访问 state 文件？需要访问日志？需要配置权限？）
   - 潜在影响范围
   - 可利用性判断
6. 当前遏制状态（Containment）
   - 公共 Issue 已去敏：是/否
   - 公共 Issue 已发布政策回复：是/否
7. 初步修复建议（Proposed Fix）
   - 修复方向（Schema 修复 / 代码修复 / 日志清理 / 依赖升级）：
   - 拟修改的 Go 文件：
   - 回归风险：
   - 兼容性影响：

## 7. 禁止事项

1. 禁止在公开 Issue 中确认漏洞是否可利用。
2. 禁止要求用户在公开线程粘贴密钥、配置、攻击路径、PoC 或日志中的敏感字段。
3. 禁止在公开 Issue 中给出补丁时间承诺或披露未发布修复细节。
4. 禁止把安全漏洞按普通 Bug 流程长期公开跟踪。

## 8. 行动计划（需看板批准后执行）

以下动作默认由 Agent 产出执行计划，但**必须由维护者在看板中批准后**再执行：

1. 编辑公开 Issue，删除或隐藏漏洞细节
   - 删除内容：PoC、利用步骤、敏感配置、密钥、可直接复现路径。
   - 保留内容：非敏感摘要、感谢语、政策指引。
2. 在公开 Issue 发布政策回复
   - 使用第 5 节模板，说明不能在公开渠道暴露细节。
   - 明确维护者会通过安全通道提交并跟进。
3. 更新标签和状态
   - 添加 `security`。
   - 移除 `needs-triage`。
   - 在完成去敏和政策回复后关闭 Issue。
4. 维护者手工提交安全报告
   - 使用第 6 节 case 模板提交至 HashiCorp Security。

完成标准：

1. 看板中批准记录可追溯。
2. 公共 Issue 不再包含敏感细节。
3. 公共 Issue 已发布政策回复并关闭。
4. 维护者已完成安全报告提交。

## 9. 关闭标准

满足以下条件即可关闭公开 Issue：

1. 已在公开线程通知报告人，维护者将通过安全渠道处理。
2. 未在公开线程保留敏感细节。
3. 标签/状态已更新。
4. 维护者备注中仅保留非敏感元数据。
