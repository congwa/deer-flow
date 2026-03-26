# DeerFlow 多 Agent 架构深度解析

> 本系列以源码为唯一依据，从系统全景到核心机制逐层展开，聚焦父子 Agent 编排体系的每一个设计决策。

---

## 阅读前提

本系列文章不讲 LangChain 和 LangGraph 的基础用法。阅读时你应该已经具备以下背景：

- 理解 LangChain v1 的核心抽象：`BaseTool`、`@tool` 装饰器、工具调用（tool calling）的执行机制；
- 了解 LangGraph 的 Agent 构造方式：`create_agent`、`AgentState`、`AgentMiddleware` 的四个钩子（`before_agent`、`wrap_model_call`、`after_model`、`after_agent`、`wrap_tool_call`）；
- 熟悉 Python 的 `TypedDict`、`Annotated`、`dataclass`，以及异步编程（`async/await`、`asyncio`、`ThreadPoolExecutor`）的基本概念。

如果对以上内容还不熟悉，建议先阅读 [LangChain 官方文档](https://python.langchain.com/docs/) 和 [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/)，再回来阅读本系列。

---

## 文章列表

### 第一层：系统全景

| 编号 | 文件 | 标题 |
|------|------|------|
| 01 | [01_system_overview.md](./01_system_overview.md) | 三个进程分工协作，让 Agent 运行与 API 服务互不阻塞 |
| 02 | [02_thread_state.md](./02_thread_state.md) | Annotated Reducer 让父子 Agent 操作同一份状态而不互相覆盖 |

### 第二层：工具与执行环境

| 编号 | 文件 | 标题 |
|------|------|------|
| 03 | [03_tool_system.md](./03_tool_system.md) | 三层来源按需组装，让运行时工具集在灵活扩展的同时保持可控 |
| 04 | [04_sandbox.md](./04_sandbox.md) | 虚拟路径映射把磁盘访问限制在线程目录内，不依赖进程隔离也能防越权 |

### 第三层：提示词与中间件

| 编号 | 文件 | 标题 |
|------|------|------|
| 05 | [05_system_prompt.md](./05_system_prompt.md) | 十个 XML 块按运行时条件拼合，让单条系统提示承载多种能力开关 |
| 06 | [06_subagent_system_prompt.md](./06_subagent_system_prompt.md) | `<subagent_system>` 把编排规则写进提示词，用 LLM 推理代替独立规划服务 |
| 07 | [07_middleware_chain.md](./07_middleware_chain.md) | 四个生命周期钩子把横切逻辑从 Agent 核心剥离，Lead 与子 Agent 按需裁剪 |

### 第四层：父子 Agent 编排核心

| 编号 | 文件 | 标题 |
|------|------|------|
| 08 | [08_task_tool.md](./08_task_tool.md) | task 工具把一次工具调用变成一段独立子会话，让父 Agent 无感知地完成委派 |
| 09 | [09_subagent_config.md](./09_subagent_config.md) | 白黑名单加系统提示切分两种子 Agent 的能力范围，防止能力过载和递归委派 |
| 10 | [10_state_sync.md](./10_state_sync.md) | 消息历史不透传保证上下文隔离，沙盒目录共享保证文件结果可交接 |
| 11 | [11_concurrency.md](./11_concurrency.md) | 软约定计数加硬截断两道防线，保证单轮并发不超出线程池承载上限 |

### 第五层：上下文管理

| 编号 | 文件 | 标题 |
|------|------|------|
| 12 | [12_summarization.md](./12_summarization.md) | 触发阈值和保留策略分开配置，让对话压缩在不丢关键上下文的前提下控制 token |
| 13 | [13_memory_system.md](./13_memory_system.md) | 防抖队列批处理加 LLM 摘要原子写入，让记忆更新不阻塞主对话并持久化到文件 |

### 第六层：设计取舍

| 编号 | 文件 | 标题 |
|------|------|------|
| 14 | [14_design_tradeoffs.md](./14_design_tradeoffs.md) | 提示词约束意图、中间件截断超限，两层防线各自守住不同的故障域 |

### 附录：全局状态流转图

| 编号 | 文件 | 标题 |
|------|------|------|
| 15 | [15_state_flow_diagram.md](./15_state_flow_diagram.md) | 完整状态流转图——从一次用户请求到所有状态变迁（14 个节点，含父子 Agent 对比） |

---

## 阅读路径

**想理解系统全貌**：01 → 02 → 03 → 04

**想搞清楚父子 Agent 怎么跑**：05 → 06 → 07 → 08 → 09 → 10 → 11

**想解决对话过长或记忆不准的问题**：12 → 13

**想理解这些设计决策背后的取舍**：14

---

## 源码对照

```
backend/packages/harness/deerflow/
├── agents/
│   ├── lead_agent/
│   │   ├── agent.py          ← 01 07 11
│   │   └── prompt.py         ← 05 06
│   ├── middlewares/          ← 07 11 12
│   ├── memory/               ← 13
│   │   ├── queue.py
│   │   ├── updater.py
│   │   └── prompt.py
│   └── thread_state.py       ← 02
├── subagents/                ← 08 09 10 11
│   ├── executor.py
│   ├── config.py
│   └── builtins/
├── tools/                    ← 03 08
│   ├── tools.py
│   └── builtins/task_tool.py
├── sandbox/                  ← 04 10
│   ├── sandbox.py
│   ├── sandbox_provider.py
│   ├── middleware.py
│   ├── tools.py
│   └── local/
└── config/                   ← 贯穿全文
    ├── paths.py
    ├── memory_config.py
    ├── summarization_config.py
    └── subagents_config.py
```
