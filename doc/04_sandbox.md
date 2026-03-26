# 虚拟路径映射把磁盘访问限制在线程目录内，不依赖进程隔离也能防越权

## 要解决的问题

Agent 要能执行 bash 命令、读写文件。如果直接把真实的宿主机路径暴露给模型，有几个问题：

1. **安全**：模型可能写 `rm -rf /`，或者访问 `/etc/passwd`、其他线程的目录；
2. **泄露**：错误信息里带着 `/Users/cong/.deer-flow/threads/abc123/user-data/workspace` 这样的宿主机路径，用户不该看到这些；
3. **多线程**：每个对话线程有自己的工作目录，工具需要知道当前线程对应哪个目录；
4. **一致性**：本地部署和容器部署的物理路径完全不同，Agent 的工具代码不能有硬编码路径。

DeerFlow 的解法是：在宿主机和 Agent 之间引入一层**虚拟路径**，Agent 始终看到固定的 `/mnt/user-data/...`，工具层负责把它翻译成真实路径，并在翻译前后做安全校验。

---

## 虚拟路径与宿主机路径的对应关系

```
Agent 看到的路径（虚拟）             宿主机真实路径（per-thread）
────────────────────────────────────────────────────────────────────
/mnt/user-data/workspace/    ←→    {base_dir}/threads/{id}/user-data/workspace/
/mnt/user-data/uploads/      ←→    {base_dir}/threads/{id}/user-data/uploads/
/mnt/user-data/outputs/      ←→    {base_dir}/threads/{id}/user-data/outputs/
/mnt/skills/public/...       ←→    {skills_path}/public/...
/mnt/skills/custom/...       ←→    {skills_path}/custom/...
```

`{base_dir}` 默认是 `~/.deer-flow`（或 `backend/.deer-flow`），`{id}` 是 LangGraph 的 `thread_id`，全部来自 `ThreadDataState` 里的三个字段。

---

## 路径翻译发生在工具调用里，而不是 Agent 框架里

翻译逻辑在 `sandbox/tools.py` 的每个工具函数里，以 `bash_tool` 为例：

```python
@tool("bash", parse_docstring=True)
def bash_tool(runtime, description: str, command: str) -> str:
    sandbox = ensure_sandbox_initialized(runtime)
    ensure_thread_directories_exist(runtime)
    thread_data = get_thread_data(runtime)

    if is_local_sandbox(runtime):
        validate_local_bash_command_paths(command, thread_data)  # 第一步：校验
        command = replace_virtual_paths_in_command(command, thread_data)  # 第二步：翻译
        output = sandbox.execute_command(command)
        return mask_local_paths_in_output(output, thread_data)  # 第三步：脱敏
    return sandbox.execute_command(command)  # 容器沙盒直接执行（路径已挂载）
```

三步顺序是不可互换的：先校验再翻译，确保校验的是模型看到的虚拟路径；翻译后执行，确保 subprocess 收到真实路径；执行完脱敏，确保错误信息不泄露宿主机目录结构。

---

## 三道安全校验

### 第一道：路径遍历拦截

```python
def _reject_path_traversal(path: str) -> None:
    normalised = path.replace("\\", "/")
    for segment in normalised.split("/"):
        if segment == "..":
            raise PermissionError("Access denied: path traversal detected")
```

在所有路径进入翻译流程之前，先检查是否含 `..`。`/mnt/user-data/../../../etc/passwd` 这类攻击在第一道就被拒掉。

### 第二道：路径前缀白名单

```python
def validate_local_tool_path(path, thread_data, *, read_only=False):
    _reject_path_traversal(path)

    if _is_skills_path(path):
        if not read_only:
            raise PermissionError("Write access to skills path is not allowed")
        return                          # 技能目录：只读通过

    if path.startswith("/mnt/user-data/"):
        return                          # 用户数据目录：读写通过

    raise PermissionError("Only paths under /mnt/user-data/ or /mnt/skills/ are allowed")
```

只允许两类虚拟路径：用户数据目录（读写）、技能目录（只读）。其他任何路径——包括 `/tmp`、`/etc`——都直接拒绝。

### 第三道：翻译后二次验证（resolve 后再检查）

```python
def _resolve_and_validate_user_data_path(path, thread_data):
    resolved_str = replace_virtual_path(path, thread_data)
    resolved = Path(resolved_str).resolve()              # 解析符号链接
    _validate_resolved_user_data_path(resolved, thread_data)  # 验证是否在允许根下
    return str(resolved)

def _validate_resolved_user_data_path(resolved, thread_data):
    allowed_roots = [Path(p).resolve() for p in (
        thread_data.get("workspace_path"),
        thread_data.get("uploads_path"),
        thread_data.get("outputs_path"),
    ) if p is not None]

    for root in allowed_roots:
        try:
            resolved.relative_to(root)   # 路径必须是 root 的子路径
            return
        except ValueError:
            continue

    raise PermissionError("Access denied: path traversal detected")
```

翻译成真实路径后，还要调用 `Path.resolve()` 展开所有符号链接，再检查展开后的路径是否真的在允许的三个目录下面。这一步防止的是通过软链接绕过第一道白名单。

---

## `is_local_sandbox` 的分叉

上面的翻译和校验逻辑只对本地沙盒生效：

```python
def is_local_sandbox(runtime) -> bool:
    sandbox_state = runtime.state.get("sandbox")
    return sandbox_state.get("sandbox_id") == "local"
```

原因注释里写得很清楚：

```python
# Path replacement is only needed for local sandbox since aio sandbox
# already has /mnt/user-data mounted in the container.
```

容器沙盒（AIO）里，`/mnt/user-data` 是真实挂载点，不需要翻译。模型传 `/mnt/user-data/workspace/file.txt`，容器里就能直接访问到。两种沙盒用同一个 `bash_tool`，但通过 `is_local_sandbox()` 走不同路径。

---

## 输出脱敏：宿主机路径不泄露给用户

```python
output = sandbox.execute_command(command)
return mask_local_paths_in_output(output, thread_data)
```

`mask_local_paths_in_output` 会把输出里出现的宿主机路径（比如 `/Users/cong/.deer-flow/threads/abc123/user-data/workspace`）替换回虚拟路径（`/mnt/user-data/workspace`）。

这处理了两种情况：某些命令会把当前路径打印出来（如 `pwd`、`git rev-parse`）；报错信息里可能包含完整路径。用户和模型都只看到虚拟路径，宿主机目录结构对外不可见。

---

## 目录的懒创建

注意 `ensure_thread_directories_exist` 这个调用：

```python
def ensure_thread_directories_exist(runtime) -> None:
    if not is_local_sandbox(runtime):
        return          # 容器沙盒不需要，目录在容器里已存在

    thread_data = get_thread_data(runtime)
    if runtime.state.get("thread_directories_created"):
        return          # 已经创建过，跳过

    for key in ["workspace_path", "uploads_path", "outputs_path"]:
        path = thread_data.get(key)
        if path:
            os.makedirs(path, exist_ok=True)

    runtime.state["thread_directories_created"] = True
```

目录在第一次工具调用时才创建，而不是线程开始时就创建。这样没有实际操作文件的轻量对话不会在磁盘上留下空目录。`ThreadDataMiddleware` 在 `lazy_init=True` 时只计算路径、不 `mkdir`，真正需要磁盘的时刻由工具层触发。

---

## 本地沙盒 vs 容器沙盒的本质区别

```
本地沙盒（LocalSandbox）
  ├── execute_command: subprocess.run(command, shell=True)   ← 宿主机进程，无隔离
  ├── 安全靠：路径翻译 + 白名单 + 二次验证 + 脱敏
  └── sandbox_id = "local"（单例，一个进程一个实例）

容器沙盒（AioSandbox）
  ├── execute_command: HTTP → 容器内运行
  ├── 安全靠：Docker/Apple Container 的 namespace 隔离 + 挂载卷限制
  └── sandbox_id = UUID（每个线程一个容器）
```

本地沙盒不是真正意义上的安全隔离，它依赖白名单而不是 OS 级隔离。开发测试用本地沙盒，生产环境对安全有要求时用容器沙盒，切换只需改 `config.yaml` 里的 `sandbox.use`。

---

## bash_tool 完整执行路径（本地沙盒）

```
模型生成 tool_call:
  bash(description="列出输出目录", command="ls /mnt/user-data/outputs")
         │
         ▼
bash_tool(runtime, "列出输出目录", "ls /mnt/user-data/outputs")
         │
         ├── ensure_sandbox_initialized → sandbox_id="local"
         ├── ensure_thread_directories_exist → mkdir workspace/uploads/outputs（首次）
         ├── is_local_sandbox → True
         │
         ├── validate_local_bash_command_paths("ls /mnt/user-data/outputs", thread_data)
         │     ├── _reject_path_traversal → 无 ".." → 通过
         │     ├── /mnt/user-data/ 前缀 → 通过
         │     └── 无其他绝对路径 → 通过
         │
         ├── replace_virtual_paths_in_command(command, thread_data)
         │     "ls /mnt/user-data/outputs"
         │     → "ls /Users/cong/.deer-flow/threads/abc123/user-data/outputs"
         │
         ├── sandbox.execute_command("ls /Users/cong/.deer-flow/threads/abc123/user-data/outputs")
         │     subprocess.run → 返回文件列表
         │
         └── mask_local_paths_in_output(output, thread_data)
               "/Users/cong/.deer-flow/threads/abc123/user-data/outputs"
               → "/mnt/user-data/outputs"
               返回对模型可见的输出
```

---

## 关键源码位置

| 文件 | 内容 |
|------|------|
| `sandbox/tools.py` | 所有工具函数，三道安全校验，翻译与脱敏 |
| `sandbox/sandbox.py` | `Sandbox` 抽象基类，定义能力接口 |
| `sandbox/local/local_sandbox.py` | `LocalSandbox`，subprocess 实现 |
| `sandbox/local/local_sandbox_provider.py` | 单例 provider，`release` 为空操作 |
| `sandbox/middleware.py` | `SandboxMiddleware`，按线程 `acquire` 沙盒 |
| `config/paths.py` | `Paths` 类，宿主机目录布局，`VIRTUAL_PATH_PREFIX = "/mnt/user-data"` |
| `agents/middlewares/thread_data_middleware.py` | 计算三条路径并写入 `thread_data` |
