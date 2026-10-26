# 03 - 架构深入解析

> 🎯 **本章目标**：看完你能回答"一条消息从进来到回复经过谁"，并一句话说清 `AgentLoop` 与 `AgentRunner` 的分工边界；同时能指出旧文档"五层架构"错在哪、为什么误导人。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

**一句话结论**：nanobot 不是"五层"，而是**一条调用链 + 三个外设**。

- 调用链：`Channel → MessageBus → AgentLoop → AgentRunner → Provider/Tools → MessageBus → Channel`；
- 三个外设：**Channel**（接哪些平台）、**Provider**（用哪个模型）、**Tool**（有哪些工具）。三者都是"接口 + 注册表 + 配置注入"，互不为"层"。

---

## 目录

- [术语表](#术语表)
- [3.1 整体架构概览](#31-整体架构概览)
- [3.2 四大核心模块详解](#32-四大核心模块详解)
- [3.3 数据流图](#33-数据流图)
- [3.4 模块间的协作关系](#34-模块间的协作关系)
- [3.5 关键设计模式](#35-关键设计模式)
- [3.6 架构对比](#36-架构对比)
- [3.7 面试高频题](#37-面试高频题)
- [练习题与思考题](#练习题与思考题)
- [3.8 本章总结](#38-本章总结)
- [附录 A：与旧版文档的差异](#附录-a与旧版文档的差异)
- [附录 B：进阶理解（为什么这么设计）](#附录-b进阶理解为什么这么设计)
- [附录 C：Python 语法加油站](#附录-cpython-语法加油站)
- [附录 D：原文勘误](#附录-d原文勘误)

---

## 术语表

| 词 | 一句人话 |
|---|---|
| **Channel** | 聊天平台适配器：收平台事件、发平台消息。 |
| **Provider** | 大模型客户端：把统一格式翻译成各家 API 请求。 |
| **Tool** | Agent 的手脚。模型只能"请求调用"，执行的是 `ToolRegistry`。 |
| **Session** | 一段连续对话，键是 `{channel}:{chat_id}`。 |
| **Turn** | 一条用户消息引发的完整处理过程。 |
| **MessageBus** | 两个 `asyncio.Queue`：`inbound` 进、`outbound` 出。 |
| **上下文治理** | 保证发给模型的请求不超窗口预算的裁剪/摘要机制。 |

![Nanobot 架构漫画](../../../comics/02-nanobot-architecture.png)

## 3.1 整体架构概览

### 类比：一家只有一条传送带的小馆子

- **接单窗口**（Channel）有好几个：堂食、外卖、电话。它们只做一件事——把单子写成统一格式递进厨房。
- **传送带**（MessageBus）两条：一条往里送单子，一条往外送菜。
- **值班经理**（AgentLoop）决定单子归哪桌（会话）、要不要先清台（压缩上下文）、是不是一句指令（`/new`）、做完记账（持久化）、怎么上菜（回复）。**他不做菜。**
- **大厨**（AgentRunner）拿着"这一单做什么"的说明书（`AgentRunSpec`）干活：问大脑（Provider）→ 需要就伸手拿工具（Tool）→ 再问，直到大脑说"好了"。
- **供应商**（Provider）和**工具箱**（Tool）是外部设备，插上就能用；换供应商不用改经理和大厨的代码。

记住两句话：**经理不掌勺，大厨不管账**。

### 真正的架构图（v0.3.5）

```
              ┌─────────── 三个外设（可插拔）───────────┐
              │ [Channel ×17] [Provider ×N] [Tool ×29] │
              └──┬───────────────┬───────────────┬─────┘
                 │ 注入           │ 注入           │ 注入
  Channel ──publish_inbound──► (bus.inbound) ──► AgentLoop ──► AgentRunner
      ▲                                             │              │
      │                                             │              ├─ provider.chat
      └──send── (bus.outbound) ◄──publish_outbound──┘              └─ tools.execute
```

七个锚点：

| 环节 | 位置 |
|---|---|
| 渠道收消息 | `nanobot/channels/base.py:255`（`_handle_message`） |
| 入队 | `nanobot/bus/queue.py:37`（`publish_inbound`） |
| 出队 + 派发 | `nanobot/agent/loop.py:1261`（`run`）、`:1391`（`_dispatch`） |
| 回合编排 | `nanobot/agent/loop.py:1594`（`_process_message`） |
| 模型循环 | `nanobot/agent/runner.py:386`（`_run_core`） |
| 工具执行 | `nanobot/agent/tools/registry.py:187`（`execute`） |
| 出队 + 发送 | `nanobot/channels/manager.py:767`（`_dispatch_outbound_loop`） |

### 为什么"五层架构"会误导

旧文档画的是 UI → Gateway → Core Agent → Provider → Tool 五层。四个具体问题：

**① 把平行协作者画成了上下层。** `AgentRunner` 同时持有 `spec.runtime.provider` 和 `spec.tools`（`nanobot/agent/runner.py:411`、`:512`），两者同级、互不调用。画成上下层会让人以为"Provider 之下是 Tool"。

**② 把管道和管道管理员混成一层。** 旧图写"ChannelManager + MessageBus"。但 `MessageBus` 是被注入到几乎每个组件的**数据管道**（`nanobot/bus/queue.py:19`），`ChannelManager` 是**消费出站队列并分发**的循环（`nanobot/channels/manager.py:82`、`:767`）。一个是队列，一个是循环。

**③ 暗示"改下层不影响上层"，但真实方向是接口注入。** `AgentLoop.__init__` 接了四十来个参数，`bus`/`provider`/`workspace`/`tool_registry`/`cron_service`/`session_manager` 都是外面造好传进来的（`nanobot/agent/loop.py:263` 起）。决定系统长什么样的地方不在 Agent 层，而在**组合根** `nanobot/cli/gateway_runtime.py`。

**④ 掩盖了真正的骨架——七阶段流水线。** `_process_message` 不是"消费 + 处理"两步：

```python
# nanobot/agent/loop.py:1679 一个回合的七个阶段，顺序硬编码
await self._run_turn_stage(ctx, "restore", self._restore_turn)     # 恢复 checkpoint / 附件引用
await self._run_turn_stage(ctx, "compact", self._compact_session)  # 空闲压缩：换入摘要
if await self._run_turn_stage(ctx, "command", self._dispatch_command):  # 斜杠命令短路
    return ctx.outbound
await self._run_turn_stage(ctx, "build", self._build_turn)         # 定 runtime / 取 history / 备料
await self._run_turn_stage(ctx, "run", self._run_turn)             # 交给 AgentRunner
await self._run_turn_stage(ctx, "save", self._persist_turn)        # 落 session + summary checkpoint
await self._run_turn_stage(ctx, "respond", self._prepare_outbound) # 造 OutboundMessage
```

---

## 3.2 四大核心模块详解

### 3.2.1 AgentLoop —— 回合编排器（不是"消息消费者"）

**一句话**：把"一条进来的消息"变成一个"回合"，收尾后交回渠道。它管**会话、阶段、持久化、回复**；不管 prompt 细节、不管模型循环、不执行工具。

```python
# nanobot/agent/loop.py:196 / __init__ :263（节选，其余 30 余个为可选注入参数）
def __init__(self, bus: MessageBus, provider: LLMProvider, workspace: Path,
             model: str | None = None, max_iterations: int | None = None, ...):
    ...
    self.context = ContextBuilder(workspace, timezone=timezone, ...)  # :374 自己 new，不是注入
    self.sessions = session_manager or SessionManager(workspace)      # :375
    self.tools = tool_registry if tool_registry is not None else ToolRegistry()  # :383
    self.runner = AgentRunner()   # :385 只造一次，全程复用
```

**消费循环为什么带 1 秒超时**：

```python
# nanobot/agent/loop.py:1261
while self._running:
    try:
        msg = await asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)  # :1269
    except asyncio.TimeoutError:
        self._check_expired_sessions_if_due()   # 空闲时顺手检查哪些会话该压缩
        continue
    ...
    task = asyncio.create_task(self._dispatch(msg))   # :1376 主循环继续收
```

这个超时把**空闲时间**利用起来做后台清理（`AutoCompact` 的过期扫描）。写成无限的 `await queue.get()` 就什么都做不了。

**并发控制**：

```python
# nanobot/agent/loop.py:1418
async with lock, gate:
```

- `lock` 是**会话锁**（`_get_session_lock`，`nanobot/agent/loop.py:2382`），存在 `WeakValueDictionary`（`:403`）：同会话串行，跨会话并行。
- `gate` 是**全局闸门**，但默认根本不存在（`nanobot/agent/loop.py:428`）：

```python
# NANOBOT_MAX_CONCURRENT_REQUESTS: unset or <=0 means unlimited.
_max = int(os.environ.get("NANOBOT_MAX_CONCURRENT_REQUESTS", "0"))
self._concurrency_gate = asyncio.Semaphore(_max) if _max > 0 else None
```

使用处 `gate = self._concurrency_gate or nullcontext()`（`:1412`）。

> ⚠️ **旧文档最大的硬伤**：它写 `self._concurrency_gate = asyncio.Semaphore(3)`、"默认最多 3 个并发"。v0.3.5 没有硬编码的 3——不设环境变量就是**不限并发**。

**三个阶段多讲两句**：`build`（`:1865`）备料（定 `runtime`、跑 `auto_compact.prepare_session`、取 `session.get_history()`、把本轮输入提前持久化 `:1960`）；`run`（`:1974`）把料交给 Runner 再回收结果字段；`save`（`:2014`）记账（修空回复、算延迟、调 `_save_turn`、保存 session、发落库事件）。

### 3.2.2 AgentRunner —— 模型循环（唯一"思考"的地方）

**一句话**：拿到 `AgentRunSpec`，反复"请求模型 → 执行工具 → 回灌结果"，直到模型不再请求工具。

```python
# nanobot/agent/runner.py:141
class AgentRunner:
    """Run a tool-capable LLM loop without product-layer concerns."""

    def __init__(self) -> None:
        self.context_governor = ContextGovernor()   # 整个对象只有这一个字段
```

注意这个 `__init__`：**没有 provider、没有 tools、没有 workspace、没有 session**。它是**无状态协作者**——输入全从 `spec` 拿，输出塞进 `AgentRunResult` 返回。

```python
# nanobot/agent/runner.py:89（节选）
@dataclass(slots=True)
class AgentRunSpec:
    initial_messages: list[dict[str, Any]] | None   # 与 transcript_input 二选一
    tools: ToolRegistry
    runtime: LLMRuntime                             # provider + model + 窗口 + 生成参数
    max_iterations: int
    max_tool_result_chars: int
    transcript_input: TranscriptInput | None = None
    hook: AgentHook | None = None
```

同一份 Spec 类型被两处使用，这是理解架构的关键：

| 调用方 | 传什么 | 位置 |
|---|---|---|
| 主 Agent（有会话、要压缩） | `transcript_input` + `consolidate_history` | `nanobot/agent/loop.py:1170` |
| 子 Agent（一次性任务） | `initial_messages=[system, user]` | `nanobot/agent/subagent.py:427` |

Runner 会校验"两者只能给一个"（`nanobot/agent/runner.py:369`）。**这就是无状态的回报**：主 Agent 与子 Agent 共用一个循环实现，只换输入形态。

```python
# nanobot/agent/runner.py:386 _run_core（结构示意）
for iteration in range(spec.max_iterations):              # :435 默认 200
    await hook.before_iteration(context)                  # :441
    response, raw_usage = await self._request_model(...)  # :448 → provider.chat_stream_with_retry
    if response.should_execute_tools:                     # :482
        messages.append(assistant_message)                # :497
        await self._emit_checkpoint(spec, {...})          # :498 phase=awaiting_tools
        results, _ = await execute_tool_calls(            # :512
            spec.tools, response.tool_calls, concurrent=spec.concurrent_tools, ...)
        for tool_call, result in zip(response.tool_calls, results):
            messages.append({"role": "tool", "tool_call_id": tool_call.id,
                "content": self.context_governor.normalize_tool_result(  # :537
                    governance_config, tool_call.id, tool_call.name, result)})
        continue                                          # :579 带着工具结果再问一次
    break                                                 # :789 模型不再请求工具
else:
    stop_reason = "max_iterations"                        # :791
```

三点：① 工具结果不是直接塞回 messages，先过 `normalize_tool_result`（`:537`），这是治理入口；② 每轮都发 checkpoint（`:498`、`:554`、`:767`），这是崩溃恢复与 WebUI 重连的基础；③ `for ... else` 是真语法，跑完 200 次没 `break` 才进 `else`（`:790`）。

### 3.2.3 ContextBuilder + 上下文治理 —— 决定"给模型看什么"

MemoryStore 细节交给第 07 章，这里只讲**上下文怎么拼、怎么裁**——旧文档错得最多的地方。

```python
# nanobot/agent/context.py:89（真实签名，注意没有 config 参数）
class ContextBuilder:
    BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md"]

    def __init__(self, workspace: Path, timezone: str | None = None,
                 disabled_skills: list[str] | None = None):     # :95
        self.memory = MemoryStore(workspace)      # 组合，不是注入
        self.skills = SkillsLoader(workspace, disabled_skills=...)
```

旧文档写的是 `__init__(self, config, memory_store, skill_loader)`——**完全不是这个签名**。

`build_system_prompt`（`:101`）的真实顺序：

```python
parts = [self._get_identity(channel=channel, workspace=root)]   # :111 身份 + 平台策略
bootstrap = self._load_bootstrap_files(root)                    # :113 AGENTS/SOUL/USER.md
parts.append(render_template("agent/tool_contract.md"))         # :117 工具契约
...                                                             # :121 跨项目时才加 "# Current Project"
memory = self.memory.read_memory()                              # :128 MEMORY.md（模板内容跳过）
active_skills = self.skills.get_always_skills()                 # :132 常驻技能全文
skills_summary = self.skills.build_skills_summary(...)          # :138 其余技能只给摘要
...                                                             # :145 归档摘要
return "\n\n---\n\n".join(parts)                                # :152
```

即 **Identity → Bootstrap 文件 → 工具契约 →（当前项目）→ Memory → 常驻技能 → 技能清单 → 归档摘要**。旧文档的"Identity → Bootstrap → Memory → Skills"少了工具契约、技能清单、归档摘要——后两段正是"技能渐进披露"和"压缩后可续聊"的关键。

`build_messages`（`:231`）只是兼容包装：它调 `build_transcript`（`:276`，拼 `[system] + history + current_message`），再把相邻同角色消息合并。单条用户消息由 `build_current_message`（`:310`）造，运行时上下文塞进 `_meta` 而不污染正文。

#### 上下文治理：预算与工具结果截断

`ContextGovernor`（`nanobot/agent/context_governance.py:207`）三句话：**先算预算、再裁历史、最后裁工具结果；但永远只改"给模型的副本"，不改持久化 transcript。**

（1）预算：

```python
# nanobot/agent/context_governance.py:693
budget = config.context_window_tokens - max_output - SNIP_SAFETY_BUFFER   # :67 → 1024
return budget if budget > 0 else 0
```

即 **输入预算 = 窗口(默认 200000) − 最大输出(默认 8192) − 1KB 安全垫**。

（2）超预算两条路。路 A **裁历史**：`snip_history`（`:951`）保留所有 `system`，从最新往前塞，塞不下就断；再用 `_legal_history_tail`（`:1001`）保证尾巴从一条 `user` 开始（否则会留孤立的 `tool` 结果）。路 B **换摘要**：`_compact_request_history`（`:521`）把"模型已接受过的历史 H"换成 LLM 摘要，只保留未发送的增量 Δ：

```python
# nanobot/agent/context_governance.py:542
summary = await compaction.consolidate_history(deepcopy(consolidation_prefix), compaction.active_summary)
prepared = self.prepare_messages_for_model(state.config, [
    *self._summary_transcript(compaction, summary),           # 摘要 + 稳定前缀
    {"role": "user", "content": SUMMARY_CONTINUATION_TEXT},    # "继续"口令
    *delta_messages,                                           # 没发出去的增量
])
```

最后 `ensure_request_fits`（`:358`）再验一遍，还超就抛 `ContextWindowExceededError`（`:76`），不发必然失败的请求。

（3）工具结果不砍，而是**落盘 + 给引用**：

```python
# nanobot/agent/context_governance.py:709
def normalize_tool_result(config, tool_call_id, tool_name, result):
    if tool_name in TOOL_RESULT_OFFLOAD_EXEMPT_TOOLS:   # :69 目前只有 {"read_file"}
        return result
    content = maybe_persist_tool_result(                # nanobot/utils/helpers.py:580
        config.workspace, config.session_key, call_id, text,
        max_chars=config.max_tool_result_chars)         # 配置项，默认 16_000（config/schema.py:131）
```

`maybe_persist_tool_result` 把超长内容写进 workspace 文件，返回"路径 + 预览 + 原始大小"的引用；只有**没有 workspace** 时才硬截断（`truncate_text`，`nanobot/utils/helpers.py:402`）。好处是模型还能用 `read_file` 把内容读回来——所以 `read_file` 在豁免名单里。硬截断的信息是真丢了。

### 3.2.4 MessageBus —— 双队列，不止"一进一出"

```python
# nanobot/bus/queue.py:19
class MessageBus:
    def __init__(self):
        self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()    # :32
        self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue()  # :33
        self._handlers: list[EventHandler] = []
    async def publish_inbound(self, msg): await self.inbound.put(msg)    # :37
    async def consume_inbound(self): return await self.inbound.get()     # :41
    async def publish_outbound(self, msg): await self.outbound.put(msg)  # :45
    async def consume_outbound(self): return await self.outbound.get()   # :67
```

两个易忽略的点：① 出站队列里不只文本，`publish_event`（`:49`）把 `AgentEvent`（流式增量、工具进度、推理片段）也包装入队；② 消费出站队列的是 `ChannelManager` 而不是每个 Channel（`nanobot/channels/manager.py:767`），重试、限流、流式合并（`:933`）只写一遍。

```python
# nanobot/bus/events.py:25
@dataclass
class InboundMessage:
    channel: str; sender_id: str; chat_id: str; content: str      # 真实字段名
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    session_key_override: str | None = None
    require_existing_session: bool = False
    @property
    def session_key(self) -> str:                                  # :40
        return self.session_key_override or f"{self.channel}:{self.chat_id}"
```

旧文档的 `InboundMessage{session_id, user_id, text, ...}` **五个字段名错三个**：没有 `session_id`（是 `session_key` 属性，由 `channel:chat_id` 算出）、没有 `user_id`（是 `sender_id`）、没有 `text`（是 `content`）。

---

## 3.3 数据流图

```
① 平台事件
   channels/telegram/runtime.py:603   TelegramChannel.start（长连接）
   channels/telegram/runtime.py:1059  TelegramChannel.send（回程）
        ▼
② 翻译成统一格式
   channels/base.py:255  BaseChannel._handle_message(...)
        ├─ :275  is_allowed(permission_id)   ← allowFrom / pairing 授权
        └─ :310  构造 InboundMessage
        ▼
③ 入队        bus/queue.py:37   publish_inbound
        ▼
④ 收消息      loop.py:1269  wait_for(consume_inbound(), timeout=1.0)
              loop.py:1376  create_task(self._dispatch(msg))
        ▼
⑤ 串行        loop.py:1391  _dispatch → :1411 _get_session_lock → :1418 async with lock, gate
        ▼
⑥ 七阶段      loop.py:1594  _process_message
        ├─ :1679 restore  ├─ :1680 compact  ├─ :1681 command（命中则 return）
        ├─ :1683 build（:1865） ├─ :1684 run（:1974） ├─ :1685 save（:2014） └─ :1686 respond（:2063）
        ▼
⑦ 交给 Runner  loop.py:1170  await self.runner.run(AgentRunSpec(...))
        ▼
⑧ 模型/工具循环
   runner.py:435  for iteration in range(spec.max_iterations)     # 默认 200
   runner.py:448  _request_model → provider.chat_stream_with_retry
   runner.py:482  should_execute_tools?
   runner.py:512  execute_tool_calls(...) → tools/registry.py:187 ToolRegistry.execute
   runner.py:537  normalize_tool_result（超长落盘 → 引用）
   runner.py:579  continue（再问一次）  /  :789 break（收工）
        │  AgentRunResult(final_content, messages, stop_reason, usage, ...)
        ▼
⑨ 持久化      loop.py:2014 _persist_turn → :2140 _save_turn（session.messages.append）
              loop.py:2054 self.sessions.save(session)  → JSONL 落盘
        ▼
⑩ 造回复      loop.py:2063 _prepare_outbound → loop.py:1435 delivery.complete → queue.py:45 publish_outbound
        ▼
⑪ 分发        channels/manager.py:767 _dispatch_outbound_loop → :998 _send_with_retry → channel.send
```

一次真实回合里，第 ⑧ 步可能循环 1 次（直接回答）也可能 20 次（边查边写）；第 ⑨ 步永远发生，哪怕模型只回了一个字。

---

## 3.4 模块间的协作关系

```
AgentLoop
 ├─ MessageBus(:307) ContextBuilder(:374) SessionManager(:375)
 ├─ ToolRegistry(:383) SubagentManager(:386) Consolidator(:433) AutoCompact(:443)
 └─ AgentRunner(:385，无状态，一次构造全程复用)

AgentRunner（只有 ContextGovernor）
 └─ 每次 run 从 AgentRunSpec 取 tools / runtime / 上限 / hook

AgentRunner ──► LLMProvider（抽象，providers/base.py:623）
AgentRunner ──► ToolRegistry ──► Tool（抽象，agent/tools/base.py:159）
ToolRegistry ──► MCPToolWrapper（远程工具伪装成本地 Tool，agent/tools/mcp.py:595）
ChannelManager ──► MessageBus（消费 outbound，manager.py:782）+ 17 个 BaseChannel 子类
```

关键点：**Loop 与 Runner 是"调用一次、拿回结果"，不共享可变状态**。交互只通过 `AgentRunSpec`（进）和 `AgentRunResult`（出）。

启动顺序（组合根 `nanobot/cli/gateway_runtime.py` 的 `_run_gateway`，`:340`）：

```python
bus = MessageBus()                                      # :411
provider_snapshot = build_provider_snapshot(config)     # :437 按 backend 造 Provider
session_manager = SessionManager(config.workspace_path) # :441
cron = CronService(cron_store_path)                     # :464
tools = ToolRegistry()                                  # :472
mcp_provider = MCPProvider.from_config(config, tools)   # :473 MCP 工具进同一个 registry
agent = AgentLoop.from_config(config, bus, tool_registry=tools, ...)  # :482
channels = ChannelManager(config, bus, ...)             # :715
# :917 _run_agent(): connect()（:919）→ agent.run()（:920）→ aclose()（:922）
# :941 create_task(channels.start_all())
```

对比旧文档的 12 步清单，三处事实差异：① 配置是 JSON（`~/.nanobot/config.json`），不是 `nanobot.yml`；② 没有"注册内置工具"这一步——`_register_default_tools`（`nanobot/agent/loop.py:615`）交给 `ToolLoader` 扫描包自动发现（`nanobot/agent/tools/loader.py:36`）；③ MCP 生命周期不属于 Loop，`connect()/aclose()` 包着 `agent.run()`。

---

## 3.5 关键设计模式

**① 生产者—消费者（双队列）**

```python
# 生产：channels/base.py:321        消费：agent/loop.py:1269
await self.bus.publish_inbound(msg)
msg = await asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)
```

渠道不知道 Agent 存在，Agent 不知道消息从哪来。新增渠道 = 新增子包，零改动核心。

**② 注册表（不是"类字典"）**

```python
# nanobot/providers/registry.py:146 —— 是元数据元组，不是 {name: Class}
PROVIDERS: tuple[ProviderSpec, ...] = (...)
```

决定实现的是 `ProviderSpec.backend`（`registry.py:55`），由工厂分派：`factory.py:150` 的 `if backend == "openai_codex" / "anthropic" / ... else OpenAICompatProvider`。所以"新增供应商"通常只是**加一条 ProviderSpec**，连类都不用写。工具侧是标准注册表，并且顺手让缓存失效：`registry.py:30` 的 `register` 里 `self._cached_definitions = None`。

**③ 适配器（Channel）**：`BaseChannel(ABC)`（`channels/base.py:21`）只有 `start`（`:74`）/`send`（`:91`）两个抽象方法，授权、配对、入队这些公共逻辑写在 `_handle_message`（`:255`）里一次。

**④ 包装器**：`MCPToolWrapper`（`agent/tools/mcp.py:595`）把远程工具包成本地 `Tool`，工具名 `mcp_<server>_<tool>`，对 Runner 与内置工具无区别。

**⑤ 作用域过滤**：

```python
# nanobot/agent/tools/loader.py:100
if scope not in getattr(tool_cls, "_scopes", {"core"}):
    continue
```

`Tool` 可声明 `_scopes`（`agent/tools/base.py:208`，默认 `{"core"}`）。子 Agent 用 `scope="subagent"` 加载，于是拿不到 `spawn`/`message`/`cron`——**不需要 clone 一份 registry 再 remove**。

**⑥ 无状态协作者 + Spec**：见 3.2.2。收益：主 Agent、子 Agent、SDK、TUI 共用一个循环；单测造一个 Spec 就能跑。

**⑦ 配置驱动 + 显式组合根**：装配只有一句 `AgentLoop.from_config(...)`，三处出现：`cli/gateway_runtime.py:482`、`nanobot/nanobot.py:136`（SDK）、`cli/commands.py:391`（`nanobot agent`）。

---

## 3.6 架构对比

请用 v0.3.5 的真实规模（约 **99,398 行 SLOC / 329 个非测试 `.py`**），不要再说"4000 行"。

| 维度 | nanobot v0.3.5 | LangChain / LangGraph |
|---|---|---|
| 核心抽象 | MessageBus + AgentLoop + AgentRunner + Spec | Chain / Runnable / Graph / Tool |
| 编排方式 | 固定七阶段 + 模型自主决定工具循环 | 显式图编排或 AgentExecutor |
| 消息传递 | 进程内双 `asyncio.Queue` + 事件订阅 | Callback / 事件流 |
| 工具注册 | `Tool` 子类 + `pkgutil` 自动发现 + `scope` 过滤 | `BaseTool` + Toolkit，需显式绑定 |
| 配置 | JSON（pydantic 校验） | 代码里构造对象为主 |
| 记忆 | `MEMORY.md` + `history.jsonl` + Dream | Memory 组件 / VectorStore |
| 抽象层数 | 少：一条链路 + 三个接口 | 多：可组合的 Runnable 协议栈 |

| 维度 | nanobot | CrewAI |
|---|---|---|
| Agent 模型 | 一个主 Agent + N 个后台子 Agent | 多角色 Agent 团队 |
| 任务分配 | 主 Agent 用 `spawn` 自行决定 | 用户预定义角色 + Task |
| 协作方式 | 子 Agent 结果作为注入消息回主会话 | Agent 间显式传递 |
| 适合场景 | 个人/小团队常驻助手 | 流程化多角色模拟 |

---

## 3.7 面试高频题

### Q1：请描述 nanobot 的架构

> 一条调用链加三个外设，不是分层。链路是 `Channel → MessageBus → AgentLoop → AgentRunner → Provider/Tools → MessageBus → Channel`。
>
> `AgentLoop`（`agent/loop.py:196`）负责回合：会话锁、七阶段、持久化、回复。`AgentRunner`（`agent/runner.py:141`）负责模型循环，无状态，输入全来自 `AgentRunSpec`。三个外设：渠道 17 个子包、Provider 靠 `ProviderSpec.backend` 分派、工具 29 个文件按 `scope` 过滤；装配集中在 `cli/gateway_runtime.py`。

### Q2：AgentLoop 和 AgentRunner 的边界在哪？

> 分界线是**会话**。Loop 知道会话、历史、落库、回复；Runner 只知道这一次运行要什么（messages/tools/runtime/上限）。
>
> 硬证据：`AgentRunner.__init__` 只初始化了一个 `ContextGovernor`（`runner.py:144`）。所以主 Agent（`loop.py:1170` 传 `transcript_input`）和子 Agent（`subagent.py:427` 传 `initial_messages`）复用同一个循环。收益：可测试、可重放、可复用。

### Q3：为什么用双队列而不是单队列？

> 收和发的节奏不同、消费者也不同：入站只有 AgentLoop，出站是 ChannelManager（`manager.py:767`）——它要处理重试、限流、流式合并。合成一个队列，渠道得自己区分"这条给谁"，还要和 Agent 抢消费。而且出站不只跑文本，`publish_event`（`queue.py:49`）把 `AgentEvent` 也塞进去。

### Q4：nanobot 怎么做并发控制？

> 两层，只有第一层默认生效。会话锁按 `{channel}:{chat_id}` 加 `asyncio.Lock`，锁对象放 `WeakValueDictionary`（`loop.py:403`），没人用了自动回收。全局闸门在 `loop.py:1418` 的 `async with lock, gate:`，但 `gate` 只有设了 `NANOBOT_MAX_CONCURRENT_REQUESTS`（>0）才存在（`loop.py:428-432`），否则是 `nullcontext()`——默认**不限并发**。

### Q5：上下文超了怎么办？

> 三层。事前 `normalize_tool_result`（`context_governance.py:709`）把超长工具结果落盘成文件只留引用，`read_file` 豁免；事中 `prepare_request`（`:593`）先算预算（窗口 − 最大输出 − 1024），超了 `snip_history` 裁尾或 `_compact_request_history` 换摘要 + "继续"口令 + 未发送增量；事后 `ensure_request_fits`（`:358`）再验，还超就抛 `ContextWindowExceededError`。前提是这些都只改"给模型的副本"。

---

## 练习题与思考题

**练习 1（读代码）**：从 `nanobot/agent/loop.py:1594` 读 `_process_message`，写出七阶段顺序；`/new` 命中时哪几个阶段被跳过？
<details><summary>提示</summary>`:1681` 命中命令就 `return ctx.outbound`，build/run/save/respond 都不执行；命令自己在 `_dispatch_command` 里落库（`:1847`）。</details>

**练习 2（辨析）**："当前第几轮迭代"存在哪？为什么不能放在 `self` 上？
<details><summary>提示</summary>`runner.py:435` 的局部变量 `iteration`，再传给 `AgentHookContext`（`:436`）。放 `self` 上会让并发 run 互相污染——同一个 Loop 同时服务多个会话。</details>

**练习 3（动手）**：把 `agents.defaults.maxToolResultChars` 改成 200，让 Agent 读一个大文件，观察 `tool` 消息内容与 workspace 下多出来的文件。
<details><summary>提示</summary>入口是 `maybe_persist_tool_result`（`nanobot/utils/helpers.py:580`）。</details>

---

## 3.8 本章总结

```
┌──────────────────────────────────────────────────────────────┐
│  nanobot v0.3.5：一条调用链 + 三个外设                          │
│                                                              │
│  调用链                                                       │
│  ├─ Channel        收平台事件 → InboundMessage                │
│  ├─ MessageBus     双 asyncio.Queue（inbound / outbound）      │
│  ├─ AgentLoop      回合编排：七阶段 + 会话锁 + 持久化           │
│  ├─ AgentRunner    模型循环：请求 → 工具 → 回灌 → 结束判定       │
│  └─ 回到 Channel   ChannelManager 统一消费 outbound 并发送      │
│                                                              │
│  三个外设（接口 + 注册表 + 配置注入）                           │
│  ├─ Channel   17 个子包，BaseChannel(ABC)                     │
│  ├─ Provider  ProviderSpec.backend → factory 分派             │
│  └─ Tool      29 个文件，Tool(ABC) + 自动发现 + scope 过滤      │
│                                                              │
│  两个核心边界                                                 │
│  ├─ 回合（Loop）vs 模型循环（Runner）：靠 Spec 解耦             │
│  └─ 持久 transcript vs 给模型的副本：治理只改后者               │
└──────────────────────────────────────────────────────────────┘
```

三句话记住本章：

1. **"五层"把平行协作者画成了上下层**，改成"一条链 + 三个插座"才和源码对得上。
2. **`AgentLoop` 管回合，`AgentRunner` 管模型循环**，两者只通过 `AgentRunSpec` / `AgentRunResult` 交互，Runner 无状态。
3. **上下文治理只改"给模型的副本"**：工具结果超长落盘给引用，历史超预算裁尾或换摘要。

### 下一步读什么

本章讲"谁在哪、为什么这么分工"。下一章把同样的链路**逐文件、逐类、逐签名**读一遍：

➡️ [04 - 源码逐行解读](../04-source-code-walkthrough/README.md)

等不及的话先打开三个文件：`nanobot/agent/loop.py:1594` → `nanobot/agent/runner.py:386` → `nanobot/agent/context_governance.py:593`。

---

## 附录 A：与旧版文档的差异

> 这节讲"旧版本本来是对的、后来演进变了"的内容。旧文档本身的硬错误见附录 D。

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 五层架构 | 一条调用链 + 三个外设，协作者平行注入 | `agent/loop.py:1594`、`agent/runner.py:386` | 换成新模型 |
| `Semaphore(3)` 硬编码并发闸门 | 默认 `None`，由 `NANOBOT_MAX_CONCURRENT_REQUESTS` 控制 | `agent/loop.py:428`-`432`、`:1412` | 改为"默认不限并发" |
| 主 40 次 / 子 15 次迭代 | 默认 200，子 Agent 继承同一值 | `config/schema.py:129`、`agent/subagent.py:140` | 数字全部更新 |
| `_save_turn` 写 `HISTORY.md` | 写 `session.messages`，由 `SessionManager` 落 JSONL；`HISTORY.md` 只是迁移源 | `agent/loop.py:2140`、`session/manager.py:548`、`agent/memory.py:106` | 记忆链路看第 07 章 |
| `save_memory` 虚拟工具 | 不存在；记忆靠 `Consolidator` + `Dream` | `agent/memory.py:1072`、`agent/tools/registry.py:19` | 删掉整段 |
| 约 4000 行 | 约 99,398 行 SLOC / 329 个非测试 `.py` | `git ls-tree -r v0.3.5` | 引用真实规模 |
| `nanobot.yml`（YAML） | `~/.nanobot/config.json`（JSON，camelCase 落盘） | `config/schema.py:422` | 全部改 JSON |
| MCP 工具由 Loop 连接管理 | 组合根创建 `MCPProvider`，`connect()/aclose()` 包住 `agent.run()` | `agent/tools/mcp.py:1346`、`cli/gateway_runtime.py:473`、`:917` | 生命周期归组合根 |
| Prompt Cache 只靠"system 放最前" | 还有工具定义**稳定排序**（内置在前、MCP 在后，各自按名排序并缓存） | `agent/tools/registry.py:86` | 补上缓存友好排序 |

---

## 附录 B：进阶理解（为什么这么设计）

**1. 为什么把"回合"和"模型循环"拆成两个类？** 合成一个类会同时背两种变化原因：产品需求（新斜杠命令、WebUI 事件、交付方式）和 AI 需求（多轮工具、长度恢复、空回复重试）。拆开后 Runner 被三处复用，产品改动只落在 Loop 的七阶段里。

**2. 为什么 Runner 要"无状态 + 纯数据 Spec"？** 因为要**重放与恢复**。Spec 和 RunResult 都是纯数据，可序列化、可落库反查"这一轮发了什么、工具返回了什么"。Runner 一旦持有 session 引用，这些做不了，并发 run 还会互相污染。

**3. 为什么上下文治理坚持"只改副本"？** 因为**持久化数据是审计记录**。`snip_history`、`normalize_tool_result`、`_compact_request_history` 全部返回新对象，transcript 本身不动。代价是内存占用高一些，收益是"用户看到的历史"永远是真话。

**4. 为什么工具结果要"落盘 + 引用"而不是截断？** 截断是**不可逆的信息损失**。落盘之后模型还能用 `read_file` 读回来（所以它在豁免名单里，`context_governance.py:69`）。代价是 workspace 里会产生文件，需要清理策略（`helpers.py:598` 的 `_cleanup_tool_result_buckets`）。

**5. 为什么三个外设只有接口、没有继承树？** 对比 LangChain 的 `BaseTool → Tool/StructuredTool/Toolkit`，nanobot 的选择是"一个抽象 + 一个注册表 + 一个 scope"。代价是扩展点少，收益是**读源码时不用在继承树里跳**。

**6. 为什么入站消费要 1 秒超时？** 因为需要"空闲时间"。`AutoCompact.check_expired`（`agent/autocompact.py:68`）要压缩久未活动的会话，而无限阻塞的消费没有触发点。固定超时 + `continue` 是最便宜的定时器。

---

## 附录 C：Python 语法加油站

### C.1 `async` / `await` + `create_task` / `wait_for`

```python
import asyncio

async def handle(msg):          # async def 定义协程：调用只得到协程对象，不执行
    await asyncio.sleep(0.1)    # await 把控制权交还事件循环
    return msg.upper()

async def main():
    task = asyncio.create_task(handle("hi"))        # 立刻排队，不阻塞当前协程
    try:
        await asyncio.wait_for(task, timeout=1.0)   # 给"等待"加超时
    except asyncio.TimeoutError:
        print("没等到")
```

用到：`agent/loop.py:1269`（`wait_for`）、`:1376`（`create_task`）。读法：`async def` = 可被打断的任务；`create_task` = 开始并发；`wait_for` = 超时兜底。

### C.2 `@dataclass` / `@dataclass(slots=True)`

```python
from dataclasses import dataclass, field

@dataclass
class InboundMessage:                 # 自动生成 __init__ / __repr__ / __eq__
    channel: str
    media: list[str] = field(default_factory=list)  # 可变默认值必须用 field

@dataclass(slots=True)
class AgentRunSpec:                   # slots=True：不能动态加属性，省内存、防打错字
    max_iterations: int
```

用到：`bus/events.py:24`、`agent/runner.py:88`、`agent/loop.py:132`、`agent/context_governance.py:112`。读法：dataclass 就是"带类型标注的字典"，先扫字段名与默认值。

### C.3 `@property`

```python
class InboundMessage:
    def __init__(self, channel, chat_id):
        self.channel, self.chat_id = channel, chat_id

    @property
    def session_key(self):            # 像读属性一样调用：msg.session_key
        return f"{self.channel}:{self.chat_id}"
```

用到：`bus/events.py:39`、`agent/loop.py:208`（`tool_names`）、`agent/tools/base.py:171`（抽象 `Tool.name`）。读法：`@property` 是"算出来的字段"，别找赋值语句。

### C.4 `ABC` + `@abstractmethod`

```python
from abc import ABC, abstractmethod

class BaseChannel(ABC):
    @abstractmethod
    async def start(self) -> None: ...   # 子类必须实现
    async def _handle_message(self): ... # 普通方法：公共逻辑写一次

class TelegramChannel(BaseChannel):
    async def start(self): ...           # 不实现则实例化直接 TypeError
```

用到：`channels/base.py:21`、`agent/tools/base.py:159`、`providers/base.py:623`。读法：ABC 是接口说明书，看抽象方法就知道最少要实现什么。

### C.5 `async with` 与"可能为 None 的上下文管理器"

```python
import asyncio
from contextlib import nullcontext

lock, gate = asyncio.Lock(), None

async def work():
    ctx = gate or nullcontext()   # 没有闸门时用空上下文，语法不变
    async with lock, ctx:         # 依次进入，逆序退出
        await do_something()
```

用到：`agent/loop.py:1412`、`:1418`。读法：`async with A, B` 是"依次进入"；`nullcontext()` 让"可能为 None"也能统一写法。

---

## 附录 D：原文勘误

> 下面这些错误**在任何版本都不成立**，不是版本演进，是记错了。

| # | 旧文档原文 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| D1 | 3.1「五层架构」，Layer 2 Provider 在 Layer 1 Tool 之上，箭头 `Provider → Tool` | Provider 与 Tool **没有任何调用关系**，都是 Runner 的平行依赖 | 一条调用链 + 三个平行外设 | `agent/runner.py:411`、`:512` |
| D2 | 3.2.1 / 4.2「并发闸门 `asyncio.Semaphore(3)`，默认最多 3 个并发」 | 代码里从来没有 `Semaphore(3)`；不设环境变量时 `gate` 是 `None` | 默认不限；由 `NANOBOT_MAX_CONCURRENT_REQUESTS` 控制 | `agent/loop.py:428`-`432`、`:1412` |
| D3 | 3.2.2「`_TOOL_RESULT_MAX_CHARS = 16000`，超过就 `result[:16000] + "...(truncated)"` | 全仓 `grep _TOOL_RESULT_MAX_CHARS` 无结果；截断逻辑在治理层不在 Runner | 配置项 `max_tool_result_chars`（默认 16000），超长**落盘 + 引用**，`read_file` 豁免 | `context_governance.py:709`、`:69`、`utils/helpers.py:580`、`config/schema.py:131` |
| D4 | 3.3 / 4.8「`InboundMessage { session_id, user_id, text, ... }`」 | 字段名全错，`session_key` 是算出来的属性不是字段 | `channel / sender_id / chat_id / content / timestamp / media / metadata` + `session_key` | `bus/events.py:25`、`:39` |
| D5 | 3.2.1 / 4.2「`_save_turn()` 把摘要追加到 `HISTORY.md`」 | 它只往 `session.messages` 追加条目并处理边界，不碰 `.md` | 会话消息落 JSONL；`HISTORY.md` 只在首次发现时迁移成 `history.jsonl` | `agent/loop.py:2140`、`session/manager.py:548`、`agent/memory.py:106` |
| D6 | 3.2.3 / 3.5「`save_memory` 虚拟工具 + `ToolRegistry.register_virtual`」 | `Tool` 没有"虚拟工具"概念，`ToolRegistry` 也没这个 API | 长期记忆由 `Consolidator` + `Dream` 维护，模型没有"存记忆"工具 | `agent/tools/registry.py:19`（全部 API）、`agent/memory.py:1072` |
| D7 | 4.2「每条消息创建一个新的 `AgentRunner` 实例」 | Runner 在 `AgentLoop.__init__` 里只创建一次 | 一个 Loop 一个 Runner，状态都在 Spec/RunResult 里 | `agent/loop.py:385` |
| D8 | 4.2「`_register_default_tools()` 手工注册 MessageTool / SpawnTool / save_memory」 | 真实实现是 `ToolLoader` 扫描包 + entry-point 插件 | 自动发现 + `scope` 过滤 | `agent/loop.py:615`、`agent/tools/loader.py:36`、`:92` |
| D9 | 4.7「`ToolRegistry.clone()` 后 `remove("message")` 给子 Agent」 | `ToolRegistry` 没有 `clone`，也没有 `remove`（只有 `unregister`） | 子 Agent 用 `scope="subagent"` 过滤，各自建 registry | `agent/tools/registry.py:19`-`:42`、`agent/tools/loader.py:100` |
| D10 | 4.5「`MemoryConsolidator` 类，阈值 50 条」 | 类名不存在；触发条件是"空闲 TTL + 有未归档消息"，没有 50 条常量 | `Consolidator` / `MemoryArchiver` / `AutoCompact`（TTL 触发） | `agent/memory.py:758`、`:1072`、`agent/autocompact.py:68` |
