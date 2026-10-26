# 10 - 子Agent与定时任务

> 🎯 **本章目标**：弄清 v0.3.5 的后台任务三件套——`spawn` 子 Agent、`cron` 定时任务、heartbeat 心跳——各自怎么启动、怎么回收结果、为什么不会递归失控。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

---

## 目录

- [10.1 为什么需要子Agent和定时任务](#101-为什么需要子agent和定时任务)
- [10.2 子Agent（SubAgent）系统](#102-子agentsubagent系统)
- [10.3 定时任务（Cron）系统](#103-定时任务cron系统)
- [10.4 Heartbeat 心跳服务](#104-heartbeat-心跳服务)
- [10.5 三者的协作关系](#105-三者的协作关系)
- [10.6 实战练习](#106-实战练习)
- [10.7 面试高频题](#107-面试高频题)
- [10.8 本章小结](#108-本章小结)
- [附录 A：与旧版文档的差异](#附录-a与旧版文档的差异)
- [附录 B：进阶理解](#附录-b进阶理解)
- [附录 C：Python 语法加油站](#附录-cpython-语法加油站)
- [附录 D：原文勘误](#附录-d原文勘误)

---

## 10.1 为什么需要子Agent和定时任务

主 Agent 是"回合制"的：用户发一条，它处理一条。但有三类需求它一个人扛不动：

```
场景 1  用户："详细调研 Rust 和 Go 的对比"
        → 可能要搜 20 个网页、跑 10 分钟；用户不想干等

场景 2  用户："每天早上 8 点给我一份新闻摘要"
        → 需要到点自己醒来，而不是等用户开口

场景 3  用户："每半小时看看有没有新邮件，有事再告诉我"
        → 周期唤醒 + 自己判断要不要打扰用户
```

对应三套机制：

| 需求 | 机制 | 一句话 |
|---|---|---|
| 耗时的独立任务 | **子 Agent** | 主 Agent 用 `spawn` 工具把活派出去，后台跑完再回来汇报 |
| 到点执行 | **Cron** | 会话里用 `cron` 工具建任务，落盘到 `jobs.json`，到点作为该会话的一轮输入执行 |
| 周期性自主检查 | **Heartbeat** | 一个受保护的系统 cron 任务，定时读 `HEARTBEAT.md`，有活才叫醒 Agent |

术语：**子 Agent（subagent）**＝由 `SubagentManager` 创建、跑在后台 Task 里的一个独立 Agent 回合；**cron job**＝一条落盘的定时任务；**系统任务（system job）**＝Dream/Heartbeat 这类由程序注册、不能被用户删除的 cron 任务。

---

## 10.2 子Agent（SubAgent）系统

### 10.2.1 SpawnTool：入口是一个叫 `spawn` 的工具

一句话：**v0.3.5 的工具名就是 `spawn`**（不是 `subagent`），模型看到的名字在 `nanobot/agent/tools/spawn.py:63`。

```python
# 来源：nanobot/agent/tools/spawn.py:24-47（节选）
@tool_parameters(tool_parameters_schema(
    task=StringSchema("The task for the subagent to complete"),
    label=StringSchema("Optional short label (for display)"),
    temperature=NumberSchema(..., minimum=0.0, maximum=2.0),
    wait=BooleanSchema("Wait for the subagent and return its result directly...", default=False),
    required=["task"],
))
class SpawnTool(Tool):
    @property
    def name(self) -> str: return "spawn"
```

四个参数：`task`（必填）、`label`（展示用短标签）、`temperature`（采样温度）、`wait`（是否阻塞等结果）。`wait` 是 v0.3.5 新增的"咨询模式"：

```python
# 来源：nanobot/agent/tools/spawn.py:96-107（节选）
method = self._manager.run_inline if wait else self._manager.spawn
return await method(
    task=task, runtime=request_ctx.runtime, label=label,
    origin_channel=origin_channel, origin_chat_id=origin_chat_id,
    session_key=session_key, origin_message_id=request_ctx.message_id,
    temperature=temperature, workspace_scope=current_workspace_scope(),
)
```

- `wait=false`（默认）→ `SubagentManager.spawn()`：`asyncio.create_task` 后台跑，主 Agent 立刻拿到 "started" 字符串继续对话。
- `wait=true` → `SubagentManager.run_inline()`：同步等结果，适合"我需要这个答案才能继续"的咨询。

### 10.2.2 SubagentManager：生命周期与并发上限

`SubagentManager`（`nanobot/agent/subagent.py:93`）是子 Agent 的大管家。构造参数里直接体现两个上限：

```python
# 来源：nanobot/agent/subagent.py:140-150（节选）
self.max_iterations = max_iterations if max_iterations is not None else defaults.max_tool_iterations
self.max_concurrent_subagents = (
    max_concurrent_subagents if max_concurrent_subagents is not None
    else defaults.max_concurrent_subagents
)
self._run_slots = asyncio.Semaphore(self.max_concurrent_subagents)
```

默认值来自 `AgentDefaults`（`nanobot/config/schema.py:116` 起）：`max_tool_iterations=200`（`nanobot/config/schema.py:129`）、`max_concurrent_subagents=4`（`:130`）。主 Agent 创建管理器时把自己的限制传进去（`nanobot/agent/loop.py:386-395`），运行期还会同步（`_sync_subagent_runtime_limits`，`:524-526`）。**所以子 Agent 不是"15 次迭代"，而是继承主 Agent 的 200 次。**

`spawn()` 本体（`:227`）做四件事：生成 8 位 `task_id`、建 `SubagentStatus`、`asyncio.create_task(self._run_subagent(...))`、注册完成回调清理状态。真正的并发闸门在 `_run_subagent`（`:354`）：

```python
# 来源：nanobot/agent/subagent.py:368-380（节选）
status.phase = "queued"
async with self._run_slots:
    status.phase = "initializing"
    return await self._run_admitted_subagent(...)
```

超过 4 个的请求不会失败，而是停在 `phase="queued"`，等前面的跑完。

### 10.2.3 子 Agent 与主 Agent 的区别

| 维度 | 主 Agent | 子 Agent |
|---|---|---|
| 迭代上限 | `max_tool_iterations=200` | **继承 200**（可被主 Agent 改小） |
| 工具集 | `core` scope（完整） | `subagent` scope（受限） |
| `spawn` | 可用 | **不可用**（防递归） |
| `message` / `cron` | 可用 | **不可用**（不与用户交互、不建定时任务） |
| 并发 | 每会话串行 | 全局最多 4 个（`max_concurrent_subagents`） |
| 执行方式 | 前台回合 | 后台 Task（或 `wait=true` 时同步等待） |
| 结果汇报 | 直接回复 | 经 MessageBus 以 `system` 消息注入父会话 |
| 生命周期 | 常驻 | 任务完成即销毁（`_cleanup`，`:278-286`） |

### 10.2.4 为什么限制工具集：scope 过滤发生在加载时

子 Agent 的注册表**不是**主注册表的子集拷贝，而是重新建一个：`_build_tools`（`nanobot/agent/subagent.py:205-225`）先 `ToolRegistry()`，再 `ToolLoader().load(ctx, registry, scope="subagent")`。

`ToolLoader.load` 在注册前逐个过滤（`nanobot/agent/tools/loader.py:100`）：

```python
# 来源：nanobot/agent/tools/loader.py:99-104
try:
    if scope not in getattr(tool_cls, "_scopes", {"core"}):
        continue
    if not tool_cls.enabled(ctx):
        continue
    tool = tool_cls.create(ctx)
```

默认 `_scopes = {"core"}`（`nanobot/agent/tools/base.py:208`），所以**没显式声明 `subagent` 的工具一律进不去**。子 Agent 能拿到的是文件读写、`exec`、`apply_patch`、web 搜索/抓取、`exec_session`、`run_cli_app` 等（它们的 `_scopes` 都含 `"subagent"`，见 `nanobot/agent/tools/filesystem.py:272`、`nanobot/agent/tools/shell.py:164`、`nanobot/agent/tools/search.py:209`）。`spawn`、`message`、`cron` 都沿用默认 `{"core"}`，天然被挡。

```
主 Agent：read_file / exec / spawn / message / cron / ...
子 Agent：read_file / exec / apply_patch / web / exec_session / ...
           （没有 spawn、message、cron）
```

这比"在 prompt 里叫模型别用"可靠得多：**能力不在注册表里，模型连调用的机会都没有**。

### 10.2.5 迭代与并发的设计取舍

200 次迭代听起来很多，但它不是"给子 Agent 放权"，而是**统一默认值**：主 Agent 和子 Agent 共用 `max_tool_iterations`，避免出现"主 Agent 能跑 200 轮、派出去的活 15 轮就断"的割裂。真正防失控的是另外三道闸：

1. **工具 scope**：子 Agent 没有 `spawn`，不能指数繁殖。
2. **并发信号量**：`max_concurrent_subagents=4`，超出的排队而非拒绝。
3. **会话级取消**：`cancel_by_session`（`nanobot/agent/subagent.py:556`）能一次性取消某会话的所有子任务，`close()`（`:567`）在关停时全取消。

如果你确实想收紧，改 `agents.defaults.maxToolIterations` 或 `maxConcurrentSubagents` 即可，不用改代码。

### 10.2.6 结果怎么回到主 Agent

子 Agent 跑完后，`_announce_result`（`:488`）把结果包装成一条**入站消息**：

```python
# 来源：nanobot/agent/subagent.py:514-530（节选）
override = origin.get("session_key") or f"{origin['channel']}:{origin['chat_id']}"
msg = InboundMessage(
    channel="system", sender_id="subagent",
    chat_id=f"{origin['channel']}:{origin['chat_id']}",
    content=announce_content, session_key_override=override,
    metadata={"injected_event": "subagent_result", "subagent_task_id": task_id},
)
await self.bus.publish_inbound(msg)
```

三个细节值得记住：

- `channel="system"`，不是旧文档说的 `"internal"`；`sender_id="subagent"`。
- `session_key_override` 保证结果回到**父会话**（包括统一会话模式下的正确 key）。
- `metadata["injected_event"]="subagent_result"` 让 AgentLoop 认出这是注入事件，并在历史里标记为隐藏（`nanobot/agent/loop.py:1045-1054`），用户不会看到一条"假用户消息"。

主 Agent 收到后，就像收到一条系统提示一样继续处理，把结果转述给用户。

### 10.2.7 使用场景

| 场景 | 适合子 Agent | 原因 |
|---|---|---|
| 长时间调研/多网页汇总 | 是 | 不阻塞当前对话 |
| 批量文件处理 | 是 | 后台跑，失败可单独看 |
| 代码分析/生成报告 | 是 | 耗时长但不需要中途交互 |
| "先查清楚再回答我" | 是（`wait=true`） | 需要结果才能继续 |
| 需要用户确认的操作 | 否 | 子 Agent 不能与用户对话 |
| 实时闲聊 | 否 | 后台反而增加延迟 |

---

## 10.3 定时任务（Cron）系统

### 10.3.1 CronService：不是 APScheduler

一句话：**v0.3.5 的 `CronService` 是自研的 asyncio 定时器，没有 APScheduler 依赖。**

```python
# 来源：nanobot/cron/service.py:168-193（节选）
class CronService:
    def __init__(self, store_path: Path, on_job=..., max_sleep_ms: int = 300_000):
        self.store_path = store_path
        self._timer_task: asyncio.Task[None] | None = None
        self.max_sleep_ms = max_sleep_ms
```

调度靠"睡到下一个到点时间"：

```python
# 来源：nanobot/cron/service.py:533-545（节选）
next_wake = self._get_next_wake_ms()
delay_ms = self.max_sleep_ms if next_wake is None else min(
    self.max_sleep_ms, max(0, next_wake - _now_ms()))
async def tick():
    await asyncio.sleep(delay_ms / 1000)
    if self._running: await self._on_timer()
self._timer_task = asyncio.create_task(tick())
```

`max_sleep_ms` 默认 5 分钟，意味着**即使没有任务也会每 5 分钟醒一次**，顺便重读磁盘上的 `jobs.json`（支持热更新/多进程协作）。任务存储路径是 `<workspace>/cron/jobs.json`（`nanobot/cli/gateway_runtime.py:463`），旁边的 `action.jsonl` 记录增量操作、`runs/` 存每次执行记录（`nanobot/cron/service.py:184-185`）。

### 10.3.2 cron 工具：add / list / remove

工具名是 `cron`（`nanobot/agent/tools/cron.py:56`）。**参数是扁平的，不是嵌套的 `schedule` 对象**：

```python
# 来源：nanobot/agent/tools/cron.py:22-44（节选）
action=StringSchema(..., enum=["add", "list", "remove"]),
message=StringSchema("REQUIRED when action='add'..."),
every_seconds=IntegerSchema(description="Interval in seconds..."),
cron_expr=StringSchema("Cron expression like '0 9 * * *'..."),
tz=StringSchema("Optional IANA timezone..."),
at=StringSchema("ISO datetime for one-time execution..."),
job_id=StringSchema("REQUIRED when action='remove'..."),
required=["action"],
```

三个动作：

**add** —— 至少给 `message` + 一种调度（`every_seconds` / `cron_expr` / `at`）：

```json
{ "action": "add", "name": "daily_news",
  "cron_expr": "0 8 * * *", "tz": "Asia/Shanghai",
  "message": "搜索今天的科技新闻，整理成摘要发给我" }
```

**list** —— 只传 `action`，返回任务、下次运行时间、上次状态。

**remove** —— 必须用 `job_id`（从 list 里拿），**不能用 `name`**：

```json
{ "action": "remove", "job_id": "a1b2c3d4" }
```

### 10.3.3 三种调度方式

| 方式 | 参数 | 语义 |
|---|---|---|
| 固定间隔 | `every_seconds: 3600` | 每 3600 秒一次 |
| Cron 表达式 | `cron_expr: "0 8 * * *"` + `tz` | 每天 8:00（按指定时区） |
| 一次性 | `at: "2026-03-17T15:30:00"` | 到点执行一次后自动删除 |

实现里 `tz` **只能和 `cron_expr` 一起用**（`nanobot/agent/tools/cron.py:177-178` 直接报错）；`at` 如果是 naive 时间，会按工具默认时区补上（`:199-203`）；`at` 任务会设置 `delete_after=True`（`:205`），执行完自己消失。

### 10.3.4 执行流程：cron 任务是"原会话的一轮输入"

这是 v0.3.5 与旧文档差别最大的地方。定时任务不是"新开一个匿名会话"，而是**回到创建它的那个会话**（`nanobot/cron/bound_runner.py:65`）：

```python
# 来源：nanobot/cron/bound_runner.py:120-128（节选）
resp = await agent.submit_cron_turn(InboundMessage(
    channel=channel, sender_id="cron", chat_id=chat_id,
    content=prompt, metadata=metadata, session_key_override=session_key,
))
```

流程：

```
gateway 启动 → CronService.start()（service.py:474）加载 jobs.json
    → _arm_timer() 睡到最近一个 next_run_at_ms
到点 → _on_timer() 找 due_jobs（service.py:567-570）
    → on_job(job) 回调（gateway_runtime 里注册）
        ├── 普通任务 → run_bound_cron_job：
        │     渲染 cron_reminder 模板 → submit_cron_turn 作为原会话的一轮
        └── 系统任务 → Dream / Heartbeat 分支
    → 写 run record、推进 next_run、_save_store()
```

任务还带"会话绑定"约束：创建时必须能拿到 `session_key` + `origin_channel` + `origin_chat_id`，否则报错"must be created from a chat session"（`nanobot/agent/tools/cron.py:172-176`）。启动时若发现历史遗留的"无绑定"任务，会把它禁用而不是瞎跑（`nanobot/cron/service.py:202-225`）。

### 10.3.5 防递归：ContextVar，而不是看 channel

旧思路是"检查消息的 `channel == "cron"`"，v0.3.5 用的是**上下文变量**：

```python
# 来源：nanobot/agent/tools/cron.py:62 与 :147-150
self._in_cron_context: ContextVar[bool] = ContextVar("cron_in_context", default=False)
...
if self._in_cron_context.get():
    return ToolResult.error("Error: cannot schedule new jobs from within a cron job execution")
```

谁把它设成 True？cron 执行器在跑任务前：

```python
# 来源：nanobot/cron/bound_runner.py:115-118
cron_tool = agent.tools.get("cron")
cron_token = None
if isinstance(cron_tool, CronTool): cron_token = cron_tool.set_cron_context(True)
```

执行完在 `finally` 里 reset（`:141-143`）。这么做的好处：**cron 触发的回合里 channel 仍是原来的 telegram/websocket，不能靠 channel 判断**；ContextVar 精确标记"当前这一轮是定时任务触发的"，哪怕它长得和普通用户消息一模一样。

为什么必须防：一个"每分钟创建一条每分钟任务"的 job 会让任务数指数增长；禁掉 add 后，定时任务只能执行、不能繁殖。

---

## 10.4 Heartbeat 心跳服务

### 10.4.1 是什么：cron 的一种特殊用法

```
Cron 普通任务：   "每天 8 点，发今日新闻"   → 明确任务，结果回原会话
Heartbeat：       "每 30 分钟醒来，读 HEARTBEAT.md，有活就干" → 自主检查
```

v0.3.5 **没有 `HeartbeatService` 这个类**。心跳是一个**受保护的 cron 系统任务**，由 gateway 注册（`nanobot/cli/gateway_runtime.py:834-845`）：

```python
# 来源：nanobot/cli/gateway_runtime.py:834-845（节选）
if hb_cfg.enabled:
    cron.register_system_job(CronJob(
        id="heartbeat", name="heartbeat",
        schedule=CronSchedule(kind="every", every_ms=hb_cfg.interval_s * 1000, ...),
        payload=CronPayload(kind="system_event"),
    ))
else:
    cron.remove_system_job("heartbeat")
```

`payload.kind == "system_event"` 让它成为"受保护任务"：`remove_job` 会拒绝删除它（`nanobot/cron/service.py:769-777`），只有启动时的 `remove_system_job` 能按配置移除。

### 10.4.2 配置方式

配置在 `gateway.heartbeat`，**不在 `agents.defaults` 下面**：

```json
{ "gateway": { "heartbeat": { "enabled": true, "interval_s": 1800 } } }
```

字段定义在 `nanobot/config/schema.py:326-330`（`enabled` 默认 `true`，`interval_s` 默认 `30*60`），挂在 `GatewayConfig` 上（`nanobot/config/schema.py:353`，`heartbeat` 字段在 `nanobot/config/schema.py:359`）。只有两个字段——**没有 `keep_recent_messages`**。

### 10.4.3 触发机制：读文件 → 跑一轮 → 评估要不要打扰你

```python
# 来源：nanobot/cli/gateway_runtime.py:621-639（节选）
if job.name == "heartbeat":
    content = (config.workspace_path / "HEARTBEAT.md").read_text(encoding="utf-8")
    if not _heartbeat_has_active_tasks(content):
        return                                    # 没活就跳过，不叫醒模型
    channel, chat_id = _pick_heartbeat_target()
    prompt = _HEARTBEAT_PREAMBLE + "You are executing periodic heartbeat tasks..." + content
```

三个关键点：

1. **只认 `## Active Tasks` 段**。`_heartbeat_has_active_tasks`（`:175`）会跳过标题、空行、HTML 注释，只在这个二级标题下找任务行；没有就完全跳过，连一次模型调用都不发生。
2. **投递目标从最近会话里挑**（`_pick_heartbeat_target`，`:735-748`）：优先选已启用渠道里最近活跃的 `channel:chat_id`；挑不到就 `cli`，而 `cli` 目标直接返回不执行（`:634-635`）。
3. **跑完还有一道"要不要通知"的评估**（`:665-686`）：

```python
# 来源：nanobot/cli/gateway_runtime.py:668-683（节选）
with llm_usage_source("cron"):
    should_notify = await evaluate_response(
        response=response, task_context=prompt, provider=agent.provider,
        model=agent.model, evaluator_prompt=evaluator_prompt,
        default_notify=False,          # 失败就闭嘴（fail closed）
    )
if should_notify:
    await _deliver_to_channel(OutboundMessage(
        channel=channel, chat_id=chat_id, content=response), record=True)
```

`evaluate_response`（`nanobot/utils/evaluator.py:92`）用一次轻量 LLM 调用判断"这条结果值得打扰用户吗"。为了不让模型用 `message` 工具绕过评估，执行期间还会临时抑制 message 投递（`nanobot/cli/gateway_runtime.py:644-658`，配合 `nanobot/agent/tools/message.py:108`）。

### 10.4.4 HEARTBEAT.md 怎么写

模板在 `nanobot/templates/HEARTBEAT.md`，核心结构：

```markdown
# Heartbeat Tasks
<!-- 只有 "## Active Tasks" 下的任务会被执行；完成后请删除，不要留档 -->
## Active Tasks

- 检查 workspace 里的 TASKS.md，有到期项就提醒我
- 看看 GitHub 上有没有新的 review request
```

规则：

- **只读 `## Active Tasks` 段**，其他段落当成注释；
- 任务做完就删掉，不要积累"已完成"清单；
- 没有任务就留空，心跳会安静跳过；
- 想改变"什么时候该通知"，可以写 workspace 里的 evaluator prompt 覆盖（`nanobot/utils/evaluator.py:38-66`），默认失败时保持沉默。

### 10.4.5 使用场景

| 场景 | `interval_s` | 说明 |
|---|---|---|
| 待办巡检 | 1800 | 每 30 分钟看一次 待办/到期项 |
| 邮件/消息摘要 | 3600 | 每小时汇总一次 |
| 项目监控 | 900 | 15 分钟看一次 CI/PR |
| 日报 | 86400 | 每天一次 |

注意：心跳间隔太短 + `HEARTBEAT.md` 任务太重，会持续消耗 token；因为每次都要跑一轮完整 Agent，还要多一次评估调用。

---

## 10.5 三者的协作关系

### 10.5.1 对比

| 维度 | 子 Agent | Cron | Heartbeat |
|---|---|---|---|
| 触发方式 | 主 Agent 调 `spawn` | 时间表达式 | 固定间隔的系统 cron 任务 |
| 任务内容 | 模型现场生成 | 用户预设的 `message` | `HEARTBEAT.md` 里的任务清单 |
| 生命周期 | 跑完即销毁 | 长期存在（`at` 类型执行后删除） | 长期存在（受保护） |
| 与用户交互 | 不能，只能回报父会话 | 结果回原会话 | 先评估，值得才发 |
| 并发 | 最多 4 个 | 按调度串行触发 | 单实例 |
| 防递归 | 无 `spawn` 工具 | `ContextVar` 禁止 add | 只读文件，不建任务 |
| 落盘 | 不落盘（内存态） | `<workspace>/cron/jobs.json` | 同 cron（系统任务） |

### 10.5.2 协作场景

```
Heartbeat（每 30 分钟）
  ├── 发现新邮件 → 值得说 → 直接通知用户
  └── 发现一份长文档 → 调 spawn 派给子 Agent 慢慢读 → 结果回会话

Cron（每天 8:00）
  └── "发今日新闻" → 作为该会话的一轮输入执行 → 结果回会话

用户的即时请求
  └── "帮我调研 X" → spawn 后台跑 → 完成后 system 消息注入 → 主 Agent 转述
```

### 10.5.3 设计模式

- **子 Agent ≈ Actor**：独立运行、通过消息通信、任务完成即销毁；区别是它跑在同一进程的 Task 里，不是独立进程。
- **Cron ≈ 定时器 + 观察者**：只负责"到点叫一声"，业务逻辑仍由 Agent 跑。
- **Heartbeat ≈ 轮询 + 门控**：轮询文件，门控（evaluator）决定要不要打扰用户，避免"没事也刷屏"。

---

## 10.6 实战练习

### 练习 1：让两个子 Agent 并行干活

```bash
nanobot agent        # 本地 TUI（或 nanobot gateway + 任意渠道）
```

对话：

```
你：帮我同时做两件事，都用后台任务：
    1) 调研 Python 3.13 的新特性
    2) 调研 Rust 2024 的进展
    完成后分别告诉我。
```

预期：模型调用两次 `spawn`（两次返回 "Subagent [...] started"），主对话不被阻塞；两个任务最多 4 个并发槽位，先完成的先注入结果。用 `wait=true` 的版本则是"先查完再回答"。

### 练习 2：创建一个定时任务

```bash
nanobot gateway
```

对话：

```
你：每天早上 8 点提醒我看一下 CI 状态，时区 Asia/Shanghai
```

模型应调用 `cron`：

```json
{ "action": "add", "name": "ci_check", "cron_expr": "0 8 * * *",
  "tz": "Asia/Shanghai", "message": "检查 CI 状态并汇报失败项" }
```

然后验证：

```
你：列出所有定时任务          → cron {action:"list"}
你：删掉 ci_check             → cron {action:"remove", job_id:"..."}
```

也可以直接看 `<workspace>/cron/jobs.json`，能看到这条任务的 `schedule`、`payload.message`、`session_key`。

### 练习 3：配置 Heartbeat

```json
{ "gateway": { "heartbeat": { "enabled": true, "interval_s": 300 } } }
```

在 `<workspace>/HEARTBEAT.md` 写：

```markdown
## Active Tasks
- 检查 workspace 里的 TASKS.md，有今天到期的条目就提醒我
```

```bash
nanobot gateway     # 启动日志会打印 "Heartbeat: every 300s"
```

验收：把任务删空后，心跳不再产生任何模型调用；有任务且确实有到期项时，才会收到一条消息。注意第一次启动时 `HEARTBEAT.md` 不存在也没关系，日志里只有一条 debug。

---

## 10.7 面试高频题

### 题目 1：nanobot 的 SubAgent 是怎么设计的？

> "入口是 `spawn` 工具（`nanobot/agent/tools/spawn.py:48`），参数 `task` 必填，另有 `label`、`temperature`、`wait`。`SubagentManager`（`nanobot/agent/subagent.py:93`）负责创建 Task、维护状态、限制并发：全局 `max_concurrent_subagents=4`，用 `asyncio.Semaphore` 排队；迭代上限继承主 Agent 的 `max_tool_iterations=200`，不是单独写死一个小数字。工具集按 scope 过滤：`ToolLoader.load(scope="subagent")` 只注册 `_scopes` 含 `"subagent"` 的工具，所以子 Agent 拿不到 `spawn`、`message`、`cron`，从能力层面杜绝递归和越权交互。结果通过 `InboundMessage(channel="system", sender_id="subagent", session_key_override=父会话)` 注入回主会话，并由 `metadata["injected_event"]` 标记为注入事件。"

### 题目 2：如何防止 Agent 系统的递归与失控？

> "三个层次。第一，**能力隔离**：子 Agent 的 ToolRegistry 里根本没有 `spawn`，想递归也没有工具；这比在 prompt 里叮嘱可靠。第二，**并发与数量限制**：`max_concurrent_subagents=4` 的信号量、任务完成即销毁、`cancel_by_session` 可一键取消。第三，**上下文标记**：cron 执行时把 `ContextVar` 置为 True，`cron` 工具的 `add` 直接报错，防止定时任务繁殖定时任务。另外 cron 任务必须绑定创建它的会话，无绑定的历史任务启动时会被禁用，避免'不知道往哪投递'的僵尸任务。"

### 题目 3：Cron 和 Heartbeat 有什么区别？

> "两者底层是同一套 `CronService`：Cron 是用户/模型通过 `cron` 工具创建的任务，携带明确的 `message`，到点作为原会话的一轮输入执行，结果回原会话；Heartbeat 是 gateway 注册的**受保护系统任务**（`id="heartbeat"`，`payload.kind="system_event"`），到点读 `HEARTBEAT.md` 的 Active Tasks，有活才叫醒 Agent，跑完还要过一次 LLM 评估决定要不要通知用户。
>
> 类比：Cron 像闹钟（8 点叫我起床），Heartbeat 像保安巡逻（每 30 分钟看一圈，有事才报告）。配置上 Heartbeat 只有 `enabled` 和 `interval_s`，没有 `keep_recent_messages` 这种字段。"

### 题目 4：如果让你设计一个后台任务系统，你会注意什么？

> "我会关注五点：**资源上限**（并发数、单任务迭代数、超时）、**递归防护**（能力层面拿掉繁殖工具）、**结果回传**（可靠的会话绑定，而不是广播）、**可观测性**（任务状态、运行记录，nanobot 用 `SubagentStatus` 和 `cron/runs/`）、**关停语义**（取消在跑的任务、清理子进程/会话）。还有一个容易被忽略的点：任务创建时的路由必须持久化，否则进程重启后不知道该把结果发到哪——这正是 v0.3.5 要求 cron 任务"从聊天会话创建"的原因。"

---

## 10.8 本章小结

```
主 Agent（单会话一轮）
 ├── spawn 工具 ──→ SubagentManager
 │                   ├── 全局信号量 4
 │                   ├── subagent scope 工具集
 │                   └── 完成后 system 消息注入父会话
 ├── cron 工具  ──→ CronService
 │                   ├── <workspace>/cron/jobs.json
 │                   ├── asyncio 定时器（最长睡 5 分钟）
 │                   └── 到点 → 原会话的一轮输入
 └── 无 spawn/cron 的回合 ← 防递归由能力隔离保证

gateway.heartbeat → 受保护系统 cron 任务
    └── 读 HEARTBEAT.md → 有活才跑 → evaluator 决定发不发
```

| 考点 | 一句话回答 |
|---|---|
| 子 Agent 入口 | `spawn` 工具（`nanobot/agent/tools/spawn.py:48`，名称 `:63`） |
| 管理器 | `SubagentManager`（`nanobot/agent/subagent.py:93`） |
| 迭代上限 | 继承主 Agent 的 `max_tool_iterations=200` |
| 并发上限 | `max_concurrent_subagents=4`（`nanobot/config/schema.py:130`） |
| 工具限制 | `ToolLoader` 按 `_scopes` 过滤（`nanobot/agent/tools/loader.py:100`） |
| 结果回报 | `channel="system"` + `session_key_override`（`nanobot/agent/subagent.py:521-527`） |
| Cron 引擎 | 自研 asyncio 定时器（`nanobot/cron/service.py:168`） |
| 任务落盘 | `<workspace>/cron/jobs.json` |
| Cron 参数 | 扁平：`action/message/every_seconds/cron_expr/tz/at/job_id` |
| 防递归 | `ContextVar` 标记 cron 上下文（`nanobot/agent/tools/cron.py:62`） |
| Heartbeat 实现 | `gateway.heartbeat` + 受保护 cron 系统任务，读 `HEARTBEAT.md` |
| Heartbeat 配置 | `enabled` + `interval_s`（`nanobot/config/schema.py:326-330`） |

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 子 Agent 用 `spawn` 工具 | ✅ 仍然叫 `spawn`（这条旧文档是对的） | `nanobot/agent/tools/spawn.py:63` | 保留 |
| `spawn` 只有 `task` 一个参数 | 还有 `label` / `temperature` / `wait`（`wait` 是同步咨询模式） | `nanobot/agent/tools/spawn.py:24-47`、`:96` | 补充参数说明 |
| 子 Agent 工具集"受限"但没讲机制 | 由 `_scopes` 在 `ToolLoader` 注册阶段过滤，默认 `{"core"}` | `nanobot/agent/tools/loader.py:100`；`nanobot/agent/tools/base.py:208` | 讲"能力隔离" |
| 没有并发上限的概念 | 新增 `max_concurrent_subagents=4` + 信号量排队 | `nanobot/config/schema.py:130`；`nanobot/agent/subagent.py:150` | 补充并发控制 |
| Cron 只在内存里调度 | 落盘 `jobs.json` + `action.jsonl` + `runs/`，支持热重载 | `nanobot/cli/gateway_runtime.py:463`；`nanobot/cron/service.py:184-185` | 讲持久化与可观测性 |
| Cron 任务不绑定会话 | 必须绑定 `session_key`，无绑定历史任务会被禁用 | `nanobot/agent/tools/cron.py:172-176`；`nanobot/cron/service.py:202-225` | 强调"从会话创建" |
| Heartbeat 是独立服务类 | 已改为 gateway 注册的受保护 cron 系统任务 | `nanobot/cli/gateway_runtime.py:834-845` | 按"cron 系统任务"理解 |
| 结果回报用 `channel="internal"` | 统一走 `channel="system"` + `injected_event` 标记 | `nanobot/agent/subagent.py:521-528`；`nanobot/agent/loop.py:1045-1054` | 统一讲 system 注入 |

## 附录 B：进阶理解

**1. 为什么子 Agent 复用 `AgentRunner`，而不是另写一套循环？** 子 Agent 本质上就是"一次独立的模型+工具回合"，复用 `AgentRunner.run(AgentRunSpec(...))`（`nanobot/agent/subagent.py:427-444`）能免费获得重试、上下文治理、工具执行、停止原因等所有逻辑，只需要换一份消息、一份工具注册表、一个 hook。另写一套只会导致行为分叉。

**2. 为什么工具限制放在加载期而不是执行期？** 执行期拦截需要"模型先看到工具 → 调用 → 被拒绝"，既浪费 token 又可能被绕过；加载期直接不注册，模型看到的工具列表里就没有 `spawn`。这是"最小权限"在工具系统里的落地（`nanobot/agent/tools/loader.py:100`）。

**3. 为什么 Cron 不用 APScheduler？** APScheduler 功能全但引入额外依赖和线程/事件循环适配问题；nanobot 的需求很窄（每 N 秒/每天/某时刻），用一个 `asyncio.sleep` 到最近到点时间就够了，还能顺便每 5 分钟重读磁盘、支持多进程通过 `action.jsonl` 合并操作。少一个依赖，行为也更可控。

**4. 为什么系统任务要"受保护"？** Heartbeat 和 Dream 是程序注册的基础设施。如果用户一句"删掉所有定时任务"就能删掉它们，心跳/记忆整理会静默失效。`remove_job` 检查 `payload.kind == "system_event"` 并拒绝（`nanobot/cron/service.py:769-777`），把"用户任务"和"系统任务"分开。

**5. 为什么心跳要做"事后评估"而不是直接发结果？** 心跳每 30 分钟跑一次，大部分结果是"没事"。直接发会把聊天窗口变成日志。`evaluate_response` 用一次低温 LLM 调用做门控，并且失败时选择沉默（`default_notify=False`），保证"宁可不打扰，也不刷屏"。

**6. 为什么 cron 任务要回到原会话执行，而不是新建会话？** 任务里的 `message` 往往依赖上下文（"把昨天的报告补完"）。回到原会话，模型能看到历史、能延续语气；新建会话则每次都是"失忆"的。代价是必须持久化 `session_key`/`origin_channel`/`origin_chat_id`，所以"从会话创建"是硬约束。

## 附录 C：Python 语法加油站

### C.1 `asyncio.create_task` + `add_done_callback`

```python
import asyncio
async def work(x): return x * 2
async def main():
    task = asyncio.create_task(work(21))     # 立即开始调度
    task.add_done_callback(lambda t: print("done:", t.result()))
    await asyncio.sleep(0)                   # 让出一次
```

用在哪：`SubagentManager.spawn` 创建后台任务并注册清理回调（`nanobot/agent/subagent.py:262-286`）。读法：`create_task` 只是"排进事件循环"，不会等你；回调在任务完成时触发，适合做状态清理——但别在回调里 `await`。

### C.2 `asyncio.Semaphore` 与 `async with`

```python
sem = asyncio.Semaphore(2)          # 最多 2 个并发
async def job(i):
    async with sem:                 # 拿不到就挂起等待
        await asyncio.sleep(1)
        print(i)
```

用在哪：子 Agent 并发闸门（`nanobot/agent/subagent.py:150`、`:369`）。读法：`async with sem` 相当于"进门领号"，出来自动还号；超出的任务不会失败，只是排队。

### C.3 `ContextVar`

```python
from contextvars import ContextVar
in_job = ContextVar("in_job", default=False)
async def run():
    token = in_job.set(True)        # 当前任务上下文内有效
    try:
        print(in_job.get())         # True
    finally:
        in_job.reset(token)
```

用在哪：cron 防递归标记（`nanobot/agent/tools/cron.py:62`，设置点在 `nanobot/cron/bound_runner.py:118`）。读法：它解决"没有参数可传、但下游需要知道当前处境"的问题；与线程局部变量不同，协程之间天然隔离，`reset` 保证不泄漏。

### C.4 `dataclass(slots=True)` 与 `TypedDict`

```python
from dataclasses import dataclass
from typing import TypedDict

@dataclass(slots=True)
class Status:            # slots 省内存、防手滑加属性
    phase: str = "queued"
    iteration: int = 0

class Origin(TypedDict):  # 只做类型提示，运行时仍是 dict
    channel: str
    chat_id: str
```

用在哪：`SubagentStatus`（`nanobot/agent/subagent.py:50-64`）和 `_SubagentOrigin`（`:43-47`）。读法：`slots=True` 的 dataclass 没有 `__dict__`，适合创建成百上千个的短命状态对象；`TypedDict` 是"给 dict 加类型注解"，不改变运行时行为。

## 附录 D：原文勘误

| # | 原文位置 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| D1 | 10.2.3 / 10.2.5 | "主 Agent 40 次、SubAgent 15 次迭代" | 主默认 **200**，子 Agent **继承** 200（可被主改） | `nanobot/config/schema.py:129`；`nanobot/agent/subagent.py:140-144`；`nanobot/agent/loop.py:393`、`:526` |
| D2 | 10.3.1 | "CronService 基于 APScheduler" | 自研 **asyncio 定时器**，无 APScheduler 依赖 | `nanobot/cron/service.py:168`、`:520-545` |
| D3 | 10.4.3 | 定义并展示 `class HeartbeatService` | **没有这个类**；心跳是 gateway 注册的受保护 cron 系统任务 | `nanobot/cli/gateway_runtime.py:834-845`；`nanobot/config/schema.py:326` |
| D4 | 10.4.2 | Heartbeat 配在 `agents.defaults.heartbeat` | 配在 **`gateway.heartbeat`** | `nanobot/config/schema.py:353-359` |
| D5 | 10.4.2 / 10.4.5 | 有 `keep_recent_messages` 字段 | **不存在**；`HeartbeatConfig` 只有 `enabled` 和 `interval_s` | `nanobot/config/schema.py:326-330` |
| D6 | 10.3.2 | cron 参数是 `schedule: {cron_expr, tz}` 嵌套对象 | 参数**扁平**：`action/message/every_seconds/cron_expr/tz/at/job_id` | `nanobot/agent/tools/cron.py:22-52` |
| D7 | 10.3.2 | `remove` 用任务名删除 | 必须传 **`job_id`**，`name` 只是展示标签 | `nanobot/agent/tools/cron.py:43`、`:132-133`、`:154` |
| D8 | 10.3.5 | 防递归靠检查 `channel == "cron"` | 用 **`ContextVar`** `_in_cron_context`，执行器设置、工具读取 | `nanobot/agent/tools/cron.py:62`、`:147-150`；`nanobot/cron/bound_runner.py:115-118` |
| D9 | 10.2.6 | 结果回报用 `channel="internal"` | 真实是 **`channel="system"`**，并带 `injected_event="subagent_result"` | `nanobot/agent/subagent.py:521-528` |
| D10 | 10.2.2 | `SubagentManager(agent_config, message_bus)` 这种位置参数 | 关键字参数：`workspace`/`bus`/`max_tool_result_chars` 等，缺了直接 `TypeError` | `nanobot/agent/subagent.py:96-118` |
| D11 | 10.2.1 | `spawn` 只接受 `task` | 还有 `label`、`temperature`、`wait` | `nanobot/agent/tools/spawn.py:24-47` |
| D12 | 10.4.3 | 心跳直接构造 `InboundMessage` 并 `publish_inbound` | 走 **cron 系统任务** → 读 `HEARTBEAT.md` → `process_direct` → evaluator 门控 | `nanobot/cli/gateway_runtime.py:621-686` |
| D13 | 10.3.2 / 10.6 | 暗示可以在任意上下文 `add` cron | `add` 必须能拿到会话路由，否则报"must be created from a chat session" | `nanobot/agent/tools/cron.py:172-176` |

---

> **下一步读什么**：继续读 [11 - 安全与部署](../11-security-and-deploy/README.md)，看 `restrict_to_workspace`、`exec` 沙箱与 Docker 部署；想直接读源码，从 `nanobot/agent/tools/spawn.py` 到 `nanobot/agent/subagent.py` 顺着读一遍，再读 `nanobot/cron/service.py` 的 `_arm_timer`，就能把本章三条链路串起来。
