---
name: thermos
description: 并行运行两个热核级审查流程，然后综合它们的发现。用于用户明确要求 thermos、double thermo review、两个 thermo reviewer、并行 review agent，或同时覆盖 bug、安全问题、测试可靠性与代码质量的分支审计。
---

# Thermos

当用户明确要求 `thermos`、double thermo review、两个 reviewer、subagents、delegation 或 parallel agent work 时，并行运行两个热核级审查流程，然后综合结果。

如果用户要求组合审计，但没有明确要求并行 agent 或 `thermos`，则本地顺序执行两个审查流程，不启动 subagents。

## 工作流

1. 根据用户请求、PR、当前分支或相关变更文件确定审查范围。
2. 收集 diff，以及 reviewer 不靠猜测就能评估改动所需的文件或上下文摘录。这一步只做一次，并把相同的 scoped context 传给两个 reviewer。
3. 使用当前宿主环境提供的并行 agent/subagent 能力启动两个只读 reviewer。不要硬编码具体工具名、agent API 名称或本机文件系统路径；通过当前环境的 skill 发现机制、skill 名称，或会话中可用的 skill 引用来加载对应审查指令：
   - 一个 reviewer 使用 `thermo-nuclear-review`，负责 bug、破坏性变更、安全问题、开发体验回退、feature gate 泄漏，以及其他分支审计风险
   - 一个 reviewer 使用 `thermo-nuclear-code-quality-review`，负责可维护性、结构、文件体量增长、意大利面式复杂度、抽象、重复、低价值测试、清理机会，以及代码库健康风险
   - 如果当前环境没有可用的并行 agent/subagent 能力，则不要伪造并行；在父 agent 中按两个审查视角顺序执行并综合结果
4. 要求两个 subagent 返回带文件引用和证据的优先级排序发现。subagent 不能编辑文件。
5. 在 subagents 运行期间，继续做不会重叠的本地检查，以提升最终综合质量。
6. 两边完成后，按 findings first 的方式综合结果，去重并合并 reviewer 间的重复发现。重叠发现权重更高；分歧由父 agent 自行判断；summary 保持简短。
7. 如果 review 后用户要求修复，由父 agent 负责实现和验证。

如果单个 background summary 已经对用户可见，不要整段复述。只输出统一结论、最高信号的发现，以及剩余不确定性。
