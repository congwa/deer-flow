# 防抖队列批处理加 LLM 摘要原子写入，让记忆更新不阻塞主对话并持久化到文件

## 要解决的问题

一个「有记忆」的 Agent 需要在对话结束后把关键信息提取并保存，以便在下次对话时能个性化地回应。但这件事有几个约束：

1. **不能阻塞当前对话**：记忆更新需要调用 LLM，这个 LLM 调用不应该让用户等待；
2. **同一线程多轮对话可能快速发生**：用户连续发多条消息，每条都触发一次记忆更新会浪费资源；
3. **记忆文件需要原子写入**：如果写到一半进程崩了，记忆文件不能损坏；
4. **注入系统提示时要按 token 预算裁减**：记忆内容不能无限增长占满上下文。

---

## 五个组件的分工

```
对话结束
    │
    ▼
MemoryMiddleware.after_agent
（过滤消息，去掉工具调用噪声，入队）
    │
    ▼
MemoryUpdateQueue
（防抖：同线程重复入队只保留最新，30 秒后批量处理）
    │
    ▼
MemoryUpdater.update_memory
（读 JSON → LLM → 解析 delta → apply → 去上传噪声 → 原子写入）
    │
    ▼
memory.json / agents/{name}/memory.json
    │
下次对话开始时
    ▼
apply_prompt_template → _get_memory_context
（按 token 预算格式化 → 注入 <memory> 块）
```

---

## 第一步：`MemoryMiddleware` 过滤消息

`after_agent` 在每轮对话结束后执行：

```python
def after_agent(self, state, runtime):
    config = get_memory_config()
    if not config.enabled:
        return None

    thread_id = runtime.context.get("thread_id")
    messages = state.get("messages", [])

    filtered_messages = _filter_messages_for_memory(messages)

    user_messages = [m for m in filtered_messages if m.type == "human"]
    assistant_messages = [m for m in filtered_messages if m.type == "ai"]

    if not user_messages or not assistant_messages:
        return None     # 至少需要一组 user+assistant 才入队

    queue = get_memory_queue()
    queue.add(thread_id=thread_id, messages=filtered_messages, agent_name=self._agent_name)
```

`_filter_messages_for_memory` 去掉不适合写入长期记忆的内容：
- **ToolMessage**：工具执行结果，流水账，不是用户真正的信息；
- **带 `tool_calls` 的 AIMessage**：中间步骤，不是最终答案；
- **`<uploaded_files>` 块**：上传文件是当次会话的，下次会话找不到这些文件，写进记忆反而有害；
- **纯上传轮次**（human 消息去掉上传块后什么都没剩）：整轮人+AI 都跳过。

---

## 第二步：防抖队列

```python
class MemoryUpdateQueue:
    def add(self, thread_id, messages, agent_name):
        context = ConversationContext(thread_id=thread_id, messages=messages, agent_name=agent_name)

        with self._lock:
            # 同一 thread_id 已有待处理的，用最新消息替换旧的
            self._queue = [c for c in self._queue if c.thread_id != thread_id]
            self._queue.append(context)

            self._reset_timer()   # 重置防抖定时器

    def _reset_timer(self):
        if self._timer is not None:
            self._timer.cancel()

        self._timer = threading.Timer(
            config.debounce_seconds,   # 默认 30 秒
            self._process_queue,
        )
        self._timer.daemon = True
        self._timer.start()
```

**去重逻辑**：同一个 `thread_id` 在队列里只保留最新的一条。用户在 30 秒内发了 5 条消息，只有最后一条（含所有 5 轮的 messages）会被处理，避免对同一线程重复更新记忆 5 次。

**防抖定时器**：每次入队都会重置定时器。30 秒内如果又有新消息入队，计时器从头开始，这样快速发消息的对话不会频繁触发记忆更新。如果 30 秒内没有新消息，定时器到期，批量处理所有待处理项。

---

## 第三步：LLM 摘要更新

`MemoryUpdater.update_memory` 的流程：

```python
def update_memory(self, messages, thread_id, agent_name):
    # 1. 读当前记忆
    current_memory = get_memory_data(agent_name)

    # 2. 把对话格式化成文本
    conversation_text = format_conversation_for_update(messages)

    # 3. 构造 prompt，让 LLM 分析对话，返回 JSON delta
    prompt = MEMORY_UPDATE_PROMPT.format(
        current_memory=json.dumps(current_memory, indent=2),
        conversation=conversation_text,
    )

    # 4. 调用 LLM
    model = self._get_model()
    response = model.invoke(prompt)
    response_text = _extract_text(response.content).strip()

    # 5. 解析 JSON（处理 markdown 代码块格式）
    if response_text.startswith("```"):
        lines = response_text.split("\n")
        response_text = "\n".join(lines[1:-1] if lines[-1] == "```" else lines[1:])
    update_data = json.loads(response_text)

    # 6. 应用更新
    updated_memory = self._apply_updates(current_memory, update_data, thread_id)

    # 7. 去掉上传事件相关的句子
    updated_memory = _strip_upload_mentions_from_memory(updated_memory)

    # 8. 原子写入
    return _save_memory_to_file(updated_memory, agent_name)
```

`MEMORY_UPDATE_PROMPT` 要求 LLM 返回特定格式的 JSON：

```json
{
  "user": {
    "workContext": {"summary": "...", "shouldUpdate": true},
    "personalContext": {"summary": "...", "shouldUpdate": false},
    "topOfMind": {"summary": "...", "shouldUpdate": true}
  },
  "history": {
    "recentMonths": {"summary": "...", "shouldUpdate": true},
    "earlierContext": {"summary": "...", "shouldUpdate": false},
    "longTermBackground": {"summary": "...", "shouldUpdate": false}
  },
  "newFacts": [
    {"content": "用户使用 Python 3.12", "category": "knowledge", "confidence": 0.95}
  ],
  "factsToRemove": ["fact_abc123"]
}
```

`shouldUpdate: false` 的字段原样保留，不会被新内容覆盖——这避免了每次对话都重写所有记忆字段，只更新有新信息的部分。

---

## 第四步：`_apply_updates` 的合并逻辑

```python
def _apply_updates(self, current_memory, update_data, thread_id):
    config = get_memory_config()
    now = datetime.utcnow().isoformat() + "Z"

    # 更新 user 段落
    user_updates = update_data.get("user", {})
    for section in ["workContext", "personalContext", "topOfMind"]:
        section_data = user_updates.get(section, {})
        if section_data.get("shouldUpdate") and section_data.get("summary"):
            current_memory["user"][section] = {
                "summary": section_data["summary"],
                "updatedAt": now,
            }

    # 更新 history 段落（同逻辑）

    # 删除指定的 facts
    facts_to_remove = set(update_data.get("factsToRemove", []))
    current_memory["facts"] = [
        f for f in current_memory.get("facts", [])
        if f.get("id") not in facts_to_remove
    ]

    # 添加新 facts（去重 + 置信度过滤 + 数量上限）
    existing_fact_keys = {_fact_content_key(f.get("content")) for f in current_memory["facts"]}
    for fact in update_data.get("newFacts", []):
        if fact.get("confidence", 0) < config.fact_confidence_threshold:  # 默认 0.7
            continue
        normalized_content = fact.get("content", "").strip()
        fact_key = _fact_content_key(normalized_content)
        if fact_key in existing_fact_keys:
            continue    # 去重（内容规范化后相同则跳过）
        fact_entry = {
            "id": f"fact_{uuid4().hex[:8]}",
            "content": normalized_content,
            "category": fact.get("category", "context"),
            "confidence": fact.get("confidence"),
            "createdAt": now,
            "source": thread_id or "unknown",
        }
        current_memory["facts"].append(fact_entry)
        existing_fact_keys.add(fact_key)

    # 超出 max_facts 时按置信度排序，保留最高的
    if len(current_memory["facts"]) > config.max_facts:  # 默认 100
        current_memory["facts"] = sorted(
            current_memory["facts"],
            key=lambda f: f.get("confidence", 0),
            reverse=True,
        )[:config.max_facts]

    return current_memory
```

---

## 第五步：原子写入

```python
def _save_memory_to_file(memory_data, agent_name):
    file_path = _get_memory_file_path(agent_name)
    file_path.parent.mkdir(parents=True, exist_ok=True)

    memory_data["lastUpdated"] = datetime.utcnow().isoformat() + "Z"

    # 先写临时文件，再 rename（原子性）
    temp_path = file_path.with_suffix(".tmp")
    with open(temp_path, "w", encoding="utf-8") as f:
        json.dump(memory_data, f, indent=2, ensure_ascii=False)

    temp_path.replace(file_path)  # 在大多数系统上是原子操作
```

`rename` 在 POSIX 系统（Linux、macOS）上是原子操作：要么旧文件，要么新文件，不会出现半写状态。如果进程在写临时文件中途崩了，旧的 `memory.json` 不受影响，`memory.tmp` 只是一个孤立的临时文件。

**mtime 缓存失效**：

```python
_memory_cache: dict[str | None, tuple[dict, float | None]] = {}

def get_memory_data(agent_name):
    file_path = _get_memory_file_path(agent_name)
    current_mtime = file_path.stat().st_mtime if file_path.exists() else None

    cached = _memory_cache.get(agent_name)
    if cached is None or cached[1] != current_mtime:
        memory_data = _load_memory_from_file(agent_name)
        _memory_cache[agent_name] = (memory_data, current_mtime)
        return memory_data

    return cached[0]
```

Gateway API 的 `POST /api/memory/reload` 或直接编辑 `memory.json` 后，下次 `get_memory_data` 检测到 mtime 变了，自动重新加载，不需要重启进程。

---

## 第六步：注入系统提示

在下一次对话开始时，`apply_prompt_template` 调用 `_get_memory_context`：

```python
def _get_memory_context(agent_name):
    config = get_memory_config()
    if not config.enabled or not config.injection_enabled:
        return ""

    memory_data = get_memory_data(agent_name)
    memory_content = format_memory_for_injection(
        memory_data,
        max_tokens=config.max_injection_tokens,    # 默认 2000
    )

    if not memory_content.strip():
        return ""

    return f"<memory>\n{memory_content}\n</memory>\n"
```

`format_memory_for_injection` 按置信度降序排列 facts，然后依次累加 token 计数，装不下就停。最后如果总长度仍超预算，按字符比例估算后截断并加 `\n...`。

---

## 记忆文件的数据结构

```json
{
  "version": "1.0",
  "lastUpdated": "2026-03-26T10:00:00Z",
  "user": {
    "workContext": {"summary": "...", "updatedAt": "..."},
    "personalContext": {"summary": "...", "updatedAt": "..."},
    "topOfMind": {"summary": "...", "updatedAt": "..."}
  },
  "history": {
    "recentMonths": {"summary": "...", "updatedAt": "..."},
    "earlierContext": {"summary": "...", "updatedAt": "..."},
    "longTermBackground": {"summary": "...", "updatedAt": "..."}
  },
  "facts": [
    {
      "id": "fact_a1b2c3d4",
      "content": "用户主要使用 Python 和 TypeScript",
      "category": "knowledge",
      "confidence": 0.95,
      "createdAt": "2026-03-01T00:00:00Z",
      "source": "thread_abc"
    }
  ]
}
```

**全局记忆**：`{base_dir}/memory.json`  
**Per-agent 记忆**：`{base_dir}/agents/{name}/memory.json`，不同 agent 各自维护，互不影响。

---

## 完整时序图

```
对话第 N 轮结束
    │
    ▼
MemoryMiddleware.after_agent
    ├── 过滤：去掉工具调用、上传块
    ├── 检查：至少有 1 user + 1 assistant
    └── queue.add(thread_id, filtered_messages)
                        │
                        ▼
                MemoryUpdateQueue
                    ├── 同 thread_id 去重（替换旧条目）
                    └── 重置定时器（30s）
                                │
                  30 秒内无新消息 │
                                ▼
                        _process_queue()
                            │
                            ▼
                    MemoryUpdater.update_memory
                        ├── get_memory_data（mtime 缓存）
                        ├── format_conversation（对话格式化）
                        ├── LLM 调用（MEMORY_UPDATE_PROMPT）
                        ├── json.loads（解析 delta）
                        ├── _apply_updates（合并、去重、限数）
                        ├── _strip_upload_mentions（清洗上传事件）
                        └── _save_memory_to_file（tmp → rename）
                                                │
下次对话开始                                    ▼
    │                               memory.json 已更新
    ▼
apply_prompt_template
    └── _get_memory_context
            ├── get_memory_data（mtime 缓存，自动检测更新）
            ├── format_memory_for_injection（按 token 预算格式化）
            └── 返回 "<memory>...</memory>" 注入系统提示
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `agents/middlewares/memory_middleware.py` | `MemoryMiddleware`，消息过滤，入队 |
| `agents/memory/queue.py` | `MemoryUpdateQueue`，防抖定时器，批量处理 |
| `agents/memory/updater.py` | `MemoryUpdater`，LLM 调用，`_apply_updates`，原子写入，mtime 缓存 |
| `agents/memory/prompt.py` | `MEMORY_UPDATE_PROMPT`，`format_memory_for_injection`（token 预算） |
| `agents/lead_agent/prompt.py` | `_get_memory_context`，注入时机 |
| `config/memory_config.py` | `MemoryConfig`，所有配置项 |
