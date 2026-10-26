# 04 - 源码逐行解读

> 🎯 **本章目标**：给你一张**真实**的源码地图——每个目录放什么、核心类签名是什么、一条消息在代码里怎么走。看完你能打开任意文件，立刻知道它在链路里的位置。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

**阅读约定**：行号以 v0.3.5 为准；代码块首行注释标来源。行号会漂移，**类名/函数名才是稳定锚点**——对不上就 `grep -n "def _process_message" nanobot/agent/loop.py`。

---

## 目录

- [4.1 源码目录结构总览](#41-源码目录结构总览)
- [4.2 AgentLoop - loop.py](#42-agentloop---looppy)
- [4.3 AgentRunner - runner.py](#43-agentrunner---runnerpy)
- [4.4 ContextBuilder - context.py](#44-contextbuilder---contextpy)
- [4.5 MemoryStore - memory.py](#45-memorystore---memorypy)
- [4.6 SubagentManager - subagent.py](#46-subagentmanager---subagentpy)
- [4.7 工具系统 - tools/](#47-工具系统---tools)
- [4.8 MessageBus - bus/](#48-messagebus---bus)
- [4.9 Channel 适配层 - channels/](#49-channel-适配层---channels)
- [4.10 配置系统 - config/](#410-配置系统---config)
- [4.11 关键代码片段解读](#411-关键代码片段解读)
- [4.12 面试高频题](#412-面试高频题)
- [4.13 本章总结](#413-本章总结)
- [附录 A：与旧版文档的差异](#附录-a与旧版文档的差异)
- [附录 B：进阶理解（为什么这么设计）](#附录-b进阶理解为什么这么设计)
- [附录 C：Python 语法加油站](#附录-cpython-语法加油站)
- [附录 D：原文勘误](#附录-d原文勘误)

---

## 4.1 源码目录结构总览

三个数先记住：**329 个非测试 `.py`、约 99,398 行 SLOC、17 个渠道子包**（含测试共 399 个 `.py`）。

```
nanobot/
├── __main__.py  nanobot.py(SDK 门面 :136)  events.py
├── agent/            50 个 .py = 19 顶层 + 29 tools/ + 2 hooks/
│   ├── loop.py               ★ 2388 行  AgentLoop：回合编排
│   ├── runner.py             ★ 1374 行  AgentRunner + AgentRunSpec
│   ├── context.py            ★  363 行  ContextBuilder
│   ├── context_governance.py ★ 1018 行  ContextGovernor：预算/裁剪/工具结果
│   ├── memory.py             ★ 1297 行  MemoryStore/Archiver/Consolidator
│   ├── subagent.py           ★  586 行  SubagentManager
│   └── tools/            29 个 .py（4.7 详讲）
├── bus/              6 个 .py
│   ├── queue.py              ★  148 行  MessageBus（双队列）
│   └── events.py             ★   68 行  InboundMessage / OutboundMessage
├── channels/         17 个子包 + 8 个框架文件（含测试 149 个 .py）
│   ├── base.py               ★  341 行  BaseChannel(ABC)
│   ├── manager.py            ★ 1101 行  ChannelManager
│   ├── registry.py               discover_plugins（pkgutil 扫子包）
│   ├── contracts.py              plugin → 实例 spec（enabled 判定）
│   └── telegram/ discord/ slack/ feishu/ dingtalk/ email/ matrix/ mattermost/
│       mochat/ msteams/ napcat/ qq/ signal/ wecom/ weixin/ whatsapp/ websocket/
│       └── 每个子包 = manifest.py（声明字段/依赖）+ runtime.py + validation.py
├── providers/        24 个 .py
│   ├── base.py               ★ 1922 行  LLMProvider(ABC)/LLMResponse/LLMUsage
│   ├── registry.py           ★  821 行  ProviderSpec + PROVIDERS
│   ├── factory.py            ★  412 行  make_provider（按 backend 分派）
├── session/manager.py        ★ 1964 行  Session / JsonlSessionStore / SessionManager
├── config/schema.py          ★  707 行  pydantic 模型
├── cli/gateway_runtime.py    ★ _run_gateway(:340) 网关组合根
├── cli/commands.py           ★ 子命令（nanobot agent 在 :391 组装 AgentLoop）
```

### 核心类速查（v0.3.5 实测）

| 概念 | 位置 | 备注 |
|---|---|---|
| `AgentLoop`（回合编排） | `agent/loop.py:196`；`run` :1261，`_dispatch` :1391，`_process_message` :1594 | `__init__` :263 |
| `AgentRunner`（模型循环） | `agent/runner.py:141`；`run` :308，`_run_core` :386，循环 :435 | 无状态 |
| `AgentRunSpec` | `agent/runner.py:89` | 一次运行的完整输入 |
| `ContextBuilder` | `agent/context.py:89`；`build_system_prompt` :101，`build_messages` :231 | `__init__` :95 |
| `MessageBus` | `bus/queue.py:19` | `publish_inbound` :37，`consume_inbound` :41，`publish_outbound` :45 |
| `Session` / `SessionManager` | `session/manager.py:276` / :1644 | JSONL 存储 `JsonlSessionStore` :548 |
| `MemoryStore` | `agent/memory.py:58`；`MemoryArchiver` :758，`Consolidator` :1072 | — |
| `ToolRegistry` | `agent/tools/registry.py:19` | `register` :30，`execute` :187 |
| `SpawnTool`（工具名 **`spawn`**） | `agent/tools/spawn.py:48`，名称 :63 | 参数 task/label/temperature/wait |
| `BaseChannel` | `channels/base.py:21` | `send` :91，`_handle_message` :255 |
| `ChannelManager` | `channels/manager.py:82` | 出站分发 :767 |
| `ProviderSpec` | `providers/registry.py:35` | `PROVIDERS` 元组 :146 |

> 这张表就是旧文档 4.1 目录树的"正确版本"。旧树里的 `cli.py`、`bus/message_bus.py`、平铺的 `channels/telegram.py`、`providers/openai.py` 都不存在，逐条见附录 D。

---

## 4.2 AgentLoop - loop.py

`nanobot/agent/loop.py` 共 **2388 行**，全仓最复杂。它只做"回合"，但回合周边的一切（会话、锁、压缩、持久化、交付）都在这里。七阶段共享同一个可变上下文 `TurnContext`（`:132`，近 40 个字段），所以每个 `_xxx_turn(ctx)` 都能独立测试；`TurnKind`（`:127`）区分用户回合与系统回合。

```python
# nanobot/agent/loop.py:263（节选；完整签名 40 余个参数）
def __init__(self, bus: MessageBus, provider: LLMProvider, workspace: Path,
             model: str | None = None,
             max_iterations: int | None = None,        # None → AgentDefaults().max_tool_iterations = 200
             context_window_tokens: int | None = None, # None → 200000
             max_tool_result_chars: int | None = None, # None → 16000
             restrict_to_workspace: bool = False,      # 注意：默认 False
             tool_registry: ToolRegistry | None = None, ...):
```

**代码里不写死数字**，全部回落 `AgentDefaults`（`config/schema.py:116`；`max_tool_iterations=200` 在 `:129`，`max_tool_result_chars=16_000` 在 `:131`），见 `loop.py:321`/`:344` 的三元表达式。

`run()`（`:1261`）负责消费：1 秒超时轮询 `bus.consume_inbound()`（`:1269`），空闲时 `_check_expired_sessions_if_due()`；命令与运行时控制就地处理，其余 `create_task(self._dispatch(msg))`（`:1376`）。`_dispatch`（`:1391`）在 `async with lock, gate`（`:1418`）里调用 `_process_message`，最后 `delivery.complete(...)`（`:1435`）；会话锁存在 `WeakValueDictionary`（`:403`，`_get_session_lock` 在 `:2382`），同会话正在跑时新消息进 `_pending_queues`（`:1339`）被当前回合"注入"，而不是排队等锁。

### _process_message()：七阶段

```python
# nanobot/agent/loop.py:1594
ctx = TurnContext(msg=msg, session=None, session_key=key,
                  turn_id=f"{key}:{time.time_ns()}", ...)            # :1626
await self._run_turn_stage(ctx, "restore", self._restore_turn)      # :1679
await self._run_turn_stage(ctx, "compact", self._compact_session)   # :1680
if await self._run_turn_stage(ctx, "command", self._dispatch_command):
    return ctx.outbound                                             # :1681
await self._run_turn_stage(ctx, "build", self._build_turn)          # :1683
await self._run_turn_stage(ctx, "run", self._run_turn)              # :1684
await self._run_turn_stage(ctx, "save", self._persist_turn)         # :1685
await self._run_turn_stage(ctx, "respond", self._prepare_outbound)  # :1686
```

| 阶段 | 方法 | 干什么 |
|---|---|---|
| `restore` | `:1748` | 附件引用化、`get_or_create` 会话、按 `session.policy.disabled_tools` 收窄工具 |
| `compact` | `:1805` | `auto_compact.prepare_session` → 换入摘要 |
| `command` | `:1813` | 斜杠命令；命中就落库并 `return`，跳过后面四阶段 |
| `build` | `:1865` | 定 runtime、取 `session.get_history()`、处理 provider 续聊状态 |
| `run` | `:1974` | `_run_agent_loop`(:939) → `runner.run(spec)`(:1170)，回收结果字段 |
| `save` | `:2014` | 修空回复、算延迟、`_save_turn`、`sessions.save`、清标记 |
| `respond` | `:2063` | 造 `OutboundMessage`，决定是否压制（模型已用 message 工具发过） |

### _save_turn()：不写任何 Markdown

```python
# nanobot/agent/loop.py:2140
if role == "assistant" and not content and not entry.get("tool_calls"):
    continue                       # :2212 空 assistant 会污染上下文，直接丢
if role == "tool" and (not tool_call_id_str or ... not in declared_tool_call_ids):
    continue                       # :2221 没有对应 tool_call 的结果也是毒药
session.messages.append(entry)     # :2251 真正写入会话
```

**它只是把消息追加进 `session.messages`，一个 `.md` 都不碰。** 落盘由 `sessions.save(session)`（`:2054`）走 `JsonlSessionStore`。工具注册也不是手写清单：

```python
# nanobot/agent/loop.py:615
ctx = ToolContext(config=self.tools_config, workspace=str(self.workspace), bus=self.bus, ...)  # :624
registered = ToolLoader().load(ctx, self.tools)                                                # :639
```

![AgentLoop 循环漫画](../../../comics/03-agent-loop.png)

---

## 4.3 AgentRunner - runner.py

`nanobot/agent/runner.py` 共 **1374 行**：两个数据类 + 一个无状态类。

```python
# nanobot/agent/runner.py:88
@dataclass(slots=True)
class AgentRunSpec:
    """Configuration for a single agent execution."""
    initial_messages: list[dict[str, Any]] | None   # 与 transcript_input 二选一
    tools: ToolRegistry
    runtime: LLMRuntime                             # provider + model + 窗口 + 生成参数
    max_iterations: int
    max_tool_result_chars: int
    transcript_input: TranscriptInput | None = None
    hook: AgentHook | None = None
    checkpoint_callback: CheckpointCallback | None = None
    consolidate_history: HistoryConsolidator | None = None
    injection_callback: InjectionCallback | None = None
    finalize_on_max_iterations: bool = True

@dataclass(slots=True)
class AgentRunResult:                    # :119 final_content / messages / tools_used /
    ...                                  #      usage / stop_reason / error / summary_checkpoint

class AgentRunner:                       # :141
    """Run a tool-capable LLM loop without product-layer concerns."""
    def __init__(self) -> None:          # :144
        self.context_governor = ContextGovernor()
```

`__init__` 只有一行——这就是"无状态协作者"的字面意思。旧文档里 `AgentRunner.__init__(provider, tool_registry, context_builder, memory_store, max_iterations, is_subagent)` 的签名完全不存在。

`_initial_transcript_and_compaction`（`:363`）做"二选一"校验：给了 `transcript_input` 就不能给 `initial_messages`。**这是主 Agent 与子 Agent 共用同一循环的接缝。** 钩子清单在 `agent/hook.py:66`：`before_run`/`after_run`(`:75`/`:78`)、`on_error`/`on_finally`(`:81`/`:84`)、`before_iteration`/`after_iteration`(`:87`/`:147`)、`before_execute_tools`(`:104`)、`before_execute_tool`/`after_execute_tool`/`on_execute_tool_error`(`:107`/`:116`/`:126`)、`on_stream`/`on_stream_end`(`:90`/`:93`)、`finalize_content`(`:150`)；`CompositeHook`(`:157`) 负责合成。

```python
# nanobot/agent/runner.py:386 _run_core
for iteration in range(spec.max_iterations):                            # :435 默认 200
    await hook.before_iteration(context)                                # :441
    response, raw_usage = await self._request_model(...)                # :448
    if response.should_execute_tools:                                   # :482
        results, new_events = await execute_tool_calls(...)             # :512
        for tool_call, result in zip(response.tool_calls, results):
            messages.append({"role": "tool", ..., "content":
                self.context_governor.normalize_tool_result(...)})      # :537
        continue
    break                                                               # :789
else:
    stop_reason = "max_iterations"                                      # :791
```

### 三种收场方式（失败路径全貌）

1. **正常**：模型不再请求工具 → 追加 assistant → `stop_reason="completed"`（`:756`-`:789`）。
2. **模型报错/异常输出**：`finish_reason == "error"` → `stop_reason="error"` 并追加占位消息（`:715`-`:736`，欠费走 `:716`）；空回复最多重试 `_MAX_EMPTY_RETRIES` 次后强行走收尾（`:594`-`:629`）；`length` 截断最多续写 `_MAX_LENGTH_RECOVERIES` 次（`:631`-`:656`）。
3. **迭代用尽**：`for...else` 进 `:790`，先做一次"无工具收尾"——`_try_finalize_after_max_iterations`（`:1163`）追加"预算用尽请收尾"提示后调 `_request_no_tools`（`:1219`，`tools=None`）；仍失败则用模板兜底 `_max_iterations_fallback`（`:1259`，`agent/max_iterations_message.md`）。

---

## 4.4 ContextBuilder - context.py

`nanobot/agent/context.py` 只有 **363 行**，却决定模型"看到的世界"。

```python
# nanobot/agent/context.py:89（真实签名，注意没有 config 参数）
class ContextBuilder:
    BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md"]        # :92
    _SKIPPABLE_DEFAULTS = {"AGENTS.md", "USER.md"}               # :93
    def __init__(self, workspace: Path, timezone: str | None = None,
                 disabled_skills: list[str] | None = None):       # :95
        self.memory = MemoryStore(workspace)                      # :98 组合，不是注入
        self.skills = SkillsLoader(workspace, disabled_skills=...) # :99
    def build_system_prompt(self, *, channel=None, session_summary=None,
                            workspace=None, include_memory=True) -> str:   # :101
    def build_messages(self, history, current_message, *, media=None, ...):  # :231
    def build_transcript(self, transcript: TranscriptInput, *, ...):         # :276
    def build_current_message(self, current_message, *, media=None, ...):    # :310
```

四个方法：`build_system_prompt` 出一整段 system；`build_transcript` 拼 `[system] + history + current`；`build_messages` 是兼容包装（再把相邻同角色合并）；`build_current_message` 只造本轮那条 user 消息（运行时上下文塞 `_meta`，不污染正文）。

```python
# nanobot/agent/context.py:101 拼装顺序 = 代码顺序
parts = [self._get_identity(...)]                       # :111 身份 + 平台策略
parts.append(self._load_bootstrap_files(root))          # :113 AGENTS/SOUL/USER.md
parts.append(render_template("agent/tool_contract.md")) # :117 工具契约
memory = self.memory.read_memory()                      # :128（:130 用 "# Memory" 包起来）
active_skills = self.skills.get_always_skills()         # :132 常驻技能全文
skills_summary = self.skills.build_skills_summary(...)  # :138 其余技能只给摘要
if session_summary: parts.append("[Archived Context Summary] ...")   # :145
return "\n\n---\n\n".join(parts)                        # :152
```

一个易忽略的设计：`_is_template_content`（`:224`）与包内模板比对，**内容等于模板就跳过**——用户没改过的 `AGENTS.md`/`MEMORY.md` 不白占 prompt。

### 治理层在另一个文件

`context.py` 只负责拼，裁在 `context_governance.py`（1018 行）：

| 机制 | 位置 | 触发 |
|---|---|---|
| 输入预算 | `:693` | 窗口 − 最大输出 − `SNIP_SAFETY_BUFFER`(1024, `:67`) |
| 请求前准备（可能压缩） | `:593` `prepare_request` | 每轮请求 |
| 历史裁剪 | `:951` `snip_history` | 估算超预算且无 compaction |
| 摘要替换 H 段 | `:521` `_compact_request_history` | 估算超预算且有 compaction |
| 最终校验 | `:358` `ensure_request_fits` | 发请求前，超了抛 `ContextWindowExceededError`(`:76`) |

```python
# nanobot/agent/context_governance.py:709
if tool_name in TOOL_RESULT_OFFLOAD_EXEMPT_TOOLS:      # :69 = frozenset({"read_file"})
    return result
content = maybe_persist_tool_result(config.workspace, config.session_key, call_id,
                                    text, max_chars=config.max_tool_result_chars)
# helpers.py:580 → 超长就写到 workspace 文件，返回 "路径 + 预览 + 原始大小" 的引用
```

**没有 `_TOOL_RESULT_MAX_CHARS` 这个常量**：长度来自 `ContextGovernanceConfig.max_tool_result_chars`（`:113`/`:119`），源头是 `agents.defaults.maxToolResultChars`。

---

## 4.5 MemoryStore - memory.py

`nanobot/agent/memory.py` 共 **1297 行**，四个主要类：

| 类 | 位置 | 职责 |
|---|---|---|
| `MemoryStore` | `:58` | 纯文件 I/O：`MEMORY.md`、`history.jsonl`、`SOUL.md`、`USER.md`、cursor、GitStore |
| `MemoryArchiver` | `:758` | 把一段对话交给 LLM 摘要 |
| `Consolidator` | `:1072` | 编排：拿锁、算预算、调 Archiver、提交 checkpoint |

```python
# nanobot/agent/memory.py:58
class MemoryStore:
    """Pure file I/O for memory files: MEMORY.md, history.jsonl, SOUL.md, USER.md."""
    def __init__(self, workspace: Path, max_history_entries: int = 1000):       # :72
        self.history_file = ensure_dir(workspace / "memory") / "history.jsonl"  # :77
        self.legacy_history_file = self.memory_dir / "HISTORY.md"               # :78 仅迁移源
        self._maybe_migrate_legacy_history()                                    # :91
```

`_maybe_migrate_legacy_history`（`:106`）做一次性升级：有 `HISTORY.md` 且 `history.jsonl` 为空时，解析旧格式写进 jsonl，再把 `HISTORY.md` 改名备份。**这是 `HISTORY.md` 在 v0.3.5 的唯一用途。**

```python
# nanobot/agent/memory.py:1222 Consolidator.compact_idle_session
lock = self.get_lock(session_key)                 # :1235 同一会话只允许一个压缩
async with lock:
    archive_start = session.last_archived          # :1240
    summary = await self.archive_session(session, archive_end=archive_end, runtime=runtime)  # :1256
    if summary:
        session.commit_summary_checkpoint(summary, insert_at=archive_end, ...)  # :1261
        session.provider_state = None              # :1265 旧的原生续聊状态作废
```

> ⚠️ 记忆题三个高频错误：类名不是 `MemoryConsolidator` 而是 `Consolidator`；压缩**不写** `MEMORY.md`（只写 `history.jsonl` + checkpoint）；没有 `save_memory` 工具。

---

## 4.6 SubagentManager - subagent.py

`nanobot/agent/subagent.py` 共 **586 行**。核心事实：**子 Agent 复用同一个 `AgentRunner`，只换 Spec 形态**。

```python
# nanobot/agent/subagent.py:427
result = await self.runner.run(AgentRunSpec(
    initial_messages=[{"role": "system", "content": system_prompt},   # :411 子 Agent 专用 prompt
                      {"role": "user", "content": task}],
    max_iterations=self.max_iterations, max_tool_result_chars=self.max_tool_result_chars,
    hook=_SubagentHook(task_id, status),
    max_iterations_message="Task completed but no final response was generated.",  # :434
    finalize_on_max_iterations=False,     # :435 不做额外收尾请求，省 token
))
```

三点与旧文档不同：① 上限继承 200 而不是 15；② 不用 `clone()`+`remove()`，用 `scope="subagent"` 过滤；③ 结果通过 `bus` 作为"注入消息"回主会话（`_announce_result`，`:488`），不直接回复用户。

---

## 4.7 工具系统 - tools/

`nanobot/agent/tools/` 有 **29 个 `.py`**（含 `__init__.py` 与私有辅助 `_windows_job.py`；以 `git ls-tree -r v0.3.5 -- nanobot/agent/tools` 的 29 为准）。

```python
# nanobot/agent/tools/registry.py:19
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}
        self._cached_definitions: list[dict[str, Any]] | None = None
    def register(self, tool: Tool) -> None:      # :30  注册（顺手清缓存）
        self._tools[tool.name] = tool; self._cached_definitions = None
    def unregister(self, name: str) -> None:     # :35  唯一的移除 API
    def get(self, name: str) -> Tool | None:     # :40
    def has(self, name: str) -> bool:            # :71
    def get_definitions(self) -> list[dict[str, Any]]:   # :86  给模型看的 schema 列表
    def prepare_call(self, name, params):        # :110  找不到/参数非法 → ToolResult.error
    async def execute(self, name: str, params: Any) -> Any:   # :187
        result = await tool.execute(**params)                 # :196
    @property
    def tool_names(self) -> list[str]:           # :203
```

**没有 `virtual` / `clone` / `remove` / `register_virtual` / `get_tool_schemas`。** 想做受限工具集，正确做法是另建 registry + `scope` 过滤（见 4.6）。


排序是为**跨请求前缀稳定**，让 provider 的 prompt cache 能命中（`mcp_` 前缀是判定依据）。

```python
# nanobot/agent/tools/base.py:144 / :159
class ToolResult(str):                     # 继承 str，能直接当字符串回灌给模型
    is_error: bool
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...              # :171
    @property
    @abstractmethod
    def description(self) -> str: ...       # :177
    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]: ...   # :183 JSON Schema
    @abstractmethod
    async def execute(self, **kwargs: Any) -> Any: ...   # :227
    @property
    def concurrency_safe(self) -> bool: return self.read_only and not self.exclusive   # :194
    _scopes: set[str] = {"core"}            # :208  ← scope 过滤依据
```

新增工具 = 放一个 `Tool` 子类，实现四个成员。`ToolLoader.discover`（`loader.py:36`）用 `pkgutil.iter_modules` 扫包，`load`（`:92`）按 `enabled()` → `create(ctx)` → `scope`（`:100`）注册。

```python
# nanobot/agent/tools/spawn.py:24（参数由装饰器声明）
@tool_parameters(tool_parameters_schema(
    task=StringSchema("The task for the subagent to complete"),
    label=StringSchema(...), temperature=NumberSchema(..., 0.0, 2.0),
    wait=BooleanSchema(..., default=False), required=["task"]))
class SpawnTool(Tool):                       # :48
    @property
    def name(self) -> str: return "spawn"    # :63  工具名就叫 spawn
    async def execute(self, task, label=None, temperature=None, wait=False, **kw)  # :81
```

MCP 侧 `MCPToolWrapper`（`mcp.py:595`）把远程工具包成本地 `Tool`，命名 `mcp_<server>_<tool>`；连接由 `MCPProvider.from_config`（`:1365`）+ `connect_mcp_servers`（`:1000`）完成。并发调度在 `execution.py:56`：`_partition_tool_batches`（`:292`）按 `concurrency_safe`/`exclusive` 分批，安全工具用 `asyncio.gather` 并行（`:83`），**返回顺序仍与调用顺序一致**（`:109`）。

---

## 4.8 MessageBus - bus/

`nanobot/bus/queue.py` 共 **148 行**，全仓最短的核心文件。

```python
# nanobot/bus/queue.py:19
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()      # :32
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()    # :33
        self._handlers: list[EventHandler] = []
    async def publish_inbound(self, msg): await self.inbound.put(msg)             # :37
    async def consume_inbound(self): return await self.inbound.get()              # :41
    async def publish_outbound(self, msg): await self.outbound.put(msg)           # :45
    async def publish_event(self, event, *, channel, chat_id, metadata=None)      # :49 事件也走 outbound
    async def consume_outbound(self): return await self.outbound.get()            # :67
    def subscribe(self, handler, event_type=None) -> Callable[[], None]           # :92
    async def publish(self, event) -> None: ...    # :118 按注册顺序 await 本地订阅者
    async def drain(self) -> None: ...             # :145 退出前跑完挂起的本地分发
```

```python
# nanobot/bus/events.py:25 / :53
@dataclass
class InboundMessage:
    channel: str; sender_id: str; chat_id: str; content: str     # 真实字段名
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    session_key_override: str | None = None
    require_existing_session: bool = False
    input_role: Literal["user", "system"] | None = None
    @property
    def session_key(self) -> str: return self.session_key_override or f"{self.channel}:{self.chat_id}"  # :40

@dataclass
class OutboundMessage:
    channel: str; chat_id: str; content: str
    reply_to: str | None = None; media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    buttons: list[list[str]] = field(default_factory=list)
    event: "AgentEvent | None" = None
```

为什么用 `asyncio.Queue` 而不是 Redis/RabbitMQ：单进程、零依赖、个人助手场景够用；真要跨进程，只有 5 个方法需要改。

---

## 4.9 Channel 适配层 - channels/

```python
# nanobot/channels/base.py:21
class BaseChannel(ABC):
    name: str = "base"; display_name: str = "Base"       # :29  ← 标识是 name，不是 channel_name
    send_progress: bool = True; send_tool_hints: bool = True; show_reasoning: bool = True
    @abstractmethod
    async def start(self) -> None: ...                    # :74
    @abstractmethod
    async def stop(self) -> None: ...                     # :86
    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...  # :91 失败必须抛异常，便于统一重试
    async def _handle_message(self, sender_id, chat_id, content, ..., is_dm=False):  # :255
        if not self.is_allowed(permission_id): ...; return    # :275 allowFrom / pairing
        msg = InboundMessage(channel=self.name, sender_id=str(sender_id), ...)  # :310
        await self.bus.publish_inbound(msg)                # :321
```

`_handle_message` 一次做完**授权 + 构造 + 入队**，子类只需关心"怎么从平台拿事件、怎么把消息发出去"。

```python
# nanobot/channels/telegram/manifest.py（节选）
SETUP_SPEC = ChannelSetupSpec(
    fields={"token": field("secret"), "allowFrom": field("list"),
            "mode": field("enum", choices={"polling", "webhook"}, default="polling"), ...},
    required=(required("token"),), verifies_connection=True)
PLUGIN = ChannelPlugin(name="telegram", display_name="Telegram",
    runtime=f"{__package__}.runtime:TelegramChannel",     # 字符串路径，延迟导入
    setup=SETUP_SPEC, dependencies=("python-telegram-bot>=22.6,<23.0", ...),
    webui="webui/index.ts")
```

`ChannelManager._init_channels`（`manager.py:234`）用 `discover_plugins()`（`channels/registry.py:30`，`pkgutil` 扫子包）拿到 17 个 plugin，再按 `enabled` 过滤：`channel_instance_specs(plugin, section)`（`manager.py:253` → `contracts.py:337`），内部 `:349` 判定 `enabled_only and not activation.resolve(default=plugin.default_enabled)` 就 `continue`。**所以没写 `"enabled": true`（且 plugin 默认不启用）的渠道不会启动**。

```python
# nanobot/channels/manager.py:767 出站：一个循环管 17 个渠道
msg = await asyncio.wait_for(self.bus.consume_outbound(), timeout=1.0)   # :781
...
await self._send_with_retry(channel, msg)                                # :998
```

`_send_with_retry`（`:998`）统一退避重试；`_coalesce_stream_deltas`（`:933`）把流式碎片合并成一次编辑，避免被平台限流。

---

## 4.10 配置系统 - config/

`nanobot/config/schema.py` 共 **707 行**，全是 pydantic 模型。

```python
# nanobot/config/schema.py
class AgentDefaults(Base):                 # :116
    workspace: str = "~/.nanobot/workspace"
    model: str = "anthropic/claude-opus-4-5"; provider: str = "auto"
    max_tokens: int = 8192; context_window_tokens: int = 200_000; temperature: float = 0.1
    max_tool_iterations: int = 200         # :129
    max_concurrent_subagents: int = Field(default=4, ge=1)   # :130
    max_tool_result_chars: int = 16_000    # :131
class ToolsConfig(Base):                   # :384
    web: WebToolsConfig; exec: ExecToolConfig; file: FileToolsConfig
    restrict_to_workspace: bool = False    # :401 默认 false
    mcp_servers: dict[str, MCPServerConfig] = Field(default_factory=dict)  # :418
class Config(BaseSettings):                # :422
    agents; channels; providers; tools; gateway; api; modelPresets
```

三个必记事实：① **格式是 JSON**，路径 `~/.nanobot/config.json`（`--config` 可覆盖，cwd 不参与）；② 字段以 **camelCase** 落盘（`apiKey`、`maxToolIterations`…），snake_case 兼容（`validation_alias=AliasChoices(...)`）；③ `${VAR}` 在内存解析、不回写。

---

## 4.11 关键代码片段解读

### 片段 1：一条消息的完整调用栈（真实行号）

```
用户（Telegram）
 ▼
TelegramChannel.start                     channels/telegram/runtime.py:603
 └─ BaseChannel._handle_message           channels/base.py:255
      └─ bus.publish_inbound()            bus/queue.py:37
 ▼
AgentLoop.run                             agent/loop.py:1261
 ├─ wait_for(consume_inbound(), 1.0s)     agent/loop.py:1269
 └─ create_task(_dispatch(msg))           agent/loop.py:1376
      ▼
   _dispatch                              agent/loop.py:1391
    └─ async with lock, gate              agent/loop.py:1418
         ▼
       _process_message                   agent/loop.py:1594
        ├─ _restore_turn                  :1748
        ├─ _compact_session               :1805 → autocompact.py:116
        ├─ _dispatch_command              :1813（命中则 return）
        ├─ _build_turn                    :1865
        ├─ _run_turn                      :1974
        │    └─ _run_agent_loop           :939 → runner.run(AgentRunSpec) :1170
        │         ▼
        │       AgentRunner.run           agent/runner.py:308
        │         └─ _run_core            agent/runner.py:386
        │              ├─ _request_model           :448
        │              ├─ execute_tool_calls       :512 → tools/execution.py:56
        │              │    └─ ToolRegistry.execute  tools/registry.py:187
        │              └─ normalize_tool_result    :537
        ├─ _persist_turn                  :2014
        └─ _prepare_outbound              :2063
         ▼
       delivery.complete()                agent/loop.py:1435
        └─ bus.publish_outbound()         bus/queue.py:45
```

### 片段 2：Provider 工厂按 backend 分派

```python
# nanobot/providers/registry.py:146 —— 元数据元组，不是 {name: Class}
PROVIDERS: tuple[ProviderSpec, ...] = (...)
# nanobot/providers/factory.py:150
if   backend == "openai_codex":   ... OpenAICodexProvider(...)
elif backend == "xai_grok":       ... XAIGrokProvider(...)
elif backend == "azure_openai":   ... AzureOpenAIProvider(...)
elif backend == "github_copilot": ... GitHubCopilotProvider(...)
elif backend == "anthropic":      ... AnthropicProvider(...)
elif backend == "bedrock":        ... BedrockProvider(...)
else:                             ... OpenAICompatProvider(...)   # 大多数供应商
```

---

## 4.12 面试高频题

### Q1：AgentLoop 的执行流程？

> 六步。① `run()`（`loop.py:1261`）1 秒超时轮询 `bus.consume_inbound()`，空闲时做会话过期检查；② 收到消息先做运行时控制/优先级命令/自动化延时判定，再 `create_task(self._dispatch(msg))`；③ `_dispatch`（`:1391`）拿会话锁 + 可选闸门，并为该会话建"回合内注入队列"；④ `_process_message`（`:1594`）跑七阶段 restore→compact→command→build→run→save→respond；⑤ `run` 阶段把 `AgentRunSpec` 交给 `AgentRunner` 拿回 `AgentRunResult`；⑥ `save` 落库、`respond` 造 `OutboundMessage`，`_dispatch` 里 `delivery.complete(...)` 推回总线。

### Q2：`AgentRunner` 为什么可以无状态？

> 因为"这一次运行需要的一切"都收进了 `AgentRunSpec`（`runner.py:89`）：messages 或 transcript、tools、runtime、上限、hook、checkpoint 回调、注入回调。`__init__`（`:144`）只初始化一个 `ContextGovernor`。
>
> 好处：同一实例服务多个会话（`agent/loop.py:385` 只造一次）；主 Agent 与子 Agent 共用一份循环（`subagent.py:427` 传 `initial_messages`）；单测造一个 Spec 就能跑完整循环。

---

## 4.13 本章总结

```
┌──────────────────────────────────────────────────────────┐
│ 核心文件速查（v0.3.5）                                     │
│ agent/loop.py          2388  AgentLoop       回合编排      │
│ agent/runner.py        1374  AgentRunner     模型循环      │
│ agent/context.py        363  ContextBuilder  prompt 拼装   │
│ agent/context_governance.py 1018  预算/裁剪/工具结果治理     │
│ agent/memory.py        1297  Memory/Consolidator/Dream    │
│ 规模：329 非测试 .py / 约 99,398 行 SLOC / 17 渠道子包      │
└──────────────────────────────────────────────────────────┘
```

### 下一步读什么

➡️ [05 - MCP 协议详解](../05-mcp-protocol/README.md)

想先把工具链吃透，也可以直接跳第 08 章，从 `nanobot/agent/tools/base.py:159` 开始给自己写一个工具。

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 核心文件 10 个、约 4000 行 | 329 个非测试 `.py`、约 99,398 行 SLOC；单 `loop.py` 就 2388 行 | `git ls-tree -r v0.3.5` | 引用真实规模 |
| `AgentRunner` 每次运行 new 一个 | `self.runner = AgentRunner()` 只造一次 | `agent/loop.py:385` | 记住"无状态可复用" |
| `AgentRunner.__init__` 有 6 个参数 | 只有 `self`，仅初始化 `ContextGovernor` | `agent/runner.py:144` | 用 Spec 讲输入 |
| `ToolRegistry` 有 `register_virtual`/`clone`/`remove`/`get_tool_schemas` | 只有 `register`/`unregister`/`get`/`has`/`get_definitions`/`prepare_call`/`execute`/`tool_names` | `agent/tools/registry.py:19`-`:212` | 换真实 API |
| 子 Agent 用 `clone()` 后 `remove()` | 用 `scope="subagent"` + 新 registry | `agent/subagent.py:224`、`agent/tools/loader.py:100` | 讲 scope 机制 |
| 子 Agent 15 次迭代 | 继承 `max_tool_iterations=200` | `agent/subagent.py:140`、`config/schema.py:129` | 数字更新 |
| `MessageBus` 只有两个 put/get | 还有 `publish_event`/`subscribe`/`publish`/`drain` | `bus/queue.py:49`/`:92`/`:118`/`:145` | 补上事件通道 |
| `BaseChannel` 用 `channel_name` 标识 | 类属性是 `name`（小写渠道名） | `channels/base.py:29` | 字段名更新 |
| 配置类叫 `NanobotConfig`/`ProviderConfig` | 是 `Config` / `AgentsConfig` / `ToolsConfig` / `ProviderSpec` | `config/schema.py:422` | 名称更新 |
| `providers/registry.py` 是 `{name: Class}` 字典 | `PROVIDERS: tuple[ProviderSpec, ...]`，实现由 `backend` 决定 | `providers/registry.py:146`、`providers/factory.py:150` | 讲 backend 分派 |
| 工具文件 30 个 | 29 个 `.py` | `git ls-tree -r v0.3.5 -- nanobot/agent/tools` | 用 29 |

---

## 附录 B：进阶理解（为什么这么设计）

**1. 为什么 `loop.py` 敢写到 2388 行？** 因为它把回合的全部周边事务收进一个文件，换来其余文件都很薄：`runner.py` 只管循环、`context.py` 只管拼装、`queue.py` 只有 148 行。代价是新人第一次打开会被吓到——所以本章按"七阶段"切开读。

**2. 为什么 `TurnContext` 是可变 dataclass？** 七阶段每个都可能产出后续要用的字段（`session`/`history`/`final_content`/`stop_reason`）。用参数列表会变成十几层传参；用可变上下文则"谁产出谁赋值"，且阶段签名统一成 `(ctx)`，能塞进 `_run_turn_stage`（`loop.py:1689`）统一计时记日志。

**3. 为什么 `get_definitions` 要排序再缓存？** prompt cache 命中的前提是**前缀逐字节相同**。工具顺序若随 `dict` 插入顺序漂移，每次请求的 tool 定义区都会变，缓存全废。所以 `register` 时清缓存、`get_definitions` 时排序（`registry.py:93`-`:106`）。

**4. 为什么 `execute_tool_calls` 要分批而不全并行？** 工具分三类：只读且安全（可并行）、独占（必须单独跑）、有副作用（串行）。`_partition_tool_batches`（`execution.py:292`）用 `read_only`/`concurrency_safe`/`exclusive` 决定分批，既拿并行收益，又不让两个写操作互踩。

---

## 附录 C：Python 语法加油站

### C.1 `@dataclass(slots=True)` 与 `field(default_factory=...)`

```python
@dataclass(slots=True)                 # slots：只允许声明过的属性，省内存
class AgentRunSpec:
    tools: object
    max_iterations: int
    hook: object | None = None         # 有默认值的字段必须排在无默认值之后
    events: list = field(default_factory=list)   # 可变默认值不能直接写 []
```

用到：`agent/runner.py:88`、`agent/loop.py:132`、`session/manager.py:275`。读法：dataclass = 带类型的记录；`slots=True` 表示不能乱加字段。

### C.2 `ABC` + `@property @abstractmethod`

```python
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...          # 叠加：抽象 + 只读
class SpawnTool(Tool):
    @property
    def name(self) -> str: return "spawn"
```

用到：`agent/tools/base.py:159`、`channels/base.py:21`、`providers/base.py:623`。读法：`@property` 让取值变成方法，`@abstractmethod` 强制实现，顺序固定（property 在外层）。

### C.3 pydantic `BaseModel` / `BaseSettings`

```python
class AgentDefaults(BaseModel):
    max_tool_iterations: int = 200
    max_concurrent_subagents: int = Field(default=4, ge=1)   # 必须 >= 1
cfg = AgentDefaults(max_tool_iterations="300")   # 自动把 "300" 转成 int
```

用到：`config/schema.py:116`/`:384`/`:422`（`Config` 继承 `BaseSettings`，会自动读环境变量）。读法：pydantic 模型 = "带校验与类型转换的 dataclass"。

---

## 附录 D：原文勘误

| # | 旧文档原文 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| D1 | 目录树有 `nanobot/cli.py` | 不存在；CLI 是**包**，入口 `nanobot/cli/commands.py`，另有 19 个兄弟模块 | 改为 `cli/` 目录 | `nanobot/cli/`（20 个 `.py`） |
| D2 | 目录树有 `bus/message_bus.py` | 不存在；是 `bus/queue.py`（148 行），同目录还有 `events.py` | `nanobot/bus/queue.py` | `bus/queue.py:19` |
| D3 | 目录树里 `channels/telegram.py`、`discord.py`、`feishu.py`… 平铺 | 17 个渠道全是**子包**，含 `manifest.py`/`runtime.py`/`validation.py` | `channels/<name>/` | `ls nanobot/channels/telegram` |
| D4 | 目录树里 `providers/openai.py`、`anthropic.py`、`deepseek.py`、`ollama.py`、`dashscope.py`、`groq.py` | 这些文件都不存在。实际是 `openai_compat_provider.py`/`anthropic_provider.py`/`azure_openai_provider.py`/`bedrock_provider.py`/`xai_grok_provider.py`/`github_copilot_provider.py`/`openai_codex_provider.py` + `factory.py`/`registry.py`；DeepSeek/Ollama/DashScope 走 OpenAI 兼容实现，靠 `ProviderSpec` 区分 | 看 `providers/` 真实清单 | `providers/registry.py:146`、`providers/factory.py:150` |
| D5 | 目录树里 `config/loader.py # YAML 配置加载` | 加载的是 JSON；`loader.py` 还负责 `${VAR}` 与路径解析 | `~/.nanobot/config.json` | `config/loader.py`、`config/schema.py:422` |
| D6 | 4.7.1 `ToolRegistry.register_virtual` / `clone()` / `remove()` / `get_tool_schemas()` | 四个方法都不存在 | 见 4.7 的真实 API | `agent/tools/registry.py:19`-`:212` |
| D7 | 4.7.4 `class SpawnTool: name = "spawn_agent"` | 工具名是 `"spawn"`，且 `name` 是 property 不是类属性 | `"spawn"` | `agent/tools/spawn.py:63` |
| D8 | 4.7.3 `MessageTool.name = "message"`、`parameters_schema` 类属性、构造函数传 `bus` | `name`/`parameters` 是 property；依赖通过 `ToolContext` 注入（`create(ctx)`） | 用 property + `create(ctx)` | `agent/tools/base.py:171`、`:219` |
| D9 | 4.5 / 4.12「`MemoryConsolidator`、阈值 50 条、`_summarize_with_llm`」 | 类名/方法名/常量都不存在 | `Consolidator`(:1072) + `MemoryArchiver`(:758) + `AutoCompact`(TTL) | `agent/memory.py`、`agent/autocompact.py:39` |
| D10 | 4.13「总代码量约 4000 行、核心文件 10 个」 | 规模严重低估 | 329 个非测试 `.py` / 约 99,398 行；单 `agent/` 就 50 个 `.py` | `git ls-tree -r v0.3.5` |
