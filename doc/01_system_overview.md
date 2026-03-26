# 三个进程分工协作，让 Agent 运行与 API 服务互不阻塞

## 要解决的问题

一个 AI Agent 系统对外需要提供两种完全不同性质的服务：

1. **长时间运行的 Agent 会话**——一次对话可能持续数分钟，期间不断有工具调用、子任务委派、SSE 事件流输出；
2. **快速响应的管理 API**——查配置、上传文件、读记忆、改技能开关，这类请求通常几十毫秒内完成。

如果把这两件事放在同一个进程里，长耗时的 Agent 任务会占满线程，导致管理 API 排队等待；如果合用同一个端口对外暴露，每次前端发请求都要自己判断该走哪条路。DeerFlow 的解法是把这两件事**拆成两个独立进程**，再用 Nginx 在入口统一分拣路由。

---

## 三个进程的分工

```
┌────────────────────────────────────────────────────────────────┐
│                    Nginx  :2026（统一入口）                      │
│                                                                  │
│   /api/langgraph/*  ────────────────→  LangGraph Server :2024  │
│   /api/models                                                    │
│   /api/mcp          ────────────────→  Gateway API     :8001   │
│   /api/memory                                                    │
│   /api/skills                                                    │
│   /api/threads/*                                                 │
│   /api/agents                                                    │
│   /         ────────────────────────→  Frontend        :3000   │
└────────────────────────────────────────────────────────────────┘
```

### LangGraph Server（:2024）

**职责**：运行 Agent，管理对话状态，输出 SSE 事件流。

配置入口是 `backend/langgraph.json`：

```json
{
  "graphs": {
    "lead_agent": "deerflow.agents:make_lead_agent"
  },
  "checkpointer": {
    "path": "./packages/harness/deerflow/agents/checkpointer/async_provider.py:make_checkpointer"
  }
}
```

这个文件做了三件事：
- 声明哪个函数负责创建 Agent（`make_lead_agent`）；
- 声明用哪个 checkpointer 持久化对话状态（支持 memory、sqlite、postgres 三种后端）；
- 被 `langgraph-cli` 读取并启动进程。

LangGraph Server 是 Agent 唯一的运行载体。每次对话都在这里的事件循环里跑，SSE 流也从这里吐出去。

### Gateway API（:8001）

**职责**：管理配置、文件、记忆、技能，不接触 Agent 本身。

用 FastAPI 实现（`backend/app/gateway/app.py`），路由覆盖：

```
GET/PUT  /api/models          模型配置查询
GET/PUT  /api/mcp             MCP 服务器配置
GET      /api/memory          读记忆文件
GET/PUT  /api/skills          技能开关管理
POST     /api/threads/{id}/uploads   上传文件
DELETE   /api/threads/{id}    删除线程目录
GET      /api/threads/{id}/artifacts 取产出文件
POST     /api/threads/{id}/suggestions 生成后续问题建议
```

Gateway 进程里**没有**任何 Agent 运行逻辑，注释里也明确说明了这一点：

```python
# NOTE: MCP tools initialization is NOT done here because:
# 1. Gateway doesn't use MCP tools - they are used by Agents in the LangGraph Server
# 2. Gateway and LangGraph Server are separate processes with independent caches
```

Gateway 和 LangGraph Server **共享同一套磁盘目录和配置文件**，但各自维护独立的内存缓存。

### Frontend（:3000）

Next.js 应用。所有 API 请求从这里发出，经 Nginx 分拣到后端两个服务。前端不直连 LangGraph Server，不直连 Gateway，只认 Nginx 的统一地址。

---

## Nginx 如何分拣请求

配置来自 `docker/nginx/nginx.conf`，规则很简单：

```
/api/langgraph/*  →  rewrite 掉前缀，转发给 LangGraph Server
/api/models       →  Gateway
/api/mcp          →  Gateway
/api/memory       →  Gateway
/api/skills       →  Gateway
/api/threads/*    →  Gateway
/api/agents       →  Gateway
/*                →  Frontend
```

LangGraph 那一段还专门开了 SSE 所需的配置：

```nginx
proxy_buffering off;
proxy_cache off;
proxy_set_header X-Accel-Buffering no;
proxy_connect_timeout 600s;
proxy_read_timeout 600s;
```

关掉缓冲是因为 SSE 事件流不能被 Nginx 攒够了再发，必须实时透传到浏览器。

---

## 两种 Agent 角色

在系统里存在两种 Agent，它们用同一套代码库里的 `create_agent` 构造，但配置和生命周期完全不同：

```
┌──────────────────────────────────────────────────────┐
│  Lead Agent（父 Agent）                               │
│  · 由 LangGraph Server 管理生命周期                    │
│  · 带 checkpointer，对话状态跨轮持久化                  │
│  · 挂完整 middleware 链（12 个）                       │
│  · 拥有 task 工具，可以委派给子 Agent                   │
└──────────────────────────┬───────────────────────────┘
                           │ 调用 task 工具
                           ↓
┌──────────────────────────────────────────────────────┐
│  子 Agent（Subagent）                                 │
│  · 在 task_tool 内部用 SubagentExecutor 构造          │
│  · 跑在后台线程池里，不在 LangGraph 事件循环里           │
│  · 无 checkpointer，无 title/memory/subagent 等中间件  │
│  · 不能再调 task 工具（递归委派被代码切断）              │
│  · 结束后把结果字符串返回给 Lead Agent 的 ToolMessage   │
└──────────────────────────────────────────────────────┘
```

子 Agent 不是 LangGraph 里注册的独立图节点，没有独立的线程 ID 和 checkpoint，它的存在只对 Lead Agent 可见，用户看到的只有 SSE 里的 `task_started / task_running / task_completed` 事件。

---

## 请求从发出到模型响应的完整链路

以用户在前端发一条消息、Lead Agent 拆成两个子任务并行执行为例：

```
浏览器
  │
  │  POST /api/langgraph/threads/{id}/runs/stream
  │
  ▼
Nginx :2026
  │
  │  rewrite → LangGraph Server :2024
  │
  ▼
LangGraph Server
  │  make_lead_agent(config)  ← langgraph.json 注册的入口
  │  checkpointer 加载历史状态
  │  拼系统提示 + 历史消息 + 当轮 user message
  │  调模型 → 模型输出两个 task tool_calls
  │
  ├─→  task_tool("分析财务")  →  SubagentExecutor → 后台线程 → 子 Agent A
  └─→  task_tool("分析舆情")  →  SubagentExecutor → 后台线程 → 子 Agent B
         │
         │  task_tool 轮询结果，每隔 5s 检查
         │  用 get_stream_writer() 把 task_running 事件推进 SSE
         │
         ▼
     子 Agent A/B 各自完成 → 返回结果字符串
         │
         ▼
  Lead Agent 收到两个 ToolMessage → 再调模型 → 汇总 → 输出最终回复
  checkpointer 把本轮新状态写回存储
         │
         ▼
  SSE 事件流 → Nginx → 浏览器
```

---

## 这样设计的好处

**LangGraph Server 可以专注事件流**。Nginx 的 SSE 超时配置设到 600 秒，而 Gateway 的普通 API 请求不需要长连接，两者的超时策略、并发模型、缓冲行为都不同，拆开进程才能各自调优。

**Gateway 的重启不影响进行中的对话**。修改配置、重启 Gateway 的时候，LangGraph Server 里的 Agent 任务继续跑，因为它们不依赖 Gateway 进程。

**配置文件和磁盘目录共享，缓存独立**。两个进程读同一个 `config.yaml` 和 `.deer-flow/` 目录，但各自在内存里缓存配置的 mtime，当文件变化时分别按需重载，不需要进程间通信。

**前端只认一个地址**。Nginx 统一了 CORS、超时、路由，前端代码里不需要维护两套后端 URL。

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `backend/langgraph.json` | LangGraph Server 的进程配置，注册 `make_lead_agent` 和 checkpointer |
| `backend/app/gateway/app.py` | Gateway 进程入口，FastAPI 路由注册 |
| `docker/nginx/nginx.conf` | Nginx 路由规则，SSE 透传配置 |
| `backend/packages/harness/deerflow/agents/checkpointer/async_provider.py` | checkpointer 工厂，支持 memory/sqlite/postgres |
| `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | `make_lead_agent`，Lead Agent 的构造函数 |
| `backend/packages/harness/deerflow/subagents/executor.py` | `SubagentExecutor`，子 Agent 的构造和运行 |
