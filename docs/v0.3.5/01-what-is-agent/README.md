# 01 - 什么是 AI Agent

> 🎯 **本章目标**：搞清 AI Agent 与 LLM 的本质区别、核心三要素、ReAct 循环，并能把"自主循环"这句话落到 nanobot v0.3.5 的真实代码位置上。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

---

## 目录

- [1.1 AI Agent 的定义](#11-ai-agent-的定义)
- [1.2 LLM vs AI Agent：本质区别](#12-llm-vs-ai-agent本质区别)
- [1.3 Agent 核心三要素](#13-agent-核心三要素)
- [1.4 ReAct 框架详解](#14-react-框架详解)
- [1.5 Agent 工作流程](#15-agent-工作流程)
- [1.6 主流 Agent 框架对比](#16-主流-agent-框架对比)
- [1.7 Agent 的发展历史](#17-agent-的发展历史)
- [1.8 面试话术](#18-面试话术)
- [1.9 面试常考点总结](#19-面试常考点总结)
- [1.10 练习题与思考题](#110-练习题与思考题)

---

![插图](../../../comics/01-agent-concept.png)

*能自己思考、自己动手、还会记事儿的助手，才叫 Agent*

## 1.1 AI Agent 的定义

### 一句话定义

**AI Agent（智能体）是以大语言模型为"大脑"，能自己决定下一步做什么、调用工具真正去做、并根据结果继续调整的系统。**

教材里常见的公式是：

> **AI Agent = LLM（大脑）+ Planning（规划）+ Memory（记忆）+ Tool Use（工具使用）**

但对工程师来说还差一层：这四个东西必须被一个"循环"串起来，否则 Planning 只是聊天，Tool Use 只是函数调用。所以更贴近代码的写法是：

> **Agent = LLM + 循环（Loop）+ 工具（Tools）+ 记忆（Memory）+ 运行时（Runtime）**

### 一个生活化的类比

把 Agent 想成一位刚入职的助理：

| 助理身上的东西 | Agent 里对应的部件 | 在 nanobot 里是谁 |
|---|---|---|
| 大脑、常识、表达能力 | LLM（大语言模型） | `nanobot/providers/` 里的 Provider |
| 判断"下一步干什么" | 循环 + 提示词 | `nanobot/agent/runner.py` |
| 手和脚 | 工具（读写文件、执行命令、搜网页） | `nanobot/agent/tools/` |
| 记事本 | 长期记忆 + 历史记录 | `nanobot/agent/memory.py` |
| 耳朵和嘴 | 渠道（Telegram、飞书、WebUI…） | `nanobot/channels/` |
| 接活、派活、收尾 | 回合编排 | `nanobot/agent/loop.py` |

助理之所以是助理，不是因为他"读过很多书"，而是因为**他会在做事的过程中不断看结果、改下一步**。这个"不断看、不断改"就是 Agent 的灵魂。

学术上常引用四个特征（Wooldridge & Jennings, 1995）：自主性（Autonomy）、反应性（Reactivity）、主动性（Pro-activeness）、社会性（Social Ability）。翻译成工程语言就是：**有循环、有输入、有目标、有交互**。少一个，系统就会退化成"更花哨的聊天框"。

### 落到 nanobot：Agent 其实"没有自己的类"

你在 nanobot 里搜 `class Agent`，是搜不到的。v0.3.5 把它拆成了两层：

| 层 | 类 | 位置 | 职责 |
|---|---|---|---|
| 回合编排（产品层） | `AgentLoop` | `nanobot/agent/loop.py:196` | 收消息、找会话、拼上下文、调 Runner、落盘、发回复 |
| 模型/工具循环（执行层） | `AgentRunner` | `nanobot/agent/runner.py:141` | 反复请求模型、执行工具、判断何时停 |

`AgentRunner` 的类注释写得很直白：*"Run a tool-capable LLM loop without product-layer concerns."*（跑一个能用工具的 LLM 循环，不掺产品层的事。）这句话就是两层拆分的理由：**循环逻辑要能复用，产品逻辑（谁发的消息、发到哪个平台、要不要流式输出）不该污染循环。**

> **术语速查**
> - **LLM**：大语言模型，输入文本/图片，输出文本和"工具调用请求"。
> - **Provider**：把不同厂商的模型 API 包装成统一接口的适配层。
> - **Tool**：有名字、有参数说明、能被模型"点名调用"的函数。
> - **Session**：一段连续对话的容器，nanobot 用 `{channel}:{chat_id}` 作键（`nanobot/bus/events.py:40`）。
> - **Turn（回合）**：从收到一条用户消息，到发出回复为止的一整轮处理。
> - **Workspace**：Agent 的工作目录，记忆、技能、产物都放在这里。

---

## 1.2 LLM vs AI Agent：本质区别

| 维度 | LLM（纯模型） | AI Agent（智能体系统） |
|---|---|---|
| 本质 | 文本生成器 | 带循环的决策系统 |
| 输入 | 提示词 | 提示词 + 工具结果 + 记忆 + 环境状态 |
| 输出 | 文本（可能含工具调用 JSON） | 文本 + 真实世界的副作用（文件、消息、订单） |
| 记忆 | 只在上下文窗口里 | 短期（上下文）+ 长期（文件/数据库） |
| 行动力 | 不能，除非外面有人执行 | 能，运行时替它执行工具 |
| 纠错 | 一次成型 | 看结果、改计划、重试 |
| 主动性 | 被动回答 | 可被定时任务、心跳、子任务唤醒 |
| 类比 | 一位知识渊博的顾问 | 一位能跑腿、会记事、能复盘的项目经理 |

### 一个具体例子

用户说："帮我看看这个仓库里 Agent 的循环写在哪。"

**纯 LLM** 只能回："你可以用 grep 搜索 `for iteration`……"——它没有手。

**Agent** 会这样做：

```
第 1 轮
  思考：先找到循环所在的文件，用 grep 搜关键词。
  行动：grep(pattern="for iteration in range", path="nanobot/agent")
  观察：nanobot/agent/runner.py:435 命中

第 2 轮
  思考：读一下上下文，确认这是不是主循环。
  行动：read_file(path="nanobot/agent/runner.py", offset=420, limit=40)
  观察：看到 `for iteration in range(spec.max_iterations)`，确认是主循环
  最终回答：循环在 runner.py:435，外层由 _run_core 驱动。
```

`read_file` 的参数就是 nanobot 真实使用的参数名：`path`、`offset`、`limit`（`nanobot/agent/tools/filesystem.py:270` 起的 `ReadFileTool`）。Agent 的"行动"最终就是一次普通的函数调用。

### 三个关键区别

**1. 行动力来自"外面那圈代码"。** 模型只输出一段 JSON，说"我想调用 `read_file`，参数是这些"。真正打开文件的是运行时的 `ToolRegistry`（`nanobot/agent/tools/registry.py:187`）。"Agent 会用工具"的准确说法是：**模型提出请求，运行时负责执行，再把结果喂回去**。

**2. 记忆不只靠上下文窗口。** 窗口再大也会满，Agent 需要把值得长期记住的事写到窗口之外。nanobot 用 `memory/MEMORY.md`（长期事实，每次拼 system prompt 都注入）加 `memory/history.jsonl`（追加式历史日志），见 `nanobot/agent/memory.py:76` 和 `:77`。

**3. 自主循环是硬条件。** 没有循环，模型无法"看到工具结果再决定下一步"。nanobot 的循环只有一行 `for`，但上限来自配置，默认 **200**（`nanobot/config/schema.py:129`），不是旧文档说的 40：

```python
# nanobot/agent/runner.py:435（节选）
for iteration in range(spec.max_iterations):
    ...
```

### 面试回答模板

> "LLM 是文本生成模型，只能根据输入产出输出。Agent 是以 LLM 为核心搭出来的系统，在模型外面加了三样东西：一个能反复'请求模型—执行工具—回灌结果'的循环，一套把重要信息写到上下文之外的记忆，以及一组能真正改变外部状态的工具。所以 LLM 是大脑，Agent 是大脑加手脚加记事本，再加一个负责调度它们的流程。"

---

## 1.3 Agent 核心三要素

Lilian Weng 在 2023 年的博文《LLM Powered Autonomous Agents》里，把 Agent 的能力拆成 Planning、Memory、Tool Use 三块。这个拆法至今仍是面试的标准答案。

```
                    ┌──────────────┐
                    │   LLM 大脑   │
                    └──────┬───────┘
            ┌──────────────┼──────────────┐
     ┌──────▼──────┐ ┌─────▼─────┐ ┌──────▼──────┐
     │  Planning   │ │  Memory   │ │  Tool Use   │
     └─────────────┘ └───────────┘ └─────────────┘
```

### 1.3.1 Planning（规划）

**定义**：把"要做什么"变成"先做什么、再做什么"的能力。

| 策略 | 一句话解释 | 适合什么任务 |
|---|---|---|
| Chain of Thought（CoT） | 让模型把推理步骤写出来 | 数学、逻辑、需要解释的判断题 |
| Tree of Thoughts（ToT） | 同时展开多条思路，再选最好的一条 | 开放性问题、方案设计 |
| Plan-and-Solve | 先列完整计划，再按计划执行 | 步骤稳定、可预先枚举的任务 |
| ReAct | 想一步、做一步、看结果，再想下一步 | 需要外部信息、步骤会变的任务 |

**规划一定要写成代码吗？** 不一定。很多框架会实现 Planner、TaskGraph、状态机；nanobot 的选择是**不实现独立规划器**，让模型在每一轮自己决定下一步。你在 `nanobot/agent/` 下找不到 `planner.py`，能找到的是循环（`runner.py`）和上下文拼装（`context.py`）。

那 nanobot 里的 `create_goal` / `update_goal` 算规划器吗？不算。它们（`nanobot/agent/tools/long_task.py:159`、`:290`）做的是把"持续目标"记进会话状态，之后每轮把目标提醒注入上下文，防止长任务跑着跑着忘了目标。**它是目标备忘录，不是任务分解器。**

> 类比：规划器像项目经理画甘特图；nanobot 更像给助理一张写着"本周目标"的便利贴，具体今天先干哪件，助理自己判断。

### 1.3.2 Memory（记忆）

**定义**：把当前对话之外的信息保存下来，并在需要时取回。

| 记忆类型 | 存什么 | 容量 | 典型实现 |
|---|---|---|---|
| 感觉记忆 | 刚收到的原始输入 | 几秒 | 消息对象 |
| 短期/工作记忆 | 当前任务的上下文 | 受上下文窗口限制 | 消息列表 |
| 长期记忆 | 跨会话的事实、偏好、决策 | 理论上不限 | 文件、数据库、向量库 |

长期记忆常见的四种做法：文件（人可读、无需基础设施）、向量数据库（按语义检索）、知识图谱（实体 + 关系）、关系型数据库（结构化数据）。nanobot 选的是文件，结构是**三个文件 + 两个阶段**：

- `SOUL.md`（人格）、`USER.md`（用户画像）、`memory/MEMORY.md`（长期事实）+ `memory/history.jsonl`（历史日志）。
- **回合结束后**：`Consolidator`（`nanobot/agent/memory.py:1072`）把摘要**追加**到 `history.jsonl`。
- **定时（Dream）**：`build_dream_prompt`（`nanobot/agent/memory.py:543`）读取还没处理过的历史增量，连同 `SOUL.md`、`USER.md`、`MEMORY.md` 一起交给模型，让模型对长期文件做**最小改动**，并交给 Git 记录版本。

两个容易记错的点：

1. **没有 `save_memory` 这个工具。** v0.3.5 不存在"虚拟工具 save_memory"，模型想记住什么，是在 Dream 阶段通过读写文件工具完成的。
2. **压缩不等于写 `MEMORY.md`。** 回合后的 `Consolidator` 只写 `history.jsonl`；只有 Dream 才会动 `MEMORY.md`。

> 类比：`history.jsonl` 是行车记录仪，什么都记；`MEMORY.md` 是贴在挡风玻璃上的便签，只写"以后每天都用得上"的事。Dream 就是每晚停车后，看一遍录像，把值得记的事抄到便签上。

### 1.3.3 Tool Use（工具使用）

**定义**：让模型能调用外部函数，从而读取信息、改变世界。这是 Agent 与纯 LLM 最硬的分界线。

工具调用的完整链路：

```
1. 运行时把工具的"名称 + 功能描述 + 参数 schema"放进请求
2. 模型判断需要某个工具，输出一段结构化调用请求（JSON）
3. 运行时解析请求，找到对应工具
4. 执行工具，拿到结果（字符串/对象/错误）
5. 把结果作为一条 tool 消息追加进对话
6. 模型看到结果，继续推理或给出最终答案
```

调用请求就是一段 JSON，例如 `{"name": "read_file", "arguments": {"path": "nanobot/agent/runner.py", "offset": 420, "limit": 40}}`。

**工具从哪来？** 两种：

- **内置工具**：v0.3.5 有 24 个内置工具名，如 `read_file`、`write_file`、`edit_file`、`list_dir`、`find_files`、`grep`、`apply_patch`、`exec`、`exec_session`、`list_exec_sessions`、`message`、`spawn`、`cron`、`generate_image`、`my`、`run_cli_app`、`create_goal`、`update_goal`、`web_search`、`web_fetch`，以及 4 个会话类工具。
- **MCP 工具**：通过 Model Context Protocol 接进来的外部工具，注册后名字变成 `mcp_<server>_<tool>`（`nanobot/agent/tools/mcp.py:609`）。

**不是所有工具都对所有 Agent 开放。** `ToolLoader.load` 会按 `scope` 过滤（`nanobot/agent/tools/loader.py:100`）：子 Agent 拿不到 `spawn`、`message`、`cron` 这类只给主 Agent 的工具，避免"子 Agent 再 spawn 子 Agent"的无限递归。

> 类比：工具是公司里的各种系统权限。实习生（子 Agent）能查资料、能改文件，但不一定有权限发全员邮件（`message`）或再招一个实习生（`spawn`）。

**三者的关系**：Planning 决定"调用哪个工具"，Tool Use 产生"新的观察"，Memory 决定"哪些观察值得留下"，下一轮 Planning 又会用到 Memory 和观察。三者通过循环咬合，缺一环系统就会退化。

---

## 1.4 ReAct 框架详解

### 什么是 ReAct

**ReAct = Reasoning + Acting**，2022 年由 Yao 等人在论文《ReAct: Synergizing Reasoning and Acting in Language Models》中提出。它的核心不是"让模型更聪明"，而是**让推理和行动交替发生**：

```
┌───────────────────────────────────────────────────────┐
│   ┌──────────┐    ┌──────────┐    ┌─────────────┐    │
│   │ Thought  │───→│  Action  │───→│ Observation │    │
│   └──────────┘    └──────────┘    └──────┬──────┘    │
│        ↑                                  │           │
│        └──────────────────────────────────┘           │
│   停止条件：模型不再请求工具 / 达到迭代上限            │
└───────────────────────────────────────────────────────┘
```

在 ReAct 之前，两条路都有短板：只推理（CoT）无法获取新信息，容易"一本正经地编"；只行动（Act-Only）机械调工具，不复盘结果，错了也发现不了。ReAct 把两者接起来：**思考指导行动，观察修正思考**。

### 一个完整例子

用户："帮我看看明天从杭州到上海的高铁，哪个时间段最合适？"

```
=== 第 1 轮 ===
Thought : 先查车次，需要日期和出发到达城市。
Action  : search_trains(from="杭州", to="上海", date="明天")
Observation: 共 137 趟；早 7 点、8 点、9 点各有 5 趟以上。

=== 第 2 轮 ===
Thought : 用户没说时间偏好，给出"早/中/晚"三档，并提醒早晚高峰。
Action  : get_train_detail(train_id="G7301")
Observation: 07:12 出发，08:05 到达，二等座有票。
Final Answer: 信息够了，按时间段整理，并注明"余票随时变化"。
```

注意：**Thought 不是必须被打印出来的字段**。工程实现里，"思考"通常就是模型的一次普通文本输出；真正被程序解析的是它附带的工具调用。

### 在 nanobot 里，ReAct 到底在哪

旧文档给了一段"简化伪代码"，还写着"最多 40 次"。v0.3.5 的真实实现分三层：

**第一层：真正跑循环的地方是 `_run_core`（`nanobot/agent/runner.py:386`）。** 它维护消息列表、调用模型、执行工具、处理重试和流式输出，最后返回 `AgentRunResult`。

**第二层：循环本体。**

```python
# nanobot/agent/runner.py:435（节选）
for iteration in range(spec.max_iterations):
    ...
    response, raw_usage = await self._request_model(...)   # Thought：请求模型
    if response.should_execute_tools:                      # 模型要工具
        ...
        results, new_events = await execute_tool_calls(    # Action：执行
            spec.tools,
            response.tool_calls,
            concurrent=spec.concurrent_tools,
            ...
        )
        ...
        messages.append(tool_message)                      # Observation：回灌
        continue                                            # 下一轮
```

对应行号：`:448` 请求模型、`:512` 执行工具、`:544` 把结果追加为 tool 消息、`:579` `continue` 进入下一轮；工具分发在 `nanobot/agent/tools/execution.py:56`。

**第三层：上限从哪来。**

```python
# nanobot/agent/loop.py:1170-1174（节选）
result = await self.runner.run(AgentRunSpec(
    ...
    max_iterations=self.max_iterations,
```

而 `self.max_iterations` 的默认值来自配置：

```python
# nanobot/config/schema.py:129
max_tool_iterations: int = 200
```

所以正确说法是：**v0.3.5 默认最多 200 轮工具迭代**（可配置），不是 40 轮。子 Agent 也不再是"15 轮"，而是继承主 Agent 的值（`nanobot/agent/subagent.py:140-144`）。

**什么时候停？** 当某一轮响应里**没有**工具调用时，`should_execute_tools` 为假，代码走到"产出最终内容"的分支（`runner.py:588` 附近），循环结束。还有几种兜底：空回复重试、输出截断后的续写、达到上限后的收尾。记住"停止条件不是模型说'我完成了'，而是模型不再要工具"。

### ReAct 与其他范式对比

| 范式 | 运行方式 | 优点 | 代价 |
|---|---|---|---|
| CoT | 只在脑内推理 | 便宜、快 | 无法获取新信息 |
| Act-Only | 直接调工具 | 实现简单 | 不会根据结果调整 |
| ReAct | 思考与行动交替 | 能纠错、能探索 | 轮次多、token 贵 |
| Plan-then-Act | 先出完整计划再执行 | 全局视角、可审查 | 计划容易和现实脱节 |
| Reflexion | ReAct + 失败复盘 | 能从错误中学习 | 更慢、更复杂 |

---

## 1.5 Agent 工作流程

### 通用流程：六个阶段

```
用户输入
   ↓
① 感知：接收消息、识别来源、解析附件
   ↓
② 取上下文：读长期记忆、取历史、加载技能说明
   ↓
③ 规划：模型判断任务类型，决定第一步
   ↓
④ 执行循环（ReAct）：调用工具 → 看结果 → 再判断
   ↓
⑤ 落盘：保存本回合消息、更新摘要/记忆 → ⑥ 响应：发回来源渠道
```

### 在 nanobot 里，一条消息是这样走的

```
Channel（channels/*/runtime.py）
  → MessageBus.publish_inbound            nanobot/bus/queue.py:37
  → AgentLoop.run                          nanobot/agent/loop.py:1261
  → AgentLoop._process_message             nanobot/agent/loop.py:1594
  → ContextBuilder.build_system_prompt     nanobot/agent/context.py:101
  → ContextBuilder.build_messages          nanobot/agent/context.py:231
  → AgentRunner.run                        nanobot/agent/runner.py:308
  → AgentRunner._run_core                  nanobot/agent/runner.py:386
       └ ReAct 循环                        nanobot/agent/runner.py:435
  → AgentLoop._persist_turn                nanobot/agent/loop.py:2014
  → MessageBus.publish_outbound            nanobot/bus/queue.py:45
  → Channel.send                           nanobot/channels/base.py:92
```

把它想成一家餐厅：`Channel` 是前台，`MessageBus` 是传菜口，`AgentLoop` 是店长，`AgentRunner` 是厨师，Tools 是冰箱和备菜，Memory 是老顾客的口味本。

**为什么要分"店长"和"厨师"？** 店长要管很多和做菜无关的事：这桌是不是 VIP、要不要拼桌、几点打烊、客人中途加菜怎么办；厨师只关心"这道菜怎么做"。代码里也一样：`AgentLoop` 管会话键、渠道、流式输出、权限、落盘、后台任务；`AgentRunner` 管消息列表、模型请求、工具执行、重试。

### 一个回合的边界

1. **开始**：`_process_message` 收到一条 `InboundMessage`，算出会话键（默认 `{channel}:{chat_id}`，`nanobot/bus/events.py:40`）。
2. **准备**：`_build_turn`（`nanobot/agent/loop.py:1865`）准备会话、运行时、历史、待处理摘要。
3. **执行**：`_run_turn`（`:1974`）调用 `_run_agent_loop`，最终进入 `AgentRunner.run`。
4. **收尾**：`_persist_turn`（`:2014`）把消息写回会话，并触发后续的摘要/记忆流程。
5. **回复**：`OutboundMessage` 回到总线，由对应渠道发送。

落盘发生在回复之前/同时，而不是"聊完再存"。这样即使发送失败，对话历史也不会丢。

### 上下文是怎么拼出来的

```
system prompt（身份/工具契约/MEMORY.md/技能清单）
+ 历史消息（可能被压缩）
+ 当前用户消息（+ 图片/文件）
+ 本轮新产生的 assistant / tool 消息
```

`build_system_prompt`（`nanobot/agent/context.py:101`）负责上半部分，`build_messages`（`:231`）负责把历史和当前消息拼成模型可用的列表。所以"记忆"不是玄学：**它最终就是一段被拼进 prompt 的文本**。

### 一个容易忽略的点：工具结果是"消息"，不是"返回值"

工具执行完，结果不会直接变成"函数返回值"，而是被包装成一条 `role="tool"` 的消息，追加进对话列表（`runner.py:533-544`），下一轮模型像读聊天记录一样读到它。这个设计让工具调用完全复用对话协议，不需要额外的状态机。

---

## 1.6 主流 Agent 框架对比

先说明口径：**除 nanobot 的数字外，下表其他框架的代码量只给量级**。它们版本迭代快，精确行数没有比较意义；Star 数也随时间变化，本章不写死。

| 维度 | **Nanobot** | **LangChain / LangGraph** | **CrewAI** | **AutoGPT** |
|---|---|---|---|---|
| 开发者 | 香港大学 HKUDS | LangChain 团队 | CrewAI 团队 | Significant Gravitas |
| 语言 | Python（+ TS WebUI） | Python / JS | Python | Python |
| 代码规模 | **99,398 行 SLOC / 329 个非测试 py 文件** | 数十万行量级 | 数万行量级 | 十万行量级 |
| 核心定位 | 可自托管的个人 Agent 运行时 | Agent 开发框架/编排库 | 多 Agent 角色协作 | 自主通用 Agent 实验 |
| MCP 支持 | 原生（`tools.mcpServers`） | 通过集成包 | 有限 | 无原生 |
| 记忆 | 文件：`MEMORY.md` + `history.jsonl` + Dream | 多种后端可选 | 内置短期记忆 | 内置 |
| 渠道 | **17 个内置渠道** | 需自行集成 | 无 | Web UI 为主 |
| 配置 | **JSON**（`~/.nanobot/config.json`） | 代码为主 | 代码 + 装饰器 | 代码 + 环境变量 |
| 学习曲线 | 中（核心清晰，外围大） | 高（抽象多） | 中 | 中 |

### 怎么理解 nanobot 的"轻"

v0.3.5 的 README 仍然自称 *ultra-lightweight*，但它**已经不是 4,000 行**。这两个说法不矛盾，只是"轻"的定义变了：

- **部署轻**：一个 Python 进程 + 一个 JSON 配置 + 一个 workspace 目录就能跑，不需要数据库、消息队列、容器编排。
- **核心轻**：真正决定 Agent 行为的链路（Channel → Bus → Loop → Runner → Tools → Bus）只有几个类，主干约 6,500 行。
- **外围重**：17 个渠道、46 个 Provider 条目、WebUI、TUI、OpenAI 兼容 API、插件系统，把总行数推到近十万。

所以面试时不要说"它只有 4000 行"，而要说"**它的核心循环很小，外围适配很多；规模增长主要来自渠道和 Provider**"。

### 各框架的取舍

**Nanobot**：优势是"一条消息的生命周期"能从头读到尾，且自带渠道、记忆、定时任务、MCP，适合学习"完整系统怎么落地"。局限是生态和第三方集成不如 LangChain，复杂多 Agent 编排也不是它的主战场。

**LangChain / LangGraph**：集成极多、抽象齐全，适合快速接各种模型和向量库；代价是抽象层多，读源码容易迷路。LangGraph 用图结构表达有状态流程，适合需要明确分支和回滚的场景。

**CrewAI**：把"角色 + 任务 + 协作"做得直观，适合多 Agent 分工；单 Agent 场景会显得重。**AutoGPT**：历史意义大于工程意义，它让"自主循环"出圈，但长程自主的可靠性和成本问题至今没有完全解决。

### 面试推荐说法

> "我重点读了 nanobot，因为它把 Agent 的完整生命周期摊开给你看：消息怎么进来、上下文怎么拼、循环怎么跑、工具怎么执行、记忆怎么落盘。它 v0.3.5 大约 99,398 行，但核心链路很集中，读完 `loop.py` 和 `runner.py` 就能抓住主干。LangChain 我也用过，它的优势是集成和抽象，但学习时容易停留在'会调 API'，不容易理解系统整体是怎么闭合的。"

---

## 1.7 Agent 的发展历史

```
2022
├── 01月  Chain-of-Thought：让模型"写出推理步骤"
└── 10月  ReAct：推理与行动交替，Agent 的理论雏形

2023
├── 03月  AutoGPT 开源，引爆"自主 Agent"热潮
├── 06月  Lilian Weng 博文系统总结 Planning / Memory / Tool Use
└── 11月  OpenAI 推出 GPTs，Agent 走向大众

2024
├── 06月  Function Calling 成为主流模型标配
└── 11月  Anthropic 发布 MCP，工具接入开始标准化

2025-2026
├── 2025   MCP 生态扩张；企业级 Agent 平台与"运行时"概念兴起
├── 2026-02  HKUDS/nanobot 首次发布（v0.1.x）
└── 2026-09  nanobot v0.3.5（本套文档对应的版本）
```

三个阶段的主线：**2022-2023 范式确立**（ReAct + 三要素）；**2024-2025 接口标准化**（Function Calling 与 MCP 让工具接入不再私有化）；**2025-2026 工程化**（上下文压缩、权限边界、沙箱、可观测性、成本统计成为重点，nanobot v0.3.5 里大量治理代码正是这一趋势的体现）。

> 旧文档的"4000 行"来自仓库根目录早期一个只统计 `nanobot/agent/*.py` 顶层文件的脚本，不是全仓库规模；v0.3.5 已经不在这个量级。

---

## 1.8 面试话术

### 话术一：30 秒解释什么是 Agent

> "Agent 是把大模型放进一个循环里，再给它工具和记忆的系统。模型负责判断下一步，循环负责让它能反复尝试，工具让它能真正读取信息和改变世界，记忆让它跨会话不忘事。所以它不是'更聪明的模型'，而是'模型 + 运行时'。"

### 话术二：Agent 和 LLM 最大的区别

> "最大的区别是**谁来决定下一步**。纯 LLM 的下一步由用户在 prompt 里规定；Agent 的下一步由模型在每一轮根据工具结果自己决定。工程上这意味着你需要一个循环、一套工具协议、一套上下文管理，还要有上限和兜底，否则它会一直转下去。"

### 话术三：为什么选 nanobot 学习

> "因为它把 Agent 的完整链路做全了，而且核心链路能读穿。v0.3.5 有 99,398 行，听起来不少，但真正的主干是 `AgentLoop`（回合编排，`loop.py:1594`）和 `AgentRunner`（模型/工具循环，`runner.py:386` 起），再加上工具、记忆、渠道几个模块。读它最大的收获不是背 API，而是理解'一条消息从进来到出去'的每一层为什么存在。"

---

## 1.9 面试常考点总结

| 编号 | 考点 | 频率 | 难度 |
|---|---|---|---|
| 1 | Agent 与 LLM 的区别 | ⭐⭐⭐⭐⭐ | 低 |
| 2 | Agent 三要素 | ⭐⭐⭐⭐⭐ | 低 |
| 3 | ReAct 原理与停止条件 | ⭐⭐⭐⭐ | 中 |
| 4 | 工具调用的完整链路 | ⭐⭐⭐⭐ | 中 |
| 5 | 记忆分层与长期记忆方案 | ⭐⭐⭐⭐ | 中 |
| 6 | 上下文窗口管理与压缩 | ⭐⭐⭐⭐ | 高 |
| 7 | 多 Agent / 子 Agent 的边界 | ⭐⭐⭐ | 高 |
| 8 | Agent 的安全边界（沙箱、权限） | ⭐⭐⭐ | 高 |
| 9 | Agent 的评测与可观测性 | ⭐⭐ | 高 |
| 10 | 框架选型与设计取舍 | ⭐⭐⭐ | 中 |

### 必须能回答的 3 个问题

**Q1：Agent 与 LLM 有什么区别？**

> 见 [1.1](#11-ai-agent-的定义) 和 [1.2](#12-llm-vs-ai-agent本质区别)。一句话：Agent 是"模型 + 循环 + 工具 + 记忆 + 运行时"，LLM 只是其中的推理部件。

**Q2：三要素分别怎么实现？**

> Planning：把目标、约束、可用工具写进 prompt，让模型每轮自定下一步；Memory：短期靠上下文，长期靠外部存储（nanobot 是 `MEMORY.md` + `history.jsonl` + Dream）；Tool Use：工具注册表 + 结构化调用 + 结果回灌。

**Q3：ReAct 的循环什么时候停？**

> 模型返回的响应不再包含工具调用时停；工程上还有迭代上限（nanobot 默认 200）、空回复重试、输出截断续写等兜底。

---

## 1.10 练习题与思考题

### 基础题

**1. 填空**：Agent 的三要素是 ______、______、______。

> 答案：Planning（规划）、Memory（记忆）、Tool Use（工具使用）。

**2. 判断**："Agent 就是一个参数更多的 LLM。" 对还是错？

> 错。Agent 是系统，核心是模型外面的循环、工具和记忆。换一个更强的模型不会自动得到 Agent。

**3. 简答**：用一句话解释 ReAct 中 Thought、Action、Observation 分别是什么。

> Thought：模型对"现在该干什么"的判断；Action：一次工具调用请求；Observation：工具执行后回灌给模型的结果。

### 进阶题

**4. 读代码**：打开 `nanobot/agent/runner.py`，找到 `for iteration in range(spec.max_iterations)`，回答：循环在哪个函数里？默认上限是多少？工具执行完后靠什么进入下一轮？

> 参考：在 `_run_core`（`:386`）里；默认 200（`nanobot/config/schema.py:129`）；工具结果 append 成 `role="tool"` 消息后 `continue`（`runner.py:579`）。

**5. 对比**：为什么 nanobot 不实现 Planner/TaskGraph，而让模型自己规划？说出至少一个优点和一个缺点。

> 优点：实现简单，能随工具结果动态调整，不需要预先把任务拆成固定图。缺点：不可预测、token 成本高、难以静态审查计划。

**6. 场景设计**：用户说"帮我调研 2026 年最流行的 5 个 Python Web 框架，写成对比报告"。请写出前三轮的 Thought / Action / Observation。

> 提示：第一轮搜索候选；第二轮逐个抓文档；第三轮补数据或开始写。每轮都要说明"为什么现在做这个"。

### 思考题

**7.** 如果 Agent 调用了错误的工具，系统应该在哪些层做兜底？（提示：参数校验、工具错误回灌、迭代上限、人工确认。）

**8.** 为什么 nanobot 要把"回合编排"和"模型/工具循环"拆成两个类？如果合并成一个类，会遇到什么问题？

---

## 本章总结

```
1. Agent = LLM + 循环 + 工具 + 记忆 + 运行时
   → 不是"更大的模型"，是"模型外面的系统"

2. 三要素：Planning / Memory / Tool Use
   → 通过循环咬合，不能孤立理解

3. ReAct：Thought → Action → Observation → ...
   → 停止条件是"模型不再请求工具"

4. nanobot：编排 loop.py:1594，循环 runner.py:386/435
   → 迭代上限默认 200（schema.py:129）

5. 规模：99,398 行 SLOC / 329 个非测试 py 文件
```

**自检清单**

- [ ] 能说出 Agent 与 LLM 的三个区别
- [ ] 能画出 ReAct 循环并说明停止条件
- [ ] 能说出 nanobot 里"谁负责编排、谁负责循环"
- [ ] 能解释为什么"工具结果是一条消息"
- [ ] 能纠正"4000 行 / 40 轮 / save_memory"

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 约 4,000 行 | 99,398 行 SLOC / 329 个非测试 py 文件 | `_work/facts-v035.md` §1；对 `nanobot/` 包按 SLOC 口径统计 | 讲"核心链路小、外围适配多" |
| 最多 40 轮迭代 | 默认 `max_tool_iterations=200` | `nanobot/config/schema.py:129` | 说 200，并说明可配置 |
| 子 Agent 15 轮 | 继承主 Agent 的 200 | `nanobot/agent/subagent.py:140-144` | 不讲"15 轮" |
| `AgentRunner` 一个类包办 | 拆成 `AgentLoop`（编排）+ `AgentRunner`（循环） | `nanobot/agent/loop.py:196`、`runner.py:141` | 讲两层分工 |
| 记忆是 `MEMORY.md` + `HISTORY.md` | `MEMORY.md` + `history.jsonl` + Dream | `nanobot/agent/memory.py:76-78`、`:543` | 讲三文件两阶段 |
| 有 `save_memory` 虚拟工具 | 不存在该工具 | `nanobot/agent/tools/` 下无对应工具类 | 改为 Dream 写长期文件 |
| 配置是 YAML | JSON `~/.nanobot/config.json` | `nanobot/config/loader.py:39` | 第 02/06 章展开 |

---

## 附录 B：进阶理解

**1. 为什么把"编排"和"循环"分开？** `AgentLoop` 要处理渠道、会话、权限、流式输出、落盘这些"产品问题"；`AgentRunner` 只处理"消息列表怎么变"。分开之后，Dream、子 Agent、CLI 单次问答都能复用同一个 Runner，而不用各自复制一份循环。这是把易变部分和稳定部分隔开的典型做法。

**2. 为什么停止条件是"不再请求工具"，而不是"模型说完成"？** 自然语言里的"完成了"不可靠，模型可能一边说完成一边还留着未执行的调用。而"响应里有没有 tool_calls"是结构化字段，程序可以精确判断。Agent 的可靠性很大程度上来自这种"把语义判断换成结构判断"的设计。

**3. 为什么工具结果要伪装成一条消息？** 因为主流模型 API 的对话协议本来就支持 `role="tool"`。复用协议意味着：不需要为工具结果设计新的传输格式，历史记录天然可回放，压缩/截断逻辑也能统一处理。代价是工具结果会占用上下文，所以才有 `max_tool_result_chars=16000` 之类的治理参数（`nanobot/agent/context_governance.py:119`）。

**4. 为什么 nanobot 不给每个任务写 Planner？** 因为 Planner 把"计划"固化成了代码里的结构，而工具返回往往会推翻计划。把规划留给模型，换来灵活性；代价是不可预测和成本上升。这是一个明确的取舍，不是"没想到要做"。

**5. 为什么长期记忆用 Markdown、为什么上下文治理是"第二战场"？** 个人 Agent 的记忆量小、要求可读可改，Markdown 可以被用户直接打开、编辑、用 Git 追踪；向量库则引入额外服务、embedding 成本和"检索不准"的新问题。而每一轮请求都会把历史、工具结果、记忆重新塞回去，轮次一多 token 会爆炸——所以真正决定 Agent 能不能长时间工作的，不是模型多聪明，而是"什么时候压缩、压缩什么、压缩后还能不能接着干"。这也是 `context_governance.py`、`autocompact.py` 存在的原因。

---

## 附录 C：Python 语法加油站

### C.1 `dataclass(slots=True)`：给配置对象"瘦身"

```python
from dataclasses import dataclass

@dataclass(slots=True)
class AgentRunSpec:
    max_iterations: int
    session_key: str | None = None

spec = AgentRunSpec(max_iterations=200)
print(spec.max_iterations)  # 200
```

- **是什么**：`@dataclass` 自动生成 `__init__`、`__repr__`；`slots=True` 让实例只能有声明过的字段，省内存、防手滑拼错。
- **在哪用**：`nanobot/agent/runner.py:89` 的 `AgentRunSpec`。
- **怎么读**：把 dataclass 当成"带类型的字典"。看到 `spec.xxx` 就去类定义里找同名字段。

### C.2 `async` / `await`：一个人怎么同时接待多桌客人

```python
import asyncio

async def fetch(name: str) -> str:
    await asyncio.sleep(0.1)      # 模拟等网络，不占线程
    return f"{name} done"

async def main() -> None:
    results = await asyncio.gather(fetch("a"), fetch("b"))
    print(results)                # ['a done', 'b done']

asyncio.run(main())
```

- **是什么**：`async def` 定义协程；`await` 表示"这里要等，但等的期间可以去干别的"。
- **在哪用**：`AgentRunner.run`、`_run_core`、`execute_tool_calls`、各渠道的 `send` 全是异步。
- **怎么读**：看到 `await self._request_model(...)`，就理解成"发一次模型请求，等它回来"；不要纠结事件循环细节。

### C.3 `ABC` + `@property`：定义"工具必须长什么样"

```python
from abc import ABC, abstractmethod

class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

class ReadFileTool(Tool):
    @property
    def name(self) -> str:
        return "read_file"
```

- **是什么**：抽象基类规定子类必须实现哪些成员；`@property` 让方法可以像属性一样访问。
- **在哪用**：`nanobot/agent/tools/base.py:159` 的 `Tool`，每个工具实现 `name` / `description` / `parameters` / `execute`。
- **怎么读**：想知道模型看到的工具名叫什么，直接找子类里的 `return "..."`。

---

## 附录 D：原文勘误

> 说明：附录 A 记"版本变了"，附录 D 记"旧文档本身写错了"。以下按旧文档小节列出。

**D.1（1.4 节 / 1.6 节 / 1.8 节）"最多 40 次迭代"**

- **错在哪**：把 v0.1.x 时代的默认值当成了永久事实，而且写成了固定值。
- **正确说法**：v0.3.5 默认 `max_tool_iterations=200`，可通过 `agents.defaults.maxToolIterations` 配置。
- **证据**：`nanobot/config/schema.py:129`；主 Agent 在 `nanobot/agent/loop.py:1174` 把它传进 `AgentRunSpec`。

**D.2（1.3.2 节 / 1.10 节）"Nanobot 用 save_memory 虚拟工具写 MEMORY.md"**

- **错在哪**：v0.3.5 没有名为 `save_memory` 的工具，也没有"虚拟工具直接写记忆"的机制。
- **正确说法**：回合后的 `Consolidator` 把摘要追加到 `history.jsonl`；Dream 阶段再由模型用文件工具对 `MEMORY.md` 做最小改动。
- **证据**：`nanobot/agent/memory.py:1072`（`Consolidator`）、`:543`（`build_dream_prompt`）、`:76`（`MEMORY.md` 路径）。

**D.3（1.3.2 节）"长期记忆 = MEMORY.md + HISTORY.md"**

- **错在哪**：`HISTORY.md` 在 v0.3.5 只是**迁移来源**，不是运行时使用的文件。
- **正确说法**：运行时是 `memory/MEMORY.md` + `memory/history.jsonl`；首次发现旧 `HISTORY.md` 时会迁移成 JSONL。
- **证据**：`nanobot/agent/memory.py:78`（`legacy_history_file`）、`:106`（迁移函数）、`:77`（`history_file`）。

**D.4（1.6 节表格）"Nanobot 代码量 ~4,000 行，是 LangChain 的 1/125"**

- **错在哪**：4,000 行的口径只统计 `nanobot/agent/*.py` 顶层文件，不含 `agent/tools/`、`providers/`、`channels/`、`cli/`；用这个数字和其他框架的全量代码比，属于拿苹果比西瓜。
- **正确说法**：v0.3.5 约 99,398 行 SLOC / 329 个非测试 py 文件；"轻"体现在部署形态和核心链路，而不是总行数。
- **证据**：`_work/facts-v035.md` §1；本仓库 `nanobot/` 包统计；旧口径来源为早期仓库根目录的 `core_agent_lines.sh`（v0.3.5 已无此文件）。

**D.5（1.3.3 节）"子 Agent 的工具集只是少几个"**

- **错在哪**：旧文档没有说明过滤机制，读者容易以为"少几个工具"是产品策略。
- **正确说法**：过滤是**代码级**的：`ToolLoader.load` 按 `scope` 跳过不匹配的工具类；`spawn`/`message`/`cron` 等只在 `core` scope 注册。
- **证据**：`nanobot/agent/tools/loader.py:100`；`spawn` 工具名见 `nanobot/agent/tools/spawn.py:63`。

---

## 下一章

概念打完之后，下一步是把这个项目"摸清楚"：它到底有多大、由哪些模块组成、17 个渠道和 46 个 Provider 是怎么组织的、为什么配置是 JSON 而不是 YAML。

➡️ [02 - Nanobot 项目概览](../02-nanobot-overview/README.md)

如果想直接看代码，建议先打开 `nanobot/agent/runner.py` 的 `_run_core`（第 386 行），对照本章 1.4 节读一遍循环。

---

> 📝 **本章小结**：Agent 的本质是"模型 + 循环 + 工具 + 记忆 + 运行时"。ReAct 提供了循环的范式，三要素提供了能力的分类。落到 nanobot v0.3.5：回合编排在 `loop.py:1594`，循环在 `runner.py:386/435`，上限 200 轮，规模 99,398 行。记住这三个数字和两个类，后面的章节都是在它们周围展开。
