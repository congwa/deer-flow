# `<subagent_system>` 把编排规则写进提示词，用 LLM 推理代替独立规划服务

## 要解决的问题

先说两个基本事实：

1. **模型本身没有「编排」的概念**。没有任何训练让它知道「遇到复杂任务就拆成并行子任务、发多个 `task` 调用」这种工作方式。不加引导的话，模型会默认一步一步地自己干，把所有中间步骤都放进主对话。

2. **工程上不想另外维护一套规划服务**。另起一个服务专门分析用户意图、输出执行计划、管理子任务依赖关系，工程量大且引入新的故障点。

`<subagent_system>` 是解决第一个问题的方案：**把编排规则、角色定义、工作流程全部写成自然语言注入系统提示**，让模型在每次处理请求时，按照这些规则自己决策。

解决不了的问题（代码层处理）：模型有时候就是不遵守提示词，这时候 `SubagentLimitMiddleware` 在模型输出之后物理截断超量的 `task` 调用——两层各守一道。

---

## 模型凭什么知道「能拆/不能拆」

这是很多人困惑的地方：代码里没有任何「分析用户问题是否可并行」的逻辑，那模型怎么做出这个判断？

答案是：**模型读到的信息本身就包含了做判断所需的全部内容**，再加上 `<subagent_system>` 给它一套判断标准。

模型每次生成回复前，上下文里有：

```
1. 用户这轮说了什么
      ↓ 是否有「比较 A/B/C」「从多角度看」「同时分析」这类结构？
      ↓ 如果有 → 多维任务，考虑拆

2. 之前的对话历史（messages 列表）
      ↓ 前几轮是否已经澄清清楚了？是否有中间结论？
      ↓ 如果任务刚开始、没有背景 → 先澄清或直接拆

3. <subagent_system> 里的判断标准
      ↓ 提供了「✅ 什么情况拆 / ❌ 什么情况不拆」的分类
      ↓ 提供了三个具体例子供类比
      ↓ 提供了计数 + 分批的思维模式
```

没有任何算法在背后做这个判断，**判断的过程就是模型的推理过程**。`<subagent_system>` 的作用是给这个推理过程提供「正确答案长什么样」的参照。

---

## `<subagent_system>` 是怎么出现在提示词里的

源码在 `agents/lead_agent/prompt.py`，函数名叫 `_build_subagent_section(max_concurrent: int)`：

```python
def _build_subagent_section(max_concurrent: int) -> str:
    n = max_concurrent   # 默认 3，从运行时配置读取
    return f"""<subagent_system>
    ...完整内容，所有「{n}」处动态填入实际并发上限...
    </subagent_system>"""
```

然后在 `apply_prompt_template` 里按条件拼入：

```python
subagent_section = _build_subagent_section(n) if subagent_enabled else ""
```

**`subagent_enabled`** 这个开关同时控制两件事：
- 提示词里有没有 `<subagent_system>` 这段；
- `get_available_tools` 里有没有 `task` 工具。

两者必须同步，不能出现「提示词说可以用 task 但工具不存在」或「工具存在但提示词没有说怎么用」。

除了 `{subagent_section}` 这个主块，同一个开关还会在两处追加更短的约束：

- **`<thinking_style>` 里的 `{subagent_thinking}`**：要求模型在思考阶段就数清楚子任务
- **`<critical_reminders>` 里的 `{subagent_reminder}`**：对并发上限的简短重申

三处合计每次系统提示里关于子 Agent 的约束出现三次，这是刻意的——越关键越要重复。

---

## `<subagent_system>` 的完整内容结构

下面是提示词的完整逻辑结构（中文概括原文英文，保留关键词）：

```
<subagent_system>
│
├── 角色定义（Role）
│     你是任务编排器（task orchestrator），角色只有三件事：
│     DECOMPOSE → DELEGATE → SYNTHESIZE
│
├── 并发上限（Hard Limit）
│     每轮回复最多 N 个 task 调用（N 从配置读取，默认 3）
│     超出的被系统静默丢弃，你会丢失那部分工作
│     发之前必须在思考里数清楚：
│       ≤ N → 本轮全发
│       > N → 本轮发最重要的 N 个，剩下留下一轮
│     多批执行模板：
│       轮 1：发第一批 → 等结果
│       轮 2：发下批 → 等结果
│       ...
│       最后一轮：汇总所有批次的结果
│
├── 可用子 Agent 类型（Available Subagents）
│     general-purpose：研究、代码探索、文件操作、分析等广用途
│     bash：git、构建、测试、部署等命令执行
│
├── 编排策略（Orchestration Strategy）
│     ✅ 应该用并行子 Agent：
│       - 复杂研究问题（需要多信息来源）
│       - 多维分析（任务有多个独立方向）
│       - 大型代码库（需同时看多块区域）
│       - 全面调查（需从多角度充分覆盖）
│     ❌ 不应该用子 Agent：
│       - 拆不出 2+ 个有意义的并行子任务
│       - 极简单操作（读一个文件、改一处代码）
│       - 需要先向用户澄清
│       - 关于对话历史本身的元问题
│       - 强串行依赖（后步必须等前步结果）
│
├── 工作流程（Critical Workflow，STRICTLY 必须遵守）
│     1. COUNT：在思考里列出所有子任务并数清楚个数
│     2. PLAN BATCHES：若 > N 则明确规划哪些进哪批
│     3. EXECUTE：只发当前批（最多 N 个 task）
│     4. REPEAT：结果回来后再发下批
│     5. SYNTHESIZE：所有批完成后汇总
│     6. 无法分解 → 直接用 bash/read_file/web_search 等工具执行
│
├── 执行机制说明（How It Works）
│     task 工具在后台异步跑子 Agent
│     后端自动轮询完成状态（不需要你手动轮询）
│     从你的角度看：task() 是一个同步调用，发出去等结果
│
└── 代码示例（Usage Examples）
      单批示例：腾讯股价（3 个子任务，一轮搞定）
      多批示例：比较 5 家云厂商（5 个子任务，两轮发完）
      反例：「跑测试」→ 直接 bash()，不需要 task()
</subagent_system>
```

---

## 三步核心流程的完整执行路径

以用户问「为什么腾讯股价下跌」为例，追踪完整的执行过程：

```
用户：「分析腾讯股价下跌原因」
    │
    ▼
模型读上下文：
  - 用户的问题
  - 历史消息（若有）
  - 系统提示里的 <subagent_system>
    │
    ▼
模型在思考区推理（这部分用户看不到）：
  「用户问股价原因，可以从财务、舆情、行业三个方向并行研究。
   共 3 个子任务，等于 N（限制为 3），本轮全发。」
    │
    ▼
─────── DECOMPOSE ───────
模型决定：拆成 3 个子任务
  1. 近期财报、营收数据、业绩趋势
  2. 负面新闻、监管政策、争议事件
  3. 行业走势、竞品表现、市场情绪
    │
    ▼
─────── DELEGATE ───────
模型生成一条 AIMessage，里面包含 3 个 task 工具调用：

AIMessage.tool_calls = [
  {name:"task", args:{description:"财务数据", prompt:"分析腾讯近三季营收...", subagent_type:"general-purpose"}},
  {name:"task", args:{description:"舆情监管", prompt:"搜集负面新闻和监管动态...", subagent_type:"general-purpose"}},
  {name:"task", args:{description:"行业趋势", prompt:"分析竞品和市场整体...", subagent_type:"general-purpose"}},
]
    │
    │    （SubagentLimitMiddleware 检查：3 ≤ 3，不截断）
    │
    ▼
框架并发执行 3 个 task：

  task_tool("财务数据", ...)  task_tool("舆情监管", ...)  task_tool("行业趋势", ...)
       │                           │                          │
       ▼                           ▼                          ▼
  子 Agent A               子 Agent B               子 Agent C
  （独立的消息历史）        （独立的消息历史）        （独立的消息历史）
  用 web_search/bash        用 web_search            用 web_search
  查财务数据                查新闻和监管              查竞品数据
       │                           │                          │
       ▼                           ▼                          ▼
  返回结果字符串              返回结果字符串              返回结果字符串
    │           │           │
    └───────────┼───────────┘
                │
    ▼（三个 ToolMessage 都回来后，模型才能继续）

ToolMessage(tool_call_id="call_1", content="财务分析：Q1-Q3 营收连续下滑...")
ToolMessage(tool_call_id="call_2", content="舆情分析：监管收紧、游戏版号问题...")
ToolMessage(tool_call_id="call_3", content="行业分析：字节跳动市场份额上升...")
    │
    ▼
─────── SYNTHESIZE ───────
模型拿到三份材料，生成最终回复：
「腾讯股价下跌主要有三方面原因：
  1. 财务端：近三季营收环比下降...
  2. 监管端：游戏行业强化审批...
  3. 竞争端：短视频平台分流用户...」
    │
    ▼
用户看到最终答案（也看到了三个子 Agent 跑过的 task_running 事件）
```

---

## 多批执行的完整流程（子任务 > N 时）

以「比较 5 家云厂商」为例（N=3）：

```
轮 1
  模型思考：「5 个子任务 > 3，必须分两批。先发 AWS/Azure/GCP。」
  发出 3 个 task 调用
    │
    ▼
  task(AWS)  task(Azure)  task(GCP)   ← 3 个并行跑
    │           │            │
    └───────────┼────────────┘
                ▼（等待）
  3 个 ToolMessage 返回
    │
    ▼
轮 2
  模型看到前 3 个结果，记起「还有阿里云和甲骨文」
  再发 2 个 task 调用
    │
    ▼
  task(阿里云)  task(甲骨文)   ← 2 个并行跑
    │               │
    └───────────────┘
                ▼（等待）
  2 个 ToolMessage 返回
    │
    ▼
轮 3
  模型拥有 5 份材料，汇总成综合比较报告
```

这里「模型记起还有两个没发」的能力依赖于：
- 第一轮结果的 ToolMessage 都在 `messages` 历史里
- 系统提示里「COUNT → PLAN BATCHES」流程让模型在第一轮就规划好了分批

如果模型第一轮规划时没写好批次，第二轮可能会忘掉还有任务——这是提示词约束（软）的局限性。

---

## 从模型视角看 task 工具

这段出现在 `<subagent_system>` 里，专门为了防止模型误解执行机制：

```
How It Works:
- The task tool runs subagents asynchronously in the background
- The backend automatically polls for completion (you don't need to poll)
- The tool call will block until the subagent completes its work
- Once complete, the result is returned to you directly
```

为什么需要说这段？因为有时候模型会联想「既然是异步的，我应该先发任务、然后轮询状态」——这是错误的行为，之前版本里有 `task_status` 工具专门用来轮询，现在已经废弃。

现在的设计是：**后端的 `task_tool` 函数内部有一个 `while True` 轮询循环**，它对模型透明。模型只看到「我发了 `task()`，一段时间后 ToolMessage 回来了」，不知道中间经历了 5 秒/10 秒/15 秒……的等待。

从模型角度：`task()` 和 `bash()` 的使用体验一样，同步调用，结果回来才能继续。

---

## 三重强化：为什么同一条规则要出现三次

「每轮最多 N 个 task」是最重要的一条规则，它出现在三个不同位置：

**位置 1：`<subagent_system>` 主块**（在功能说明里）

```
⛔ HARD CONCURRENCY LIMIT: MAXIMUM {n} `task` CALLS PER RESPONSE. THIS IS NOT OPTIONAL.
Any excess calls are silently discarded by the system — you will lose that work.
```

用大写字母和 ⛔ 图标，在功能介绍时就先建立认知。

**位置 2：`<thinking_style>` 里**（在思考阶段检查清单里）

```
- DECOMPOSITION CHECK: Can this task be broken into 2+ parallel sub-tasks? If YES, COUNT them.
  If count > {n}, you MUST plan batches of ≤{n} and only launch the FIRST batch now.
  NEVER launch more than {n} `task` calls in one response.
```

模型在思考阶段就要做这个检查，而不是等生成工具调用时才想起。

**位置 3：`<critical_reminders>` 里**（在最终提醒清单里）

```
- Orchestrator Mode: ... HARD LIMIT: max {n} `task` calls per response.
  If >{n} sub-tasks, split into sequential batches of ≤{n}.
```

最后一道提醒，在所有其他规则之后再确认一次。

**这样做的原因**：模型的「注意力」在长提示词里是分散的，越靠后、越孤立的规则越容易被忽视。把同一条规则放在不同的语境里重复出现，覆盖了「读功能描述时」「思考时」「最后检查时」三个时机。

---

## 提示词约束（软）和代码约束（硬）的分工

| 约束内容 | 提示词做了什么 | 代码做了什么 |
|---------|--------------|------------|
| 每轮最多 N 个 task | 三次重申，示例说明，分批方案 | `SubagentLimitMiddleware.after_model()` 截断超量调用 |
| 子 Agent 不能再委派 | 无专门说明（靠工具不存在） | `get_available_tools(subagent_enabled=False)` 不给子 Agent 加 `task` |
| 先澄清再行动 | `<clarification_system>` 五种场景规则 | `ClarificationMiddleware` 拦截，`Command(goto=END)` 中断 |
| 不用 task 包单一任务 | 反例说明，「单任务=直接执行」 | 无对应代码约束（靠提示词） |
| 分批后汇总 | 工作流程五步 | 无对应代码约束（靠提示词） |

提示词负责引导正常情况，代码负责兜底极端情况。如果模型生成了 5 个 task（超出限制），`SubagentLimitMiddleware` 删掉最后 2 个；模型下一轮收到 3 个 ToolMessage，会发现少了 2 个结果，再发第二批——前提是它能记住第一轮说要发 5 个。这个「记住」同样靠提示词里的分批规划引导，不是代码保证的。

---

## `<subagent_system>` 在整个系统提示词里的位置

```
系统提示词的完整顺序：
  ┌──────────────────────────────────────┐
  │ <role>          ← 你是谁             │
  │ {soul}          ← 你的个性（可选）   │
  │ {memory_context}← 你记得什么（可选） │
  │ <thinking_style>← 怎么思考问题       │ ← 这里已经插入了 subagent_thinking
  │ <clarification> ← 什么时候先问用户  │
  │ {skills_section}← 有哪些技能（可选） │ ← 先介绍工具/技能
  │ {deferred_tools}← 有哪些延迟工具     │ ← 再介绍工具列表
  │ {subagent_section} ← 这里            │ ← 然后才讲怎么编排这些工具
  │ <working_directory>← 文件目录规则    │
  │ <response_style>← 怎么回复用户       │
  │ <citations>     ← 引用格式           │
  │ <critical_reminders>← 最终提醒       │ ← 这里再次出现 subagent_reminder
  │ <current_date>  ← 今天是几号         │
  └──────────────────────────────────────┘
```

`<subagent_system>` 放在技能区和延迟工具区之后，是因为「怎么编排子任务」依赖「知道有哪些工具和技能」才有意义。先让模型知道自己能用什么工具，再告诉它面对复杂任务时怎么分配这些工具。

---

## 这段提示词的设计局限

坦诚说，这套设计有清晰的边界：

**可以保证的**：模型在大多数标准场景下按三步骤（分解/委派/汇总）工作，不会把所有中间步骤堆在主对话。

**不能保证的**：
- 模型对「能不能并行」的判断在非标准情况下可能出错（强行并行有依赖的子任务）
- 模型在超复杂任务时可能遗忘分批计划，第二批没有被发出来
- 分批后的汇总质量取决于模型能否理解 N 份不同来源的结果并整合

**代码层无法弥补这些局限**——代码只能保证「超出了就截断、子 Agent 没有 task 工具」，不能保证分析判断的正确性。这是「用提示词代替规划器」这一设计选择带来的固有权衡，第 14 篇文章里有更完整的讨论。

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `agents/lead_agent/prompt.py` | `_build_subagent_section(n)` — 生成完整的 `<subagent_system>` 块 |
| `agents/lead_agent/prompt.py` | `apply_prompt_template` — 把 `subagent_thinking` / `subagent_reminder` / `subagent_section` 按条件拼入模板 |
| `agents/lead_agent/agent.py` | `make_lead_agent` — 从 `config.configurable.subagent_enabled` 读开关，同步控制工具和提示词 |
| `agents/middlewares/subagent_limit_middleware.py` | `SubagentLimitMiddleware` — 硬约束，`after_model()` 截断超量 task 调用 |
| `tools/builtins/task_tool.py` | `get_available_tools(subagent_enabled=False)` — 切断子 Agent 递归委派 |
| `subagents/executor.py` | `_build_initial_state` — 子 Agent 初始只有一条 HumanMessage，不继承父的历史 |
