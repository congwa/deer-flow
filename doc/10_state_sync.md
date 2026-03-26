# 消息历史不透传保证上下文隔离，沙盒目录共享保证文件结果可交接

## 要解决的问题

父子 Agent 之间需要某种形式的「信息交换」：
- 子 Agent 要能看到父交给它的任务；
- 子 Agent 写出的文件，父 Agent 要能读到；
- 子 Agent 不该把父的整段对话历史都吃进去，否则上下文会爆炸；
- 子 Agent 结束后，父 Agent 要能用结果继续工作。

这四点有矛盾。全部共享会让子 Agent 上下文过大；完全隔离则父子无法协作。DeerFlow 的解法是：**在进程内存层面隔离（消息历史不共享），在磁盘层面共享（沙盒目录、文件系统是同一块）**。

---

## 父 → 子：哪些信息被传递

子 Agent 启动时，初始状态由 `_build_initial_state(prompt)` 构造：

```python
def _build_initial_state(self, task: str) -> dict:
    state: dict = {
        "messages": [HumanMessage(content=task)],   # ← 只有一条消息
    }

    if self.sandbox_state is not None:
        state["sandbox"] = self.sandbox_state        # ← 父的沙盒 ID

    if self.thread_data is not None:
        state["thread_data"] = self.thread_data      # ← 父的三条磁盘路径

    return state
```

**传了什么**：
1. **委派的任务文本**（`prompt`），以 `HumanMessage` 的形式作为子会话的第一条也是唯一一条初始消息；
2. **`sandbox_state`**：父的沙盒 ID（本地模式下是 `"local"`）；
3. **`thread_data`**：三条真实磁盘路径（workspace、uploads、outputs）。

**没传什么**：父的完整 `messages` 历史、`title`、`artifacts`、`todos`、`uploaded_files`、`viewed_images`——这些字段都不在子的初始状态里。

---

## 消息历史不共享：隔离上下文

父和子各自维护一套 `messages`：

```
父 Agent 的 messages 历史：
  HumanMessage("帮我分析腾讯股价下跌原因")
  AIMessage(tool_calls=[task("财务"), task("舆情"), task("行业")])
  ToolMessage("Task Succeeded. Result: [财务分析结果]", id="call_1")
  ToolMessage("Task Succeeded. Result: [舆情分析结果]", id="call_2")
  ToolMessage("Task Succeeded. Result: [行业分析结果]", id="call_3")
  AIMessage("综合三方面分析，腾讯股价下跌的原因是...")

子 Agent（财务）的 messages 历史：
  HumanMessage("分析腾讯最近三个季度的营收趋势...")
  AIMessage(tool_calls=[bash("python report.py")])
  ToolMessage("营收 Q1: 100亿, Q2: 98亿, Q3: 95亿...")
  AIMessage("分析结果：营收连续下滑，主要因为...")
```

子 Agent 看不到父的那些用户对话，也看不到其他两个子 Agent 的过程。每个子会话的上下文是干净的，只有委派任务的 prompt 和自己执行过程中产生的消息。

**为什么这样好**：子 Agent 用更少的上下文 token 完成专注的任务；多个子 Agent 并行跑时，各自的历史不会因为别人的对话变长而被挤占。

---

## 沙盒目录共享：文件结果可交接

`sandbox_state` 和 `thread_data` 传给子 Agent 之后，子 Agent 的工具调用会走同一套 `replace_virtual_path` 逻辑：

```
父 Agent 写文件：
  write_file("/mnt/user-data/outputs/report.csv")
    → 实际写到 /home/user/.deer-flow/threads/abc123/user-data/outputs/report.csv

子 Agent 读文件（thread_data 与父相同）：
  read_file("/mnt/user-data/outputs/report.csv")
    → 实际读 /home/user/.deer-flow/threads/abc123/user-data/outputs/report.csv
    → ✓ 能读到父写的文件
```

这就是文件协作的基础：**同一 `thread_id` 决定同一块磁盘目录**，父写子能读，子写父能读。

---

## 子 → 父：结果如何回来

子 Agent 跑完之后，结果通过 `ToolMessage` 回到父：

```python
# task_tool.py 里，子 Agent 完成后：
return f"Task Succeeded. Result: {result.result}"
# 框架把这个字符串包成 ToolMessage(content=..., tool_call_id="call_abc")
```

`result.result` 是子 Agent 最后一条 AIMessage 的内容——子 Agent 的整个执行过程压缩成了一段文字。父 Agent 只看到这段结论，不看到子 Agent 跑了哪些工具、中间产生了什么消息。

但如果子 Agent 写了文件到 `/mnt/user-data/outputs/`，这些文件在磁盘上存在，父可以通过 `read_file` 访问。结论文字是「软」交接，磁盘文件是「硬」交接。

---

## `artifacts` 不会自动同步

子 Agent 的 `state["artifacts"]` 即使在子运行期间被 `present_file` 工具写入，这些值也不会自动合并回父的 `state["artifacts"]`。

原因：`present_files` 被加入子 Agent 的 `disallowed_tools`，子 Agent 无法调用它。子 Agent 执行过程中即使产生了文件，也无法通过 `present_file` 工具把文件路径写进自己的 `artifacts`。如果需要父知道文件在哪里，子 Agent 应该在 `result.result` 文字里提到文件路径，或者父自己在汇总时调用 `present_file`。

---

## 子的 checkpoint 不写入 LangGraph

父 Agent 的状态被 `checkpointer` 持久化（sqlite / postgres）：每轮之后快照一次，对话可以跨时间继续。

子 Agent 没有 `checkpointer`：

```python
# executor.py
run_config: RunnableConfig = {
    "recursion_limit": self.config.max_turns,
}
if self.thread_id:
    run_config["configurable"] = {"thread_id": self.thread_id}
# ← 注意：没有 checkpointer，run_config 里没有 checkpointer 配置
```

子 Agent 的执行结果只活在内存里的 `_background_tasks` 字典，以及最终返回给父的 `ToolMessage` 字符串。子 Agent 中途崩了、超时了，不会产生残留的 checkpoint，也不需要额外的清理逻辑。

---

## 完整的信息流图

```
父 Agent（有 checkpointer，历史持久化）
    │
    ├── state["messages"] = [父的完整对话历史]
    ├── state["sandbox"] = {sandbox_id: "local"}
    ├── state["thread_data"] = {workspace: ".../abc123/...", ...}
    ├── state["artifacts"] = ["report.pdf"]
    │
    │ 模型调用 task：
    │   prompt="分析财务趋势..."
    │   sandbox_state = state["sandbox"]      ← 复制
    │   thread_data = state["thread_data"]    ← 复制
    │
    ▼
子 Agent（无 checkpointer，内存里跑）
    ├── state["messages"] = [HumanMessage("分析财务趋势...")]  ← 只有一条
    ├── state["sandbox"] = {sandbox_id: "local"}              ← 复制自父
    ├── state["thread_data"] = {workspace: ".../abc123/...", ...} ← 复制自父
    │
    │ 子 Agent 执行：
    │   bash("python analyze.py")
    │   write_file("/mnt/user-data/outputs/analysis.csv")   ← 写到共享目录
    │   AIMessage("分析结果：Q1 营收下滑 5%...")
    │
    │ 子 Agent 结束，result.result = "分析结果：Q1 营收下滑 5%..."
    │
    ▼
父 Agent 收到：
    ToolMessage("Task Succeeded. Result: 分析结果：Q1 营收下滑 5%...")
    │
    ├── 可以直接用文字结论
    └── 可以用 read_file("/mnt/user-data/outputs/analysis.csv") 读详细数据
              ↑ 同一块磁盘，能读到
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `subagents/executor.py` | `_build_initial_state`，初始状态构造；无 checkpointer 的 run_config |
| `subagents/executor.py` | `_aexecute`，`result.result` 从最后一条 AIMessage 提取 |
| `tools/builtins/task_tool.py` | 从 `runtime.state` 提取 sandbox / thread_data / thread_id |
| `sandbox/tools.py` | `replace_virtual_path`，同一 `thread_data` 决定同一磁盘目录 |
| `agents/thread_state.py` | `ThreadState` 定义，`artifacts` 有 Reducer 但子 Agent 不调 present_file |
