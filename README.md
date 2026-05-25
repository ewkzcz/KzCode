# KzCode Desktop

KzCode 是一个**本地代码智能体（Coding Agent）桌面应用**，也是您的本地代码协作助手。灵感来自 Claude Code、Codex 等 AI 编程工具。

与普通 AI 编程工具不同，KzCode 围绕 **Agent Harness** 思想设计：将模型推理、工具调用、上下文管理、记忆系统、安全边界和运行审计整合成一条可复盘、可恢复、可评测的执行链路。在此基础上，通过 Electron 提供原生桌面体验，让您在自己的代码仓库中获得安全、可控、高效的人工智能协作。



## 项目总览

```
用户输入（终端 / UI）
        │
        ▼
┌─────────────────────────────────────────────┐
│              前端（Electron + Vue）           │
│  • 桌面文件系统访问   • 会话管理   • 流式对话   │
│  • 审批面板          • Diff 对比  • 多主题     │
│  • 项目文件树        • Git 集成   • 内置编辑器  │
└─────────────────────────────────────────────┘
        │ HTTP + SSE（IPC / Web）
        ▼
┌─────────────────────────────────────────────┐
│          后端（FastAPI + Agent Harness）      │
│  • 主控制循环        • 上下文管理（预算裁剪）    │
│  • 分层记忆          • 工具调用与安全治理       │
│  • Checkpoint/Resume • 运行审计与工件           │
│  • 多模型聚合          • @引用文件上下文         │
└─────────────────────────────────────────────┘
        │
        ▼
    本地代码仓库
```



## 后端：Agent Harness 核心

后端基于 **FastAPI**，实现了一个完整的本地代码智能体运行框架（Harness）。它的设计目标不是"接一个模型、配几个工具"，而是解决代码仓库长链路任务中的核心工程问题：**上下文膨胀、重复读取、状态丢失、工具副作用不可控、结果难复盘**。

### 1、架构与主循环

KzCode 的后端是一个有状态、有边界、可恢复的 Agent 运行时。每轮循环经历**感知 → 决策 → 行动 → 记录**四个阶段：

```python
# 主控制循环
while tool_steps < max_steps and attempts < max_attempts:
    prompt, metadata = build_prompt_and_metadata(user_message)    # 1. 感知：拼 prompt
    raw = model_client.complete(prompt, max_new_tokens)           # 2. 决策：模型输出
    kind, payload = parse(raw)                                    # 3. 解析：工具调用还是最终答案
    if kind == "tool":
        result = run_tool(payload.name, payload.args)             # 4. 行动：执行工具
        update_memory_and_history(result)                         # 5. 记录：回写历史+记忆
        save_checkpoint_and_artifacts()
        continue
    # kind == "final" → 返回最终答案
```



#### 缓存策略

系统提示词（角色 + 工具清单 + 工作区基线）相对稳定，生成快照并记录工作区指纹。

每次运行前检查工作区指纹（Git 状态 + 项目文件）和工具注册表是否变化，无变化时复用缓存的系统提示词，减少重复构建开销。



#### 重试策略

| 重试类型     | 触发条件                    | 策略                                                    |
| ------------ | --------------------------- | ------------------------------------------------------- |
| 不可恢复错误 | 认证/鉴权失败、模型不可用等 | **直接终止**，不重试                                    |
| API 错误     | 模型调用异常（超时/限流等） | 指数退避 1s→2s→4s→8s→9s，连续失败达最大步数后停止       |
| 格式错误     | 模型返回无效格式            | 递增计数，达上限后停止                                  |
| 工具失败     | 工具执行返回错误            | 按工具名+参数分别计数和全局计数，任一达上限后停止该方向 |
| 降级         | 重试耗尽                    | 如有上一次成功的工具结果，生成降级回复代替报错          |



### 2、上下文管理

KzCode 将 prompt 拆成**系统提示词**（角色 + 工具清单 + 工作区基线，相对稳定）和**用户提示词**（每轮动态重建），由统一的组装器拼接。

#### 用户提示词结构

按固定顺序拼接 6 个片段，稳定基线放前面、最新请求放最后：

| 片段           | 内容                                      | 稳定性 | 能否裁剪         |
| -------------- | ----------------------------------------- | ------ | ---------------- |
| **运行上下文** | 当前工作区信息、重试次数、checkpoint 状态 | 极稳定 | 不参与           |
| **执行进度**   | 当前任务完成了什么、还剩什么              | 动态   | 不参与           |
| **记忆**       | 工作记忆 + 文件摘要                       | 较稳定 | 可裁（第三顺位） |
| **相关记忆**   | 从记忆中召回与当前任务相关的条目          | 动态   | 可裁（最先裁）   |
| **历史**       | 对话历史（滑动窗口 + 压缩）               | 动态   | 可裁（第二顺位） |
| **当前请求**   | 用户最新输入                              | 极动态 | **永不裁**       |



#### 预算控制

- 默认总预算 **12000 字符**。超预算时按**相关记忆 → 历史 → 记忆**顺序逐个收缩，各片段触底（动态下限 = `max(20, 预算/4)`）后切下一个，直至满足预算。
- 运行上下文、执行进度、当前请求三个片段不参与裁剪。
- 最近 6 轮对话保留较完整摘要（900 字符上限）；之外的进行压缩：同一文件反复读取只留最近一次、工具调用缩成单行、普通消息压到 60 字符。



#### 工具结果与进度追踪

每次工具执行后收集执行状态、影响文件列表、diff 预览等元数据：

- 写入**对话历史**（供后续 prompt 使用）
- 写入**审计日志** `trace.jsonl`（含耗时、安全事件）
- 异常/失败状态沉淀为**记忆笔记**
- **SSE 实时推送**到前端

同时每步更新执行进度（已完成、进行中、阻塞点），注入 prompt 让模型感知进度，避免重复已完步骤。进度变化也通过 SSE 推送到前端。



#### 异常快速失败

以下情况直接终止，不重试：

| 异常类型      | 触发条件                                       |
| ------------- | ---------------------------------------------- |
| 认证/鉴权错误 | 模型返回 401、invalid api key、unauthorized 等 |
| 模型不可用    | 模型未配置或配额耗尽                           |
| API 重试超限  | 连续调用失败达最大步数                         |
| 格式重试超限  | 模型连续输出无效格式达最大步数                 |
| 工具连续失败  | 同一工具连续失败达上限                         |



#### 效果

- 平均 prompt 长度从 6964 压缩至 5418，平均压缩率 **18%**，最高 **35%**。
- 当前请求被裁坏率 **0%**。



### 3、分层记忆系统

KzCode 的记忆不是"聊天历史摘要"，而是**分层、可失效、工具驱动**的轻量工作记忆 + 长期事实沉淀。

| 记忆层       | 存储内容                                                     | 生命周期             |
| ------------ | ------------------------------------------------------------ | -------------------- |
| **工作记忆** | 任务摘要、最近操作过的文件（上限 8 条）                      | 当前会话，跨轮保留   |
| **文件摘要** | 每个文件对应短摘要 + 内容 SHA256 指纹（上限 500 字符/条）    | 文件修改后自动失效   |
| **过程笔记** | 工具执行过程记录（上限 12 条，带标签和来源）                 | 当前会话，跨轮保留   |
| **持久记忆** | 项目约定、关键决策、依赖事实、用户偏好，存入 `.KzCode/memory/` | **跨会话**，自动沉淀 |



#### 文件摘要自动失效

**读文件时**：不保存全文，只生成短摘要并记录内容的 SHA256 指纹。

**写文件或修改文件后**：对应文件摘要被标记为失效。

**下次会话恢复时**：批量校验所有摘要的指纹是否匹配当前文件内容，不匹配的直接丢弃。

这样，模型看到的永远是有效的文件信息，不会拿着过时内容做决策。



#### 长期记忆自动沉淀

当用户请求包含"记住/保存/记录"等持久化意图时，自动从最终答案中提取结构化事实，多步处理后写入磁盘：

- **意图检测** — 匹配 `capture/remember/save/store/persist/note/记住/保存/记录/沉淀/长期记忆/持久记忆`
- **格式匹配** — 从答案中逐行提取 4 种固定格式：项目约定、决策、依赖、偏好等
- **拒绝过滤** — 空内容、含 API key/token/secret 等敏感信息、含 checkpoint 状态字段、含 stdout/stderr 等噪声输出，均不写入
- **写入** — 存入 `.KzCode/memory/topics/` 目录，同名主题自动去重更新



#### 记忆检索

同时检索过程笔记和持久记忆，按 **标签精确匹配 > 关键词重叠数 > 时间新旧** 排序。不依赖向量模型，基于分词匹配。最多返回 3 条，无匹配时提示为空。



#### 效果

- 开记忆后，后续对话阶段重复读文件次数从 **60 次降为 0 次**。
- 平均工具步数从 1.0 降为 0，不再需要额外调用工具来确认已拿到的事实。



### 4、工具调用与安全控制

KzCode 将工具层视为**受控执行链**，而不是简单的函数映射。所有工具调用必须经过统一网关 `run_tool()`，经过多道护栏。

#### 当前可用工具集

| 工具         | 作用                      | 风险 | 约束                                                  |
| ------------ | ------------------------- | ---- | ----------------------------------------------------- |
| `list_files` | 列出工作区目录内容        | 只读 | 沙箱隔离环境下，禁止逃逸出当前工作目录                |
| `read_file`  | 按行范围读取 UTF-8 文件   | 只读 | 行号范围合法，结果自动生成摘要并进入记忆              |
| `search`     | 在工作区中搜索文本模式    | 只读 | pattern 非空，优先使用 rg（ripgrep），python 兜底     |
| `run_shell`  | 在当前目录执行 shell 命令 | 高危 | 超时 1-120s，环境变量白名单过滤，防敏感信息泄露       |
| `write_file` | 写入文本文件              | 高危 | 不能写入目录；写入后使对应文件摘要失效                |
| `patch_file` | 精确替换文件中一段文本    | 高危 | `old_text` 必须精确出现且仅出现一次，防止模糊修改     |
| `delegate`   | 派生子 agent 只读调研     | 只读 | 子 agent 只读 + 默认 3 步上限，深度超限后不暴露该工具 |



#### 审批判定流程

```
工具调用
   │
   ▼
┌──────────┐  不合法
│ 参数校验 ├─────────→ 拦截并返回错误
└────┬─────┘
     │ 通过
     ▼
┌──────────┐  重复
│ 重复检测 ├─────────→ 拦截
└────┬─────┘
     │ 新调用
     ▼
┌──────────────────┐  是
│ read_only/never? ├─────────→ 拦截
└────┬─────────────┘
     │ 否
     ▼
┌──────────────┐  逃逸
│ 沙箱隔离检测 ├─────────→ 需要审批
└────┬─────────┘
     │ 安全
     ▼
┌──────────┐  命中
│ 黑名单   ├─────────→ 需要审批
└────┬─────┘
     │ 未命中
     ▼
┌──────────┐  命中
│ 白名单   ├─────────→ 自动放行
└────┬─────┘
     │ 未命中
     ▼
┌──────────────────┐
│ approval_policy  │── ask ──→ 需要审批
│      == ask?     │── auto ─→ 自动放行
└──────────────────┘

需要审批
   │
   ▼
SSE approval_request
   │
   ▼
┌──────────────┐  允许
│ 用户审批     ├─────────→ 自动放行
│ (600s 超时)  │
└──────────────┘
   │ 拒绝或超时
   ▼
 拦截
```



#### 权限判定

| 优先级 | 规则             | 命中后果                               | 匹配方式                                                     |
| ------ | ---------------- | -------------------------------------- | ------------------------------------------------------------ |
| 1      | **沙箱隔离**     | 必须审批                               | 路径逃逸检测： 1、拦截包含 .. 父目录引用的路径；2、审核绝对路径是否逃逸；3、审核以 ~ 开头（home 目录）的路径； |
| 2      | **黑名单**       | 必须审批                               | `fnmatch` glob 匹配：`git*` `rm*` `del*` `rmdir*` `Remove-Item*` |
| 3      | **白名单**       | 自动放行                               | `fnmatch` glob 匹配：`ls*` `cat*` `grep*` `rg*` `find*`      |
| 4      | **是否需要审批** | `ask`→审批； `auto`→放行；`never`→拒绝 | "ask" = 沙箱/黑名单外的常规命令也要问用户<br/>"auto" = 只有沙箱逃逸和黑名单命中的才弹审批，其余静默放行<br/>"never" = 实际类似于官网模式（连读文件都做不了） |



#### 环境隔离

```python
# run_shell 执行时只传递 allowlist 中的环境变量
env = {name: os.environ[name] for name in SHELL_ENV_ALLOWLIST if name in os.environ}
env["PWD"] = workspace_root  # 覆盖 PWD
# SENSITIVE_ENV_NAME_MARKERS = ("API_KEY", "TOKEN", "SECRET", "PASSWORD")
```

- 敏感变量值在 trace/report 写入前通过 `redact_text()` / `redact_artifact()` 替换为 `<redacted>`。
- 敏感信息正则拦截（匹配 `api_key`、`token`、`secret`、`sk-` 等）在记忆沉淀前拦截，防止泄漏到持久化存储。



#### 治理效果

在 11 个治理场景测试下，

- 拦截路径逃逸 3 次，无效参数拒绝 5 次，重复调用拦截 2 次。
- 固定回归任务通过率 100%，预算内完成率 100%，verifier 通过率 100%。



### 5、任务恢复机制

KzCode 主要采用了两套任务恢复机制

- **Session**：保存会话历史 + 工作记忆 + 长期记忆，**用于"下次还能接着聊"**。
- **Checkpoint**：保存任务状态快照（当前目标、卡点、下一步、关键文件指纹、运行身份），**用于"中断后恢复现场"**。



#### 运行时身份

checkpoint 保存时记录 11 个身份字段。恢复时逐一比对，任何字段不匹配即判定为 `workspace-mismatch`，避免误信旧状态继续执行。

| 字段                | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| `cwd`               | 工作区当前目录                                               |
| `model`             | 模型标识（如 `gpt-4`、`claude-sonnet-4`）                    |
| `model_client` 类名 | 模型客户端类（如 `OpenAIClient`）                            |
| `approval_policy`   | 审批策略（`ask`/`auto`/`never`）                             |
| `read_only`         | 是否只读模式                                                 |
| `max_steps`         | 最大工具迭代步数                                             |
| `max_new_tokens`    | 模型输出最大 token 数                                        |
| `feature_flags`     | 特性开关：`memory`、`relevant_memory`、`context_reduction`、`prompt_cache` |



#### 覆盖场景

多个恢复场景（基础恢复、部分状态过期、工具半成功恢复等）。

恢复成功率 **100%**，无误信旧状态继续执行的情况。



### 6、运行工件与评测闭环

每次 `ask()` 调用会在 `.KzCode/runs/{run_id}/` 下写入：

| 工件              | 格式  | 内容                                                         |
| ----------------- | ----- | ------------------------------------------------------------ |
| `task_state.json` | JSON  | 运行状态机快照（步数、状态、进度、停止原因、checkpoint_id）  |
| `trace.jsonl`     | JSONL | 逐事件时间线（prompt 组装、模型调用、工具执行、前缀缓存命中、预算裁剪日志） |
| `report.json`     | JSON  | 运行摘要（最终答案、指标、记忆沉淀记录、secret env 摘要）    |



#### 工作区快照 Diff

每次高风险工具执行前后，对整个工作区做文件级别的快照（相对路径 → SHA256 + UTF-8 文本缓存），再对比前后差异生成 unified diff：

- **Snapshot 范围**：递归扫描工作区全部文件，跳过 `.git` / `node_modules` / `__pycache__` / `.venv` 等忽略目录。
- **变更检测**：文件缺失 / 新增 / hash 变化三种状态，仅检测UTF-8 文本（二进制跳过）。
- **Diff 预览**：基于 `difflib.SequenceMatcher` 生成 unified diff，含上下文行。上限 `MAX_DIFF_PREVIEW_FILES=20`、`MAX_DIFF_PREVIEW_LINES=1000`、`MAX_DIFF_SNAPSHOT_BYTES=256KB`。
- 变更结果写入 `trace.jsonl` 和 `report.json`，同时通过 SSE 推送给前端实时展示。



#### 多层评测

KzCode 将评测拆成多层，不混成一个总分：

- **Harness regression**：固定任务集，验证运行时合同稳定性（工具、预算、verifier）。
- **上下文治理**：压缩率、当前请求是否被裁坏。
- **记忆收益**：重复读文件次数、follow-up 工具步数。
- **恢复正确性**：恢复成功率、漂移识别率、误信旧状态次数。
- **模型后端对照**：同一任务在不同 provider（OpenAI / Anthropic / Ollama / DeepSeek）上的 pass rate / attempts / tool_steps。



### 7、多模型聚合

KzCode 不绑定特定模型提供商，支持通过统一接口切换后端：

| 提供商    | 协议               | 配置方式           |
| --------- | ------------------ | ------------------ |
| OpenAI    | OpenAI 兼容        | API Key + Base URL |
| Anthropic | Anthropic 消息 API | API Key + Base URL |
| DeepSeek  | OpenAI 兼容        | API Key + Base URL |
| Ollama    | Ollama API         | Host URL，本地模型 |

所有模型客户端共享同一组调用接口，Agent 运行时无需感知底层模型差异。



### 8、SSE 流式事件系统与审计

`POST /api/chat-stream` 端点将 KzCode 控制循环与前端实时同步。通过 `_stream_agent_response()` 在独立线程中运行 `agent.ask()`，通过事件队列向 SSE 流推送 7 种事件类型：

| 事件               | 触发时机            | 关键载荷                                                     |
| ------------------ | ------------------- | ------------------------------------------------------------ |
| `chunk`            | 流式文本增量        | `{text}`                                                     |
| `tool_call`        | 工具开始执行        | `{tool_call_id, name, args, sequence}`                       |
| `tool_result`      | 工具执行完成        | `{tool_call_id, status, content, workspace_changed, affected_paths, diff_summary, diff_preview, tool_error_code}` |
| `approval_request` | 高危工具需审批      | `{approval_id, toolName, args, status}`                      |
| `assistant_notice` | 进度通知 / 异常通知 | `{content, thinking}`                                        |
| `done`             | ask() 返回最终答案  | `{content}`                                                  |
| `error`            | 不可恢复错误        | `{message}`                                                  |



#### Hooked Record 模式

核心思想：不修改 agent 核心代码，而是在运行前用钩子函数替换 record() 方法，避免耦合。

`hooked_record()` 拦截 agent 的 `record()` 方法，在每次工具执行结果写入 history 时同步生成 SSE 事件和审计记录：

- 记录工具执行时间线（tool_call_id、开始时间、完成时间、状态、diff 摘要）。
- 记录审批请求时间线（approval_id、状态、解决时间）。
- 审计事件持久化到 `session.json`，供前端查询回放。



## 前端：桌面应用

前端基于 **Electron + Vue 3 + Pinia + Element Plus**，提供原生桌面体验。

- **会话与流式对话**
- **工具执行与审批**
- **文件变更 Diff**
- **@引用文件及上传文件附件功能**

- **项目文件树与文件编辑器**

- **Git 集成**

- **设置与主题**



## 技术栈

| 层级     | 技术                                         |
| -------- | -------------------------------------------- |
| **后端** | Python 3.10+、FastAPI、Pydantic、uv          |
| **前端** | Electron、Vue 3、Vite 6、Pinia、Element Plus |
| **协议** | HTTP + SSE（流式）、Electron IPC（文件系统） |
| **测试** | pytest                                       |



## 项目结构

```
KzCodeDesktop/                    # Electron 前端
├── src/
│   ├── main/index.js             # Electron 主进程（窗口 + IPC + 后端管理）
│   ├── preload/index.js          # contextBridge 暴露安全 API
│   └── renderer/
│       ├── src/
│       │   ├── App.vue           # 根组件
│       │   ├── views/ChatView/   # 主视图
│       │   ├── store/            # Pinia（session、theme、approval）
│       │   ├── services/api.js   # HTTP + SSE 通信
│       │   ├── components/
│       │   │   ├── chat/         # ChatPanel、MessageItem、InputArea、ModelPicker
│       │   │   ├── session/      # SessionSidebar
│       │   │   ├── layout/       # HeaderBar、SettingsDialog、RuntimeBanner
│       │   │   ├── approval/     # ApprovalPanel
│       │   │   ├── diff/         # DiffViewer、FileEditPreview
│       │   │   ├── mention/      # MentionDropdown
│       │   │   └── system/       # ToastStack、ConfirmDialog、ProcessingIndicator
│       │   ├── composables/      # useMention
│       │   ├── constants/        # models.js
│       │   ├── utils/            # assistantText.js
│       │   └── assets/themes/    # 12 套 CSS 主题
│       └── vite.config.js
├── electron.vite.config.mjs
└── package.json

KzCode/                           # Python 后端
├── app/
│   ├── main.py                   # FastAPI HTTP + SSE 服务器
│   ├── runtime.py                # Agent Harness 核心控制循环（KzCode 类）
│   ├── context_manager.py        # 上下文管理（section 预算裁剪）
│   ├── memory.py                 # 分层记忆（LayeredMemory + DurableMemoryStore）
│   ├── tools.py                  # 工具定义与执行（list/read/search/shell/write/patch/delegate）
│   ├── task_state.py             # 运行状态机快照
│   ├── run_store.py              # 运行工件持久化（task_state/trace/report）
│   ├── workspace.py              # 工作区快照（Git 状态 + 项目文档）
│   ├── models.py                 # 多模型客户端（OpenAI/Anthropic/Ollama/DeepSeek）
│   ├── config.py                 # 配置加载
│   └── cli.py                    # CLI 入口
├── tests/                        # 测试套件（含安全、记忆、上下文、恢复等）
└── pyproject.toml
```



## 快速开始

### 后端

```bash
cd KzCode
uv sync                     # 安装 Python 依赖
uv run python -m app.api_server --port 11435
```

### 前端

```bash
# 安装 Node 依赖
npm install
```

```bash
# 开发模式（Electron 桌面应用，自动启动后端）
npm run dev:electron
```

```bash
# 开发模式（仅 Web，无 Electron）
npm run dev       # → http://localhost:5173
```

```bash
# 生产构建
npm run build:electron
```



## 总结

KzCode 不是"聊天 + 文件读写"的简单组合，而是一个**工程化的本地代码智能体系统**：

- **后端**：实现了 Agent Harness 的核心——上下文分层治理与预算裁剪、结构化记忆与过期淘汰、工具执行安全隔离、检查点恢复与运行时身份校验、可审计的运行工件体系。这些设计使其在多轮长链路任务中能够保持稳定、可控、可复盘。多模型后端抽象使应用不绑定特定 AI 提供商。

- **前端**：通过 Electron 提供原生桌面体验，将 Agent 能力以流式对话、工具执行面板、审批控制、Diff 文件变更预览、项目文件树与 Git 集成等形式直观呈现，同时内置文件编辑器、@文件引用、附件上传等便捷交互。

