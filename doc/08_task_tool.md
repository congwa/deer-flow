# task 工具把一次工具调用变成一段独立子会话，让父 Agent 无感知地完成委派

## 要解决的问题

父 Agent 要把一个子任务交给子 Agent 完成，这件事从框架角度看只有一种可能的接口——工具调用。模型已经知道怎么发工具调用，运行时已经有工具执行机制，不需要为委派单独建立一套 RPC 层。

关键在于：**`task` 工具的执行函数内部要能完整地构造并运行一个子 Agent，还要把子 Agent 的进度实时同步到父 Agent 的 SSE 流里**。

---

## `task_tool` 的完整签名

```python
@tool("task", parse_docstring=True)
def task_tool(
    runtime: ToolRuntime[ContextT, ThreadState],   # 注入父状态和上下文
    description: str,                               # 任务简述（显示给用户）
    prompt: str,                                    # 给子 Agent 的任务正文
    subagent_type: Literal["general-purpose", "bash"],  # 子 Agent 类型
    tool_call_id: Annotated[str, InjectedToolCallId],   # 框架注入，不由模型填写
    max_turns: int | None = None,                   # 可选覆盖 max_turns
) -> str:                                           # 返回子 Agent 的结果字符串
```

`InjectedToolCallId` 是关键：框架把本次 `tool_calls[i].id` 自动注入为 `tool_call_id`，这个 id 既是唯一追踪键（`task_id`），也是 SSE 事件和 `ToolMessage` 对齐的依据，不需要模型自己编一个。

---

## 执行过程的六个阶段

### 阶段一：查配置，构造执行器

```python
config = get_subagent_config(subagent_type)    # 从注册表取 SubagentConfig
if config is None:
    return f"Error: Unknown subagent type '{subagent_type}'"

overrides = {}
skills_section = get_skills_prompt_section()
if skills_section:
    overrides["system_prompt"] = config.system_prompt + "\n\n" + skills_section
if max_turns is not None:
    overrides["max_turns"] = max_turns
if overrides:
    config = replace(config, **overrides)       # dataclasses.replace，不改原始配置
```

skills 段落在这里追加进子 Agent 的系统提示——子 Agent 也能知道有哪些技能可用。

### 阶段二：从父状态提取上下文

```python
sandbox_state = runtime.state.get("sandbox")      # 父 Agent 当前的沙盒 ID
thread_data = runtime.state.get("thread_data")    # 父 Agent 当前的磁盘路径
thread_id = runtime.context.get("thread_id")      # 同一 LangGraph 线程 ID
parent_model = runtime.config.get("metadata", {}).get("model_name")
trace_id = runtime.config.get("metadata", {}).get("trace_id") or uuid4()[:8]
```

这三个信息会传给 `SubagentExecutor`，子 Agent 用同一个 `sandbox_id`（同一块磁盘）、同一个 `thread_id`（子 Agent 的沙盒工具能找到正确路径）。

### 阶段三：准备子 Agent 的工具集

```python
from deerflow.tools import get_available_tools

tools = get_available_tools(model_name=parent_model, subagent_enabled=False)
```

`subagent_enabled=False` 这一行切断了递归。子 Agent 的工具列表里不存在 `task` 工具，无论提示词里写什么，模型的工具 schema 列表里都不会出现 `task`，不会被调用。

### 阶段四：异步启动执行

```python
executor = SubagentExecutor(
    config=config,
    tools=tools,
    parent_model=parent_model,
    sandbox_state=sandbox_state,
    thread_data=thread_data,
    thread_id=thread_id,
    trace_id=trace_id,
)

task_id = executor.execute_async(prompt, task_id=tool_call_id)
```

`execute_async` 把任务丢进后台线程池，立即返回 `task_id`（就是 `tool_call_id`），不等待子 Agent 完成。

### 阶段五：轮询 + 实时推进度

```python
writer = get_stream_writer()
writer({"type": "task_started", "task_id": task_id, "description": description})

while True:
    result = get_background_task_result(task_id)

    # 有新 AI 消息，推送给前端
    current_message_count = len(result.ai_messages)
    if current_message_count > last_message_count:
        for i in range(last_message_count, current_message_count):
            writer({
                "type": "task_running",
                "task_id": task_id,
                "message": result.ai_messages[i],
                "message_index": i + 1,
            })
        last_message_count = current_message_count

    # 检查终态
    if result.status == SubagentStatus.COMPLETED:
        writer({"type": "task_completed", "task_id": task_id, "result": result.result})
        cleanup_background_task(task_id)
        return f"Task Succeeded. Result: {result.result}"
    elif result.status == SubagentStatus.FAILED:
        writer({"type": "task_failed", ...})
        return f"Task failed. Error: {result.error}"
    elif result.status == SubagentStatus.TIMED_OUT:
        ...

    time.sleep(5)   # 每 5 秒轮询一次
```

`get_stream_writer()` 是 LangGraph 的 SSE 写入接口。在工具执行的过程中，这些 `task_*` 事件会实时推到前端，用户能看到子 Agent 的进度，而不是等父 Agent 工具调用结束后才看到结果。

从父 Agent 的角度看，`task_tool` 是一个同步调用——发出去等结果回来。等待期间这个工具函数在线程里阻塞，但轮询间隔是 5 秒，不会消耗大量 CPU。

### 阶段六：返回值成为 ToolMessage

工具函数返回字符串，框架自动包装成 `ToolMessage` 追加进 `messages`：

```
messages 最终形态：
  ...
  AIMessage(tool_calls=[{name: "task", id: "call_abc", args: {...}}])
  ToolMessage(content="Task Succeeded. Result: ...", tool_call_id="call_abc")
```

`tool_call_id` 必须和 `AIMessage.tool_calls[i].id` 对应，否则 LLM 会报「missing tool message」错误。用 `InjectedToolCallId` 注入 `tool_call_id` 保证了这个对应关系。

---

## `SubagentExecutor` 内部：子 Agent 是怎么创建的

```python
# executor.py
def _create_agent(self):
    model_name = _get_model_name(self.config, self.parent_model)
    model = create_chat_model(name=model_name, thinking_enabled=False)

    from deerflow.agents.middlewares.tool_error_handling_middleware import build_subagent_runtime_middlewares
    middlewares = build_subagent_runtime_middlewares(lazy_init=True)

    return create_agent(
        model=model,
        tools=self.tools,                   # 父传来的、已过滤掉 task 的工具集
        middleware=middlewares,             # 精简版中间件（4 个）
        system_prompt=self.config.system_prompt,
        state_schema=ThreadState,           # 与父共用同一 schema
    )

def _build_initial_state(self, task: str) -> dict:
    state = {
        "messages": [HumanMessage(content=task)],   # 只有一条 human 消息
    }
    if self.sandbox_state is not None:
        state["sandbox"] = self.sandbox_state       # 复制父的沙盒 ID
    if self.thread_data is not None:
        state["thread_data"] = self.thread_data     # 复制父的磁盘路径
    return state
```

子 Agent 的历史只有一条消息（委派的 prompt），不继承父的对话历史。

### 子 Agent 的执行用 `astream` + 实时收集

```python
async for chunk in agent.astream(state, config=run_config, stream_mode="values"):
    final_state = chunk

    messages = chunk.get("messages", [])
    if messages:
        last_message = messages[-1]
        if isinstance(last_message, AIMessage):
            message_dict = last_message.model_dump()
            # 去重后追加到 result.ai_messages
            if not is_duplicate:
                result.ai_messages.append(message_dict)
```

每次子 Agent 的状态更新都被收集进 `result.ai_messages`。task_tool 里的轮询看到 `ai_messages` 变长时，就把新消息推进 SSE。

---

## 后台线程池：两层分离

```python
# executor.py
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")
_execution_pool  = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")
```

调度和执行分两个线程池：

```
task_tool 调用 execute_async
    │
    ▼
_scheduler_pool.submit(run_task)   ← 调度线程：负责超时控制
    │
    ▼ 调度线程内
_execution_pool.submit(self.execute, task, result_holder)   ← 执行线程
    │  wait(timeout=config.timeout_seconds)
    │
    ├── 超时：FuturesTimeoutError → status = TIMED_OUT
    └── 完成：复制结果到 _background_tasks
```

执行线程里调 `asyncio.run(self._aexecute(...))` 跑子 Agent 的异步循环。每个子 Agent 在自己独立的事件循环里跑，互不影响。

---

## 任务注册表

```python
_background_tasks: dict[str, SubagentResult] = {}
_background_tasks_lock = threading.Lock()
```

全局字典，`task_id`（等于 `tool_call_id`）作为 key。`task_tool` 的轮询通过 `get_background_task_result(task_id)` 查询。任务完成后由 `cleanup_background_task(task_id)` 清除，只清除处于终态的任务，防止还在运行的任务被误删。

---

## 从 tool_call 到子 Agent 结果的完整路径

```
模型生成：
  AIMessage.tool_calls = [{
    name: "task",
    id: "call_abc123",
    args: {
      description: "分析财务数据",
      prompt: "分析最近三个季度的营收趋势...",
      subagent_type: "general-purpose"
    }
  }]
         │
         ▼
框架找到 task_tool，调用：
  task_tool(runtime, "分析财务数据", "分析最近三...", "general-purpose",
            tool_call_id="call_abc123")
         │
         │  1. get_subagent_config("general-purpose") → SubagentConfig
         │  2. 从 runtime.state 提取 sandbox / thread_data
         │  3. get_available_tools(subagent_enabled=False) → 不含 task 的工具集
         │  4. SubagentExecutor 构造
         │  5. execute_async(prompt, task_id="call_abc123")
         │     → _background_tasks["call_abc123"] = SubagentResult(PENDING)
         │     → _scheduler_pool.submit(run_task)
         │  6. 发送 SSE: task_started
         │  7. 进入 while 轮询循环
         │
         ▼ 后台线程
  _execution_pool → asyncio.run(_aexecute(prompt))
    → create_agent(子 Agent 配置)
    → agent.astream(初始状态)
    → 子 Agent 运行多轮：调工具 → 调模型 → 输出
    → 每轮新 AI 消息写入 result.ai_messages
         │
         ▼ 轮询循环
  task_tool 发现 ai_messages 新增 → SSE: task_running
  task_tool 发现 status=COMPLETED → SSE: task_completed
  cleanup_background_task("call_abc123")
  return "Task Succeeded. Result: 三个季度营收均增长..."
         │
         ▼
框架把返回值包成：
  ToolMessage(content="Task Succeeded. Result: ...", tool_call_id="call_abc123")
  → 追加进 messages
         │
         ▼
Lead Agent 下一轮调模型，看到 ToolMessage，继续汇总
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `tools/builtins/task_tool.py` | `task_tool`，六个阶段的完整实现 |
| `subagents/executor.py` | `SubagentExecutor`，`_create_agent`，`_build_initial_state`，`_aexecute`，`execute_async` |
| `subagents/registry.py` | `get_subagent_config`，从内置表查配置，应用 config.yaml 的超时覆盖 |
| `subagents/config.py` | `SubagentConfig` 数据类 |
| `subagents/builtins/general_purpose.py` | `GENERAL_PURPOSE_CONFIG` |
| `subagents/builtins/bash_agent.py` | `BASH_AGENT_CONFIG` |
