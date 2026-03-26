# 提示词约束意图、中间件截断超限，两层防线各自守住不同的故障域

## 这篇文章讲什么

前面 13 篇把各个机制讲清楚了。这篇专门谈一件事：**这个系统是在什么约束下做出哪些取舍，这些取舍的边界在哪里**。

理解这些取舍，比会复制代码更重要——下次遇到问题时才能判断是正常边界内的现象，还是需要修复的 bug，还是设计上根本就没打算支持的场景。

---

## 取舍一：用 LLM 推理代替独立规划服务

**做了什么**：把「任务分解」的逻辑写进 `<subagent_system>` 提示词，让主编排模型自己决定怎么拆、拆几个、分几批。

**得到什么**：不需要维护单独的规划运行时、DAG 执行引擎、任务队列。整个编排逻辑是一段文字，改起来只需要改 prompt，不改代码。

**失去什么**：无法保证每次行为完全符合预期。模型可能：
- 把本该一步做完的事情拆成多个 task（过度委派）；
- 把应该并行的任务串行发（失去并行收益）；
- 在强串行依赖的场景里强行并行（子 Agent 因为信息不足而质量下降）；
- 忽视分批规则，发出超过限制数量的 task（被中间件截断，丢失工作）。

**兜底在哪里**：代码层用 `SubagentLimitMiddleware` 截断超量的 task 调用，防止最坏情况（无限并发）。但截断本身不能补回被丢弃的工作，模型下一轮会收到比预期少的 ToolMessage，需要提示词里的分批规则来引导模型发现「这轮少了什么、下批再发」。

---

## 取舍二：消息历史不透传给子 Agent

**做了什么**：子 Agent 的初始消息只有父委派的一条 `HumanMessage(prompt)`，看不到父的对话历史。

**得到什么**：子 Agent 的上下文窗口干净，专注于当前子任务；多个子 Agent 并行时，各自的上下文相互独立，不会因为其他子 Agent 的中间步骤污染彼此。

**失去什么**：父 Agent 在 `prompt` 里没有写清楚的背景信息，子 Agent 不知道。如果子任务需要参考用户早期说过的内容，父必须在 `prompt` 里显式复述——这依赖父 Agent 的能力（有时候模型会省略重要上下文）。

**实际影响**：子 Agent 的质量很大程度上取决于父写的 `prompt` 的质量。这也是 `<subagent_system>` 里强调「Be specific and clear about what needs to be done」的原因——`prompt` 越详细，子 Agent 的结果越好。

---

## 取舍三：子 Agent 的执行结果只回来一段文字

**做了什么**：`task_tool` 返回子 Agent 的最后一条 AIMessage 内容作为字符串，通过 `ToolMessage` 传给父 Agent。

**得到什么**：父 Agent 拿到的是子任务的「结论」，而不是整段执行过程。父的 `messages` 历史里不会出现几十条子 Agent 的工具调用记录，主线对话保持干净。

**失去什么**：子 Agent 中间发现的信息、用过的工具、执行路径，父 Agent 都看不到。如果父想知道「子 Agent 查了哪些网站」，这信息在 `result.result` 里可能有（取决于子 Agent 的输出格式），也可能没有。

**文件是另一条通道**：子 Agent 写到 `/mnt/user-data/outputs/` 的文件，父可以通过 `read_file` 读。对于「结论文字不够，需要详细数据」的场景，让子 Agent 输出文件是正确的做法，这也是子 Agent 系统提示里「3. Any relevant file paths, data, or artifacts created」一项的意义。

---

## 取舍四：本地沙盒没有进程隔离

**做了什么**：`LocalSandbox` 用 `subprocess.run` 在宿主机上直接执行命令，通过路径白名单和虚拟路径映射限制访问范围。

**得到什么**：零基础设施依赖，不需要 Docker、不需要 k8s，`make dev` 启动就能用完整功能。

**失去什么**：任何能绕过路径白名单的方式（比如某些 shell 内建命令、符号链接攻击）都可能访问到宿主机文件系统。路径白名单是软边界，不是 OS 级隔离。

**边界在哪里**：路径白名单检查了三道（前缀、去掉 `..`、resolve 后验证在允许根下）。但这是 Agent 系统内部的防护，假设模型不是恶意的。如果需要真正的隔离，切换到 `AioSandboxProvider`，用 Docker 容器隔离。

---

## 取舍五：记忆更新是异步的，不保证实时

**做了什么**：`MemoryUpdateQueue` 防抖 30 秒，在对话结束后异步调用 LLM 更新记忆。

**得到什么**：记忆更新不会阻塞当前对话；同一线程多轮快速对话不会触发多次 LLM 调用（只处理最新的一次）。

**失去什么**：两次对话之间的间隔如果短于 30 秒，第一次对话产生的记忆可能还没写入就开始了第二次对话。第二次对话的系统提示里的 `<memory>` 是上一次记忆更新完成时的状态。

**影响**：对于连续快速对话的用户，记忆有一定滞后性。对于正常使用节奏（两次对话之间间隔超过 30 秒）没有感知差异。

---

## 取舍六：上下文压缩（摘要）默认关闭

**做了什么**：`SummarizationMiddleware` 默认 `enabled: false`，需要手动在 `config.yaml` 里开启。

**得到什么**：不会因为意外触发摘要而丢失用户感知到的对话细节；部署简单，不需要额外配置摘要模型。

**失去什么**：长对话（数百轮、包含大量工具输出的会话）没有自动的上下文管理，最终会超出模型上下文窗口，导致错误或截断。

**给用户的信号**：如果你要支持非常长的对话，需要主动开启摘要并配置合理的触发阈值和保留策略。否则默认行为是让对话自然增长，直到触及模型限制。

---

## 两层防线的分工边界

整个系统的约束从哪层来，哪层失效会发生什么：

```
约束事项                    提示词（软）           中间件/代码（硬）
──────────────────────────────────────────────────────────────────────
每轮 task 数量上限           <subagent_system> 里   SubagentLimitMiddleware
                            的计数和分批规则        截断多余 task 调用

子 Agent 不递归委派          提示词无限制           get_available_tools(subagent_enabled=False)
                                                   不给子 Agent 加 task 工具

先澄清再行动                 <clarification_system>  ClarificationMiddleware
                            的五种场景规则          拦截调用，Command(goto=END)

循环重复不停                 无专门约定              LoopDetectionMiddleware
                                                   检测哈希，warning + hard stop

工具执行异常不崩溃            无专门约定              ToolErrorHandlingMiddleware
                                                   捕获异常，返回错误 ToolMessage

路径访问越权                 提示词里说「用虚拟路径」   validate_local_tool_path
                                                   三道校验

记忆文件写入损坏              无专门约定              temp + rename 原子写入
```

提示词负责「正常情况下引导模型按预期行事」。代码/中间件负责「模型出错时系统仍然可恢复，不崩溃，不产生副作用」。

两层都失败的场景：模型生成格式完全错误的工具参数，但不是通过 `task` 调用递归委派，也不是无限重复（所以不触发任何中间件）。这类错误最终会通过 `ToolErrorHandlingMiddleware` 捕获，变成一条错误 ToolMessage，模型下一轮自己判断怎么处理。

---

## 这个系统适合什么、不适合什么

**适合**：
- 复杂的研究和分析任务（多源信息收集、整合）
- 长流程的代码任务（探索、修改、测试的多步骤）
- 需要隔离上下文的并行子任务

**不适合**：
- 需要严格保证任务分解正确性的场景（如金融、医疗决策）——提示词约束是概率性的
- 需要审计完整执行轨迹的场景——子 Agent 的执行过程不写进父的 checkpoint
- 需要进程级安全隔离的场景——本地沙盒的隔离是软边界

**扩展点**：
- 增加新的子 Agent 类型：在 `subagents/builtins/` 新建文件，注册到 `BUILTIN_SUBAGENTS`，在 `task_tool.py` 的 `Literal` 里加上新类型名；
- 增加新的中间件：实现 `AgentMiddleware[AgentState]`，在 `_build_middlewares` 或 `build_lead_runtime_middlewares` 里插入到合适位置；
- 换沙盒实现：实现 `Sandbox` 和 `SandboxProvider` 接口，在 `config.yaml` 的 `sandbox.use` 里切换。

---

## 一句话总结整个系统的设计哲学

**用已有的工具调用机制做委派（`task` 是普通工具）、用提示词定义编排规范（`<subagent_system>`）、用中间件做兜底（`SubagentLimitMiddleware` 等）、用文件系统做父子协作（共享沙盒目录）**——最大化复用现有机制，最小化新引入的基础设施，代价是「正确性」依赖模型遵守提示词。

---

## 各篇文章对应的关键源码索引

| 主题 | 关键文件 |
|------|----------|
| 系统全景 | `langgraph.json`, `app/gateway/app.py`, `docker/nginx/nginx.conf` |
| 状态模型 | `agents/thread_state.py` |
| 工具系统 | `tools/tools.py`, `tools/builtins/task_tool.py`, `mcp/cache.py` |
| 沙盒 | `sandbox/tools.py`, `sandbox/local/local_sandbox.py`, `config/paths.py` |
| 系统提示词 | `agents/lead_agent/prompt.py` |
| 子 Agent 编排提示词 | `agents/lead_agent/prompt.py` `_build_subagent_section` |
| 中间件链 | `agents/lead_agent/agent.py` `_build_middlewares`, `agents/middlewares/` |
| task 工具执行 | `tools/builtins/task_tool.py`, `subagents/executor.py` |
| 子 Agent 配置 | `subagents/config.py`, `subagents/builtins/` |
| 父子状态同步 | `subagents/executor.py` `_build_initial_state`, `sandbox/tools.py` |
| 并发控制 | `agents/middlewares/subagent_limit_middleware.py`, `subagents/executor.py` |
| 消息压缩 | `config/summarization_config.py`, `agents/middlewares/todo_middleware.py` |
| 记忆系统 | `agents/memory/queue.py`, `agents/memory/updater.py`, `agents/memory/prompt.py` |
