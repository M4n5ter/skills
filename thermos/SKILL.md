---
name: thermos
description: 并行运行两个热核级审查流程，然后综合它们的发现。用于用户明确要求 thermos、double thermo review、两个 thermo reviewer、并行 review agent，或同时覆盖 bug、安全问题、测试可靠性与代码质量的分支审计。
---

# Thermos

当用户明确要求 `thermos`、double thermo review、两个 reviewer、subagents、delegation 或 parallel agent work 时，并行运行两个热核级审查流程，然后综合结果。

## 工作流

1. 根据用户请求、PR、当前分支或相关变更文件确定审查范围。
2. 收集 diff，以及 reviewer 不靠猜测就能评估改动所需的文件或上下文摘录。这一步只做一次，并把相同的 scoped context 传给两个 reviewer。
3. 使用当前宿主环境提供的 multi-agent/spawn agent/subagent/task 能力启动两个只读 reviewer。不要硬编码具体工具名、agent API 名称或本机文件系统路径；通过当前环境的 skill 发现机制、skill 名称，或会话中可用的 skill 引用来加载对应审查指令：
   - 一个 reviewer 使用 `thermo-nuclear-review`，负责 bug、破坏性变更、安全问题、开发体验回退、feature gate 泄漏，以及其他分支审计风险
   - 一个 reviewer 使用 `thermo-nuclear-code-quality-review`，负责可维护性、结构、文件体量增长、意大利面式复杂度、抽象、重复、低价值测试、清理机会，以及代码库健康风险
   - 如果当前环境没有可用的 multi-agent/spawn agent/subagent/task 能力，则不要伪造并行；直接按两个审查视角顺序执行并综合结果
4. 要求两个 subagent 返回带文件引用和证据的优先级排序发现。每项 finding 必须说明可通过哪项受支持行为触发、必要前置条件、可观察影响、证据与置信度，并给出最小修复方向；subagent 不能编辑文件。
5. 让 reviewer 自然完成，不设置会影响判断的收束时限，也不读取其内部 rollout。在它们运行期间，只做不会与审查重复的本地检查。
6. 两边完成后，按 findings first 综合并去重。重叠发现值得优先复核，但不因两人都提到就自动成立；由主 agent 端到端验证。
7. 裁决 finding，而不是机械累加修复：
   - 能由受支持行为触发，或涉及安全、权限、数据损坏的真实问题，进入修复。
   - 暴露内部契约不清的问题，优先判断能否用更小的接口、类型或职责调整让非法状态不可表达。
   - 仅依赖违反明确契约的调用、无法到达的输入或纯理论条件的问题，默认作为残余风险或加固建议，不扩张实现。
   - 低价值测试、样式 nit 和不能提高信心的防御性代码直接拒绝。
8. 同时给出两个独立结论：正确性是否可接受；实现是否仍是内聚、最小且便于 review 的表达。如果复审持续在同一职责边界发现新的同类问题，而且修复反复新增状态与分支，把它视为底层模型可能有误的信号：暂停补丁式修复，重新审视契约、类型和所有权。轮数不是硬阈值；若后续 finding 彼此独立，且修复在减少复杂度，可以继续正常复审。
9. 如果 review 后用户要求修复，由主 agent 负责实现和验证。变更 diff 后，只在风险值得时复用上一轮 reviewer 或发起新一轮审查，不把 review 轮数本身当作质量。

如果单个 background summary 已经对用户可见，不要整段复述。只输出统一结论、最高信号的发现，以及剩余不确定性。
