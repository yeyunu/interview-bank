# AI Coding 工程化硬核实践 · 出处与依据

> 配套文件：[`ai-coding-engineering-suggestions-hardcore.md`](./ai-coding-engineering-suggestions-hardcore.md)

## 总述

核心机制来自官方文档、开源代码和真实工程实践。文档里的 YAML、Python、JSON 示例多数是我根据这些机制重新编写的落地示例，不是从原帖逐字复制，也不保证不改配置就能直接运行。

## 最主要的第一手依据

### 1. Spec 驱动、任务契约、独立验收

对应文档第 1、2、3 条。

Spec Kit 的真实工作流是：

constitution → specify → plan → tasks → implement

其模板明确要求：

- 用户故事必须可以独立测试。
- 使用 Given/When/Then 编写验收场景。
- 需求中标记 NEEDS CLARIFICATION。
- Plan 阶段执行 Constitution Check。
- Task 必须包含精确文件路径、依赖关系和可并行标记。

直接依据：

- Spec Kit 官方仓库 (https://github.com/github/spec-kit)
- spec-template.md (https://github.com/github/spec-kit/blob/57cc518d63d6f10da3dd93df1ebcadda87c59374/templates/spec-template.md)
- plan-template.md (https://github.com/github/spec-kit/blob/57cc518d63d6f10da3dd93df1ebcadda87c59374/templates/plan-template.md)
- tasks-template.md (https://github.com/github/spec-kit/blob/57cc518d63d6f10da3dd93df1ebcadda87c59374/templates/tasks-template.md)
- 项目 Constitution 实例 (https://github.com/github/spec-kit/blob/57cc518d63d6f10da3dd93df1ebcadda87c59374/.specify/memory/constitution.md)

文档里的 PAGINATION-017.yaml 是把这些要求与 Harness 场景 Schema 合并后写出的示例，不是官方原文件。

### 2. PreToolUse、PostToolUse 和 Stop Hook

对应第 3、7、8、9 条。

这些不是概念想象，Claude Code 官方仓库里有真实实现：

- PreToolUse 在执行前检查命令，退出码 2 可以阻止工具调用。
- PostToolUse 可以在文件修改后执行 lint 或质量检查。
- Stop 可以阻止代理结束，让它继续完成验证。
- Stop 循环必须设置最大迭代次数、完成条件和状态文件。
- Hook 有递归风险，因此需要检查是否已经处于 Stop Hook 循环。

直接依据：

- Bash 命令校验 Hook (https://github.com/anthropics/claude-code/blob/67f390c9a0b1440d369aebe2ff6a5023db35bf8e/examples/hooks/bash_command_validator_example.py)
- Hook 工程模式 (https://github.com/anthropics/claude-code/blob/67f390c9a0b1440d369aebe2ff6a5023db35bf8e/plugins/plugin-dev/skills/hook-development/references/patterns.md)
- Ralph Stop 循环源码 (https://github.com/anthropics/claude-code/blob/67f390c9a0b1440d369aebe2ff6a5023db35bf8e/plugins/ralph-wiggum/hooks/stop-hook.sh)
- Hookify 配置 (https://github.com/anthropics/claude-code/blob/67f390c9a0b1440d369aebe2ff6a5023db35bf8e/plugins/hookify/hooks/hooks.json)

文档中的 stop_gate.py 是我把官方 Bash Stop Hook 简化成 Python 版本的示范。

### 3. Harness 轨迹回归、预算与防投机验证

对应第 6、10、13、17 条。

参考的 Harness 测试 Schema 明确区分三层评分：

**确定性命令：** test、lint、typecheck、build

**Harness 结构断言：** 必须修改哪些文件、禁止修改哪些文件、必须使用哪些工具、最大 token、工具次数和时间、是否观察工具结果

**可选语义审查：** 保存评分 Prompt、响应、结果和 rubric_hash

仓库里还有真实的陷阱场景：

- 修改测试制造假通过。
- 超出任务范围修改文件。
- token 和工具调用爆炸。
- 三轮任务中遗忘前两轮功能。
- 测试通过后继续修改导致回归。
- 验证器文件被篡改。

直接依据：

- Harness Scenario Schema (https://github.com/Abbiirr/harness-tester/blob/a4768866485e621beadbb2540d601ac2a8a119dc/docs/scenario-schema.md)
- Token 与工具循环爆炸场景 (https://github.com/Abbiirr/harness-tester/blob/a4768866485e621beadbb2540d601ac2a8a119dc/scenarios/token-tool-loop-blowup.yaml)
- 越界修改场景 (https://github.com/Abbiirr/harness-tester/blob/a4768866485e621beadbb2540d601ac2a8a119dc/scenarios/over-editing-trap.yaml)
- 篡改验证器场景 (https://github.com/Abbiirr/harness-tester/blob/a4768866485e621beadbb2540d601ac2a8a119dc/scenarios/verifier-sabotage-trap.yaml)
- 三轮上下文保持场景 (https://github.com/Abbiirr/harness-tester/blob/a4768866485e621beadbb2540d601ac2a8a119dc/scenarios/three-turn-context-retention.yaml)

这里需要说明：该仓库目前影响力不大，所以把它当作"具体实现参考"，没有把它当成行业标准。它的价值在于 Schema 和场景可以直接检查。

### 4. 上下文压缩、工具类型、会话恢复和受限子代理

对应第 5、6、14、16 条。

一个可读的 Coding Agent 实现把 Harness 分成六个组件：

1. 实时仓库上下文。
2. 稳定 Prompt 前缀与缓存复用。
3. 结构化工具、参数验证和权限。
4. 上下文裁剪与工具输出管理。
5. Transcript、Memory 与会话恢复。
6. 有深度限制的只读子代理。

代码里可以直接看到：

- MAX_TOOL_OUTPUT 和 MAX_HISTORY。
- 旧工具结果和最近结果采用不同裁剪长度。
- 重复的旧文件读取会被折叠。
- 会话保存为 JSON。
- 高风险工具要求审批。
- 子代理有 max_depth 和 max_steps。
- 相同无效工具调用不应反复执行。

直接依据：

- Mini Coding Agent 源码 (https://github.com/rasbt/mini-coding-agent/blob/717cae4ff10d01773bd12951f62a575825053414/mini_coding_agent.py)
- Mini Coding Agent 仓库 (https://github.com/rasbt/mini-coding-agent)

文档里的 context-pack.json 和带 revision、TTL、evidence 的 Memory Schema，是基于这些原则进一步做的工程化设计，不是该仓库原样提供的格式。

### 5. Prompt Cache 与稳定前缀

对应第 4 条。

官方文档明确说明：

- 缓存对象是完整 Prompt 前缀。
- 前缀顺序是 tools → system → messages。
- 修改缓存断点之前的任意内容，会产生不同哈希。
- 静态内容应放在前面，动态内容放在后面。
- 每轮变化的时间戳、动态字段如果出现在缓存前缀中，会导致缓存无法命中。

依据：

- Prompt Caching 官方文档 (https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)

"模型黏性"部分还参考了一条生产模型路由讨论：不要每次调用都重新选择模型，应在任务开始时路由并绑定会话，避免缓存局部性被破坏。

- Model affinity 讨论 (https://x.com/_avichawla/status/2075888860115120424)

这里"固定稳定前缀"有官方机制支撑；"任务级模型黏性"属于基于缓存机制和生产路由经验做出的工程建议，不是所有平台强制规定。

### 6. 变异测试

对应第 11 条。

变异测试通过修改整数、边界比较、返回值等方式制造缺陷，再检查现有测试能不能发现它。当前 Mutmut 文档明确举例：

- 整数常量发生变化。
- < 变成 <=。
- 运行变异后查看幸存变异。
- 为幸存变异补充真正有效的断言。

依据：

- Mutmut 官方仓库 (https://github.com/boxed/mutmut)
- Mutmut README (https://github.com/boxed/mutmut/blob/32f0b4269168840ab8de3fd5c56089b3f39c2840/README.rst)

复核时发现文档中的 `mutmut run --paths-to-mutate src/order/service.py` 属于版本相关写法。当前版本更推荐在配置中设置 source_paths，然后执行 mutmut run，或者使用模块过滤形式。因此概念有依据，但这一条命令需要按实际安装版本调整。

### 7. 架构适应度函数

对应第 12 条。

Import Linter 本身就是把 Python 架构规则做成自动检查，支持：

- Forbidden contract。
- Layers contract。
- Independence contract。

依据：

- Import Linter 官方仓库 (https://github.com/seddonym/import-linter)
- Forbidden Contract 文档 (https://import-linter.readthedocs.io/contract_types/forbidden/)
- Layers Contract 文档 (https://import-linter.readthedocs.io/contract_types/layers/)

文档里的禁止 domain → api/infrastructure 依赖配置，是按 Import Linter 的真实契约格式写的示例。

### 8. 独立 Worktree 与并行代理隔离

对应第 15 条。

Git 官方文档确认，一个仓库可以拥有多个 linked worktree；各工作树拥有独立的 HEAD 和工作目录，同时共享对象数据库和部分 refs。

依据：

- Git Worktree 官方文档 (https://git-scm.com/docs/git-worktree)

"每个代理一个 worktree"是把 Git 的真实隔离能力应用到多代理并行开发。文件所有权 YAML 是我增加的治理层，不是 Git 原生功能。

### 9. 可观测轨迹和 Harness 迭代

对应第 17、18 条。

Agentic Harness Engineering 的实现强调：

evaluate → analyze → improve

它把每次任务的 step-level trace、工具调用、中间件事件和 verifier 结果保存下来。后续优化不是只看最终成功率，而是从失败轨迹提取证据，提出 Harness 修改，并预测哪些任务应该由失败翻转为成功。

依据：

- Agentic Harness Engineering 仓库 (https://github.com/china-qijizhifeng/agentic-harness-engineering)
- 固定版本 README (https://github.com/china-qijizhifeng/agentic-harness-engineering/blob/faf44bc4aea57413c520bc5711c6ebf628e0da1e/README.md)

文档里的 JSONL 事件格式和 HARNESS-031.yaml 是我根据这种"轨迹—证据—可证伪修改"流程设计的简化版本。

## 社交平台内容起了什么作用

我看到的代表性内容包括：

- 四类 Agent 循环：turn、goal、time、proactive (https://x.com/akshay_pachaar/status/2076748259377516782)
- Spec Kit 的结构化开发过程 (https://x.com/crptAtlas/status/2076754607049449633)
- 小任务、明确边界和隐藏依赖问题 (https://x.com/0xCodila/status/2077894002536226856)
- AI Coding 反思 (https://www.xiaohongshu.com/explore/6a3cd41a000000001101fb65)
- Loop Engineering 与 Harness Engineering (https://www.xiaohongshu.com/explore/6a34f8df000000000f004474)
- Agent 长任务工程化 (https://www.xiaohongshu.com/explore/6a0bf2cc0000000035033e85)
- AI Coding 工程化与 SDD (https://www.xiaohongshu.com/explore/6a3924a9000000000f032c2e)
- AI Coding 工程化范式转移 (https://www.xiaohongshu.com/explore/69c63b22000000001a02679a)

这些帖子让我确定应该重点写 Harness、Spec、长任务、上下文和验证，而具体落地细节主要用上面的源码与官方文档交叉验证。

## 最坦诚的结论

不是捏造，但也不是"18 条全部来自某一篇权威文章"。

更准确地说：

- Spec、Hook、Stop Loop、Prompt Cache、Worktree、变异测试、架构契约：有直接官方实现或官方文档。
- 轨迹预算、防篡改场景、Harness 回归：有开源实现和具体 Schema。
- Memory 版本、文件所有权、任务状态 JSON、质量映射表：是我将多个可靠机制组合后的工程设计。
- 文档中的数值，例如 30 次工具调用、500 行 Diff、18K 上下文预算，都是示例阈值，需要根据项目基线调整，不能当作行业标准。
