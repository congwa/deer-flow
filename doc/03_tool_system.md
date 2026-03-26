# 三层来源按需组装，让运行时工具集在灵活扩展的同时保持可控

## 要解决的问题

一个通用 Agent 系统面临两个矛盾的需求：工具越多越好（覆盖面广）、但工具越少越安全（上下文不被 schema 污染、子 Agent 不能乱调）。

具体来说：
- **父 Agent** 需要搜索、执行、文件操作、委派子任务等全套能力；
- **子 Agent** 只应该有任务相关的工具，绝对不能再出现委派工具（否则递归无限嵌套）；
- **MCP 工具**数量可能很多，把所有 schema 都塞进模型上下文，context 利用率会变差；
- **视觉工具**只在模型支持多模态时才有意义，硬编码进去反而出错。

DeerFlow 的解法是把工具来源分成三层，每层用不同的开关控制，在 `get_available_tools` 一个函数里完成组装。

---

## `get_available_tools` 的三层来源

```
get_available_tools(groups, include_mcp, model_name, subagent_enabled)
         │
         ├── 第一层：配置文件工具
         │     config.yaml 里 tools[] 数组
         │     用 resolve_variable 按路径加载（如 "deerflow.sandbox.tools:bash_tool"）
         │     可选按 groups 过滤，子 Agent 有自己的 tool_groups
         │
         ├── 第二层：内置工具（按开关决定哪些加入）
         │     present_file_tool    ← 始终加入（父/子皆可）
         │     ask_clarification_tool ← 始终加入（父/子皆可）
         │     task_tool            ← 仅 subagent_enabled=True 时加入（父专属）
         │     view_image_tool      ← 仅 model.supports_vision=True 时加入
         │
         └── 第三层：MCP 工具（include_mcp=True 时）
               读 extensions_config.json（从磁盘读，不用进程内缓存）
               按 enabled=true 过滤服务器
               get_cached_mcp_tools() 取工具列表
                  └─ 若 tool_search 开启：全部进延迟注册表，只暴露 tool_search 工具
                  └─ 若 tool_search 关闭：全部 schema 直接绑给模型
```

---

## 第一层：配置文件工具（config.yaml）

`config.example.yaml` 里的典型配置：

```yaml
tools:
  - use: deerflow.sandbox.tools:bash_tool
    group: default
  - use: deerflow.sandbox.tools:ls_tool
    group: default
  - use: deerflow.sandbox.tools:read_file_tool
    group: default
  - use: deerflow.sandbox.tools:write_file_tool
    group: default
  - use: deerflow.sandbox.tools:str_replace_tool
    group: default
  - use: deerflow.community.tavily.tools:web_search_tool
    group: default
```

`use` 字段是 `module_path:variable_name` 格式，由 `resolve_variable` 加载：

```python
# reflection/resolvers.py
def resolve_variable(variable_path: str, ...) -> T:
    module_path, variable_name = variable_path.rsplit(":", 1)
    module = import_module(module_path)
    return getattr(module, variable_name)
```

这个设计让工具完全不需要在代码里硬编码引用路径，只改 yaml 就能换工具实现，也不影响任何其他代码。加载失败时会给出可操作的错误提示（如 `uv add langchain-google-genai`）。

`groups` 参数允许 per-agent 配置只加载自己需要的工具组，子 Agent 配置里可以用 `tool_groups: ["search"]` 只拿搜索工具，不把沙盒工具也给它。

---

## 第二层：内置工具（硬编码，按条件挂载）

代码里固定定义了这几个：

```python
BUILTIN_TOOLS = [
    present_file_tool,       # 让产出文件对用户可见
    ask_clarification_tool,  # 向用户追问，会被中间件拦截中断
]

SUBAGENT_TOOLS = [
    task_tool,               # 委派子任务，只在父 Agent 里有
]
```

**`task_tool` 为什么要通过 `subagent_enabled` 开关而不是默认给所有 Agent**：

子 Agent 在 `task_tool` 内部被构造时，`get_available_tools` 会被调用一次，`subagent_enabled` 强制传 `False`：

```python
# tools/builtins/task_tool.py
tools = get_available_tools(model_name=parent_model, subagent_enabled=False)
```

这一行代码切断了递归委派的可能性。子 Agent 的工具集里不存在 `task` 工具，模型看不到它的 schema，即使提示词里没有任何约束，子 Agent 也无法再发起委派。

**`view_image_tool` 为什么要检查 `supports_vision`**：

把视觉工具的 schema 传给不支持图片的文字模型，会让模型以为自己能处理图片，导致它去调用一个实际上会出错的工具。检查 `model_config.supports_vision` 确保工具列表和模型能力一致。

---

## 第三层：MCP 工具与延迟加载

MCP（Model Context Protocol）工具来自外部服务器（本地 stdio 进程、远程 HTTP/SSE 端点）。和配置文件工具不同，MCP 工具的数量不固定，用户可以随时在 Gateway 的 `PUT /api/mcp` 接口里增删配置。

**为什么 MCP 配置要从磁盘读，而不是用进程内缓存**：

```python
# tools.py 里的注释说明了原因
# NOTE: We use ExtensionsConfig.from_file() instead of config.extensions
# to always read the latest configuration from disk. This ensures that changes
# made through the Gateway API (which runs in a separate process) are immediately
# reflected when loading MCP tools.
extensions_config = ExtensionsConfig.from_file()
```

Gateway 和 LangGraph Server 是两个进程，Gateway 写了 `extensions_config.json` 之后，LangGraph Server 的内存里没有收到通知。每次 `get_available_tools` 从磁盘重读，保证用户改了 MCP 配置、下一轮对话就能生效。

**工具缓存用 mtime 失效**：

`mcp/cache.py` 记录 `extensions_config.json` 的修改时间，每次调用时比较：

```python
def _is_cache_stale() -> bool:
    current_mtime = _get_config_mtime()
    if current_mtime > _config_mtime:   # 文件改过了
        return True                      # 缓存作废，重新加载
    return False
```

MCP 工具的加载（连接外部进程/服务）有成本，不能每次调用都重新建立，但配置改了也不能一直用旧缓存，mtime 比较是代价最低的失效策略。

---

## 延迟加载（tool_search）：解决 MCP 工具过多的问题

如果 MCP 工具有几十个，把它们的完整 schema 全部传给模型，系统提示词就会膨胀，有效上下文减少。

`tool_search` 是一个折中方案：

```
tool_search 开启时的工具暴露策略：

模型视角：
  ┌─────────────────────────────────┐
  │  system prompt 里可见的工具名   │
  │  <available-deferred-tools>     │
  │    filesystem_read              │
  │    github_search                │
  │    slack_send                   │
  │    ...（只是名字列表，没有 schema）│
  └─────────────────────────────────┘
           │ 模型觉得需要某个工具
           ▼
  tool_search("filesystem")   ← 模型调这个
           │
           ▼  DeferredToolRegistry.search()
  返回匹配工具的完整 schema
           │
           ▼
  模型拿到 schema，下一轮可以正确调用这个工具
```

搜索支持三种查询形式：
- `select:name1,name2`：精确名字匹配
- `+keyword rest`：名字必须含 keyword，再按 rest 排序
- `keyword`：正则匹配名字和描述字段

`DeferredToolFilterMiddleware` 负责在每次把工具列表绑给模型时，把延迟注册表里的工具从 schema 列表里剔除，确保模型只看到工具名、看不到完整 schema。

---

## 汇总：不同 Agent 拿到的工具集

```
父 Agent（subagent_enabled=True, 默认模型）
  ├── 配置文件工具：bash, ls, read_file, write_file, str_replace,
  │                 web_search, web_fetch, image_search ...
  ├── 内置工具：present_file, ask_clarification, task
  ├── 视觉工具：view_image（若模型支持）
  └── MCP 工具：用户配置的外部服务（或通过 tool_search 延迟加载）

子 Agent general-purpose（subagent_enabled=False，工具白名单=None）
  ├── 配置文件工具：（同父，但无 task）
  ├── 内置工具：present_file, ask_clarification（无 task）
  ├── 视觉工具：同父
  └── MCP 工具：同父

子 Agent bash（subagent_enabled=False，工具白名单明确指定）
  └── 仅：bash, ls, read_file, write_file, str_replace
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `tools/tools.py` | `get_available_tools`，三层组装逻辑 |
| `tools/builtins/__init__.py` | 内置工具导出 |
| `tools/builtins/task_tool.py` | `task_tool`，`subagent_enabled=False` 切断递归 |
| `tools/builtins/tool_search.py` | `DeferredToolRegistry`，延迟加载实现 |
| `mcp/cache.py` | MCP 工具缓存，mtime 失效策略 |
| `reflection/resolvers.py` | `resolve_variable`，按路径字符串加载工具 |
| `config/extensions_config.py` | MCP 配置文件读取，双进程共享 |
