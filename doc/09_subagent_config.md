# 白黑名单加系统提示切分两种子 Agent 的能力范围，防止能力过载和递归委派

## 要解决的问题

不同类型的子任务需要不同的能力范围。一个负责「执行 bash 命令」的子 Agent 不应该能发起 HTTP 请求，一个负责「代码探索」的子 Agent 不需要也不应该再去委派更多子 Agent。

如果所有子 Agent 都拿到父 Agent 的全部工具，有两个具体问题：
1. 模型面对太多工具会分心，有时候选错工具；
2. 子 Agent 如果也能调 `task`，可以无限递归地委派，直到线程池耗尽。

DeerFlow 用 `SubagentConfig` 数据类把「这种子 Agent 有什么工具、用什么提示词、最多跑几轮」封装成配置，工具过滤逻辑在 `SubagentExecutor.__init__` 里用白名单和黑名单实现。

---

## `SubagentConfig` 的结构

```python
@dataclass
class SubagentConfig:
    name: str                             # 类型名（"general-purpose" / "bash"）
    description: str                      # 给人看的，说明什么时候选这种子 Agent
    system_prompt: str                    # 子 Agent 的系统提示词
    tools: list[str] | None = None        # 允许的工具名列表（None=不限，全部从父继承）
    disallowed_tools: list[str] | None    # 禁止的工具名列表（覆盖 tools 的允许列表）
    model: str = "inherit"                # 用哪个模型（"inherit" = 跟父 Agent 一样）
    max_turns: int = 50                   # 最多执行多少轮（防止无限循环）
    timeout_seconds: int = 900            # 超时时间（15 分钟）
```

`tools=None` 和 `tools=["bash", "ls", ...]` 是两种截然不同的策略：
- `None`：不限制，子 Agent 会从父侧传来的工具集里去掉黑名单后的所有工具；
- 指定列表：白名单，只能用列表里的工具。

---

## 两种内置子 Agent 的配置对比

### `general-purpose`：广泛能力，适合探索和分析类任务

```python
GENERAL_PURPOSE_CONFIG = SubagentConfig(
    name="general-purpose",
    system_prompt="You are a general-purpose subagent... Do NOT ask for clarification...",
    tools=None,                                          # 不限制工具类型
    disallowed_tools=["task", "ask_clarification", "present_files"],
    model="inherit",
    max_turns=50,
)
```

**`tools=None`**：可以用搜索、读写文件、执行命令等所有父侧传来的工具（除黑名单外）。

**`disallowed_tools` 三个**：
- `task`：禁止递归委派（代码层也在 task_tool 里切断，双保险）
- `ask_clarification`：子 Agent 不能中断等待用户，必须用父传来的 prompt 自己判断
- `present_files`：子 Agent 不能直接向用户呈现文件，产出归父 Agent 管理

**系统提示词的核心原则**：自主完成，不问澄清，返回结构化结果（做了什么 / 关键发现 / 文件路径 / 问题）。

---

### `bash`：精简工具，只做命令执行

```python
BASH_AGENT_CONFIG = SubagentConfig(
    name="bash",
    system_prompt="You are a bash command execution specialist...",
    tools=["bash", "ls", "read_file", "write_file", "str_replace"],  # 白名单
    disallowed_tools=["task", "ask_clarification", "present_files"],
    model="inherit",
    max_turns=30,
)
```

**`tools=["bash", "ls", "read_file", "write_file", "str_replace"]`**：明确指定白名单，只有沙盒文件操作工具。没有搜索、没有 MCP、没有图片、没有委派。

**`max_turns=30`**（比 `general-purpose` 的 50 少）：bash 任务通常是有限步骤的命令序列，不需要也不应该跑太多轮。

**适用场景**：跑测试、构建项目、执行 git 操作、部署命令——这类任务输出冗长，放在父 Agent 上下文里会占满历史，隔离到 bash 子 Agent 里可以保持父侧对话干净。

---

## 工具过滤的执行逻辑

在 `SubagentExecutor.__init__` 里：

```python
self.tools = _filter_tools(
    tools,              # 父传来的工具集（已去掉 task）
    config.tools,       # 白名单（None 或具体列表）
    config.disallowed_tools,
)

def _filter_tools(all_tools, allowed, disallowed):
    filtered = all_tools

    # 第一步：白名单过滤（allowed 非 None 时才有效）
    if allowed is not None:
        allowed_set = set(allowed)
        filtered = [t for t in filtered if t.name in allowed_set]

    # 第二步：黑名单过滤（始终执行）
    if disallowed is not None:
        disallowed_set = set(disallowed)
        filtered = [t for t in filtered if t.name not in disallowed_set]

    return filtered
```

两步顺序是「先白名单再黑名单」。如果 `general-purpose` 配置了 `tools=None`（白名单不限），第一步不做任何过滤；第二步黑名单去掉 `task`、`ask_clarification`、`present_files`。如果 `bash` 配置了 `tools=["bash", "ls", ...]`，第一步只保留这五个；第二步黑名单即使重叠也没问题，是幂等的。

---

## 工具集实际的差异

```
父传来的工具集（subagent_enabled=False，已去掉 task）：
  bash, ls, read_file, write_file, str_replace
  web_search, web_fetch, image_search
  present_file, ask_clarification, view_image
  [MCP 工具...]

经过 general-purpose 过滤后（tools=None，black: task, ask_clarification, present_files）：
  bash, ls, read_file, write_file, str_replace
  web_search, web_fetch, image_search
  view_image
  [MCP 工具...]
  ← 去掉了 task / ask_clarification / present_file/files

经过 bash 过滤后（white: bash,ls,read_file,write_file,str_replace, black: task,ask_clarification,present_files）：
  bash, ls, read_file, write_file, str_replace
  ← 只剩沙盒文件操作工具
```

---

## 模型继承（`model="inherit"`）

```python
def _get_model_name(config: SubagentConfig, parent_model: str | None) -> str | None:
    if config.model == "inherit":
        return parent_model
    return config.model
```

子 Agent 默认继承父 Agent 的模型。这样用户在前端选了 Claude 3.7，子 Agent 也用 Claude 3.7，不会出现父强子弱的不一致。

如果特定类型的子任务需要不同模型（比如 bash 任务用更便宜的模型），可以在 `config.yaml` 的 `subagents.agents` 里覆盖：

```yaml
subagents:
  timeout_seconds: 900
  agents:
    bash:
      timeout_seconds: 300    # bash 任务超时更短
```

不过目前 `SubagentOverrideConfig` 只支持 `timeout_seconds`，模型覆盖需要直接修改代码里的 `SubagentConfig.model`。

---

## 系统提示词对行为的约束

工具过滤解决「模型能用什么工具」，系统提示词解决「模型应该怎么做事」。两者协同：

**`general-purpose` 的关键约束**：
- `Do NOT ask for clarification - work with the information provided`：对应黑名单里的 `ask_clarification`，提示词和工具限制双重保证
- 输出格式化：固定的五段结构（摘要 / 发现 / 文件路径 / 问题 / 引用）让父 Agent 更容易汇总

**`bash` 的关键约束**：
- `Execute commands one at a time when they depend on each other`：防止并行执行有依赖关系的命令
- `Report both stdout and stderr when relevant`：确保错误信息不被丢掉
- 输出格式化：每组命令固定四段（执行了什么 / 成功或失败 / 输出摘要 / 错误警告）

---

## 对比：父 Agent 和两种子 Agent 的能力边界

```
                          父 Agent    general-purpose    bash
─────────────────────────────────────────────────────────────
执行 bash 命令              ✓              ✓               ✓
读写文件                    ✓              ✓               ✓
网络搜索                    ✓              ✓               ✗（白名单不含）
MCP 工具                    ✓              ✓               ✗（白名单不含）
查看图片（视觉模型）         ✓              ✓               ✗（白名单不含）
向用户展示文件               ✓              ✗（黑名单）     ✗（黑名单）
向用户追问澄清               ✓              ✗（黑名单）     ✗（黑名单）
委派给子 Agent              ✓              ✗（黑名单+代码）✗（黑名单+代码）
最大执行轮数               无限制           50              30
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `subagents/config.py` | `SubagentConfig` 数据类定义 |
| `subagents/builtins/general_purpose.py` | `GENERAL_PURPOSE_CONFIG` |
| `subagents/builtins/bash_agent.py` | `BASH_AGENT_CONFIG` |
| `subagents/executor.py` | `_filter_tools`，工具过滤逻辑；`_get_model_name`，模型继承 |
| `subagents/registry.py` | `get_subagent_config`，查配置表，应用 config.yaml 超时覆盖 |
| `config/subagents_config.py` | `SubagentsAppConfig`，全局超时和 per-agent 超时覆盖 |
