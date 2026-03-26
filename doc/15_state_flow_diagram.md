# 完整状态流转图——从一次用户请求到所有状态变迁

## 说明

这张图以 **状态为主线**，追踪 `ThreadState` 的每一个字段从哪里写入、在什么时机被哪个组件修改、如何流向子 Agent，以及最终如何落盘。

**状态字段速查：**

```
ThreadState（继承自 AgentState）
  ├── messages          : list[BaseMessage]     Reducer: add_messages（追加合并）
  ├── sandbox           : SandboxState          普通赋值（{sandbox_id: str}）
  ├── thread_data       : ThreadDataState       普通赋值（{workspace_path, uploads_path, outputs_path}）
  ├── title             : str | None            普通赋值（只写一次）
  ├── artifacts         : list[str]             Reducer: merge_artifacts（追加去重）
  ├── todos             : list | None           普通赋值（plan mode）
  ├── uploaded_files    : list[dict] | None     普通赋值（每轮覆盖）
  └── viewed_images     : dict[str, base64]     Reducer: merge_viewed_images（合并 / {} 清空）
```

---

## 完整状态流转图

```
╔═══════════════════════════════════════════════════════════════════════════════════════════════╗
║                    DEERFLOW COMPLETE STATE FLOW — 从用户请求到所有状态变迁                      ║
╚═══════════════════════════════════════════════════════════════════════════════════════════════╝

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【A】 进入与路由
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 用户浏览器
   │  POST /api/langgraph/threads/{thread_id}/runs/stream
   │  （附带：user message、model_name、subagent_enabled、is_plan_mode 等 configurable）
   ▼
 ┌────────────────────────────────────────┐
 │   Nginx :2026                          │
 │   location /api/langgraph/ {           │
 │     rewrite 掉前缀                     │
 │     proxy_buffering off;  ← SSE 透传  │
 │   }                                    │
 └────────────────────────┬───────────────┘
                          │
                          ▼
 ┌────────────────────────────────────────┐
 │   LangGraph Server :2024               │
 │   注册入口: make_lead_agent            │
 │   (来自 langgraph.json)                │
 └────────────────────────┬───────────────┘
                          │
                          │ 读 configurable:
                          │   thread_id, model_name, subagent_enabled,
                          │   is_plan_mode, thinking_enabled, agent_name ...
                          ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【B】 状态加载 —— Checkpointer
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │  checkpointer.get(thread_id)                                                           │
 │  backend: memory / sqlite / postgres（由 config.yaml 决定）                            │
 │                                                                                        │
 │  加载上一轮存下的 ThreadState 快照：                                                    │
 │                                                                                        │
 │   state = {                                                                            │
 │     messages:      [... 历史对话，含所有 Human/AI/ToolMessage ...]                      │
 │     sandbox:       {sandbox_id: "local"}          ← 上轮写入，直接复用               │
 │     thread_data:   {workspace:"/path/ws",                                              │
 │                     uploads: "/path/up",                                               │
 │                     outputs: "/path/out"}         ← 上轮写入，直接复用               │
 │     title:         "财务分析报告"                 ← 首轮生成后不再变                  │
 │     artifacts:     ["report.pdf","chart.png"]     ← 历史产出物（累加的）              │
 │     todos:         [{id:..,status:..},..]         ← plan mode 下的任务列表            │
 │     uploaded_files: null                          ← 每轮开始前为 null                 │
 │     viewed_images:  {}                            ← 每轮结束后被清空                 │
 │   }                                                                                    │
 │                                                                                        │
 │  同时把本轮新的 user message 通过 add_messages reducer 追加进 messages                 │
 └──────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │ state（已含本轮 user message）
                                            ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【C】 Middleware 链 —— before_agent 阶段（按顺序依次执行）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  state 进入 before_agent 链
   │
   ├──▶ 1. ThreadDataMiddleware.before_agent
   │        检查 state["thread_data"] 是否已存在
   │        ┌─────────────────────────────────────────────────────────────┐
   │        │ lazy_init=True（默认）：                                     │
   │        │   只计算路径，不 mkdir（目录懒创建，工具调用时才建）           │
   │        │ lazy_init=False：                                            │
   │        │   立即 mkdir -p workspace/uploads/outputs，chmod 0o777      │
   │        └─────────────────────────────────────────────────────────────┘
   │         ── 如果 state["thread_data"] 为 None ──▶
   │             state["thread_data"] ← {
   │               workspace_path: "{base_dir}/threads/{id}/user-data/workspace"
   │               uploads_path:   "{base_dir}/threads/{id}/user-data/uploads"
   │               outputs_path:   "{base_dir}/threads/{id}/user-data/outputs"
   │             }
   │         ── 已有值则跳过（多轮时不重写）──
   │
   ├──▶ 2. UploadsMiddleware.before_agent
   │        扫描 uploads_path 目录，找出本轮新上传的文件
   │        ┌─────────────────────────────────────────────────────────────┐
   │        │ 检测逻辑：                                                    │
   │        │   比较当前磁盘上的文件列表 vs state["uploaded_files"]（历史）  │
   │        │   找出新增的文件                                              │
   │        └─────────────────────────────────────────────────────────────┘
   │         ── 有新文件 ──▶
   │             state["uploaded_files"] ← [{filename, size, path, ...}, ...]
   │             state["messages"] ← add_messages([HumanMessage(
   │               content="<uploaded_files>..新文件列表..</uploaded_files>\n{用户原消息}"
   │             )])
   │             （把上传信息拼进最后一条 HumanMessage）
   │         ── 无新文件 ── 跳过 ──
   │
   ├──▶ 3. SandboxMiddleware.before_agent（lazy_init=True 时，此处跳过）
   │        ┌─────────────────────────────────────────────────────────────┐
   │        │ lazy_init=False（非默认）才会在这里申请：                     │
   │        │   sandbox_id = provider.acquire(thread_id)                  │
   │        │   state["sandbox"] ← {sandbox_id: sandbox_id}              │
   │        │ lazy_init=True：sandbox 在第一次工具调用时懒申请             │
   │        └─────────────────────────────────────────────────────────────┘
   │
   └──▶ 其余 middleware 的 before_agent 为空实现，跳过
        （DanglingToolCall/Guardrail/ToolErrorHandling/Summarization/Todo 等）

  before_agent 结束，state 更新：
    messages       ← 追加了上传文件块（若有）
    thread_data    ← 三条路径（首轮写入）
    sandbox        ← sandbox_id（若 eager init）
    uploaded_files ← 本轮新上传文件列表

                                            │ 更新后的 state
                                            ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【D】 wrap_model_call —— 组装上下文，调用模型（循环执行，每轮工具完成后重复）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌───────────────────────────────────────────────────────────────────────────────────────┐
  │  wrap_model_call 是一个洋葱结构，每层中间件包一层，从外到内走 before 侧，             │
  │  调用模型，然后从内到外走 after 侧                                                    │
  └───────────────────────────────────────────────────────────────────────────────────────┘

  [before 侧，按顺序执行]
   │
   ├──▶ DanglingToolCallMiddleware（wrap_model_call before 侧）
   │        扫描 state["messages"] 历史
   │        找出「有 tool_calls 但无对应 ToolMessage」的 AIMessage（悬挂 tool_call）
   │        ── 发现悬挂 ──▶
   │            在悬挂 AIMessage 正后方插入占位 ToolMessage：
   │            {content:"[Interrupted]", status:"error"}
   │            （直接修改传给模型的 messages 列表，不经 add_messages reducer）
   │        ── 无悬挂 ── 透传 ──
   │
   ├──▶ SummarizationMiddleware（条件：enabled=true，wrap_model_call before 侧）
   │        统计 state["messages"] 的 token 数
   │        ── 触达 trigger 阈值（tokens/messages/fraction 任一） ──▶
   │            ┌──────────────────────────────────────────────────────────┐
   │            │  旧消息（超出 keep 边界的）→ 送给摘要模型（trim 到              │
   │            │  trim_tokens_to_summarize token 后再传）                    │
   │            │  摘要模型返回摘要文本                                        │
   │            │  构造新消息列表：                                            │
   │            │    [HumanMessage("Here is a summary...")]  ← 摘要          │
   │            │    [keep 范围内的近期消息原样]                               │
   │            │  注意：AI+ToolMessage 对不拆分（边界自动调整）               │
   │            └──────────────────────────────────────────────────────────┘
   │         ── 未触达 ── 透传 ──
   │
   ├──▶ TodoMiddleware（条件：is_plan_mode=true，wrap_model_call 的 before_model 钩子）
   │        检查 state["todos"] 非空 AND state["messages"] 里找不到 write_todos 痕迹
   │        ── 两者同时成立（write_todos 被摘要压缩掉了） ──▶
   │            state["messages"] ← add_messages([HumanMessage(
   │              name="todo_reminder",
   │              content="<system_reminder>Todo list still active: [当前 todos]</system_reminder>"
   │            )])
   │        ── 否则 ── 跳过 ──
   │
   ├──▶ ViewImageMiddleware（条件：model.supports_vision，wrap_model_call before 侧）
   │        检查最后一条 AIMessage 是否有 view_image 工具调用且已完成
   │        ── 条件满足 ──▶
   │            读 state["viewed_images"] 里的 base64 数据
   │            构造 HumanMessage 包含图片的 base64 内容
   │            临时追加进传给模型的 messages（不写入持久 state）
   │        ── 无图片 ── 透传 ──
   │
   ├──▶ DeferredToolFilterMiddleware（条件：tool_search.enabled）
   │        把 MCP 工具从绑给模型的 schema 列表里剔除
   │        模型只看到工具名（在 system prompt 的 <available-deferred-tools>）
   │        不看到完整 schema，直到主动调 tool_search 再加载
   │

  ─────────────────────────────────────────────────────────────────────────
  【D1】 组装系统提示词 system_prompt（每轮都重新拼一遍，不缓存）
  ─────────────────────────────────────────────────────────────────────────
  │
  │  apply_prompt_template(subagent_enabled, max_concurrent, agent_name)
  │  ├── <role>            : "{agent_name}"
  │  ├── {soul}            : 读 agents/{name}/SOUL.md（若有）
  │  ├── {memory_context}  : get_memory_data(agent_name)
  │  │                       └── 读 memory.json（带 mtime 缓存）
  │  │                           format_memory_for_injection(max_tokens=2000)
  │  │                           → "<memory>...</memory>"
  │  ├── <thinking_style>  : 固定内容 + {subagent_thinking}（条件插入）
  │  ├── <clarification>   : 固定内容
  │  ├── {skills_section}  : load_skills(enabled_only=True) → "<skill_system>..."
  │  ├── {deferred_tools}  : get_deferred_registry() → "<available-deferred-tools>..."
  │  ├── {subagent_section}: _build_subagent_section(n)（subagent_enabled=True 时）
  │  ├── <working_directory>: 固定内容（虚拟路径说明）
  │  ├── <response_style>  : 固定内容
  │  ├── <citations>       : 固定内容
  │  ├── <critical_reminders>: 固定 + {subagent_reminder}（条件插入）
  │  └── <current_date>    : datetime.now().strftime(...)
  │
  ─────────────────────────────────────────────────────────────────────────

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  【模型调用】                                                              │
  │                                                                          │
  │  model.invoke({                                                          │
  │    system: system_prompt（上面拼好的，含记忆/技能/subagent规则/日期）      │
  │    messages: [                                                           │
  │      ...历史消息（可能已被摘要压缩）...                                    │
  │      HumanMessage("用户本轮输入 [可能含上传文件块]")                       │
  │      [若有图片：HumanMessage(base64图片数据)]                             │
  │    ]                                                                     │
  │    tools: [bash, ls, read_file, write_file, str_replace,                 │
  │            web_search, present_file, ask_clarification,                  │
  │            task（subagent_enabled=True）,                                 │
  │            view_image（supports_vision=True）,                            │
  │            tool_search（tool_search.enabled）,                            │
  │            MCP工具（tool_search.enabled=false 时直接绑 schema）]           │
  │  })                                                                      │
  │                                                                          │
  │  返回：AIMessage {                                                        │
  │    content: "..." 或 ""                                                  │
  │    tool_calls: [{name, args, id}, ...]  ← 若模型决定调工具               │
  │    usage_metadata: {input_tokens, output_tokens, total_tokens}           │
  │  }                                                                       │
  └──────────────────────────────────────────────────────────────────────────┘

  state["messages"] ← add_messages([AIMessage])   ← 模型输出追加进历史

  [after 侧，按反序（从内到外）执行]
   │
   ├──▶ ViewImageMiddleware（after 侧）
   │        state["viewed_images"] ← {}
   │        （写入空字典，触发 merge_viewed_images 的「清空」语义）
   │        下一轮不会再带上本轮的图片 base64

                                            │ 模型输出已进入 messages
                                            ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【E】 after_model —— 检查和修改模型输出（before 工具执行）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   ├──▶ TokenUsageMiddleware.after_model
   │        读 messages[-1].usage_metadata → 写日志（不修改 state）
   │
   ├──▶ TitleMiddleware.after_model
   │        检查：state["title"] 为 None AND 对话里恰好有 1 条 human + ≥1 条 ai
   │        ── 条件满足（首轮结束后）──▶
   │            调轻量模型生成标题
   │            state["title"] ← "自动生成的标题"
   │        ── 已有 title 或不是首轮 ── 跳过 ──
   │
   ├──▶ SubagentLimitMiddleware.after_model（条件：subagent_enabled=True）
   │        检查 messages[-1].tool_calls 里 name=="task" 的调用数
   │        ── 数量 > max_concurrent ──▶
   │            截断：只保留前 max_concurrent 个 task 调用，丢弃多余的
   │            updated_msg = last_msg.model_copy(update={"tool_calls": truncated})
   │            state["messages"] ← add_messages([updated_msg])
   │            （同一 id 触发 add_messages 的替换语义，不是追加）
   │        ── 数量 ≤ max_concurrent ── 透传 ──
   │
   ├──▶ LoopDetectionMiddleware.after_model
   │        hash 最新一条 AIMessage 的 tool_calls（name+args，顺序无关）
   │        查 per-thread 滑动窗口（最近 20 次），统计同 hash 出现次数
   │        ── count ≥ warn_threshold（默认 3） ──▶
   │            state["messages"] ← add_messages([HumanMessage("[LOOP DETECTED] 停止重复...")])
   │        ── count ≥ hard_limit（默认 5） ──▶
   │            从 AIMessage 里清空 tool_calls（强制只输文字，不再调工具）
   │            state["messages"] ← add_messages([updated_msg])

   ─── 检查：messages[-1].tool_calls 是否为空 ───
       │                                     │
       │ 为空：无工具调用                     │ 非空：有工具调用
       ▼                                     ▼
    直接跳到【H】after_agent            进入【F】工具执行循环

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【F】 工具执行循环 —— wrap_tool_call（对每个 tool_call 重复）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  对 messages[-1].tool_calls 里的每一个 {name, args, id}：
   │
   ├──▶ GuardrailMiddleware.wrap_tool_call（条件：guardrails.enabled）
   │        用 GuardrailProvider.evaluate(tool_call) 做权限校验
   │        ── 拒绝 ──▶ 返回 ToolMessage(content="拒绝原因", status="error")，跳过实际执行
   │        ── 通过 ── 继续 ──
   │
   ├──▶ ToolErrorHandlingMiddleware.wrap_tool_call
   │        try: handler(request)
   │        except GraphBubbleUp: raise  ← 不能拦截 LangGraph 控制信号
   │        except Exception as e:
   │          返回 ToolMessage(content="Error: ...", status="error")  ← 不崩溃
   │
   ├──▶ ClarificationMiddleware.wrap_tool_call（最后执行）
   │        ── name == "ask_clarification" ──▶
   │            格式化问题 → ToolMessage(content=formatted_question)
   │            return Command(update={"messages":[tool_msg]}, goto=END)
   │                           ↑ 中断 Agent，等待用户下次输入
   │            ← 整个流程在这里暂停，不再执行后续工具 ─────────────────────────────────┐
   │        ── 其他工具 ── 正常执行 ──                                              │
   │                                                                              │
   ├─────────────────────────── 工具分支 ───────────────────────────────────────  │
   │                                                                              │
   │  ┌──────────────────────────────────────────────────────────────────────┐   │
   │  │  【F1】沙盒工具（bash/ls/read_file/write_file/str_replace）           │   │
   │  │                                                                      │   │
   │  │  ensure_sandbox_initialized(runtime)                                 │   │
   │  │    如果 state["sandbox"] 为 None 或 sandbox_id 不存在：              │   │
   │  │      sandbox_id = provider.acquire(thread_id)  ← lazy init          │   │
   │  │      runtime.state["sandbox"] ← {sandbox_id: sandbox_id}            │   │
   │  │  ensure_thread_directories_exist(runtime)                            │   │
   │  │    如果是 local sandbox 且目录还没建：                               │   │
   │  │      mkdir -p workspace/uploads/outputs                              │   │
   │  │      runtime.state["thread_directories_created"] ← True             │   │
   │  │                                                                      │   │
   │  │  is_local_sandbox？                                                  │   │
   │  │    是：validate → replace_virtual_path → execute → mask_output      │   │
   │  │    否：直接 sandbox.execute_command/read_file/write_file             │   │
   │  │                                                                      │   │
   │  │  返回 ToolMessage(content=output)                                    │   │
   │  │  state["messages"] ← add_messages([ToolMessage])                    │   │
   │  └──────────────────────────────────────────────────────────────────────┘   │
   │                                                                              │
   │  ┌──────────────────────────────────────────────────────────────────────┐   │
   │  │  【F2】present_file 工具                                              │   │
   │  │                                                                      │   │
   │  │  验证路径在 /mnt/user-data/outputs/ 下                               │   │
   │  │  return Command(update={                                             │   │
   │  │    "artifacts": [normalized_path],   ← merge_artifacts Reducer 追加  │   │
   │  │    "messages": [ToolMessage("Successfully presented files")]         │   │
   │  │  })                                                                  │   │
   │  │  state["artifacts"] ← merge_artifacts(old, new)  ← 追加去重         │   │
   │  │  state["messages"]  ← add_messages([ToolMessage])                   │   │
   │  └──────────────────────────────────────────────────────────────────────┘   │
   │                                                                              │
   │  ┌──────────────────────────────────────────────────────────────────────┐   │
   │  │  【F3】view_image 工具                                                │   │
   │  │                                                                      │   │
   │  │  读图片文件 → 转 base64                                              │   │
   │  │  state["viewed_images"] ← merge_viewed_images(old, {path: {base64}})│   │
   │  │  返回 ToolMessage（路径确认）                                         │   │
   │  │  （base64 数据等 ViewImageMiddleware 在下次 before_model 时注入）      │   │
   │  └──────────────────────────────────────────────────────────────────────┘   │
   │                                                                              │
   │  ┌──────────────────────────────────────────────────────────────────────┐   │
   │  │  【F4】write_todos 工具（plan mode）                                  │   │
   │  │                                                                      │   │
   │  │  解析新的 todo 列表                                                   │   │
   │  │  state["todos"] ← [{id, content, status}, ...]  ← 普通赋值（覆盖）   │   │
   │  │  返回 ToolMessage（确认）                                             │   │
   │  └──────────────────────────────────────────────────────────────────────┘   │
   │                                                                              │
   │  ┌──────────────────────────────────────────────────────────────────────┐   │
   │  │  【F5】tool_search 工具（tool_search.enabled）                        │   │
   │  │                                                                      │   │
   │  │  DeferredToolRegistry.search(query)                                  │   │
   │  │  返回匹配的工具完整 schema（ToolMessage）                             │   │
   │  │  模型下一轮调用时才真正绑定这个工具                                   │   │
   │  └──────────────────────────────────────────────────────────────────────┘   │
   │                                                                              │
   │  ┌──────────────────────────────────────────────────────────────────────┐   │
   │  │  【F6】task 工具 ──▶ 进入【G】子 Agent 编排                           │   │
   │  └──────────────────────────────────────────────────────────────────────┘   │
   │                                                                              │
   └─────────────────────────────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【G】 task 工具 → 子 Agent 编排（可多个并行）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  task_tool(runtime, description, prompt, subagent_type, tool_call_id)
   │
   ├─ 从 runtime.state 提取父状态快照：
   │    sandbox_state  ← state["sandbox"]
   │    thread_data    ← state["thread_data"]
   │    thread_id      ← runtime.context["thread_id"]
   │    parent_model   ← runtime.config["metadata"]["model_name"]
   │    trace_id       ← runtime.config["metadata"]["trace_id"]
   │
   ├─ get_subagent_config(subagent_type)  → SubagentConfig
   │    name, system_prompt, tools(白名单), disallowed_tools(黑名单),
   │    model("inherit"), max_turns, timeout_seconds
   │
   ├─ get_available_tools(subagent_enabled=False)
   │    ← 父侧工具集，但强制不含 task 工具
   │
   ├─ SubagentExecutor(config, tools, sandbox_state, thread_data, thread_id, trace_id)
   │    ├─ _filter_tools(all_tools, config.tools, config.disallowed_tools)
   │    │    general-purpose: tools=None, black=[task, ask_clarification, present_files]
   │    │    bash:            tools=[bash,ls,read_file,write_file,str_replace], black=[task...]
   │    └─ self.tools ← 过滤后的工具集
   │
   ├─ executor.execute_async(prompt, task_id=tool_call_id)
   │    _background_tasks[task_id] = SubagentResult(PENDING)
   │    _scheduler_pool.submit(run_task)  ← 后台线程池
   │    return task_id
   │
   ├─ SSE: writer({"type":"task_started", "task_id":task_id, "description":description})
   │
   └─ while True（轮询，5s 间隔）:
        result = get_background_task_result(task_id)
        if new ai_messages:
          SSE: writer({"type":"task_running", "task_id":task_id, "message":msg})
        if COMPLETED: break → return "Task Succeeded. Result: ..."
        if FAILED:    break → return "Task failed. Error: ..."
        if TIMED_OUT: break → return "Task timed out..."
        time.sleep(5)

  ────────────────────────────────────────────────────────────────────────────
  【G1】后台线程：子 Agent 的独立状态和执行
  ────────────────────────────────────────────────────────────────────────────

  _execution_pool（独立于调度池）
   │
   ├─ asyncio.run(_aexecute(prompt, result_holder))
   │
   ├─ _build_initial_state(prompt):
   │    子 Agent 初始 state = {
   │      messages:    [HumanMessage(content=prompt)]  ← 只有一条，不继承父历史
   │      sandbox:     {sandbox_id: "local"}           ← 从父 state 复制
   │      thread_data: {workspace:"..", uploads:"..", outputs:".."}  ← 从父 state 复制
   │      （其他字段：title/artifacts/todos/uploaded_files/viewed_images 均不初始化）
   │    }
   │
   ├─ _create_agent():
   │    model = create_chat_model(parent_model, thinking_enabled=False)
   │    middlewares = build_subagent_runtime_middlewares()
   │      [ThreadDataMiddleware, SandboxMiddleware,
   │       (GuardrailMiddleware), ToolErrorHandlingMiddleware]
   │      ← 没有：UploadsMiddleware/DanglingToolCall/Summarization/Todo/
   │               Title/Memory/ViewImage/DeferredToolFilter/SubagentLimit/
   │               LoopDetection/ClarificationMiddleware
   │    create_agent(model, self.tools, middlewares, system_prompt, state_schema=ThreadState)
   │
   ├─ run_config = {
   │    recursion_limit: config.max_turns,      ← 最多执行轮数
   │    configurable: {thread_id: thread_id}    ← 共享线程目录
   │    ← 没有 checkpointer：子 Agent 状态不持久化
   │  }
   │
   └─ async for chunk in agent.astream(state, config=run_config, stream_mode="values"):
        final_state = chunk
        if 最后一条是 AIMessage:
          result.ai_messages.append(last_ai.model_dump())  ← 收集中间过程
                                                              task_tool 轮询到后推 SSE
   │
   │  子 Agent 内部循环（与父 Agent 相同的 D→E→F 过程，但用精简中间件）：
   │  ┌────────────────────────────────────────────────────────────────────┐
   │  │  子 Agent 自己的 state（在内存里，无 checkpoint）：                 │
   │  │    messages:    逐步增长的对话（tool_calls + ToolMessages）         │
   │  │    sandbox:     {sandbox_id: "local"}  ← 初始时从父复制来的        │
   │  │    thread_data: {workspace, uploads, outputs}  ← 同父路径          │
   │  │    （其余字段在子 Agent 里基本不写）                               │
   │  │                                                                    │
   │  │  子 Agent 工具调用（同父，但无 task）：                            │
   │  │    bash → 写到 /mnt/user-data/workspace|outputs/                   │
   │  │           ← 同一块磁盘，父可以 read_file 读到                     │
   │  │    read_file → 可以读父写的文件                                    │
   │  └────────────────────────────────────────────────────────────────────┘
   │
   ▼ 子 Agent 结束
  result.result = 最后一条 AIMessage 的 content
  result.status = COMPLETED
  _background_tasks[task_id] 更新

  ────────────────────────────────────────────────────────────────────────────

  task_tool 轮询检测到 COMPLETED
  返回字符串: "Task Succeeded. Result: {result.result}"
   │
   ▼
框架把返回值包成：
  ToolMessage(content="Task Succeeded. Result: ...", tool_call_id=tool_call_id)
  state["messages"] ← add_messages([ToolMessage])   ← 子任务结论追加进父历史
   │
   ▼  （所有 task 都完成后）
返回【D】wrap_model_call，再次调模型（父 Agent 汇总）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【H】 Middleware 链 —— after_agent 阶段（整轮含所有工具结束后，按顺序执行）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   ├──▶ SandboxMiddleware.after_agent
   │        get_sandbox_provider().release(sandbox_id)
   │        ← LocalSandboxProvider.release() 是空操作（单例复用）
   │        ← AioSandboxProvider.release() 会真正释放容器（按配置）
   │
   ├──▶ TokenUsageMiddleware.after_agent → 无（日志在 after_model 里已记）
   │
   ├──▶ TitleMiddleware.after_model（实际在 after_model 阶段，此处补说明）
   │        ← 已在【E】阶段说明
   │
   ├──▶ MemoryMiddleware.after_agent
   │        _filter_messages_for_memory(state["messages"])
   │        │  保留：HumanMessage（剥离 <uploaded_files> 块）+ 无 tool_calls 的 AIMessage
   │        │  丢弃：ToolMessage、带 tool_calls 的 AIMessage、纯上传轮次
   │        ▼
   │        filtered_messages 里至少有 1 user + 1 assistant？
   │        ── 是 ──▶ memory_queue.add(thread_id, filtered_messages, agent_name)
   │                  │  同 thread_id 去重（替换旧条目）
   │                  │  重置防抖定时器（30s）
   │        ── 否 ── 跳过 ──

                                            │ after_agent 结束
                                            ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【I】 状态持久化 —— Checkpointer Save
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 ┌─────────────────────────────────────────────────────────────────────────────────────────┐
 │  checkpointer.put(thread_id, state)                                                     │
 │                                                                                         │
 │  本轮结束后的 state 快照（将在下一轮开始时被 B 阶段重新加载）：                          │
 │                                                                                         │
 │   state = {                                                                             │
 │     messages:      [...全部历史，含本轮 AI 回复及所有 ToolMessage...]                    │
 │                    ← add_messages Reducer 保证追加合并，不覆盖                          │
 │     sandbox:       {sandbox_id: "local"}                                               │
 │                    ← 下轮直接复用，不重新 acquire                                      │
 │     thread_data:   {workspace:"..", uploads:"..", outputs:".."}                        │
 │                    ← 下轮直接复用                                                      │
 │     title:         "本轮生成的标题"（或沿用上轮的）                                    │
 │                    ← 只在首轮写一次                                                    │
 │     artifacts:     ["report.pdf","chart.png","analysis.csv"]                           │
 │                    ← merge_artifacts Reducer 保证累加去重                              │
 │     todos:         [{id, content, status},...] 或 null                                │
 │                    ← plan mode 下的任务列表                                            │
 │     uploaded_files: null（或本轮的文件列表）                                           │
 │                    ← 下轮 before_agent 时用于对比新文件                               │
 │     viewed_images:  {}                                                                 │
 │                    ← ViewImageMiddleware 在 after_model 时清空，不带入下轮             │
 │   }                                                                                     │
 └─────────────────────────────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【J】 SSE 事件流回前端（贯穿整个过程的异步推送）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  LangGraph 自动推送（stream_mode="messages-tuple" & "values"）：
    ├── messages-tuple 事件: 每条新消息（含工具调用、工具结果、AI 回复）逐步推
    └── values 事件:         每次 state 变化后推完整 state 快照
                              （含 title / artifacts / todos 等字段）

  task_tool 用 get_stream_writer() 手动推送：
    ├── {type:"task_started",   task_id, description}         ← 子任务开始
    ├── {type:"task_running",   task_id, message, message_index, total_messages}
    │                                                           ← 子 Agent 每产出一条 AI 消息
    ├── {type:"task_completed", task_id, result}               ← 子任务成功
    ├── {type:"task_failed",    task_id, error}                ← 子任务失败
    └── {type:"task_timed_out", task_id}                       ← 子任务超时

  前端收到的完整事件时序（单批 3 个 task 的例子）：
  ┌──────────────────────────────────────────────────────────────────┐
  │  values: {messages:[..., AIMessage(tool_calls=[t1,t2,t3])]}     │ ← 模型决定发 3 个 task
  │  task_started(t1, "财务分析")                                    │
  │  task_started(t2, "舆情监管")                                    │
  │  task_started(t3, "行业趋势")                                    │
  │  task_running(t1, AI消息1)                                       │ ← 子 A 首轮
  │  task_running(t2, AI消息1)                                       │ ← 子 B 首轮
  │  task_running(t1, AI消息2)                                       │ ← 子 A 继续
  │  task_running(t3, AI消息1)                                       │ ← 子 C 首轮
  │  task_completed(t2, "舆情分析结果...")                            │ ← 子 B 完成
  │  task_completed(t1, "财务分析结果...")                            │ ← 子 A 完成
  │  task_completed(t3, "行业分析结果...")                            │ ← 子 C 完成
  │  messages-tuple: ToolMessage(t1, "Task Succeeded...")            │ ← 父收到结果
  │  messages-tuple: ToolMessage(t2, "Task Succeeded...")            │
  │  messages-tuple: ToolMessage(t3, "Task Succeeded...")            │
  │  messages-tuple: AIMessage("综合分析：...")                       │ ← 父汇总
  │  values: {messages:[..全部], title:"财务分析", artifacts:[...]}  │ ← 完整 state 快照
  └──────────────────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【K】 后台异步：记忆更新管线（与主对话完全解耦）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  MemoryUpdateQueue（全局单例，防抖 30 秒）

  ┌ MemoryMiddleware 入队 ──────────────────────────────────────────────────────┐
  │                                                                             │
  │  同 thread_id 旧条目被新条目替换（30s 内多轮快速对话只处理最新一次）          │
  │  重置 threading.Timer(30s, _process_queue)                                  │
  │                                                                             │
  └─────────────────────────────────────────────────────────────────────────────┘
                                │ 30 秒无新入队
                                ▼
  ┌ _process_queue ──────────────────────────────────────────────────────────────┐
  │  for context in contexts_to_process:                                        │
  │    MemoryUpdater.update_memory(messages, thread_id, agent_name)             │
  │      │                                                                      │
  │      ├─ get_memory_data(agent_name)                                         │
  │      │    ├─ 读 _memory_cache[(agent_name)]（mtime 检测失效）               │
  │      │    └─ 或 从 memory.json / agents/{name}/memory.json 加载             │
  │      │                                                                      │
  │      ├─ format_conversation_for_update(messages) → 对话文本                 │
  │      │                                                                      │
  │      ├─ LLM 调用（MEMORY_UPDATE_PROMPT）                                    │
  │      │    输入：当前 memory.json 内容 + 对话文本                            │
  │      │    输出：JSON delta {                                                 │
  │      │      "user": {workContext: {summary, shouldUpdate}, ...}             │
  │      │      "history": {recentMonths: {summary, shouldUpdate}, ...}         │
  │      │      "newFacts": [{content, category, confidence}, ...]              │
  │      │      "factsToRemove": ["fact_id_1", ...]                             │
  │      │    }                                                                 │
  │      │                                                                      │
  │      ├─ _apply_updates(current_memory, update_data, thread_id)              │
  │      │    ├─ 更新 user 段落（shouldUpdate=true 才更新）                     │
  │      │    ├─ 更新 history 段落（同上）                                      │
  │      │    ├─ 删除 factsToRemove 指定的 fact                                 │
  │      │    ├─ 添加 newFacts（confidence ≥ 0.7，内容去重，数量 ≤ max_facts）  │
  │      │    └─ 超出 max_facts(100) 时按置信度排序，保留最高的                 │
  │      │                                                                      │
  │      ├─ _strip_upload_mentions_from_memory(updated_memory)                  │
  │      │    用正则删除所有段落里关于「上传文件事件」的句子                     │
  │      │                                                                      │
  │      └─ _save_memory_to_file(updated_memory, agent_name)                   │
  │           temp_path = memory.tmp                                            │
  │           json.dump → temp_path                                             │
  │           temp_path.replace(memory.json)  ← 原子写入                       │
  │           _memory_cache[(agent_name)] ← (memory_data, new_mtime)           │
  └──────────────────────────────────────────────────────────────────────────────┘

  下次对话时，apply_prompt_template → _get_memory_context
    get_memory_data → 检测 mtime 变化 → 重新加载（自动失效）
    format_memory_for_injection(max_tokens=2000) → "<memory>...</memory>"
    注入系统提示词

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【L】 跨进程：Gateway API 对状态相关数据的读写
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Gateway :8001（独立进程，不运行 Agent，但读写同一套磁盘数据）

  ┌──────────────────────────────────────────────────────────────────────────────┐
  │  POST /api/threads/{id}/uploads                                              │
  │    ← 文件写到 {base_dir}/threads/{id}/user-data/uploads/                    │
  │    ← 下次 Agent 运行时 UploadsMiddleware 会检测到这个新文件                 │
  │                                                                              │
  │  DELETE /api/threads/{id}                                                    │
  │    ← 删除 {base_dir}/threads/{id}/ 整个目录                                 │
  │    ← state 里的 thread_data 路径指向的磁盘已清空                            │
  │                                                                              │
  │  GET /api/threads/{id}/artifacts/{path}                                     │
  │    ← 读 state["artifacts"] 列表里的文件内容                                 │
  │                                                                              │
  │  POST /api/memory/reload                                                     │
  │    ← 强制重置 _memory_cache，下次 get_memory_data 从磁盘重新加载            │
  │                                                                              │
  │  PUT /api/mcp                                                                │
  │    ← 写 extensions_config.json（mtime 变化）                                │
  │    ← LangGraph 进程下次 get_cached_mcp_tools 检测 mtime，自动重载 MCP 工具  │
  │                                                                              │
  │  PUT /api/skills/{name}                                                      │
  │    ← 写 extensions_config.json（skills.enabled）                            │
  │    ← 下次 apply_prompt_template 重新 load_skills，技能出现/消失在提示词里   │
  └──────────────────────────────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【M】 多轮对话时 state 字段的变化规律（汇总）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────┬──────────────┬──────────────────────────────────────────────┐
  │ 字段              │ 写入时机      │ 特性                                         │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ messages         │ 每轮持续追加   │ add_messages Reducer，不覆盖，含全历史         │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ sandbox          │ 第一次工具调用 │ lazy init，之后复用，不重复 acquire           │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ thread_data      │ 第一轮 before │ 只写一次，之后从 checkpoint 复用              │
  │                  │ _agent       │                                               │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ title            │ 第一轮结束后   │ after_model 首轮生成，之后不再改动            │
  │                  │ after_model  │                                               │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ artifacts        │ present_file │ merge_artifacts Reducer，跨轮累加去重         │
  │                  │ 工具调用时    │                                               │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ todos            │ write_todos  │ 普通赋值（覆盖），plan mode 下每步更新         │
  │                  │ 工具调用时    │ TodoMiddleware 在摘要后补救注入提醒           │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ uploaded_files   │ before_agent │ 每轮覆盖（本轮新增文件）                       │
  ├──────────────────┼──────────────┼──────────────────────────────────────────────┤
  │ viewed_images    │ view_image   │ merge_viewed_images Reducer                   │
  │                  │ 工具 + after │ 写入 base64；after_model 写 {} 触发清空        │
  │                  │ _model 清空  │ 下一轮不带上一轮的图片                        │
  └──────────────────┴──────────────┴──────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 【N】 父 vs 子 Agent 的状态对比
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────┬──────────────────────────────┬──────────────────────────────────┐
  │ 维度              │ 父 Agent（Lead）              │ 子 Agent（Subagent）              │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ checkpointer     │ 有（sqlite/postgres/memory）   │ 无（只在内存里跑）                │
  │ state 持久化      │ 每轮结束写盘                   │ 不写盘，跑完消失                  │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ messages 初始值   │ 从 checkpoint 加载历史         │ 只有一条 HumanMessage(prompt)    │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ sandbox          │ 从 checkpoint 复用             │ 从父 state 复制（同一个 ID）      │
  │ thread_data      │ 从 checkpoint 复用             │ 从父 state 复制（同一套路径）     │
  │ （共享磁盘）       │                              │ ← 子写的文件，父能读             │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ title            │ 由 TitleMiddleware 写入        │ 不写（中间件无 TitleMiddleware）  │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ artifacts        │ 由 present_file 工具写入        │ present_file 在黑名单，不写       │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ 结果如何回父       │ 通过 ToolMessage(task 结果)    │ 子最后一条 AIMessage.content     │
  │                  │ 追加进父 messages              │ → 字符串 → ToolMessage           │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ 记忆              │ MemoryMiddleware after_agent   │ 无 MemoryMiddleware，不更新记忆  │
  ├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
  │ 中间件数量         │ 最多 16 个                    │ 最少 4 个（基础链）               │
  └──────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## 关键源码对照

| 图中节点 | 源码位置 |
|---------|----------|
| A — Nginx 路由 | `docker/nginx/nginx.conf` |
| B — 状态加载 | `agents/checkpointer/async_provider.py` |
| C.1 — ThreadDataMiddleware | `agents/middlewares/thread_data_middleware.py` |
| C.2 — UploadsMiddleware | `agents/middlewares/uploads_middleware.py` |
| C.3 — SandboxMiddleware | `sandbox/middleware.py` |
| D — wrap_model_call 各层 | `agents/middlewares/dangling_tool_call_middleware.py` / `summarization_config.py` / `todo_middleware.py` / `view_image_middleware.py` |
| D1 — 系统提示词组装 | `agents/lead_agent/prompt.py` `apply_prompt_template` |
| E — after_model 各层 | `agents/middlewares/token_usage_middleware.py` / `title_middleware.py` / `subagent_limit_middleware.py` / `loop_detection_middleware.py` |
| F1 — 沙盒工具 | `sandbox/tools.py` `ensure_sandbox_initialized` / `bash_tool` |
| F2 — present_file | `tools/builtins/present_file_tool.py` |
| F3 — view_image | `tools/builtins/view_image_tool.py` |
| F4 — write_todos | `TodoListMiddleware`（LangChain 内置） |
| F5 — tool_search | `tools/builtins/tool_search.py` |
| F6 — task | `tools/builtins/task_tool.py` |
| G — SubagentExecutor | `subagents/executor.py` |
| G1 — 子 Agent 创建 | `subagents/executor.py` `_build_initial_state` / `_create_agent` |
| H — after_agent | `sandbox/middleware.py` / `agents/middlewares/memory_middleware.py` |
| I — 状态持久化 | `agents/checkpointer/async_provider.py` |
| J — SSE 事件 | `langgraph` 框架内置 + `tools/builtins/task_tool.py` `get_stream_writer()` |
| K — 记忆管线 | `agents/memory/queue.py` / `agents/memory/updater.py` |
| L — Gateway 跨进程读写 | `app/gateway/routers/` 各路由文件 |
