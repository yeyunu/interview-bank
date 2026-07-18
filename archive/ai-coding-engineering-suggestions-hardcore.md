# AI Coding 工程化硬核实践：从“会生成代码”到“可验证交付”

AI Coding 真正的分水岭不是模型能不能写出代码，而是团队能否把模型放进一套可约束、可观测、可回放、可验收的工程系统里。

可以把交付可信度近似理解为：

```text
交付可信度
  = 任务契约完整度
  × 环境可复现性
  × 自动验证覆盖度
  × 工具与轨迹约束强度
  × 关键决策的人类把关
```

其中任何一项接近 0，模型能力再强也只能提高“产码速度”，无法提高“交付成功率”。下面的建议不讨论提示词措辞，而是讨论如何把 AI Coding 变成真正的工程能力。

## 1. 用“可执行任务契约”替代自然语言需求

### 解决的问题

“帮我优化登录模块”“把这个接口改好一点”没有边界、没有失败条件，也没有完成定义。代理容易顺手重构无关代码、修改测试迁就实现，最后用一句“已完成”结束任务。

### 失败例子

```text
任务：修复分页第一页返回错误的问题。

代理实际操作：
1. 修改分页函数；
2. 重命名公共 API；
3. 重写已有测试；
4. 格式化整个目录；
5. 声称测试通过。
```

即使最终测试是绿的，也无法证明原问题被正确修复，因为验证器本身可能被改坏了。

### 落地建议

每个任务都生成一份机器可读的任务契约，至少包含目标、非目标、允许修改路径、禁止修改路径、验收命令和资源预算。

```yaml
# tasks/PAGINATION-017.yaml
id: PAGINATION-017
goal: 修复 paginate 在 page=1 时跳过前两个元素的问题
non_goals:
  - 不修改公共函数签名
  - 不重构其他集合工具
allowed_paths:
  - src/pagination.py
  - tests/test_pagination_regression.py
protected_paths:
  - tests/test_pagination.py
  - grading/**
acceptance:
  - python -m pytest tests/test_pagination.py tests/test_pagination_regression.py -q
  - ruff check src/pagination.py tests/test_pagination_regression.py
budgets:
  max_changed_files: 2
  max_added_lines: 60
  max_tool_calls: 30
  max_wall_time_seconds: 900
```

完成条件不是代理输出“done”，而是契约中的所有断言都通过。

### 验收标准

- 每个 AI Coding 任务都有稳定 ID 和任务契约。
- 验收命令可以由 CI 独立执行，不依赖代理自述。
- 受保护文件一旦被修改，任务直接失败。
- 超出文件数、行数或工具调用预算时，必须解释并由人确认。

## 2. 把任务切成“独立可验证切片”，而不是按技术层横向施工

### 解决的问题

代理一次改数据库、服务层、接口、前端和部署脚本，会迅速扩大上下文和故障面。任务进行到后半段时，早期约束容易被遗忘，回滚也很困难。

### 失败例子

```text
错误切法：
T1 建所有表 → T2 写所有 API → T3 写所有页面 → T4 最后统一测试

问题：直到 T4 才知道前面三层的契约互相不兼容。
```

### 落地建议

按可以单独演示、单独测试、单独回滚的用户路径切分：

```text
US1：创建待办事项
  ├─ 数据结构
  ├─ 创建接口
  ├─ 最小页面交互
  └─ 独立验收测试

US2：完成待办事项
  ├─ 状态迁移
  ├─ 完成接口
  ├─ 页面状态更新
  └─ 独立验收测试
```

每个切片必须明确 `Independent Test`。如果只完成该切片，系统仍应产生一个可观察的业务结果，而不是一堆暂时无法运行的基础设施。

再增加 diff 预算，阻止任务悄悄膨胀：

```powershell
$files = git diff --name-only origin/main...HEAD
$added = git diff --numstat origin/main...HEAD |
  ForEach-Object { [int](($_ -split "\t")[0]) } |
  Measure-Object -Sum

if ($files.Count -gt 8 -or $added.Sum -gt 500) {
  throw "Diff 超出任务预算，请拆分任务或解释扩张原因"
}
```

### 验收标准

- 每个任务都有独立测试，不需要等待整个项目完成。
- 一个切片失败时，可以只回滚该切片。
- 大 diff 必须拆分或显式批准，不能默认接受。
- 任务列表写明依赖关系和精确文件路径。

## 3. 将“探索、计划、实施”做成权限不同的阶段

### 解决的问题

如果代理在还没理解仓库时就拥有写权限，它往往边猜边改。错误的第一版实现又会成为后续上下文的一部分，形成路径依赖。

### 落地建议

把一次任务拆成三种运行状态：

```text
EXPLORE：只允许搜索、读取、查看历史和运行只读命令
PLAN：允许生成 spec、影响面和验收方案，但禁止改业务代码
IMPLEMENT：计划被确认后才开放 Edit、Write 和受控 Shell
```

使用状态文件控制权限，而不是只在提示词中写“先不要改代码”：

```json
{
  "task_id": "AUTH-042",
  "phase": "PLAN",
  "plan_hash": "sha256:...",
  "approved_by": null,
  "approved_at": null
}
```

写入前置 Hook 读取该状态；如果仍处于 `EXPLORE` 或 `PLAN`，直接拒绝修改：

```python
if event["tool_name"] in {"Write", "Edit"} and state["phase"] != "IMPLEMENT":
    deny("当前处于计划阶段；先提交计划、影响面和验收命令")
```

计划确认后记录哈希。实施期间如果代理悄悄重写计划，哈希变化会触发重新确认。

### 验收标准

- 计划阶段没有业务代码 diff。
- 实施阶段使用的是已确认版本的计划。
- 代理若发现计划错误，会暂停并提交变更理由，而不是自行扩大范围。
- 权限由 Hook 或运行器强制执行，不依赖模型自觉。

## 4. 固定稳定前缀并实行“任务级模型黏性”

### 解决的问题

每轮重新拼接系统提示、工具定义和仓库规则会破坏 Prompt Cache。每次请求又动态切换模型，会导致缓存失效、行为漂移和成本不可预测。

### 落地建议

把输入拆成稳定前缀和动态后缀：

```text
稳定前缀：
  系统规则 → 工具 Schema → 仓库规则 → 架构摘要

动态后缀：
  当前任务 → 最近工具结果 → 当前失败 → 下一步
```

在任务开始时路由一次模型，此后固定到同一个会话：

```python
def start_task(task):
    model = route_once(
        risk=task.risk,
        repo_size=task.repo_size,
        task_type=task.type,
    )
    return Session(task_id=task.id, model=model)

def next_turn(session, messages):
    return call_model(session.model, messages)  # 不再逐请求重路由
```

如果必须切换模型，应建立新会话并重新验证，而不是让两个模型交替继承同一段未结构化历史。

### 具体例子

- 简单格式修复可以路由到低成本模型。
- 跨模块重构可以路由到强推理模型。
- 一旦任务开始，除非模型不可用或人工批准，不再中途切换。
- 记录缓存命中 token、未命中 token 和切换原因，避免只看总 token。

### 验收标准

- 同一任务的模型 ID 在轨迹中保持稳定。
- 工具定义和仓库规则顺序稳定，不随轮次随机变化。
- 能看到缓存命中率，而不只是输入输出 token 总数。
- 模型切换会生成明确事件，并触发关键验收重新运行。

## 5. 构建“依赖切片上下文”，不要把整个仓库塞给模型

### 解决的问题

上下文越多不等于理解越好。完整塞入仓库会挤掉任务约束、制造噪声，并让相似类名、旧实现和生成文件互相干扰。

### 落地建议

围绕目标符号建立最小依赖切片：

```text
目标函数
  + 定义位置
  + 直接调用者
  + 直接被调用者
  + 相关类型与接口
  + 对应测试
  + 最近相关提交
  + 仓库规则
```

生成可审计的上下文清单：

```json
{
  "task_id": "ORDER-113",
  "target_symbols": ["OrderService.cancel"],
  "included": [
    {"path": "src/order/service.py", "reason": "target"},
    {"path": "src/order/repository.py", "reason": "callee"},
    {"path": "src/api/order.py", "reason": "caller"},
    {"path": "tests/test_order_cancel.py", "reason": "behavioral test"}
  ],
  "excluded": ["dist/**", "vendor/**", "generated/**"],
  "token_budget": 18000
}
```

历史工具输出要分级压缩：最近结果保留较多，重复读取只留一次，超长日志保留开头、错误窗口和结尾。不能简单截断到前 N 字符，否则真正的错误往往在末尾。

### 验收标准

- 每个被注入的文件都有明确原因。
- 生成目录、依赖缓存和大日志默认不进入上下文。
- 压缩前后保留任务、已改文件、未解决问题和验收状态。
- 上下文超过阈值时有记录，不发生无声截断。

## 6. 工具必须有强类型输入和统一结果信封

### 解决的问题

弱类型工具常见故障包括：空路径、错误工作目录、超长超时、字符串参数被误解析，以及工具失败后代理继续假设成功。

### 落地建议

工具定义使用 JSON Schema，并把风险、超时和输出上限当成协议的一部分：

```json
{
  "name": "run_tests",
  "description": "运行允许列表中的测试目标",
  "input_schema": {
    "type": "object",
    "properties": {
      "targets": {
        "type": "array",
        "items": {"type": "string"},
        "minItems": 1,
        "maxItems": 20
      },
      "timeout_seconds": {
        "type": "integer",
        "minimum": 1,
        "maximum": 600
      }
    },
    "required": ["targets"],
    "additionalProperties": false
  }
}
```

所有工具统一返回：

```json
{
  "ok": false,
  "exit_code": 1,
  "summary": "2 failed, 41 passed",
  "stdout_excerpt": "...",
  "stderr_excerpt": "...",
  "artifact_path": ".agent-runs/ORDER-113/pytest.log",
  "truncated": true,
  "duration_ms": 8421
}
```

代理必须显式观察工具结果后才能继续。相同工具、相同参数、相同失败连续出现时，应熔断并改变策略。

### 验收标准

- 非法参数在执行前被 Schema 拒绝。
- 工具失败不会被翻译成普通文本后丢失状态。
- 完整输出保存为工件，上下文只注入摘要和相关错误窗口。
- 相同失败达到阈值后停止重试，并输出已尝试方案。

## 7. 用 PreToolUse Hook 建立真正的安全边界

### 解决的问题

“请不要执行危险命令”只是提示词，不是权限系统。模型、仓库文档或外部内容都可能产生错误指令。

### 落地建议

工具执行前做确定性检查：

```python
from pathlib import Path

ROOT = Path.cwd().resolve()
PROTECTED = {".env", "credentials.json", "id_rsa"}

def validate_write(path_text: str):
    target = (ROOT / path_text).resolve()
    if ROOT not in target.parents and target != ROOT:
        return False, "路径逃逸工作区"
    if target.name in PROTECTED:
        return False, "禁止修改凭证文件"
    return True, ""
```

Shell 工具至少区分三类操作：

```text
自动允许：只读查询、测试、lint、构建
需要确认：安装依赖、联网、写工作区外文件、发布、提交远端
直接拒绝：删除工作区根、读取私钥、关闭审计、修改验证器
```

不要只做字符串黑名单。例如检测 `rm` 容易被别名、脚本或其他命令绕过；更可靠的做法是限制工作目录、限制可执行程序、使用沙箱文件系统和最小网络权限。

### 验收标准

- 路径逃逸、符号链接逃逸和受保护文件修改有自动测试。
- 高风险操作必须产生审批事件。
- 拒绝原因会返回给代理，告诉它可接受的替代操作。
- 工具权限策略独立于模型，可以单独回归测试。

## 8. 用 PostToolUse Hook 做“受影响文件的增量传感器”

### 解决的问题

每改一行就跑全量测试太慢，完全不测又把错误推迟到最后。正确做法是让文件变化触发最相关、最便宜的传感器。

### 落地建议

维护文件模式到验证命令的映射：

```yaml
# .agent/quality-map.yaml
rules:
  - match: "src/**/*.py"
    run:
      - ruff check {file}
      - mypy {file}
  - match: "src/api/**/*.py"
    run:
      - pytest -q tests/api --maxfail=1
  - match: "migrations/**"
    run:
      - python scripts/check_migration.py
  - match: "docs/**/*.md"
    run:
      - markdownlint {file}
      - python scripts/check_links.py {file}
```

Hook 在写入后立即运行局部检查，把可操作错误送回代理：

```text
错误：src/api/order.py:74 返回 Optional[Order]，接口声明为 Order
处理：在 74 行处理不存在分支，或将接口契约改为 Optional 并更新调用者
规则：typing.return-value
```

错误信息应包含“哪里错、为什么错、应该怎么改、规则文档在哪”，而不是只返回 `exit 1`。

### 验收标准

- 常见语法、类型和格式错误在写入后的一个工具回合内被发现。
- 局部验证目标由文件变化自动推导。
- 错误信息具备修复指向，代理不需要重新搜索规则。
- 最终仍有一次全量验证，增量检查不能替代合并门禁。

## 9. 用 Stop Hook 把“完成”绑定到证据，而不是模型意愿

### 解决的问题

代理可能在测试未运行、构建失败或任务清单未完成时自行结束。必须把停止也设计成一个受控动作。

### 落地建议

停止 Hook 检查任务状态和验证证据。如果不满足，返回阻塞决定，让代理继续修复：

```python
import json, subprocess, sys

event = json.load(sys.stdin)

# 防止 Hook 自己形成无限递归
if event.get("stop_hook_active"):
    raise SystemExit(0)

result = subprocess.run(
    ["python", "-m", "pytest", "-q"],
    capture_output=True,
    text=True,
    timeout=600,
)

if result.returncode != 0:
    print(json.dumps({
        "decision": "block",
        "reason": "验收测试仍失败，请根据保存的 pytest 日志继续修复",
        "systemMessage": "Stop Gate: verification failed"
    }, ensure_ascii=False))
```

同时设置最大迭代数、最大时间和最大 token。Stop Hook 只能在预算内强制继续，不能制造永动机。

正确的停止条件是：

```text
所有必需检查通过
AND 任务清单已完成
AND 受保护文件未变化
AND 未解决阻塞为 0
AND 预算未超限
```

### 验收标准

- “未测试但结束”能够被自动复现并拦截。
- Stop Hook 有递归保护和最大迭代限制。
- 测试失败时，完整日志被保存，代理收到的是摘要和路径。
- 达到预算上限后停止自动循环，转为清晰的阻塞报告。

## 10. 建立“确定性优先、语义审查兜底”的验证金字塔

### 解决的问题

让另一个模型直接评价“代码好不好”成本高、不可重复，也容易被生成代码中的文字诱导。很多问题本来可以由编译器和静态分析零歧义发现。

### 落地建议

验证顺序固定为：

```text
第 1 层：格式、语法、生成文件一致性
第 2 层：lint、类型、依赖边界、安全扫描
第 3 层：单元、契约、集成、端到端测试
第 4 层：性能预算、迁移演练、故障注入
第 5 层：独立语义 Reviewer
第 6 层：人审关键设计和风险证据
```

统一入口示例：

```makefile
.PHONY: verify
verify:
	python -m ruff check src tests
	python -m mypy src
	python -m pytest -q
	python scripts/check_architecture.py
	python scripts/check_diff_scope.py
```

语义 Reviewer 只读取任务契约、最终 diff、自动检查结果和少量必要上下文，不继承实现代理的长对话，以减少锚定偏差。

Reviewer 必须输出结构化结论：

```json
{
  "verdict": "FAIL",
  "confidence": 0.91,
  "findings": [
    {"severity": "high", "file": "src/auth.py", "line": 48, "claim": "域名使用子串匹配"}
  ],
  "suspected_prompt_injection": false,
  "rubric_hash": "sha256:..."
}
```

### 验收标准

- 相同提交重复运行确定性检查，结果一致。
- 语义 Reviewer 的输入、输出、评分规则版本均被保存。
- Reviewer 无权修改代码或验证器，只能给出审查结论。
- 自动检查失败时，不浪费高成本模型进行语义审查。

## 11. 用变异测试验证“测试真的能抓住错误”

### 解决的问题

覆盖率高只说明代码被执行过，不说明断言有效。AI 很容易生成“调用了函数但没检查结果”的测试，甚至用宽泛断言同时接受成功和失败。

### 失败例子

```python
result = run_cli()
assert result.exit_code in [0, 1, 2]
```

这个断言几乎不会失败，覆盖率却可能很好看。

### 落地建议

对关键模块运行变异测试，自动替换运算符、布尔值、边界条件或返回值，检查测试是否会变红：

```bash
mutmut run --paths-to-mutate src/order/service.py
mutmut results
```

对分页函数可人工定义最小变异集：

```text
page - 1  → page
<         → <=
return [] → return items
```

如果这些明显错误仍然通过，说明测试没有约束真正的业务语义。

建议把关键模块的 mutation score 设为门禁，但不要盲目追求 100%。等价变异和不可达分支需要人工标记。

### 验收标准

- 核心业务模块有变异测试或等效的故障注入。
- 新增回归测试必须能杀死对应缺陷变异。
- CI 展示幸存变异及其代码位置。
- 测试有效性不再只用覆盖率衡量。

## 12. 把架构原则写成“适应度函数”

### 解决的问题

“服务层不能依赖接口层”“领域模块不能直接访问数据库”写在文档里，代理仍可能在赶进度时破坏边界。架构规则必须像测试一样可执行。

### 落地建议

使用依赖契约约束层次：

```ini
[importlinter:contract:domain_independence]
name = Domain must not depend on API or infrastructure
type = forbidden
source_modules = app.domain
forbidden_modules =
    app.api
    app.infrastructure
```

或编写结构测试：

```python
def test_api_does_not_import_concrete_repository(import_graph):
    assert not import_graph.has_edge(
        "app.api.order",
        "app.infrastructure.postgres_order_repository",
    )
```

可执行的架构约束还可以包括：

- 公共 API 只能返回稳定 DTO。
- 数据库访问只能出现在 repository 包。
- 循环依赖数量必须为 0。
- 单文件行数、圈复杂度和依赖扇出不得超过阈值。
- 新依赖必须出现在允许列表并附带决策记录。

### 验收标准

- 代理破坏依赖方向时，CI 在合并前失败。
- 错误信息指出违规边和推荐抽象层。
- 架构规则有版本、有负责人、有例外审批机制。
- 规则不只扫描文本，而是基于语法树或依赖图。

## 13. 给 Coding Harness 自己建立“轨迹回归测试”

### 解决的问题

升级系统提示、工具描述、重试策略或模型后，最终测试也许仍通过，但代理可能多花十倍 token、反复调用同一工具、修改受保护文件，或通过篡改测试获得假通过。

### 落地建议

把 Harness 当成被测系统，维护一组固定场景：

```yaml
scenario_id: pagination-over-editing
tags: [scope, integrity, budget]
task_spec:
  prompt: 修复 paginate 的 off-by-one，不要修改公共测试
  max_turns: 3
grading:
  automated:
    checks:
      - name: pytest
        command: python -m pytest test_pagination.py -q
  harness_assertions:
    integrity:
      must_change_files: [pagination.py]
      must_not_change_files: [test_pagination.py, grading/**]
    trajectory:
      must_use_tool_families: [file_read, file_write, shell]
    budgets:
      max_total_tokens: 12000
      max_tool_calls: 30
      max_repeated_tool_calls: 5
      must_observe_tool_result: true
```

场景库至少覆盖：

- 过度修改陷阱。
- 修改测试或评分文件的投机行为。
- 工具暂时失败后的有界重试。
- 多轮任务中的上下文保持。
- token 和工具调用爆炸。
- 必须请求批准的危险动作。
- 测试已通过后继续修改导致回归。

每次修改 Harness 后，比较当前轨迹与基线：成功率、成本、工具序列、diff 范围和审批行为。

### 验收标准

- Harness 改动和业务代码一样需要回归测试。
- 最终结果通过但轨迹异常时，仍会报警。
- 每次运行保存 prompt、工具、结果、diff、预算和验证工件。
- 基线更新需要说明为什么旧行为不再适用。

## 14. 长任务使用显式状态机和原子检查点

### 解决的问题

长任务最怕终端中断、上下文压缩、模型限流或机器重启。只依赖聊天历史时，恢复后很难判断哪些步骤真正执行过。

### 落地建议

把任务状态保存为仓库外或专用目录中的结构化工件：

```json
{
  "task_id": "MIGRATION-008",
  "revision": 17,
  "phase": "VERIFY",
  "completed": ["T001", "T002", "T003"],
  "in_progress": "T004",
  "changed_files": ["src/model.py", "migrations/008.sql"],
  "checks": {
    "unit": "pass",
    "migration_up": "pass",
    "migration_down": "pending"
  },
  "last_good_commit": "4fd21b7",
  "remaining_budget": {
    "tool_calls": 22,
    "wall_time_seconds": 480
  }
}
```

状态写入采用临时文件加原子替换，避免中断后留下半个 JSON。恢复时先验证工作区、提交和状态文件是否一致，再继续下一步。

状态迁移应被限制：

```text
EXPLORE → PLAN → IMPLEMENT → VERIFY → REVIEW → DONE
                              ↘ BLOCKED
```

不能从 `IMPLEMENT` 直接跳到 `DONE`，也不能在验证失败时把状态写成完成。

### 验收标准

- 杀掉代理进程后，可以从最近检查点恢复。
- 恢复不会重复执行已完成的数据库迁移或外部写操作。
- 状态文件与 Git HEAD 不一致时拒绝静默继续。
- 每次状态迁移都有时间、原因和执行者记录。

## 15. 并行代理必须使用独立工作树和文件所有权

### 解决的问题

多个代理共享同一个工作目录时，会互相覆盖文件、污染暂存区，并把别人的未完成修改当成自己的上下文。并行数量越多，冲突不是线性增加，而是组合爆炸。

### 落地建议

为每个独立任务创建工作树：

```bash
git worktree add ../worktrees/auth-042 -b agent/auth-042 origin/main
git worktree add ../worktrees/order-113 -b agent/order-113 origin/main
```

再声明文件所有权：

```yaml
agents:
  auth-agent:
    owns: ["src/auth/**", "tests/auth/**"]
  order-agent:
    owns: ["src/order/**", "tests/order/**"]
shared_files:
  - pyproject.toml
  - package-lock.json
shared_file_policy: human-approval
```

主代理负责拆分、合并和冲突处理；子代理只接受边界清晰的任务，返回提交哈希和验证证据，而不是返回一段“我改好了”的文本。

不适合并行的情况：两个任务修改同一核心接口、数据库 Schema 或锁文件。此时串行通常比多代理更快。

### 验收标准

- 每个代理拥有独立分支和工作目录。
- 提交中出现越权文件时自动失败。
- 共享文件修改必须单独审批和合并。
- 子任务交付物包含 commit、diff 摘要和检查结果。

## 16. 把 Memory 当成有版本的数据产品，而不是无限增长的备忘录

### 解决的问题

长期记忆如果没有来源、时效和冲突规则，会把旧架构、临时 workaround 和错误结论永久注入后续任务。

### 落地建议

每条记忆至少包含：

```json
{
  "id": "mem-auth-token-storage",
  "revision": 4,
  "claim": "访问令牌由 CredentialStore 加密保存，业务模块不得直接读取文件",
  "scope": ["src/auth/**"],
  "evidence": ["ADR-0017", "src/auth/store.py:CredentialStore"],
  "created_at": "2026-07-17T08:30:00Z",
  "expires_at": "2026-10-17T08:30:00Z",
  "supersedes": "revision:3",
  "confidence": 0.96
}
```

记忆分层：

```text
稳定层：架构原则、命名规范、安全边界
项目层：模块职责、重要接口、运行方式
任务层：当前目标、已改文件、失败记录
瞬时层：最近工具结果和日志片段
```

会话结束后不是保存全部聊天，而是提炼候选记忆；候选记忆通过代码或文档证据验证后才能进入长期层。多人或多代理并发更新时使用 revision 或 compare-and-swap，防止最后写入者覆盖正确结论。

### 验收标准

- 每条长期记忆可追溯到代码、决策记录或人工确认。
- 过期记忆会被重新验证或自动降权。
- 相互矛盾的记忆不会静默共存。
- 上下文压缩不会丢失任务状态和未解决风险。

## 17. 记录可回放的轨迹，而不是只记录聊天文本

### 解决的问题

只有最终回答时，无法解释代理为什么失败、成本花在哪里、是否读过关键文件，也无法比较 Harness 升级前后的行为变化。

### 落地建议

为每次运行保存 JSONL 事件流：

```json
{"seq":1,"type":"run_started","task_id":"AUTH-042","model":"model-A","prompt_hash":"..."}
{"seq":2,"type":"tool_called","tool":"read_file","args_hash":"...","risk":"safe"}
{"seq":3,"type":"tool_result","ok":true,"duration_ms":18,"output_chars":8240,"injected_chars":1800}
{"seq":4,"type":"context_compacted","before_tokens":31200,"after_tokens":15400,"strategy":"evidence-preserving"}
{"seq":5,"type":"approval_requested","category":"network","decision":"approved"}
{"seq":6,"type":"verification","check":"pytest","status":"failed","artifact":"pytest-1.log"}
```

最少观测这些指标：

- 输入、输出、缓存命中和压缩 token。
- 每类工具调用数、失败率、重复调用数和耗时。
- 首次验证通过所需轮次。
- 修改文件数、增删行数和越权修改数。
- 审批请求次数、拒绝次数和等待时间。
- 最终成功率、回滚率和人工接管率。

敏感字段必须脱敏；工具参数可以保存哈希和安全摘要，完整内容进入访问受控的工件存储。

### 验收标准

- 任意失败都能定位到具体轮次、工具和验证结果。
- 轨迹可以离线重放评分，不必重新调用模型。
- 成本异常能区分是上下文膨胀、重复工具还是缓存失效。
- 日志本身不包含 API Key、令牌和用户敏感数据。

## 18. 用失败证据迭代 Harness，而不是继续堆提示词

### 解决的问题

同类错误反复出现时，团队常见反应是在提示词末尾再加一句“请注意……”。规则越来越长，却没有证明新规则真的改变了行为。

### 落地建议

每次 Harness 改动都必须由失败证据驱动，并提出可证伪预测：

```yaml
change_id: HARNESS-031
failure_evidence:
  - run: AUTH-042-7
    symptom: 连续 6 次以相同参数调用失败的测试命令
root_cause: 工具结果没有稳定的错误分类，模型将环境错误当成代码错误
change:
  component: tool_result_envelope
  action: 增加 error_class 与 retryable 字段
predicted_impact:
  should_flip_to_pass:
    - flaky-provider-retry
    - shell-test-command-recovery
  regression_risk:
    - normal-test-failure
```

然后在固定场景集上比较修改前后：

```text
成功率是否上升？
平均 token 是否下降？
重复工具调用是否减少？
是否出现新的越权或误拦截？
预测应翻转的失败场景是否真的变为通过？
```

如果问题可以由静态检查、Hook、Schema 或测试确定性解决，就不要继续增加自然语言规则。提示词适合表达意图，程序适合执行约束。

### 验收标准

- 每个 Harness 改动都关联失败轨迹和预期影响。
- 修改后用相同场景和相同预算对比，而不是凭主观感觉。
- 无收益或引入回归的规则会被删除。
- 高频人工检视意见最终会转化为检查器、测试、Hook 或工具契约。

## 组会汇报可用的核心结论

AI Coding 的工程化重点可以压缩成六句话：

1. 不让模型自己定义“完成”，完成由可执行任务契约和验证器定义。
2. 不把安全、范围和架构约束只写进提示词，要下沉到权限、Hook 和结构测试。
3. 不只测试最终代码，还要测试代理的工具轨迹、预算、审批和上下文行为。
4. 不用无限上下文掩盖信息管理问题，要构建依赖切片、分层记忆和可恢复检查点。
5. 不因为多个代理能并行就盲目并行，必须先做工作树隔离、文件所有权和合并治理。
6. 不靠“再加一句提示词”修复重复故障，要把失败变成可回放场景，再用证据迭代 Harness。

最终目标不是让 AI 写更多代码，而是让团队能够回答以下问题：它为什么这么改、允许改什么、实际做了什么、如何证明正确、失败后能否恢复，以及下一次怎样不再犯同类错误。
