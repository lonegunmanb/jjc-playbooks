# 安全漏洞类 Issue 处理指南（面向维护者）

## 1. 适用范围

当 Issue 被归类为 `安全漏洞`、`敏感信息泄露` 或 `潜在可被利用的安全风险` 时，Agent 必须使用本指南协助维护者处理。

## 2. 强制规则（来自 SECURITY.md）

1. **不得在公开 GitHub Issue 中处理漏洞细节。**
2. 维护者负责将漏洞报告提交到 Microsoft Security Response Center (MSRC)：
   - 报告入口：<https://aka.ms/opensource/security/create-report>
   - 无登录邮箱通道：<secure@microsoft.com>
3. 不在公开线程要求 PoC、利用代码、完整复现细节或敏感配置。
4. 遵循 Coordinated Vulnerability Disclosure (CVD) 原则。
5. 建议使用英文沟通（MSRC 偏好英文）。
6. Agent 无法登录 MSRC；Agent 的职责是完成技术预评估、修复建议和 handoff 材料，最终由维护者人工提交 MSRC。

## 3. 快速分流流程

对每个疑似安全 Issue，按顺序执行：

1. 识别风险级别
   - 高风险信号：密钥/令牌泄露、权限提升、绕过认证、数据泄露、远程代码执行、供应链投毒等。
2. 技术预评估（Agent 必做）
   - 研究模块代码、示例、变量约束和现有行为，判断报告是否成立。
   - 输出结论：`有效` / `部分有效` / `无效或证据不足`。
   - 给出最小修复方向（配置修复、代码修复、文档修复），并说明潜在行为影响。
3. 公开面遏制
   - 不在公开评论中复现漏洞。
   - 不扩散敏感细节；如已出现敏感信息，计划后续清理（见第 9 节）。
4. 构建维护者 handoff case
   - 由 Agent 生成可供维护者手工提交 MSRC 的 case（见第 7 节模板）。
5. 状态和标签处理
   - 添加或保留 `Type: Security Bug`（若仓库标签体系支持）。
   - 移除 `Needs: Triage`。
   - 标记为已转入安全通道后关闭公开 Issue（避免公开跟踪漏洞细节）。
6. 记录最小元数据
   - 仅记录是否已通知报告人、维护者是否已提交 MSRC、是否需要后续公开修复公告，不记录漏洞可利用细节。

## 4. Agent 输出模板（给维护者）

Agent 对安全类 Issue 的输出必须包含：

1. 分类摘要
   - 建议类别：安全漏洞
   - 置信度：高/中/低
   - 证据摘录：仅引用不敏感内容
2. 风险判断
   - 风险等级：高/中/低
   - 是否存在敏感信息暴露：是/否
   - 是否需要立即遏制：是/否
3. 有效性评估与修复建议
   - 报告有效性：有效/部分有效/无效或证据不足
   - 证据与推理：引用非敏感证据
   - 建议修复路径：代码/配置/文档
   - 修复优先级：P0/P1/P2
4. 维护者执行建议（唯一默认方案）
   - 行动：维护者手工提交 MSRC case，并按第 9 节执行公开 Issue 清理计划
   - 负责人：值班维护者或安全响应负责人
   - 时间框：同日（24 小时内）完成首轮处置
   - 完成标准：MSRC case 已由维护者提交，公开 Issue 已去敏并回复政策说明
5. 可直接发送的回复草稿
   - 使用第 6 节模板，避免技术细节
6. MSRC handoff case
   - 使用第 7 节模板生成可直接复制的提交材料

## 5. 禁止事项

1. 禁止在公开 Issue 中确认漏洞是否可利用。
2. 禁止要求用户在公开线程粘贴密钥、配置、攻击路径、PoC 或日志中的敏感字段。
3. 禁止在公开 Issue 中给出补丁时间承诺或披露未发布修复细节。
4. 禁止把安全漏洞按普通 Bug 流程长期公开跟踪。

## 6. 维护者回复模板（公开 Issue）

### 6.1 中文模板

感谢您的安全报告。根据我们的安全政策，我们无法在公开 Issue 中讨论漏洞细节。

本模块的维护者已收到此报告，将代为向 Microsoft Security Response Center (MSRC) 提交，并按 Coordinated Vulnerability Disclosure 流程跟进处理。

本 Issue 中的敏感细节将被移除，随后 Issue 将被关闭。后续如有修复发布，将通过发布说明同步。

感谢您帮助提升本模块的安全性。

### 6.2 English Template (Preferred)

Thank you for your security report. Per our security policy, we are unable to discuss vulnerability details in a public GitHub issue.

The maintainers of this module have received your report and will submit it to the Microsoft Security Response Center (MSRC) on your behalf. It will be handled through the Coordinated Vulnerability Disclosure process.

Sensitive details in this issue will be removed, and the issue will then be closed. If a fix is released, it will be communicated through release notes.

We appreciate your help in improving the security of this module.

## 7. 维护者手工提交 MSRC 的 Case 模板（Agent 生成）

> 说明：Agent 生成此模板内容，维护者人工登录 MSRC 后粘贴提交。

1. 标题（Title）
   - `[AVM][Security] <一句话漏洞摘要，不含利用细节>`
2. 受影响仓库（Repository）
   - `Azure/terraform-azurerm-avm-res-cognitiveservices-account`
3. 受影响范围（Affected Scope）
   - 模块版本/分支/提交：
   - 影响的资源或配置路径：
4. 漏洞类型（Issue Type）
   - 例如：权限提升 / 信息泄露 / 认证绕过 / 供应链风险 / 其他
5. 复现条件（Reproduction Prerequisites）
   - 特殊配置、前置权限、依赖版本
6. 复现步骤（Repro Steps）
   - 仅写私下可披露的完整步骤
7. 影响评估（Impact）
   - 攻击前提、潜在影响范围、可利用性判断
8. 当前遏制状态（Containment）
   - 公共 Issue 已去敏：是/否
   - 公共 Issue 已发布政策回复：是/否
9. 初步修复建议（Proposed Fix）
   - 代码修复方向：
   - 回归风险：
   - 预估修复时间窗：

## 8. 可请求报告人提供的信息（仅私下渠道）

仅在维护者提交 MSRC 后通过私下渠道联系报告人时，可请求补充以下内容：

1. 漏洞类型（如注入、越权、信息泄露等）
2. 受影响源文件路径
3. 受影响版本或提交（tag/branch/commit）
4. 复现所需特殊配置
5. 复现步骤
6. PoC 或利用代码（可选）
7. 影响与攻击路径说明

## 9. 行动计划（需看板批准后执行）

以下动作默认由 Agent 产出执行计划，但**必须由用户在看板中批准后**再执行：

1. 编辑公开 Issue，删除或隐藏漏洞细节
   - 删除内容：PoC、利用步骤、敏感配置、密钥、可直接复现路径。
   - 保留内容：非敏感摘要、感谢语、政策指引。
2. 在公开 Issue 发布政策回复
   - 使用第 6 节模板，说明不能在公开渠道暴露细节。
   - 明确模块维护者会通过专用安全渠道提交并跟进。
3. 更新标签和状态
   - 添加/保留 `Type: Security Bug`。
   - 移除 `Needs: Triage`。
   - 在完成去敏和政策回复后关闭 Issue。
4. 维护者手工提交 MSRC
   - 使用第 7 节 case 模板提交。

完成标准：

1. 看板中批准记录可追溯。
2. 公共 Issue 不再包含敏感细节。
3. 公共 Issue 已发布政策回复并关闭。
4. 维护者已完成 MSRC 提交。

## 10. 关闭标准

满足以下条件即可关闭公开 Issue：

1. 已在公开线程通知报告人，维护者将通过安全渠道处理。
2. 未在公开线程保留敏感细节。
3. 标签/状态已更新为安全处理路径。
4. 维护者备注中仅保留非敏感元数据。
