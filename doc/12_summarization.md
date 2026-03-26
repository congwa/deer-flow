# 触发阈值和保留策略分开配置，让对话压缩在不丢关键上下文的前提下控制 token

## 要解决的问题

长对话会让 `messages` 历史越来越长，最终超出模型的上下文窗口。常见的处理方式是直接截断，但截断会丢失重要的中间结论。

DeerFlow 用 `SummarizationMiddleware` 在触发阈值时，把旧消息**压缩成一段摘要**替换掉，同时保留最近的 N 条（或 N 个 token）原始消息。压缩和保留的策略各自独立配置，互不影响。

---

## 三种触发条件（任意满足即触发）

```yaml
summarization:
  enabled: true
  trigger:
    - type: tokens
      value: 4000      # 历史消息超过 4000 token
    - type: messages
      value: 50        # 历史消息超过 50 条
    - type: fraction
      value: 0.8       # token 超过模型最大上下文的 80%
```

配置为列表时，多个条件是 OR 关系，任意一个满足就触发摘要。配置为单个对象时就只有一个条件。

**`tokens`**：按绝对 token 数触发，适合知道自己用的模型有多大上下文的场景。

**`messages`**：按消息条数触发，不需要统计 token，但对平均消息长度有假设。

**`fraction`**：按模型最大上下文的比例触发，适合需要兼容多种模型（不同 context 窗口大小）的部署。

`SummarizationMiddleware` 的 token 统计：
- Anthropic 模型：约 3.3 字符/token 估算
- 其他模型：用 LangChain 内置的估算方法

---

## 保留策略（`keep`）

```yaml
  keep:
    type: messages
    value: 20    # 触发摘要后，保留最近 20 条消息原样
```

三种形式同触发条件一样（`messages` / `tokens` / `fraction`），但只能配置一个（不是列表）。

`keep` 决定「摘要之后留什么」：
- 比 keep 阈值更新的消息：原样保留
- 比 keep 阈值更旧的消息：送去摘要，替换为一段摘要文本

---

## 摘要后的消息结构

触发之前：

```
[HumanMessage] Turn 1 用户消息
[AIMessage]    Turn 1 模型回复
[HumanMessage] Turn 2 用户消息
...（超过 keep 的旧消息）
[HumanMessage] Turn N-9 消息
[AIMessage]    Turn N-9 回复（刚好在 keep 边界）
[HumanMessage] Turn N-8 消息  ← keep 区域开始
...
[HumanMessage] Turn N 最新消息
[AIMessage]    Turn N 最新回复
```

触发之后：

```
[HumanMessage] "Here is a summary of the conversation to date:\n\n[摘要文本]"
[HumanMessage] Turn N-8 消息  ← keep 区域原样保留
...
[HumanMessage] Turn N 最新消息
[AIMessage]    Turn N 最新回复
```

摘要被注入为一条 `HumanMessage`，放在所有被压缩消息的位置，然后紧接着是保留的近期消息。

---

## AI/Tool 消息对不拆分

工具调用结果必须紧跟在 AIMessage 之后，如果把 AIMessage 保留、对应的 ToolMessage 摘要掉（或反过来），LLM 会报格式错误。

`SummarizationMiddleware` 有专门的保护：如果 keep 边界落在一条 AI+Tool 消息对的中间，会自动调整边界，让整对消息都保留：

```
调整前（边界在 ToolMessage 里）：
  [AIMessage(tool_calls=[...])]  ← 被摘要
  [ToolMessage]                  ← 被摘要
  ← keep 边界
  [AIMessage]  ← 保留

调整后（边界移到 AI+Tool 对的外面）：
  [AIMessage(tool_calls=[...])]  ← 也保留
  [ToolMessage]                  ← 也保留
  [AIMessage]  ← 保留
```

---

## 摘要本身的 token 控制（`trim_tokens_to_summarize`）

```yaml
  trim_tokens_to_summarize: 4000
```

在把旧消息送给摘要模型之前，先把这些旧消息自身裁剪到最多 `trim_tokens_to_summarize` 个 token。

这防止了「旧消息太长，摘要请求本身也超出模型上下文」的情况。设为 `null` 可以跳过裁剪，但对极长的对话不建议这样做。

---

## 摘要模型和提示词

```python
# agent.py 里的构造逻辑
if config.model_name:
    model = config.model_name     # 用指定的轻量模型
else:
    model = create_chat_model(thinking_enabled=False)  # 用默认模型但关闭 thinking

if config.summary_prompt is not None:
    kwargs["summary_prompt"] = config.summary_prompt   # 自定义摘要提示
```

推荐用一个轻量便宜的模型做摘要（如 `gpt-4o-mini`），因为摘要任务不需要最强的模型，用默认大模型做摘要会在每次触发时产生额外成本。

默认提示词（LangChain 的）要求模型：提取最高质量的相关上下文、聚焦于对整体目标关键的信息、避免重复已完成的操作、只返回提取的内容。

---

## 触发摘要之后：Todo 状态会丢失

如果 `is_plan_mode=True`，Agent 有一个 `todos` 状态在 `ThreadState` 里维护任务列表。当对话被压缩时，之前调用 `write_todos` 的那些 AIMessage + ToolMessage 会被摘要掉，消失在 `messages` 历史里。

模型下一次看不到 `write_todos` 相关的历史，可能以为自己还没有创建过 todo 列表，重新创建或忘掉进度。

`TodoMiddleware` 专门处理这个情况：

```python
def before_model(self, state, runtime):
    todos = state.get("todos") or []
    if not todos:
        return None          # 没有 todos，不需要处理

    messages = state.get("messages") or []
    if _todos_in_messages(messages):
        return None          # write_todos 仍然可见，不需要补救

    if _reminder_in_messages(messages):
        return None          # 已经注入过提醒，不重复注入

    # write_todos 已被摘要掉，但 todos 状态还在
    # 注入一条提醒，让模型知道当前 todo 状态
    formatted = _format_todos(todos)
    reminder = HumanMessage(
        name="todo_reminder",
        content="<system_reminder>\n"
                "Your todo list from earlier is no longer visible in the current context window, "
                "but it is still active. Here is the current state:\n\n"
                f"{formatted}\n\n"
                "Continue tracking and updating this todo list as you work. "
                "Call `write_todos` whenever the status of any item changes.\n"
                "</system_reminder>",
    )
    return {"messages": [reminder]}
```

检测逻辑很简单：`state["todos"]` 里有内容（从 ThreadState 持久化的），但最近的 `messages` 里找不到 `write_todos` 调用痕迹——说明发生了「状态有但历史看不到」的断裂，需要注入提醒弥补。

---

## 完整的触发流程

```
每次模型调用前，SummarizationMiddleware 执行（在 wrap_model_call 里）：

计算当前 messages 的 token / 条数
    │
    ├── 没触达任何阈值 → 不做任何事，正常调用模型
    │
    └── 触达阈值
            │
            ▼
    确定 keep 边界
    找出边界之前的「旧消息」和边界之后的「近期消息」
            │
            ├── 调整边界：确保 AI+Tool 对不拆分
            │
            ▼
    把旧消息（trim 到 trim_tokens_to_summarize）送给摘要模型
    摘要模型返回一段摘要文本
            │
            ▼
    构造新消息列表：
      [HumanMessage("Here is a summary...")]  ← 摘要
      [近期消息原样]
            │
            ▼
    用新消息列表调用实际模型（本次推理）
            │
            ▼
    （注意：这次 summarization 不写回 ThreadState checkpoint，
     只对本次推理的输入做了替换。
     下一轮 checkpoint 会保存包含 summarization 结果的完整历史。）

然后 TodoMiddleware 在 before_model 里检查 todo 状态断裂，必要时注入提醒。
```

---

## 这样设计的好处

**触发和保留分开配置**：可以同时设「达到 4000 token 就触发」和「触发后保留最近 20 条」，两个参数针对不同问题（何时压缩 vs 保留多少），不互相影响。

**不直接截断**：截断丢失所有被裁掉的内容；摘要保留了语义信息，后续对话可以引用早期讨论的内容。

**AI/Tool 对保护**：不会因为边界划分导致 LLM 看到残缺的工具调用历史，不会报格式错误。

**摘要用轻量模型**：可以用便宜的小模型做摘要，不影响主任务模型的选择。

**Todo 补救独立**：`TodoMiddleware` 和 `SummarizationMiddleware` 各自独立，Todo 不需要知道摘要发生了，只需要检查自己的状态和历史是否一致。

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `config/summarization_config.py` | `SummarizationConfig`，三种 `ContextSize` 类型 |
| `agents/lead_agent/agent.py` | `_create_summarization_middleware`，把配置转换成中间件参数 |
| `agents/middlewares/todo_middleware.py` | `TodoMiddleware`，检测 todos 断裂并注入提醒 |
| `backend/docs/summarization.md` | 详细配置文档和 Best Practices |
