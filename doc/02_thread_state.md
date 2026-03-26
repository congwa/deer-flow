# Annotated Reducer 让父子 Agent 操作同一份状态而不互相覆盖

## 要解决的问题

一次对话里发生的事情远不止「用户说话、模型回答」。中间件要往状态里写沙盒 ID、要记录上传文件、要累积输出物列表、要临时缓存图片的 base64 数据。如果多轮对话之间、或父子 Agent 之间都往同一个字典里写数据，默认行为是后写者覆盖先写者——新加的产出物会把之前的全部清空，图片读一张就把别的忘了。

DeerFlow 用 **TypedDict + Annotated Reducer** 的组合解决了这个问题：对于需要「累加」的字段，声明字段的同时绑定一个合并函数，框架会在每次写入时自动调用这个函数而不是直接覆盖。

---

## ThreadState 的完整结构

全部内容在 `backend/packages/harness/deerflow/agents/thread_state.py`，共 56 行：

```python
from typing import Annotated, NotRequired, TypedDict
from langchain.agents import AgentState

class SandboxState(TypedDict):
    sandbox_id: NotRequired[str | None]

class ThreadDataState(TypedDict):
    workspace_path: NotRequired[str | None]
    uploads_path: NotRequired[str | None]
    outputs_path: NotRequired[str | None]

class ViewedImageData(TypedDict):
    base64: str
    mime_type: str

class ThreadState(AgentState):
    sandbox:       NotRequired[SandboxState | None]
    thread_data:   NotRequired[ThreadDataState | None]
    title:         NotRequired[str | None]
    artifacts:     Annotated[list[str], merge_artifacts]
    todos:         NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

`AgentState` 本身已经包含了 `messages` 字段（带 `add_messages` reducer），`ThreadState` 继承后再追加这七个字段。

---

## 每个字段存什么、谁来写

```
┌─────────────────┬────────────────────────────┬──────────────────────┐
│ 字段             │ 存的内容                    │ 谁来写               │
├─────────────────┼────────────────────────────┼──────────────────────┤
│ messages        │ 完整对话历史                 │ Agent 框架自动维护   │
│ sandbox         │ sandbox_id（字符串）         │ SandboxMiddleware    │
│ thread_data     │ workspace/uploads/outputs   │ ThreadDataMiddleware │
│                 │ 三条真实磁盘路径             │                      │
│ title           │ 自动生成的会话标题           │ TitleMiddleware      │
│ artifacts       │ 产出文件的虚拟路径列表       │ present_file 工具    │
│ todos           │ 计划模式下的任务列表         │ write_todos 工具     │
│ uploaded_files  │ 本轮新上传文件列表           │ UploadsMiddleware    │
│ viewed_images   │ 图片路径→base64 的映射       │ ViewImageMiddleware  │
└─────────────────┴────────────────────────────┴──────────────────────┘
```

---

## 两个 Reducer 的实现

### `merge_artifacts`：追加去重，不覆盖

```python
def merge_artifacts(existing: list[str] | None, new: list[str] | None) -> list[str]:
    if existing is None:
        return new or []
    if new is None:
        return existing
    return list(dict.fromkeys(existing + new))
```

逻辑：把旧列表和新列表拼在一起，用 `dict.fromkeys` 去重（保序）。这样 Agent 分多次调用 `present_file` 工具时，每次只需要传当次新增的路径，框架会自动合并到累积列表里，不会丢掉之前的产出物。

### `merge_viewed_images`：合并字典，空字典代表清空

```python
def merge_viewed_images(
    existing: dict[str, ViewedImageData] | None,
    new: dict[str, ViewedImageData] | None,
) -> dict[str, ViewedImageData]:
    if existing is None:
        return new or {}
    if new is None:
        return existing
    if len(new) == 0:   # 空字典是特殊信号：清空所有图片
        return {}
    return {**existing, **new}
```

`ViewImageMiddleware` 在调用模型前把需要的图片读成 base64 注入状态，调用完之后写入空字典 `{}` 来触发清空——下一轮不应该还带着上一轮的图片数据重复传给模型。如果用普通的 `new` 覆盖 `existing`，写入空字典会把还没用到的图片也丢掉；这里用特殊约定，空字典才是「清空」信号，`None` 则表示「本次没有变化，保持原样」。

---

## `NotRequired` 的作用

`NotRequired[T]` 告诉类型检查器这个字段可以不存在，从字典里取时可能得到 `None`。

这很重要：`SandboxMiddleware` 在 `before_agent` 里检查 `state.get("sandbox")` 是否为 `None` 来决定要不要申请新沙盒——如果字段必须存在，第一轮就会出错。`title` 用 `NotRequired` 也是同理，`TitleMiddleware` 判断 `state.get("title")` 非 `None` 就跳过生成。

---

## 父子 Agent 共用同一个 schema

`create_agent` 的 `state_schema` 参数在 Lead Agent 和子 Agent 里用的是同一个 `ThreadState`：

```python
# Lead Agent（agent.py）
return create_agent(
    model=...,
    tools=...,
    state_schema=ThreadState,   # ← 相同
    ...
)

# 子 Agent（executor.py）
return create_agent(
    model=...,
    tools=...,
    state_schema=ThreadState,   # ← 相同
    ...
)
```

共用同一个 schema 的好处是：`SandboxMiddleware` 和 `ThreadDataMiddleware` 的代码只需要写一套，父 Agent 和子 Agent 都能跑，无需分叉维护两套中间件逻辑。

子 Agent 在启动时，会把父 Agent 当前的 `sandbox` 和 `thread_data` 字段直接复制进初始状态：

```python
# SubagentExecutor._build_initial_state
state: dict[str, Any] = {
    "messages": [HumanMessage(content=task)],
}
if self.sandbox_state is not None:
    state["sandbox"] = self.sandbox_state       # 来自父 Agent 的 state["sandbox"]
if self.thread_data is not None:
    state["thread_data"] = self.thread_data     # 来自父 Agent 的 state["thread_data"]
```

这样子 Agent 的工具在处理路径时，能认出同一套虚拟路径（`/mnt/user-data/...`），找到同一块磁盘目录，读写父 Agent 产生的文件。

---

## 状态在多轮对话里是怎么持久化的

```
第 1 轮对话
    用户发消息
    Lead Agent 运行
    checkpointer.put(thread_id, state_after_turn_1)
         ↓
第 2 轮对话
    checkpointer.get(thread_id)  → 恢复 state_after_turn_1
    在这个状态基础上追加用户新消息
    Lead Agent 继续运行
    checkpointer.put(thread_id, state_after_turn_2)
```

checkpointer 支持三种存储后端（`async_provider.py`）：

| 类型 | 适用场景 |
|------|----------|
| `memory` | 本地开发，重启后状态丢失 |
| `sqlite` | 单机部署，重启后状态保留 |
| `postgres` | 多实例部署，状态集中存储 |

`sandbox`、`thread_data`、`title` 等字段会随着每轮的 checkpoint 一起持久化。`artifacts` 因为用了 Reducer，每轮合并后的完整列表也会写进去。`viewed_images` 因为每轮结束前会被清空，checkpoint 里一般是空字典，不占存储。

---

## 字段读写的完整流程

```
第 N 轮对话开始
    │
    ├─ checkpointer 加载上一轮 state
    │      sandbox.sandbox_id = "local"（已存在，不用重新申请）
    │      thread_data.workspace_path = "/path/to/workspace"
    │      artifacts = ["report.pdf", "chart.png"]（累积的产出物）
    │      viewed_images = {}（上轮结束时清空）
    │
    ├─ ThreadDataMiddleware.before_agent
    │      state["thread_data"] 已有值 → 不重写
    │
    ├─ SandboxMiddleware.before_agent（lazy_init=True，跳过）
    │
    ├─ UploadsMiddleware.before_agent
    │      发现新上传文件 → state["uploaded_files"] = [{...}]
    │
    ├─ 模型调用
    │
    ├─ ViewImageMiddleware.before_model
    │      state["viewed_images"] = {"img.png": {base64: ...}}  ← 注入
    │
    ├─ 模型看到图片，生成回复
    │
    ├─ ViewImageMiddleware.after_model
    │      state["viewed_images"] = {}  ← 清空
    │
    ├─ Agent 调用 present_file 工具
    │      返回 {"artifacts": ["newfile.csv"]}
    │      merge_artifacts(["report.pdf","chart.png"], ["newfile.csv"])
    │      → state["artifacts"] = ["report.pdf","chart.png","newfile.csv"]
    │
    ├─ TitleMiddleware.after_agent（首轮）
    │      state["title"] = "财务分析报告"
    │
    └─ checkpointer.put → 持久化本轮最终状态
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `agents/thread_state.py` | `ThreadState` 定义，两个 Reducer 函数 |
| `agents/lead_agent/agent.py` | `create_agent(..., state_schema=ThreadState)` |
| `subagents/executor.py` | `_build_initial_state` 复制父 Agent 的 sandbox/thread_data |
| `agents/checkpointer/async_provider.py` | checkpointer 工厂，三种持久化后端 |
