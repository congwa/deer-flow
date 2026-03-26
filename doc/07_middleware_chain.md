# 四个生命周期钩子把横切逻辑从 Agent 核心剥离，Lead 与子 Agent 按需裁剪

## 要解决的问题

Agent 的核心循环很简单：拿到消息 → 调模型 → 执行工具 → 继续。但围绕这个循环有很多「横切关注点」：
- 第一轮要初始化目录和沙盒；
- 每次调模型前要检查图片是否需要注入；
- 每次调工具前要做权限校验；
- 工具执行报错要转成 ToolMessage，不能崩溃；
- 每轮结束后要更新标题、入队记忆；
- 澄清工具调用了要中断整个 Agent。

把这些逻辑写进 Agent 核心里，核心会膨胀且难以测试。DeerFlow 用 `AgentMiddleware` 把这些逻辑封装成可组合的钩子，按需插拔。

---

## 四个钩子的执行位置

```
一轮 Agent 执行的时序：

  before_agent(state, runtime)
       │
       ▼
  [模型调用准备]
       │
  wrap_model_call(messages, ..., call_model)
    ├── before 侧：可以修改 messages（注入临时内容）
    ├── call_model(messages) → response
    └── after 侧：可以修改 response
       │
  after_model(state, runtime)      ← 可以修改模型输出后的状态
       │
       ▼
  [工具执行循环]
  for each tool_call:
    wrap_tool_call(request, handler)
      ├── 可以在工具执行前拦截
      ├── handler(request) → ToolMessage
      └── 可以捕获异常并返回 ToolMessage
       │
  [工具全部执行完]
       │
  after_agent(state, runtime)      ← 一整轮（含所有工具）结束后
```

- `before_agent`：每次 Agent 开始前。初始化类逻辑放这里。
- `wrap_model_call`：包住模型调用。注入临时内容、修改模型输入/输出。
- `after_model`：模型回复之后、工具执行之前。检查和修改模型输出。
- `wrap_tool_call`：包住每个工具调用。鉴权、异常捕获放这里。
- `after_agent`：整轮（含工具）结束后。清理、记录、入队放这里。

---

## Lead Agent 的完整中间件链

`_build_middlewares` 按固定顺序拼装，顺序就是执行顺序：

```
┌──── 基础链 build_lead_runtime_middlewares ─────────────────────┐
│  1. ThreadDataMiddleware      before_agent：计算/创建线程目录    │
│  2. UploadsMiddleware         before_agent：检测新上传文件       │
│  3. SandboxMiddleware         before_agent/after_agent：申请沙盒 │
│  4. DanglingToolCallMiddleware wrap_model_call：修补历史消息     │
│  5. GuardrailMiddleware       wrap_tool_call：工具调用鉴权       │
│  6. ToolErrorHandlingMiddleware wrap_tool_call：捕获工具异常     │
└────────────────────────────────────────────────────────────────┘

┌──── Lead 专有（按运行时配置条件追加）──────────────────────────┐
│  7. SummarizationMiddleware   wrap_model_call：压缩历史（条件）  │
│  8. TodoMiddleware            before_model：TodoList 补救（条件）│
│  9. TokenUsageMiddleware      after_agent：统计 token（条件）    │
│ 10. TitleMiddleware           after_agent：生成会话标题          │
│ 11. MemoryMiddleware          after_agent：记忆入队              │
│ 12. ViewImageMiddleware       wrap_model_call：注入图片（条件）  │
│ 13. DeferredToolFilterMiddleware before_model：隐藏延迟工具schema │
│ 14. SubagentLimitMiddleware   after_model：截断超量 task（条件） │
│ 15. LoopDetectionMiddleware   after_model：检测重复工具调用循环  │
│ 16. ClarificationMiddleware   wrap_tool_call：拦截澄清请求 ← 最后│
└────────────────────────────────────────────────────────────────┘
```

顺序意义重大，注释里有详细说明：

```python
# ThreadDataMiddleware 必须在 SandboxMiddleware 之前（沙盒需要 thread_id）
# UploadsMiddleware 在 ThreadDataMiddleware 之后（需要访问线程目录）
# DanglingToolCallMiddleware 在模型调用前修补历史
# ClarificationMiddleware 必须在最后（拦截后不再执行后续中间件的 wrap_tool_call）
```

---

## 子 Agent 的精简中间件链

子 Agent 只用 `build_subagent_runtime_middlewares`，就是 `include_uploads=False, include_dangling=False` 版本：

```
┌──── 子 Agent 中间件（4 个）────────────────────────────────────┐
│  1. ThreadDataMiddleware      before_agent：读取线程路径        │
│  2. SandboxMiddleware         before_agent/after_agent：沙盒管理│
│  3. GuardrailMiddleware       wrap_tool_call：工具鉴权（条件）  │
│  4. ToolErrorHandlingMiddleware wrap_tool_call：工具异常捕获    │
└────────────────────────────────────────────────────────────────┘
```

没有：UploadsMiddleware（父处理了）、DanglingToolCallMiddleware（子会话历史很干净）、SummarizationMiddleware、TodoMiddleware、TitleMiddleware、MemoryMiddleware、SubagentLimitMiddleware（子不能发 task）、ClarificationMiddleware（子不能中断等待用户）。

这样子 Agent 轻量、启动快、不做父 Agent 需要的那些跨轮持久化逻辑。

---

## 几个关键中间件的实现细节

### `ClarificationMiddleware`：在工具执行前拦截

放在链尾，用 `wrap_tool_call` 检查工具名：

```python
def wrap_tool_call(self, request, handler):
    if request.tool_call.get("name") != "ask_clarification":
        return handler(request)         # 其他工具正常执行
    return self._handle_clarification(request)  # 澄清：中断

def _handle_clarification(self, request) -> Command:
    formatted_message = self._format_clarification_message(args)
    tool_message = ToolMessage(content=formatted_message, ...)
    return Command(
        update={"messages": [tool_message]},
        goto=END,           # ← 发出 goto END，整个 Agent 停在这里等用户
    )
```

`Command(goto=END)` 是 LangGraph 的控制流信号，不是普通的返回值。它告诉框架：这轮 Agent 执行到此为止，等下次用户发消息再继续。中断发生在工具执行阶段，不会继续执行其他工具或调用模型。

这是 `ClarificationMiddleware` 必须放在链尾的原因——它的 `wrap_tool_call` 会短路掉原始工具执行，排在它之后的中间件 `wrap_tool_call` 就不会被调用了。

### `DanglingToolCallMiddleware`：在 `wrap_model_call` 里修补消息

当用户打断了上一轮正在执行的工具（比如刷新页面），历史里会有一条 AIMessage 带着 `tool_calls`，但后面没有对应的 ToolMessage。这种「悬挂」的结构会让 LLM 报错。

修补逻辑用 `wrap_model_call` 而不是 `before_model`：

```python
# 注释里解释了为什么
# Uses wrap_model_call instead of before_model to ensure patches are inserted
# at the correct positions (immediately after each dangling AIMessage),
# not appended to the end of the message list as before_model + add_messages reducer would do.
```

因为 `before_model` + `add_messages` reducer 会把新消息追加到末尾，而修补消息必须插在那条 AIMessage 的**紧后面**，所以用 `wrap_model_call` 直接修改传给模型的消息列表。

### `LoopDetectionMiddleware`：在 `after_model` 里检测循环

用哈希追踪最近 N 条模型输出的工具调用：

```python
def _hash_tool_calls(tool_calls):
    # 对工具名+参数做 MD5，顺序无关
    ...

# after_model 里：
hash = _hash_tool_calls(tool_calls)
count = 同一 hash 出现次数

if count >= warn_threshold:    # 默认 3
    注入 HumanMessage 警告：「你在重复调用，请给出最终答案」

if count >= hard_limit:        # 默认 5
    把 tool_calls 从 AIMessage 里清空
    → 模型下一轮没有工具调用可执行，只能输出文字答案
```

这解决了模型陷入「工具出错 → 重试 → 继续出错 → 继续重试」的死循环。

### `ToolErrorHandlingMiddleware`：在 `wrap_tool_call` 里捕获异常

```python
def wrap_tool_call(self, request, handler):
    try:
        return handler(request)
    except GraphBubbleUp:
        raise               # LangGraph 控制信号不能吃掉（如 Command(goto=END)）
    except Exception as exc:
        return self._build_error_message(request, exc)  # 转成 ToolMessage

def _build_error_message(self, request, exc):
    return ToolMessage(
        content=f"Error: Tool '{tool_name}' failed with {exc.__class__.__name__}: {detail}. Continue with available context.",
        status="error",
        ...
    )
```

工具执行失败不崩溃，返回一条描述错误的 ToolMessage。模型收到这条消息后可以决定怎么处理，比如换个工具或告知用户。

注意 `GraphBubbleUp` 必须重新抛出——它是 LangGraph 用来传递 `Command(goto=END)` 等控制信号的机制，`ClarificationMiddleware` 的中断就是通过这个信号传递的。

---

## 中间件钩子的调用顺序（以一轮完整执行为例）

```
用户消息到来
    │
    ▼
[before_agent] 按顺序调用：
    ThreadDataMiddleware.before_agent  → 写入 thread_data 路径
    UploadsMiddleware.before_agent     → 检测新文件，注入 uploaded_files
    SandboxMiddleware.before_agent     → 若 lazy_init=False，此处申请沙盒
    （其他中间件的 before_agent 为空实现）
    │
    ▼
[wrap_model_call] 按顺序包装：
    DanglingToolCallMiddleware 修补历史（若有悬挂 tool_call）
    SummarizationMiddleware 压缩历史（若触发阈值）
    ViewImageMiddleware 注入图片 base64（若模型支持视觉）
        │
        ▼ 调用模型
        │
    ViewImageMiddleware 清空 viewed_images（注入后清除）
    （call 返回，各中间件按反序 unwrap）
    │
    ▼
[after_model] 按顺序调用：
    SubagentLimitMiddleware.after_model → 截断多余 task 调用
    LoopDetectionMiddleware.after_model → 检测重复
    （其他中间件的 after_model 为空实现）
    │
    ▼
[工具执行循环，每个工具：]
    [wrap_tool_call] 按顺序包装：
        GuardrailMiddleware 鉴权（若配置了护栏）
        ToolErrorHandlingMiddleware 异常捕获
        ClarificationMiddleware 拦截澄清 → 返回 Command(goto=END) 则跳出
            │
            ▼ 执行工具
    （unwrap）
    │
    ▼
[after_agent] 按顺序调用：
    SandboxMiddleware.after_agent   → release 沙盒
    TokenUsageMiddleware.after_agent → 记录 token
    TitleMiddleware.after_agent     → 首轮生成标题
    MemoryMiddleware.after_agent    → 过滤消息，入记忆队列
    （其他中间件的 after_agent 为空实现）
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `agents/lead_agent/agent.py` | `_build_middlewares` 按顺序拼装 Lead 中间件链 |
| `agents/middlewares/tool_error_handling_middleware.py` | `build_lead_runtime_middlewares` / `build_subagent_runtime_middlewares`，以及 `ToolErrorHandlingMiddleware` |
| `agents/middlewares/clarification_middleware.py` | 澄清拦截，`Command(goto=END)` 中断 |
| `agents/middlewares/dangling_tool_call_middleware.py` | 修补悬挂 tool_call，`wrap_model_call` 实现 |
| `agents/middlewares/loop_detection_middleware.py` | 循环检测，哈希 + 滑动窗口 |
| `agents/middlewares/subagent_limit_middleware.py` | 截断多余 task 调用，`after_model` 实现 |
| `sandbox/middleware.py` | `SandboxMiddleware`，沙盒生命周期管理 |
