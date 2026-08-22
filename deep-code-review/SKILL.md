---
name: deep-code-review
description: 围绕冻结的审查目的，对代码变更或现有实现进行证据驱动、决策相关的深度审查，覆盖 correctness、安全、回归、设计质量和测试信心。用于用户要求全面 code review、correctness/design/maintainability audit，或明确要求并行 reviewer 时。默认只报告足以改变合并、修复或设计决策的问题。
---

# Deep Code Review

## 目标

找出会实质改变当前工程决策的问题，而不是尽可能多地列举风险。

默认只读。除非用户明确要求修复，否则不要修改代码、提交、推送、评论或 resolve thread。

审查完成可以得出“没有 finding”。不要为了证明审查充分而制造建议。

## 冻结 Review Charter

开始前建立一份简短的 Review Charter：

- 审查对象：PR diff、提交区间、模块或现有实现
- 基线：通常是 merge base 或用户指定 revision
- 一句话目的：这次变更必须交付的用户或系统结果
- 支持路径：为实现该目的需要成立的主要运行路径
- 非目标：本轮明确不解决的相邻问题
- Merge bar：哪些失败会阻止当前变更合并或采用
- 范围扩张条件：什么新证据才足以改变以上内容
- 主要用户路径、信任边界和持久状态
- 哪些验证在当前环境真实可行

PR 审查优先从 PR body、关联 issue、用户补充和实际 diff 提炼 Charter。Charter 建立后保持冻结；只有用户改变目标，或新证据表明原目标、边界或 Merge bar 不成立时才更新。所有 reviewer 和后续轮次共享同一份 Charter。独立审查意味着独立判断证据，不意味着重置范围。

先画一个简短 Risk Map，把注意力集中在高风险边界：

- authority、authentication、authorization
- 持久化、事务、恢复、迁移
- 并发、生命周期、重连、shutdown
- 跨进程、跨机器和不可信输入
- 用户可见的核心工作流与兼容性

不要机械遍历所有文件或为低风险代码平均分配审查预算。

## 审查方式

默认完成一轮独立审查。用户明确要求并行审查时，按互补视角拆分，例如：

- correctness / security
- design / maintainability

Reviewer 应独立阅读代码并形成判断，再裁决外部评论；不要把已有评论直接当作 backlog。

存在多个 reviewer、外部评论或多轮复核时，维护一个轻量 Finding Ledger，记录每项问题的证据和裁决：open、resolved、rejected、follow-up 或 accepted risk。综合结论是对证据的裁决，不是所有 reviewer 输出的并集。已经裁决的项目只有在出现新证据、相关代码重新变化或 Risk Map 实质变化时才重新打开。

并行 reviewer 共享同一工作区时默认只读，避免同时 build、clean、format 或写入生成目录。

## Finding 准入门槛

一项问题只有同时满足以下条件，才作为 finding：

1. **归因成立**：由审查范围引入、暴露或明确属于用户要求审查的现有设计
2. **路径成立**：有受支持的运行路径、合理的失败/并发路径，或明确的攻击者可控边界
3. **影响实质**：会造成错误行为、安全边界破坏、数据风险、明显回归，或持续的维护负担
4. **修复相称**：建议方案的复杂度和测试成本不高于它消除的风险
5. **影响决策**：会改变是否合并、是否修复或采用哪种设计
6. **当前归属**：它影响 Review Charter 的目的或 Merge bar，且根因修复属于当前范围

缺少任一项时，不应升级为当前 finding。真实性与当前性分别裁决：一个问题可以真实存在，但只适合作为 follow-up 或 accepted risk；若对决策没有帮助，则直接省略。

### 可达性

区分四类路径：

- 正常用户路径
- 合理的故障、重连、并发或恢复路径
- 攻击者可控的信任边界
- 需要多个不受支持前提同时成立的狭窄或构造路径

最后一类默认不报告。只有可能导致不可恢复数据损失、权限突破、核心 authority 破坏或广泛故障时，才值得提升。

“理论上可能”不是充分证据；“发生概率低”也不是自动忽略的理由。综合判断：

> materiality = reachability / exposure × impact × persistence / recoverability

### 证据

每个 finding 至少给出：

- 精确位置
- 触发条件和控制流
- 用户或系统影响
- 为什么现有约束不能阻止它
- 最小、根因导向的修复方向

优先使用静态调用链、稳定契约、已有测试和小型定向复现。不要为了证明极窄猜想建立庞大 harness。

安全边界可以在事故发生前报告，但仍需证明输入或 principal 能到达该边界，以及影响成立。

## Correctness 与设计判断

Correctness 重点检查：

- 输入与协议是否 fail closed
- authority、identity、revision 和 target 是否贯穿整个逻辑操作
- commit、rollback、recovery 和 shutdown 是否保持不变量
- reconnect、retry 和 replay 是否会重复 mutation 或迁移到错误 target
- 错误是否在恰当边界保留语义并可诊断

Design 只报告已经带来实质成本的问题：

- 同一事实存在多个 owner 或 representation
- 抽象没有减少理解成本，只增加转发、状态或生命周期
- 通用层解释了具体 domain policy
- 修复不断追加条件而没有消除根因
- 当前需求可以通过更小、更内聚的模型完整表达

单纯的行数、个人风格、命名偏好或未来可能扩展，不构成 design finding。

## 裁决与修复建议

外部 reviewer、静态分析器和 AI 报告都只是证据来源。逐项判断：

- 是否属于当前范围
- 是否真实可达
- 是否已有约束或后续提交封口
- 影响是否值得增加实现和测试成本

确认问题后，优先删除错误状态、合并 authority 或收紧边界。不要默认增加新状态机、兼容层或重试机制。

实施前做一次 Scope Delta 检查。若修复需要引入新的 owner、状态、协议、公共契约、依赖或跨子系统职责，只有它是恢复 Charter 目标的最小必要条件时才进入当前 PR；否则记录为 follow-up。以新增 mental model 和边界衡量扩张，不使用机械的行数阈值。

单项裁决完成后，把所有拟在当前范围实施的修复作为一个整体复核。若它们分别成立、组合后却新增多个 owner、状态、契约、边界或重复机制，优先收敛为共同根因和统一模型；无法收敛且不影响 Merge bar 的项目降为 follow-up。最终方案以交付 Charter 所需的最小整体 mental model 为准，而不是各项修复的局部最优。

合理结论也可以是：

- 接受风险，不修改
- 改进诊断或文档，而不增加运行时机制
- 删除不再需要的行为或兼容路径
- 推迟到有真实需求的后续工作

## 测试预算

修复 finding 不自动要求新增测试。只有同时满足以下条件时才添加：

- 验证稳定、用户可见的行为或关键不变量
- 回归有现实复发可能，或影响足够严重
- 现有测试不能有效覆盖
- 测试不会主要锁定调用次数、私有顺序、mock 细节或偶然竞态

优先级：

1. 扩充现有行为测试
2. 用类型或数据结构消除非法状态
3. 复用现有集成测试
4. 新增一个最小行为测试
5. 不新增测试，并说明已有验证为何足够

通常每个 finding 至多新增一个直接行为测试。删除实现路径时，也删除只保护该路径的低价值测试。

## 输出

默认先列 findings，按严重度排序：

- **Blocking**：当前不应合并或采用
- **Important**：真实且值得在当前范围修复，但不必然阻断整体方向

每项包含位置、触发路径、影响和最小修复方向。不要混入可选重构、泛化建议或纯风格意见。

只有用户要求，或记录能避免后续重复裁决时，才附一小节 **Follow-up**：列出真实但不影响当前 Merge bar 的问题。Follow-up 不进入当前修复循环。

随后给出：

- Correctness verdict：acceptable / not acceptable
- Design verdict：acceptable / not acceptable
- 已完成的验证及未验证项
- 仅与决策相关的 residual risks

## 与 Simplify Audit 联用

Deep Code Review 负责 correctness、风险和合并决策；Simplify Audit 只判断当前或拟议方案能否在 Charter 内删除 mental model。两者共享 Charter 和 Finding Ledger，并只综合一次。简化建议本身不能把 follow-up 升级为当前 finding。

## 停止条件

满足以下条件后停止：

- 高风险边界已经被独立检查
- 外部反馈已经裁决
- 已确认 finding 的修复得到相称验证
- 没有新的证据表明 Risk Map 发生实质变化

修复局部问题后，优先定向复核该边界，不要自动开启新一轮全量审查。只有修复改变了 authority、持久化、协议、并发或生命周期模型时，才重新做完整 review。

用户要求“全新一轮”时，可以由 reviewer 独立重读代码，但仍继承冻结的 Charter 和 Finding Ledger；fresh perspective 不是 scope reset。
