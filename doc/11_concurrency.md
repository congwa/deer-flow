# 软约定计数加硬截断两道防线，保证单轮并发不超出线程池承载上限

## 要解决的问题

父 Agent 可以在一轮回复里生成多个 `task` 调用，这些子 Agent 会并行执行。并行越多，潜在收益越大，但：

1. **线程池有上限**：执行线程池 `max_workers=3`，超出会排队，但消耗线程池 slot；
2. **模型可能无视提示**：提示词里说「最多 3 个」，模型有时候会生成 4-5 个甚至更多；
3. **超时也需要保证**：单个子 Agent 最多跑 900 秒（15 分钟），不能无限占用线程。

解法是两道防线：**提示词里的计数约定**（软约束）和 **`SubagentLimitMiddleware` 的实际截断**（硬约束）。两道防线各自工作，互不依赖。

---

## 第一道：提示词里的软约定

在 `<subagent_system>` 和 `{subagent_reminder}` 里多次出现的核心规则：

```
HARD CONCURRENCY LIMIT: MAXIMUM {n} `task` CALLS PER RESPONSE. THIS IS NOT OPTIONAL.
Any excess calls are silently discarded by the system — you will lose that work.
Before launching subagents, you MUST count your sub-tasks in your thinking.
```

「软」的含义：模型读了提示词之后**可能**遵守，也可能在复杂任务下忽视或误判。提示词的作用是引导模型在大多数情况下做正确的分批决策，减少截断发生的频率。

---

## 第二道：`SubagentLimitMiddleware` 的硬截断

在 `after_model` 里执行，无论模型输出什么，超出的 `task` 调用会被物理删除：

```python
def _truncate_task_calls(self, state: AgentState) -> dict | None:
    messages = state.get("messages", [])
    last_msg = messages[-1]

    # 只处理最新一条 AI 消息
    tool_calls = getattr(last_msg, "tool_calls", None)

    # 找出所有名为 "task" 的调用下标
    task_indices = [i for i, tc in enumerate(tool_calls) if tc.get("name") == "task"]

    if len(task_indices) <= self.max_concurrent:
        return None     # 没超，不做任何事

    # 超出了，从第 max_concurrent 个开始丢弃
    indices_to_drop = set(task_indices[self.max_concurrent:])
    truncated_tool_calls = [tc for i, tc in enumerate(tool_calls) if i not in indices_to_drop]

    # 用截断后的 tool_calls 替换原消息（同 id 触发 messages reducer 的替换逻辑）
    updated_msg = last_msg.model_copy(update={"tool_calls": truncated_tool_calls})
    return {"messages": [updated_msg]}
```

几个细节：

**只看最后一条 AI 消息**：每次 `after_model` 只处理刚产生的那条 AIMessage，不会影响历史消息。

**保留前 N 个，丢弃后面的**：`task_indices[self.max_concurrent:]` 取超出部分的下标，保留的是按顺序排在最前面的那些 task 调用。模型通常把最重要的任务放在前面，这个策略和提示词里「选最重要的」一致。

**非 task 的调用不受影响**：如果模型同时生成了 `bash` 和 `task`，bash 调用不会被计入 task 计数，也不会被删除。

**上限被 clamp 到 [2, 4]**：

```python
MIN_SUBAGENT_LIMIT = 2
MAX_SUBAGENT_LIMIT = 4

def _clamp_subagent_limit(value: int) -> int:
    return max(MIN_SUBAGENT_LIMIT, min(MAX_SUBAGENT_LIMIT, value))
```

从 `config.configurable.max_concurrent_subagents` 读到值后会被限制在 [2, 4] 范围内，防止配置错误（设成 0 或 100）导致问题。

---

## 两道防线的协作

```
模型生成一轮回复
    │
    ▼
AIMessage.tool_calls = [
  {name: "task", ...},    ← task 1
  {name: "bash", ...},    ← 非 task，不计入
  {name: "task", ...},    ← task 2
  {name: "task", ...},    ← task 3
  {name: "task", ...},    ← task 4（多了）
  {name: "task", ...},    ← task 5（多了）
]
    │
    ▼ SubagentLimitMiddleware.after_model（假设 max_concurrent=3）
    │
    检查：task_indices = [0, 2, 3, 4, 5]，len=5 > 3
    丢弃：indices_to_drop = {4, 5}（第 4 和第 5 个 task）
    保留：tool_calls = [task1, bash, task2, task3]
    │
    ▼ 实际执行的工具：
    task1 → SubagentExecutor → 后台线程
    bash  → bash_tool → 直接执行
    task2 → SubagentExecutor → 后台线程
    task3 → SubagentExecutor → 后台线程
    task4、task5 已不存在，不执行
```

---

## 线程池：调度和执行分两层

```python
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")
_execution_pool  = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")
```

**为什么要两个池**：

调度线程需要等待执行线程完成（`execution_future.result(timeout=...)`），如果调度和执行放在同一个池里，可能出现调度线程全被 block、等不到执行槽位的死锁。分开两个池，调度线程等待时不占用执行 worker。

```
task_tool 调用 execute_async
    │
    ├─→ _scheduler_pool.submit(run_task)   ← 调度线程，占 1 个 scheduler slot
    │      │
    │      ├─→ _execution_pool.submit(self.execute)  ← 执行线程，占 1 个 execution slot
    │      │      │
    │      │      └── asyncio.run(_aexecute)  ← 子 Agent 跑起来
    │      │
    │      └── execution_future.result(timeout=900)  ← 调度线程等待
    │
    └── task_tool 主线程轮询 _background_tasks（5 秒一次，不占池子 slot）
```

如果同时有 3 个 task 调用，就会占满两个池各 3 个 worker。第 4 个 task（如果没被截断）会在池子排队，等前面的 slot 释放。

---

## 超时机制

```python
exec_result = execution_future.result(timeout=config.timeout_seconds)  # 默认 900s
```

`future.result(timeout=900)` 会在 900 秒后抛 `FuturesTimeoutError`：

```python
except FuturesTimeoutError:
    with _background_tasks_lock:
        _background_tasks[task_id].status = SubagentStatus.TIMED_OUT
        _background_tasks[task_id].error = f"Execution timed out after {config.timeout_seconds} seconds"
    execution_future.cancel()   # best effort，可能无法停止已经在跑的协程
```

`execution_future.cancel()` 是「尽力」的，如果执行线程已经在跑了，cancel 可能无效。实际上超时后任务状态被标记为 `TIMED_OUT`，task_tool 的轮询检测到这个状态后会返回错误给父 Agent，父 Agent 可以选择告知用户或继续汇总其他子任务的结果。

超时值可以在 `config.yaml` 里全局或 per-agent 覆盖：

```yaml
subagents:
  timeout_seconds: 900    # 全局默认
  agents:
    bash:
      timeout_seconds: 300   # bash 子 Agent 更短的超时
```

---

## SSE 进度事件

并发的子 Agent 各自跑各自的，它们的进度通过 `task_tool` 里的轮询推进 SSE：

```python
# task_tool 里（每个 task 调用各自跑一个这样的 while 循环）
while True:
    result = get_background_task_result(task_id)
    if len(result.ai_messages) > last_message_count:
        for new_msg in result.ai_messages[last_message_count:]:
            writer({
                "type": "task_running",
                "task_id": task_id,     # 用 task_id 区分是哪个子任务
                "message": new_msg,
            })
```

当父 Agent 同时有 3 个 task 工具在执行时，有 3 个 `while` 循环在跑（都在父 Agent 的工具执行线程池里），各自轮询各自的 `task_id`，各自往 SSE 流里推 `task_running` 事件。前端靠 `task_id` 区分不同子任务的进度。

---

## 状态机：一个子 Agent 的生命周期

```
PENDING
  │ （调度线程开始执行）
  ▼
RUNNING
  │  ├─ 子 Agent 逐步产生 ai_messages（task_tool 轮询到时推 task_running 事件）
  │  │
  │  ├─ 子 Agent 正常结束
  │  │     ▼
  │  │   COMPLETED（task_tool 推 task_completed，返回结果字符串）
  │  │
  │  ├─ 子 Agent 抛异常
  │  │     ▼
  │  │   FAILED（task_tool 推 task_failed，返回错误信息）
  │  │
  │  └─ 超过 timeout_seconds
  │        ▼
  │      TIMED_OUT（task_tool 推 task_timed_out，返回超时信息）
  │
  ▼ 终态后 cleanup_background_task(task_id) 清除内存
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `agents/middlewares/subagent_limit_middleware.py` | `SubagentLimitMiddleware`，`after_model` 截断逻辑，clamp 范围 [2,4] |
| `subagents/executor.py` | 双层线程池，`execute_async`，`FuturesTimeoutError` 超时处理 |
| `subagents/executor.py` | `MAX_CONCURRENT_SUBAGENTS = 3`，`_background_tasks` 注册表 |
| `tools/builtins/task_tool.py` | 轮询循环，`get_stream_writer`，SSE 事件推送 |
| `config/subagents_config.py` | 全局和 per-agent 超时配置 |
