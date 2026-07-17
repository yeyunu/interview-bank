# 腾讯 PCG 候选 1 - 套 MindClaw / ClawCodex 回答

tags: #agent面试 #RAG #Agent #MindClaw #ClawCodex

> 说明：这份是把原始面经中的“候选 1”问题，套到我们自己的 MindClaw / ClawCodex 项目里。
> 原始 OCR 里有一些噪声，比如日期、章节号、答案片段，这里只保留能转成面试问题的部分。

## 项目总定位

如果面试官让你先介绍项目，可以先这样定调：

> 我做的是 MindClaw / ClawCodex 这类面向软件工程任务的 AI Agent 框架。它不是单纯的聊天机器人，而是让大模型能接入真实工程流程：读代码、查文档、调用工具、修改文件、跑测试、生成报告，甚至把结果同步到 PR 或任务系统。
>
> 架构上我会分三层：底层是 Provider 和 Tool / Skill 系统，中间是 Agent Runtime，负责 ReAct 循环、上下文和工具调用，上层是 Orchestrator / Coordinator，负责任务状态机、多 Agent 协作、重试、验证和回滚。
>
> 所以这些 RAG + Agent 问题，我会重点落到“代码/文档/日志检索、工具调用稳定性、多 Agent 调度、状态恢复、评测闭环”这几个真实工程点上。

---

## Q1：稠密检索已经有了，为什么还要加 BM25？

先纠偏：这个问题不能直接套成“我们已经用了 BGE + BM25”。从当前代码看，MindClaw / ClawCodex 没有真正实现稠密向量检索，也没有 BM25 排序器。

更准确的说法是：我们项目现在做的是工程 Agent 的多路检索和定位，不是标准知识库 RAG 里的 dense + BM25 hybrid retrieval。

当前项目里真实存在的检索/定位机制主要是：

1. `ToolSearch`：把工具名、prompt、description、search_hint、schema 描述拼成可搜索文本，再做字符串、token、短语、别名和分层排序匹配。
2. workspace search：用 `ripgrep` 做固定字符串内容搜索，用 `rg --files` 列文件，再用 fuzzy score 做 quick-open 文件过滤。
3. session retrieval：按当前 cwd、改动文件、最近用户输入、历史摘要 token 的 Jaccard 相似度和文件路径重叠做历史会话召回。
4. memdir memory recall：先扫描 memory manifest，再让 LLM 从候选 memory 文件里选择相关项。
5. orchestrator rule dedup：只有规则去重场景用了轻量 TF-IDF + cosine similarity，这不是业务知识库检索，也不是稠密 embedding。

所以面试时我会这样答：

> 我们当前项目不是 BM25 + 向量库的标准混合 RAG，而是偏代码工程场景的检索增强 Agent。对于代码符号、文件路径、错误栈、工具名这类精确证据，我们主要依赖 ToolSearch、ripgrep、文件名 fuzzy search、历史会话召回和可重建的结构化索引。  
> 如果后续要做“大规模项目文档/面试题库/知识库问答”，我会再引入 dense + BM25：dense 负责同义语义召回，BM25 负责关键词、符号、错误码和路径命中。但不能说当前已经实现了这套。

一句话总结：

> 当前项目没有稠密检索和 BM25。能讲的是“为什么未来可能需要 hybrid retrieval”，以及当前已经用工程化检索解决代码 Agent 的证据定位问题。

---

## Q2：MongoDB 和 Milvus 数据一致性怎么保证？

这个题不能硬套成“我们项目用了 MongoDB + Milvus”。当前项目没有看到 Milvus 这类向量库链路。

如果面试官问这题，我会先把它抽象成“事实源”和“派生索引”的一致性问题：

- 事实源：代码文件、配置文件、工具定义、Issue / PR 状态、session transcript、执行日志、workflow rules。
- 派生索引或缓存：ToolSearch 搜索文档、workspace 文件列表、session summary、rules TF-IDF 向量、外部接入时可重建的 CodeGraph。

套到 MindClaw / ClawCodex，可以这样回答：

1. 不把索引当事实源，事实以代码文件、日志、任务状态和配置为准。
2. 派生索引必须可重建，比如工具搜索文档可以从工具 registry 重建，workspace 文件列表可以从 `rg --files` 重扫，session summary 可以从 transcript 重新生成。
3. 写入流程上先落事实源，再构建或刷新派生索引。
4. 查询结果必须能回到源文件、源日志、源工具定义或源 transcript，否则就不能作为强证据。
5. 如果索引过期，优先降级到源数据读取、重新扫描或重新构建，而不是让 Agent 继续基于脏索引决策。

如果未来真的引入 MongoDB + Milvus，我会沿用同一个原则：MongoDB / Postgres / 文件系统做事实源，Milvus 只做可重建召回索引，用 `doc_id`、`chunk_id`、版本号、checksum、重试队列和对账任务保证最终一致。

一句话总结：

> 当前项目没有 MongoDB + Milvus。更稳的回答是：我们把检索结果当派生证据，事实源可追溯，索引可重建，脏索引要能降级和修复。

---

## Q3：复杂文档 / 表格切块如何保证完整上下文？

这个问题非常适合套到我们的项目，因为代码、Markdown、日志、表格都不能简单固定长度切。

回答：

我不会只按固定 token 长度硬切。复杂文档要先保留结构，再做 chunk。

在我们项目里，常见文档有几类：

- Markdown 技术文档。
- README / API 文档。
- 代码文件。
- 测试日志和错误栈。
- 工具说明和 Skill 文档。

我会按结构切：

1. Markdown 按标题层级切，chunk 里保留父标题路径。
2. 代码按函数、类、方法、路由、组件等 AST 单元切。
3. 表格按行组或业务实体切，不把表头和数据分开。
4. 错误日志按一次失败事件切，保留错误栈、文件路径、失败断言。
5. 每个 chunk 带 metadata，比如来源文件、标题路径、函数名、行号、版本号。

召回时也不只返回孤立 chunk，而是可以扩展上下文：

- 返回命中 chunk。
- 加上父标题 / 前后相邻 chunk。
- 对代码加上函数签名、调用方、被调用方。
- 对表格加上表头和字段解释。

一句话总结：

> 切块不是把文本切短，而是把文档结构变成 Agent 能理解和检索的最小语义单元。工程里最怕的是召回到了内容，却丢了标题、字段、前置条件和调用关系。

---

## Q4：子 Agent 跑死怎么回退，不让用户重启？

这个特别适合套到 Orchestrator / Coordinator。

回答：

我不会让子 Agent 直接裸跑，而是放在 Orchestrator 管控下。每个子 Agent 都应该有任务 ID、状态、超时、心跳和结构化输出。

具体做法：

1. Orchestrator 维护任务状态机，比如 `QUEUED / RUNNING / WAITING_TOOL / FAILED / DONE / NEED_HUMAN`。
2. 每个 Agent 有独立 scratchpad 和工作区，避免失败时污染全局状态。
3. 工具调用前后记录 checkpoint，比如这一步要改什么文件、调用什么工具、预期产物是什么。
4. 如果子 Agent 超时、循环调用工具或返回异常，Orchestrator 可以 kill 掉该 Agent。
5. 回退时优先回到最后一个有效 checkpoint，而不是让用户重新开始。
6. 对幂等任务可以自动 retry，对非幂等任务进入人工确认。

比如代码任务里：

- Agent 修改文件前记录 diff。
- 测试失败时保留日志。
- 如果 Agent 继续乱改，可以 revert 当前工作区 diff，或者从上一个 checkpoint 重新派发。

一句话总结：

> 子 Agent 可以不稳定，但外层 Orchestrator 必须稳定。回退靠状态机、checkpoint、工作区隔离和验证日志，而不是靠模型自己记住做过什么。

---

## Q5：静态词典新增词怎么处理？要不要发版？

这个可以套到“工具路由词典 / Skill 元数据 / Provider 配置 / 检索同义词表”。

回答：

我会先区分这个新增词影响的是“配置”还是“程序行为”。

如果只是新增同义词、工具描述、Skill 标签、检索关键词，比如：

- `Claudecode` 和 `Claude Code` 归一化。
- `tool search`、`工具搜索`、`ToolSearch` 归一化。
- 某个工具增加别名。

这种不应该每次都发代码版本，最好外置成配置：

- YAML / JSON / Markdown 配置。
- Skill metadata。
- 检索词典。
- 工具 description。

更新后可以热加载或灰度加载。但我不会完全无验证上线，至少要有：

1. 配置格式校验。
2. 小样本回归测试。
3. 检索命中率或工具路由准确率检查。
4. 版本号和回滚机制。

如果新增词会改变核心解析逻辑、状态机或工具调用参数，那就不能只热更新，要走发版和测试。

一句话总结：

> 词典类变化尽量配置化，不要每次发版；但只要影响 Agent 决策，就要有版本、测试和回滚。

---

## Q6：为什么基于 RAG 设计？RAG 可能有什么问题？

回答：

这里也要换一种说法：我不会把当前项目说成“标准 RAG 系统”，更准确是“检索增强的工程 Agent”。它解决的问题和 RAG 类似：LLM 本身不知道当前项目的真实状态。

它不知道：

- 当前仓库代码是什么。
- 最近 Issue / PR 发生了什么。
- 工具怎么用。
- 测试失败日志是什么。
- 项目文档里的约束是什么。

如果不做检索和证据收集，模型只能凭训练记忆和当前短上下文猜，很容易幻觉。所以这类检索增强设计的作用是把外部事实取回来，让 Agent 基于真实证据做决策。

但 RAG 也有问题：

1. 召回不到：关键文件、日志或文档没被找出来。
2. 召回错：相似但不相关的文件被放进上下文。
3. chunk 切坏：函数、表格、标题、错误栈被切断。
4. 过时：索引没更新，拿到旧代码或旧文档。
5. 上下文污染：召回太多，模型抓错重点。
6. 权限问题：不该给模型看的内容被召回。

所以在 MindClaw / ClawCodex 里，我会把检索增强和这些机制一起用：

- ToolSearch 做工具发现和工具路由。
- `ripgrep` / 文件名 fuzzy search 处理代码、日志、路径、错误栈这类精确匹配。
- session retrieval / memory recall 补充历史上下文。
- 规则过滤控制上下文质量。
- 引用文件路径、行号、日志作为证据。
- 最后用测试、lint、CI 验证结果。

一句话总结：

> 这里不要吹成“我们有完整 RAG 平台”。准确说是：我们用检索和证据收集把 Agent 接到真实工程上下文里，再用工具调用和验证闭环降低幻觉。

---

## Q7：Supervisor 同时调搜索 Agent 和比价 Agent，怎么组织？

原题像电商场景。套到我们项目，可以改成：

> Coordinator 同时调“检索/定位 Agent”和“评估/成本 Agent”，怎么组织？

回答：

我会让 Supervisor / Coordinator 只负责调度和状态，不让它亲自做所有细节。

比如一个任务是“修复某个 provider fallback 问题，并评估不同模型后端的成本和效果”，可以拆成：

- 检索 Agent：查相关代码、Issue、文档、历史修复记录。
- 代码 Agent：根据检索结果修改代码。
- 测试 Agent：运行测试、收集失败日志。
- 评估 Agent：统计模型成本、延迟、成功率或 benchmark 结果。
- Reviewer Agent：检查 diff、风险和遗漏。

Supervisor 做几件事：

1. 定义统一任务目标和验收标准。
2. 给每个 Agent 明确输入输出 schema。
3. 并行调度可以并行的任务，比如检索和成本评估。
4. 把结果写入共享状态或 artifact store。
5. 发现冲突时做裁决，比如代码修改通过测试但成本上升，需要标记风险。
6. 最终生成一个合并报告，而不是把每个 Agent 的长输出直接塞回上下文。

一句话总结：

> Supervisor 不应该变成一个巨型 Agent。它更像状态机和调度器：拆任务、分派、收集结构化结果、处理冲突、触发验证。

---

## Q8：Agent 通信走消息总线还是共享黑板？怎么避免硬编码固定链路？

回答：

我会避免让 Agent 之间直接互相私聊。更稳的是通过 Orchestrator 管控的共享状态或事件流来通信。

可以有两种方式：

1. 消息总线：Agent 产生事件，比如 `CODE_CHANGED`、`TEST_FAILED`、`NEED_MORE_CONTEXT`。
2. 共享黑板：Agent 把结构化产物写到 artifact store，比如检索结果、diff、测试日志、风险列表。

在我们项目里，我更倾向于：

- Agent 不互相依赖具体实现。
- 每个 Agent 只读自己需要的上下文。
- 产物用结构化格式输出。
- Orchestrator 决定下一步派给谁。

这样就不会硬编码成“搜索 Agent 必须调比价 Agent，比价 Agent 必须调总结 Agent”。而是根据当前状态动态决定：

- 如果缺证据，调检索 Agent。
- 如果有 diff，调测试 Agent。
- 如果测试失败，调修复 Agent。
- 如果结果通过，调 Reviewer 或 Summary Agent。

一句话总结：

> 多 Agent 协作不要靠 Agent 互相喊话，而要靠共享状态、结构化产物和 Orchestrator 状态机。这样链路可观测、可回放、可替换。

---

## 面试时的总收束

如果面试官把这些题连续追问，可以用这段收束：

> 这组问题本质上都在问一个点：Agent 系统怎么从 demo 变成稳定工程系统。我的理解是，检索和证据收集负责把真实上下文找回来，Tool / Skill 负责让模型能行动，Orchestrator 负责状态机、重试和回滚，Verification 负责兜底。  
> 在 MindClaw / ClawCodex 里，我不会把所有复杂性都交给 LLM，而是把检索、状态、权限、验证和多 Agent 通信都工程化，这样 Agent 即使单步不稳定，整体系统也能可控地完成任务。
