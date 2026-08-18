---
name: simplify-audit
description: "以高怀疑、证据优先的方式审计现有 repository 或限定范围的 architecture，系统识别可删除、可合并或可收回的 concept、authority、state、path、contract 与 representation。用于发现重复抽象、无效兼容层、镜像状态、复制变体、假想扩展点及其他维护成本超过当前价值的实现，降低全局维护复杂度，并暴露阻塞重大简化的真实决策；不用于直接实施变更、修复 bug、优化性能，或在没有简化目标时进行一般性 code review。"
---

# 简化审计（Simplification Audit）

找出值得实施的删除与合并项：它们的维护成本已经超过当前价值；同时保留高信号但尚未完全闭合的简化线索，并明确区分证据缺口与真正的产品或架构决策。

简化的基本单位是一个可以消失的 `concept`、`authority`、`state`、`path`、`contract`、`coordination mechanism` 或 `representation`，而不是目标代码行数。

采用以下总原则：

> **主动发现，分级报告，保守执行。**

仅进行审计（Audit）。不要编辑 production code，不要提交 patch，也不要把报告写成 implementation plan。可以指出决定性验证步骤，但不要在审计阶段规定最终实现方式。

## 建立覆盖范围（Establish coverage）

阅读 repository instructions，以及当前 architectural authority 的来源。在切分 `slice` 之前，先枚举：

- 已交付的 `shipped deliverables`；
- 对外或对内受支持的 `entry points`；
- 将它们组装起来的 `compositions`；
- persisted artifacts、deployed versions、background jobs、replay/resume paths 和 operational workflows；
- public contracts、extension contracts 以及已声明支持的 third-party producers 或 consumers。

将每一个仍在维护的 top-level surface 追溯到它实际交付的 behavior、supported contract 或 operational obligation，以及支撑它继续存在的当前决策。

然后按 responsibility 和 authority，而不是按文件数量，枚举一个有限的 `slice` 集合。每个 `slice` 按以下状态跟踪：

- **Reviewed** — 已调查其 `demand chains`、ownership、运行时连接方式，以及相关简化信号；可信线索均已完成分类。
- **Partial** — 仍有部分 demand chain、动态连接方式或关键证据未解决；必须明确指出缺口。
- **Unreviewed** — 未进行调查；必须说明原因。

在有帮助时，彼此独立的 slices 可以并行调查。primary agent 负责最终 evidence check、跨 slice 去重和 cross-slice synthesis。

将用户请求的 repository 或 architecture 边界保持为 candidate scope。可以阅读相邻 producers、consumers、registrations、configuration 和 compositions 作为证据，但不要在没有说明的情况下把它们扩展成额外的 candidate scope。

## 采用高怀疑姿态（High-suspicion posture）

默认认为 accidental complexity 可能广泛存在。应主动生成并保留简化线索，但不得降低对最终 implementation recommendation 的证明要求。

一个 internal concept、authority、state、path、contract 或 representation，除非当前证据把它连接到以下至少一项，否则应被视为**推定可疑（presumptively suspicious）**：

- 一条独立结束于 shipped behavior 的 `demand chain`；
- 一个当前受支持且有实际 demand 的 public、external 或 extension contract；
- 一个真实承担 ownership、isolation、policy、security 或 invariant 的边界；
- 一个 persisted-data、deployed-version 或 compatibility obligation；
- 一个当前 operational obligation，例如可观测性、故障隔离、限流、恢复、审计或合规要求。

以下内容只能构成 `retention claim`，不能单独证明保留价值：

- 代码存在并且被调用；
- 多个内部模块相互调用；
- tests 覆盖了该结构；
- documentation 只是复述当前 implementation；
- 命名、目录结构或 architectural symmetry 暗示它“应该存在”；
- hypothetical reuse、future flexibility、可能的第二实现或尚未承诺的扩展；
- 同一模式在 repository 中被反复复制、派生或变体化；
- 删除后需要修改较多内部调用点。

不要从 implementation prevalence、test coverage、复制的文档、命名习惯或重复模式中推断存在明确的当前 design decision。只有当当前 architectural authority 明确陈述其 rationale、constraints 和 intended ownership 时，才把它视为显式且仍有效的设计决策。

缺少历史 rationale 会削弱 retention case，但不能单独证明应当删除。审计不需要为每一种抽象的理论用途构造反例；保留理由必须由当前 evidence 支撑。

## 推导最小组成（Derive the minimum composition）

在调查各 slice 内部的线索之前，先针对每个 shipped deliverable 推导一组最小的 concepts、authorities、states、paths 和 contracts，使其足以满足当前已有证据支持的：

- shipped behavior；
- supported contracts；
- operational obligations；
- persisted-data 和 compatibility obligations。

优先调查这个最小集合与当前 composition 之间最大的差距。这个最小集合只是 `discovery lens`，不是 target design；每个差距首先都只是一个 `lead`。

把相互支撑的内部 consumers 视为一个 `demand cluster`：如果一条 chain 只在该 cluster 内部结束，就不能证明存在独立 demand。内部循环依赖、层层转发或多个包装层共同维持彼此，不构成保留它们的独立理由。

## 主动寻找可以消失的概念（Look for disappearing concepts）

在每个 slice 中，至少调查以下通用信号：

- 对同一事实存在重复的 authorities 或 representations；
- 存在依赖 synchronization 才能保持一致的 parallel paths；
- 存在当前没有实际 demand 的 capabilities、contracts 或 compatibility paths；
- 某些 complexity 仅由 tests、documentation 或 historical machinery 维持；
- 某些 responsibilities 被实现到了其 natural owner 之外；
- 同一业务规则在多个位置被独立解释、默认、验证或转换；
- 当前 composition 相比最小组成多出了没有独立义务的层、状态或协议。

还要主动搜索以下高频 accidental-complexity 模式：

- 只有一个 implementation 或一个 semantic consumer 的 interface、factory、registry、strategy 或 plugin system；
- 不增加 policy、invariant、ownership、isolation、compatibility 或 operational value 的 pass-through service、facade、repository、adapter、coordinator 或 wrapper；
- 表示同一事实的重复 domain model、DTO、schema、view model，以及多段 mapper chain；
- 重复的 validation、normalization、defaulting、parsing、serialization 或 error translation；
- multiple sources of truth、mirrored state、由其他 authority 可直接推导却仍被持久化的 derived state，或没有现实 operational need 的 cache；
- 只有一个已证实 live value 的 feature flag、configuration branch、fallback path、strategy variant 或 environment switch；
- 没有受支持 producer 的 old/new path、migration scaffolding、transitional representation、compatibility reader 或 dual-write machinery；
- 唯一 consumer 是 tests 的 production API、state、hook、extension point 或 helper layer；
- 不增加语义约束的 wrapper type、alias、result object、request/response object 或 helper abstraction；
- 通过 copy-and-variation 产生的近似 modules、handlers、functions、prompts、schemas、workflows 或 configuration blocks；
- 唯一理由是 future reuse、clean architecture、symmetry、flexibility 或“以后可能替换”的抽象；
- `catch-log-rethrow`、convert-and-reconvert、serialize-and-immediately-parse，或在多个等价 representation 之间往返映射的路径；
- 多个对象共同维护同一个 lifecycle、status、version、mode 或 ownership decision；
- 局部 convenience abstraction 迫使全局承担额外 contract、state 或 coordination；
- 为绕过另一层抽象而新增的 escape hatch、special case、fallback 或 bypass path。

这些模式只用于生成和排序 leads，不能单独构成删除结论。例如，只有一个 implementation 的 interface 可能仍承担外部 contract 或隔离边界；必须继续检查其 demand、ownership 和 obligations。

## 追踪 demand、ownership 与运行时连接（Trace demand and ownership）

对于 surface-sized leads，使用 `demand chains`；对于局部线索，使用 exact-symbol、exact-string、representation、configuration 和 registration 搜索。

至少检查：

- direct 和 indirect symbol references；
- entry-point reachability；
- dependency injection、registries、plugin discovery 和 framework conventions；
- reflection、dynamic import、string-based lookup、generated code 和 serialization hooks；
- configuration、feature flags、environment variables、deployment manifests 和 startup composition；
- persisted artifacts、database values、queues、events、snapshots、replay/resume inputs；
- public APIs、CLI、file formats、protocols、webhooks、extensions 和 third-party producers；
- operational tooling、migration jobs、backfills、support scripts 和 incident procedures。

Architecture 描述预期 ownership；live calls 只能证明“正在被使用”，不能证明“值得保留”。把 usage 视为 retention evidence，而不是 retention proof。必须解决 ownership、demand 和 usage 之间的冲突。

不要重新打开一个**明确且当前有效**的 design decision 或已拒绝的 alternative，除非现有证据：

- 改变了该决策的某个前提；
- 揭示了原决策未考虑的 material maintenance cost；
- 表明当初承诺的 demand、extension model 或 compatibility obligation 已不复存在；
- 证明当前 implementation 已偏离原决策的 intended ownership。

Public contracts 和 extension contracts 必须检查 ecosystem 与 supported-extension demand；它们仅仅存在或被文档记录，并不能证明 demand 仍然存在。

对于 compatibility path，当前没有 writer 并不足以作为删除依据。在判断 reader、fallback 或 migration path 可以消失之前，必须检查 supported persisted artifacts、deployed versions、replay/resume paths、rolling upgrades 和 third-party producers。

不要为了追求“完整”而阅读每一个文件。当一个 slice 的 top-level surfaces、demand chains、动态连接方式和上述信号已经被调查到足以分类所有可信 leads 时，该 slice 即可视为 Reviewed。

## 要求删除证明（Require a deletion proof）

一个 lead 只有在其 `deletion proof` 建立以下所有事项后，才能成为 `Candidate`：

- 当前 authority、production consumers、demand chain，以及最有力的 retention reason；
- 在整个受影响系统中，哪些 code、state、contract、coordination、representation 或 required change surface 会消失；
- 该变更会在其他位置新增、保留或迁移什么，以及为什么最终结果仍然是严格的 `net reduction`；
- 为什么最有力的 retention reason 已不再成立，或为什么更小的组成仍能满足它；
- 将放弃的 capability 或 behavior、impact radius，以及 material uncertainty。

必须以最有利于保留方的方式表述 strongest retention reason。不要通过忽略 dynamic usage、external demand 或 operational obligations 来制造 deletion proof。

对于 bounded internal lead，在受影响的 architectural boundary 内完成穷尽性证据检查即可；不要求找到不存在的历史 rationale，也不要求排除没有任何现实证据支持的纯理论用途。

对于 public contract、supported extension、persisted representation、deployed-version compatibility、replay/resume path 和 third-party producer，仍必须执行完整 deletion proof。

只有当 retention reason 已解决，并且 net reduction 不依赖未来某个 implementation choice 时，proof 才算 closed。Priority 永远不能替代 evidence。

## 将发现分为三类（Classify findings）

### 1. Candidate

`Candidate` 是 deletion proof 已闭合、可以进入实施优先级判断的简化项。

每个 Candidate 必须说明：

- 将消失的 concept、authority、state、path、contract 或 representation；
- 当前 authority、production consumers 和 demand chain；
- strongest retention reason，以及它为何不再成立；
- whole-system disappearance surface；
- 需要新增、保留或迁移的内容；
- 为什么结果是 strict net reduction；
- 放弃的 capability、impact radius 和 material uncertainty；
- 支撑结论的 production evidence。

### 2. Deletion probe

`Deletion probe` 是高度疑似 accidental complexity、likely disappearance surface 清晰，但仍存在一个或少数几个**范围有限且可决定性验证**的不确定项。

不要仅仅因为还缺一次 bounded verification，就省略高信号 lead。

每个 Deletion probe 必须说明：

- suspected disappearing concept；
- 为什么它更像 accidental complexity，而不是 intentional architecture；
- 当前 authority、observed consumers 和已追踪的 demand chain；
- likely whole-system disappearance surface；
- strongest plausible retention reason；
- bounded unresolved uncertainty；
- 最可能闭合该线索的**单个 decisive check**；
- 若该 check 失败，线索应如何被否决或降级。

最多报告 10 个 Deletion probes。只保留 likely disappearance surface 具有 material value，且 decisive check 明确、范围有限的 probes。不要把普通 code smell 清单包装成 probes。

### 3. Decision gate

`Decision gate` 只用于真正尚未做出的 product 或 architecture choice。缺少证据本身不是 Decision gate；可以通过搜索、运行时追踪、读取配置或检查 persisted artifacts 解决的问题，应当是 evidence gap 或 Deletion probe，而不是决策门。

每个 Decision gate 必须说明：

- 待做出的 product 或 architecture decision；
- 不同选择分别保留或消除什么；
- likely disappearance surface；
- strongest retention reason；
- decisive missing evidence 或尚未确定的 policy；
- 为什么审计无法在不替代 owner 做决策的情况下闭合它。

不要把 Decision gate 表述成 implementation recommendation。普通 false leads 直接省略。

## 过滤低价值观察（Filter low-value observations）

省略只是在说“代码看起来复杂”“层太多”或“可以更优雅”的观察。

Cosmetic defects、style inconsistencies、命名问题、格式问题和一次性 documentation fixes 不属于本次 audit，除非它们体现了持续存在的 maintenance obligation、duplicate authority 或 recurring coordination cost。

不要把以下内容单独作为发现：

- 单纯减少代码行数；
- 把多个文件机械合并成一个文件；
- 用另一套同等复杂的 abstraction 替换当前 abstraction；
- 仅改变命名、目录或 class/function 边界；
- 需要先完成未决定的 target design 才能声称存在 net reduction；
- 没有 production evidence 的偏好型重构建议。

## 分配实施优先级（Assign an implementation priority）

只给 deletion proof 已闭合的 Candidates 分配 priority。Priority 描述一个有效 simplification 何时值得实施，而不是 defect severity：

- **P0** — 正在造成 active harm、阻塞工作、破坏 correctness，或持续扩散 complexity。
- **P1** — 具有高 leverage、真实的当前成本和充分 evidence。
- **P2** — 值得实施，具有普通 urgency。
- **P3** — 价值较有限，适合 opportunistically 实施。

任何 priority 都可以为空。不要在同一个 priority 内对 findings 排名，也不要为了凑数量而填满 candidate quota。

Deletion probes 不分配 implementation priority；可以说明其 expected leverage，但不能把它伪装成已验证的实施项。

## 结束审计（Close the audit）

在报告之前，综合不同 slices 中重复、互为前提或共同指向同一底层 simplification 的 findings。一个底层 concept 只能报告一次；其他位置作为 impact surface 或 supporting evidence 合并进去。

一次 complete audit 要求：

- 请求范围内的每个 slice 都是 Reviewed；
- 每个 Candidate 都有 closed deletion proof；
- 每个 Deletion probe 都只有 bounded uncertainty 和明确 decisive check；
- 每个 Decision gate 都确实需要 product 或 architecture owner 作出选择；
- 所有结论都能追溯到 production evidence，而不是只追溯到 tests 或文档复述。

如果 evidence、access 或运行时可见性阻止闭环，则报告 partial audit，并明确哪些结论仅适用于已审查范围；不要暗示已经得出全范围结论。

返回：

1. 一张 coverage table，列出每个 slice、状态、已检查的 top-level surfaces 和 remaining gaps；
2. 按 P0–P3 分组的 actionable Candidates；
3. material Deletion probes；
4. genuine product 或 architecture Decision gates。

当输出满足以下条件时停止：

- coverage table 能解释请求 scope 中的每个 slice；
- Candidate、probe 和 gate 的分类彼此一致；
- 每项主张都能追溯到相应 evidence；
- 同一个底层 simplification 没有被重复报告；
- 没有为了提高发现数量而把 uncertainty 隐藏成 certainty；
- 没有因为无法达到绝对确定而省略具有 material value、且只剩 bounded verification 的高信号线索。
