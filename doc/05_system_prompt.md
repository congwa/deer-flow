# 十个 XML 块按运行时条件拼合，让单条系统提示承载多种能力开关

## 要解决的问题

一个 Agent 系统有很多功能维度：思考风格、澄清规则、技能系统、子 Agent 编排、工作目录说明、引用格式、关键提醒……这些内容需要以某种方式传达给模型。

选择一：固定写进系统提示，所有功能永远开着。问题是很多功能是可选的（比如子 Agent 编排需要 `subagent_enabled=True` 才有意义），永远开着会让提示词膨胀、干扰模型。

选择二：每种功能单独一个系统提示，切换时整体替换。问题是不同功能有交叉，独立维护容易出现不一致。

DeerFlow 的做法是：**用一个模板，通过 `str.format()` 条件填充**。固定的段落写在模板里，可选的部分用 `{placeholder}` 占位，`apply_prompt_template` 在运行时根据参数决定每个占位符填什么内容——空字符串表示这个功能没有开启。

---

## 模板的整体结构

`SYSTEM_PROMPT_TEMPLATE` 是一个多行字符串，内部有 9 个命名占位符，外加最后追加的 `<current_date>`：

```
SYSTEM_PROMPT_TEMPLATE =
  <role>
  You are {agent_name}, an open-source super agent.
  </role>

  {soul}                    ← 可选：agent 人格/个性（SOUL.md）

  {memory_context}          ← 可选：长期记忆注入（<memory>...</memory>）

  <thinking_style>
  ...
  {subagent_thinking}       ← 条件插入：子 Agent 分解检查提示
  ...
  </thinking_style>

  <clarification_system>    ← 固定：澄清规则（五种场景，始终存在）
  ...
  </clarification_system>

  {skills_section}          ← 可选：技能系统（<skill_system>...</skill_system>）

  {deferred_tools_section}  ← 可选：延迟工具名单（<available-deferred-tools>）

  {subagent_section}        ← 条件插入：子 Agent 编排规范（<subagent_system>）

  <working_directory>       ← 固定：工作目录说明，始终存在
  ...
  </working_directory>

  <response_style>          ← 固定：输出风格要求，始终存在
  ...
  </response_style>

  <citations>               ← 固定：引用格式规范，始终存在
  ...
  </citations>

  <critical_reminders>
  ...
  {subagent_reminder}       ← 条件插入：子 Agent 并发上限提醒
  ...
  </critical_reminders>

+ <current_date>2026-03-26, Thursday</current_date>  ← 每次构造时追加
```

---

## 每个块做什么

### 固定块（每次都在）

| 块 | 作用 |
|----|------|
| `<role>` | 声明 agent 名称（可从 per-agent 配置读取，默认 "DeerFlow 2.0"） |
| `<thinking_style>` | 规定思考优先级：先想清楚再动手；不要把最终答案写在思考过程里 |
| `<clarification_system>` | 定义五种必须澄清的场景：缺少信息、需求歧义、方案选型、高危操作、主动建议 |
| `<working_directory>` | 告知三个虚拟路径的用途和文件管理规则 |
| `<response_style>` | 输出风格：简洁、不要过度格式化、以行动为导向 |
| `<citations>` | 引用规范：用 `[citation:Title](URL)` 格式，收尾汇总 Sources 段 |
| `<critical_reminders>` | 几条兜底提醒（始终列出），可按条件追加子 Agent 提醒 |
| `<current_date>` | 每次构造时注入当天日期，让模型知道时间背景 |

### 条件块（按运行时参数决定是否填充）

**`{soul}`**：如果当前 agent 目录下有 `SOUL.md`，读取后包在 `<soul>` 标签里注入。用来给 per-agent 定制人格，默认 agent 没有。

**`{memory_context}`**：当 `memory.enabled` 且 `memory.injection_enabled` 时，从 `memory.json` 读取格式化后的记忆，包在 `<memory>` 标签里。内容按 token 预算截断（`max_injection_tokens`，默认 2000）。

**`{skills_section}`**：扫描 `skills/public/` 和 `skills/custom/` 找到启用的技能，生成 `<skill_system>` 块，包含每个技能的名称、描述、文件路径。模型读到这块后知道去哪里加载技能文件。

**`{deferred_tools_section}`**：当 `tool_search.enabled` 时，把 MCP 工具的名字列表（不含 schema）填入 `<available-deferred-tools>` 块，让模型知道这些工具存在、可以通过 `tool_search` 按需加载 schema。

**`{subagent_section}`**：当 `subagent_enabled=True` 时，填入 `_build_subagent_section(n)` 的返回值，也就是整个 `<subagent_system>` 块（详见下一篇文章）。

**`{subagent_thinking}`**：当 `subagent_enabled=True` 时，在 `<thinking_style>` 块内插入一行「分解检查」提示，要求模型在思考时数一数子任务个数、规划分批。

**`{subagent_reminder}`**：当 `subagent_enabled=True` 时，在 `<critical_reminders>` 里追加一行，重申每轮最多 N 个 `task`、超出要分批。

---

## `apply_prompt_template` 如何组装

```python
def apply_prompt_template(
    subagent_enabled: bool = False,
    max_concurrent_subagents: int = 3,
    *,
    agent_name: str | None = None,
    available_skills: set[str] | None = None,
) -> str:
    n = max_concurrent_subagents

    # 条件块：子 Agent 相关
    subagent_section  = _build_subagent_section(n) if subagent_enabled else ""
    subagent_reminder = "- **Orchestrator Mode**: ... HARD LIMIT: max {n} `task` ...\n" \
                        if subagent_enabled else ""
    subagent_thinking = "- **DECOMPOSITION CHECK**: ...\n" \
                        if subagent_enabled else ""

    # 条件块：其他
    skills_section        = get_skills_prompt_section(available_skills)   # 可能为 ""
    deferred_tools_section = get_deferred_tools_prompt_section()          # 可能为 ""
    memory_context        = _get_memory_context(agent_name)               # 可能为 ""

    # 填充模板
    prompt = SYSTEM_PROMPT_TEMPLATE.format(
        agent_name=agent_name or "DeerFlow 2.0",
        soul=get_agent_soul(agent_name),                # 可能为 ""
        skills_section=skills_section,
        deferred_tools_section=deferred_tools_section,
        memory_context=memory_context,
        subagent_section=subagent_section,
        subagent_reminder=subagent_reminder,
        subagent_thinking=subagent_thinking,
    )

    # 追加当天日期
    return prompt + f"\n<current_date>{datetime.now().strftime('%Y-%m-%d, %A')}</current_date>"
```

每个条件块如果不满足开启条件就填空字符串，模板里对应的位置就什么都没有，不会有 `None` 或多余的换行。

---

## `make_lead_agent` 调用时实际传入的参数

```python
# agent.py
return create_agent(
    model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled),
    tools=get_available_tools(model_name=model_name, subagent_enabled=subagent_enabled),
    middleware=_build_middlewares(config, model_name=model_name),
    system_prompt=apply_prompt_template(
        subagent_enabled=subagent_enabled,
        max_concurrent_subagents=max_concurrent_subagents,
        agent_name=agent_name,
    ),
    state_schema=ThreadState,
)
```

`subagent_enabled` 同时控制两件事：`get_available_tools` 里加不加 `task_tool`，以及 `apply_prompt_template` 里填不填 `subagent_section`。工具和提示词的开关保持同步，不会出现「工具能用但提示词没说怎么用」或「提示词说可以用但工具不存在」的不一致。

---

## 各块出现条件汇总

```
┌──────────────────────────┬─────────────────────────────────────────┐
│ 块                       │ 出现条件                                 │
├──────────────────────────┼─────────────────────────────────────────┤
│ <role>                   │ 始终                                     │
│ <soul>                   │ agent 目录下有 SOUL.md                   │
│ <memory>                 │ memory.enabled && memory.injection_enabled│
│ <thinking_style>         │ 始终（内部 subagent_thinking 条件插入）   │
│ <clarification_system>   │ 始终                                     │
│ <skill_system>           │ 有启用的技能                             │
│ <available-deferred-tools>│ tool_search.enabled && 有 MCP 工具      │
│ <subagent_system>        │ subagent_enabled=True                    │
│ <working_directory>      │ 始终                                     │
│ <response_style>         │ 始终                                     │
│ <citations>              │ 始终                                     │
│ <critical_reminders>     │ 始终（内部 subagent_reminder 条件追加）   │
│ <current_date>           │ 始终（每次构造时注入当天日期）             │
└──────────────────────────┴─────────────────────────────────────────┘
```

---

## 这样设计的好处

**单一维护点**：所有段落集中在一个模板字符串里，阅读顺序就是模型收到的顺序，改一处不影响其他段落。

**开关独立不耦合**：`subagent_enabled`、`memory.enabled`、`tool_search.enabled` 互不依赖，任意组合都能正常工作。

**提示词长度按需增长**：关闭子 Agent、无技能、无记忆的最小配置下，提示词只有固定几块；打开所有功能后才会变长，不影响 token 使用效率。

**工具和提示词同步**：同一个 `subagent_enabled` 参数同时控制工具加载和提示词填充，保证模型知道的和系统提供的始终一致。

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `agents/lead_agent/prompt.py` | `SYSTEM_PROMPT_TEMPLATE`、`apply_prompt_template`、各段落生成函数 |
| `agents/lead_agent/agent.py` | `make_lead_agent` 调用 `apply_prompt_template` 传入运行时参数 |
| `config/memory_config.py` | `injection_enabled` 控制记忆注入 |
| `config/tool_search_config.py` | `tool_search.enabled` 控制延迟工具暴露 |
| `skills/loader.py` | `load_skills(enabled_only=True)` 给 `get_skills_prompt_section` 提供技能列表 |
