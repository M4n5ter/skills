---
name: engineering-principles
description: 使用 24 条工程原则指导代码设计、重构、迁移、调试、测试与多步骤工程任务；也用于用户通过中英文原则名称纠正工作方向。按任务触发条件读取完整原则，说明它具体改变了哪个决定。
---

# 工程原则

## 使用方式

1. 每项多步骤工程任务开始时，阅读下方索引，匹配触发条件。
2. 应用任何原则前，完整阅读对应参考文件；不要仅凭名称或摘要运用。
3. 只读取当前任务需要的原则。用户可使用中文名或保留的英文名来引导工作，无需单独调用 24 个技能。
4. 在回复中列出真正影响决策的原则，并指出各自改变的具体选择。只引用本次会话完整读过的原则；没有对应决定的引用不算应用。

## 原则索引与触发条件

### 核心

- **[懒惰协议（Laziness Protocol）](references/laziness-protocol.md)**：重构、评估差异规模，或想增加抽象、层次、信号传递时。倾向删除，以及能解决问题的最小改动。
- **[基础性思考（Foundational Thinking）](references/foundational-thinking.md)**：编写逻辑之前：核心类型和数据结构、基础设施与功能的顺序、并发参与者共享什么。
- **[从第一性原理重新设计（Redesign from First Principles）](references/redesign-from-first-principles.md)**：把新需求整合进现有设计，或同类需求不断增加特殊分支时。重新设计，就像它从第一天起就是基础需求。
- **[质疑前提（Attack the Premise）](references/attack-the-premise.md)**：两个或更多共享同一前提的修复均未通过同一检验时。先普查哪些参与者承载失衡，再质疑前提，不要继续写依赖它的修复。
- **[先减后加（Subtract Before You Add）](references/subtract-before-you-add.md)**：安排新增、重构或重写的顺序时。先移除负担，再在更简单的基础上构建。
- **[降低阅读者负担（Minimize Reader Load）](references/minimize-reader-load.md)**：审阅或塑造难以追踪的代码时。计算层次和隐藏状态，合并单一调用方的包装层，缩小可变状态作用域。
- **[面向结果执行（Outcome-Oriented Execution）](references/outcome-oriented-execution.md)**：具有明确阶段边界的计划性重写和迁移。向目标架构收敛，不保留注定丢弃的兼容状态。
- **[体验优先（Experience First）](references/experience-first.md)**：产品、用户体验或功能范围的取舍。用户愉悦优先于实现方便。
- **[充分探索设计空间（Exhaust the Design Space）](references/exhaust-the-design-space.md)**：没有先例的新交互或架构决策。构建 2 至 3 个竞争原型，比较后再确定方案。
- **[构建杠杆工具（Build the Lever）](references/build-the-lever.md)**：任何非琐碎工作。构建完成或证明工作的工具（codemod、脚本、生成器），而不是手工做；工具是审阅者可重跑的产物。

### 架构

- **[为领域建模（Model the Domain）](references/model-the-domain.md)**：编写有状态逻辑，或代码分支很多、多个文件重复假设同一种数据形状时。用结构（状态机、类型化模型、表或注册表、reducer、边界、合适的集合）表达领域，取代散落的条件判断。
- **[边界纪律（Boundary Discipline）](references/boundary-discipline.md)**：连接校验、错误处理或框架适配器时。防护放在系统边界，信任内部类型，保持业务逻辑纯粹。
- **[类型系统纪律（Type System Discipline）](references/type-system-discipline.md)**：在任意类型化语言中设计类型或签名时。让非法状态无法表示，为基本类型标注语义身份，在边界解析外部数据。
- **[让操作幂等（Make Operations Idempotent）](references/make-operations-idempotent.md)**：设计可能遭遇崩溃和重试的命令、生命周期步骤或循环时。收敛到相同最终状态。
- **[迁移调用方，再删除旧 API（Migrate Callers Then Delete Legacy APIs）](references/migrate-callers-then-delete-legacy-apis.md)**：旧调用方仍存在时引入新内部 API。在同一轮中迁移并删除。
- **[先隔离，再串行化共享状态（Separate Before Serializing Shared State）](references/separate-before-serializing-shared-state.md)**：并发参与者可能写同一文件、分支、键或对象时。先消除共享。

### 验证

- **[证明它确实可用（Prove It Works）](references/prove-it-works.md)**：任务完成后、宣布完成前。验证真实产物，而不是替代指标或“编译通过”。
- **[修复根因（Fix Root Causes）](references/fix-root-causes.md)**：调试时。把每个症状追溯到根因，先复现，不断问为什么，直至找到根因。
- **[把工作组织为可验证单元（Sequence Work into Verifiable Units）](references/sequence-verifiable-units.md)**：多步骤工作（批量修改、迁移、一连串相似编辑），以及组织提交和 PR 时。拆成小单元，每个以检查结束，确认后再继续；让交付顺序本身成为证明。
- **[测试行为，而非实现（Test Behavior, Not Implementation）](references/test-behavior-not-implementation.md)**：编写、修改或保留测试时。像用户一样调用代码，对照字面量预期值断言结果。如果每个导入函数都返回 undefined 时测试仍会通过，就重写断言或删除测试。
- **[解释测量数字（Explain the Number）](references/explain-the-number.md)**：信任、汇报或依据测量数字（加速比、回退、吞吐量、延迟或评估结果）采取行动前。找出限制因素，排除测到了其他东西的可能性。

### 委派

- **[保护上下文窗口（Guard the Context Window）](references/guard-the-context-window.md)**：上下文逐渐填满时：大量输出、长文件、反复读取、并行分派规划。把批量阅读交给子代理，主线程保留摘要。
- **[不要因等待人类而停滞（Never Block on the Human）](references/never-block-on-the-human.md)**：对于可逆工作，想问“我应该做 X 吗？”时。先做，呈现结果，让人类纠正方向。

### 元原则

- **[把经验固化到结构中（Encode Lessons in Structure）](references/encode-lessons-in-structure.md)**：发现自己第二次写相同指令时。把它固化成 lint、元数据标志、运行时检查或脚本，而不是增加文字。
