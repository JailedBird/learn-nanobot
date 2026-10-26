# 16 - STAR 面试话术与模拟面试（v0.3.5 版）

> 🎯 **本章目标**：把你读过的 nanobot 源码变成"能在 90 秒内讲清、被追问三层还不塌"的口头表达——给你 3 份完整话术（项目介绍 / 一次改动 / 一次失败复盘），每份后面配 3 个真实追问，并教你用 `nanobot/...:行号` 给自己上证据。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，commit `1bb712d3`，2026-09-16）
> **前置知识**：[03 - 架构深潜](../03-architecture-deep-dive/README.md)、[08 - 技能与工具](../08-skills-and-tools/README.md)、[13 - 面试八股文](../13-interview-bagua/README.md)
> **阅读时间**：约 1 小时（建议朗读两遍，把书面语改成你自己的口气）

> ⚠️ **数据诚实条款（先看这条）**：本章出现的"日均 500 次对话""准确率 78% → 95%""成本降低 25%"之类数字，**全部是格式占位示例，不是 nanobot 项目的真实数据**。
> **示例成果数字不可虚构，投递前必须替换为你自己的真实数据**；如果没有真实数据，就删掉那一句，只讲"机制 + 你的判断 + 边界"。面试官一旦顺着数字往下问"怎么测的、样本多大"，编的数字会当场崩。

---

## 目录

- [一、STAR 面试法介绍](#一star-面试法介绍)
- [二、话术一：90 秒项目介绍](#二话术一90-秒项目介绍)
- [三、话术一展开：标准版项目 STAR](#三话术一展开标准版项目-star)
- [四、话术二：一次改动的深挖故事](#四话术二一次改动的深挖故事)
- [五、话术三：失败与复盘](#五话术三失败与复盘)
- [六、高频追问的分类与答法](#六高频追问的分类与答法)
- [七、模拟面试片段](#七模拟面试片段)
- [八、面试技巧与表达节奏](#八面试技巧与表达节奏)
- [九、数据诚实与投递前自检](#九数据诚实与投递前自检)
- [附录 A：与旧版文档的差异](#附录-a与旧版文档的差异)
- [附录 B：进阶理解](#附录-b进阶理解)
- [附录 C：如何把技术细节讲成故事](#附录-c如何把技术细节讲成故事)
- [附录 D：原文勘误](#附录-d原文勘误)

---

![面试场景漫画](../../../comics/07-interview-scene.png)

*STAR 不是背稿，是给面试官一条"能跟着走"的路*

![STAR 面试法详解漫画](../../../comics/08-star-method.png)

*S → T → A → R：前 30 秒定框架，中间 60 秒给证据，最后 20 秒给结果*

## 一、STAR 面试法介绍

### 1.1 STAR 是什么，以及它为什么对你特别有用

| 要素 | 含义 | 时间占比 | 常见错误 |
|---|---|---|---|
| **S** Situation | 在什么场景下 | 15% | 讲成公司介绍，30 秒还没进入正题 |
| **T** Task | 你要解决什么 | 15% | 说"我参与了"，不说"我负责哪一块" |
| **A** Action | 你具体做了什么 | 50% | 全是形容词（"深入优化""全面提升"），没有动作 |
| **R** Result | 产生了什么效果 | 20% | 数字是编的，或者只有数字没有代价 |

对你（读了源码但没在生产环境跑过 nanobot）来说，STAR 的真正价值不是"包装"，而是**逼你把"我读过"翻译成"我判断过"**。读过源码的人最容易犯的错是复述架构（"它有个 AgentLoop"），面试官听完不知道你会干什么。好的 STAR 一定落在"我发现了什么、我改了什么、我怎么验证"。

### 1.2 三种时长，三种颗粒度

- **90 秒版**：面试官说"简单介绍下这个项目"。只讲一条主链路 + 一个你核过的事实。
- **3-5 分钟版**：面试官说"详细讲讲"。90 秒版 + 你的改动 + 一个坑。
- **任意深度的追问**：每个追问用 mini-STAR（30-60 秒）答，永远"结论 → 机制 → 证据 → 边界"四段。

### 1.3 三条纪律（比话术本身重要）

1. **先说结论，再说过程**。"默认 200 次，子 Agent 继承"比"这个要看情况"强十倍。
2. **每个技术结论挂一个可验证锚点**：文件名、类名、配置键、行号。说不出类名就别说"我读过源码"。
3. **主动交代边界**。"这是 v0.3.5 的行为，更早版本我没核过""这段我只读到调用方，没进实现"——这句话会让你整体可信度上升。

---

## 二、话术一：90 秒项目介绍

> 面试官："简历上写你研究过 nanobot，简单介绍下这个项目和你做的事。"

### 2.1 完整话术（约 90 秒，可直接念）

> 我研究的对象是香港大学 HKUDS 开源的 nanobot，版本是 v0.3.5，2026-09-16 的 tag。它不是玩具 demo，是一个能跑起来的多渠道 Agent 运行时：17 个渠道子包、24 个内置工具、约 329 个 Python 文件、近 10 万行实现代码。
>
> 我最想讲的是它**一条消息的生命周期**，因为这条链路能说明它的设计取舍。消息从渠道进来，先进 `MessageBus`——它不是消息队列中间件，就是两个 `asyncio.Queue`（`nanobot/bus/queue.py:19`），`publish_inbound` 在第 37 行，渠道和 Agent 内核因此完全解耦。
>
> 然后是 `AgentLoop`。它接一条消息后走**七个固定阶段**：restore、compact、command、build、run、save、respond（`nanobot/agent/loop.py:1679-1686`），每段都包了耗时日志，出问题能一眼定位到阶段。真正的模型/工具循环不在 Loop 里，在 `AgentRunner`：`runner.py:435` 是一个 `for iteration in range(spec.max_iterations)` 的循环，默认上限 200（`config/schema.py:129`），每轮解析 `tool_calls`、经 `ToolRegistry.execute`（`tools/registry.py:187`）执行、把结果回灌。
>
> 有两个细节我专门核过，因为很多教程写错了。第一，工具名是**写死的类属性**：`web_search` 和 `web_fetch` 分别在 `tools/web.py:367` 和 `:1114`，搜索后端换成谁都改不了这两个名字。第二，MCP 的连接**不在 Loop 里**——组合根 `cli/gateway_runtime.py:473` 从配置建 `MCPProvider`、`:579` 去 connect、关停时 `:327` 统一 aclose，Loop 只拿到已经注册好的工具。
>
> 我做的事是：按这条链路读源码、给每一环写可复现的验证命令，然后动手改了一处工具可见性——这个待会儿可以展开。

**节奏拆解**：0-15 秒身份与规模；15-55 秒主链路（三个锚点）；55-75 秒两个"反常识细节"（钩子）；75-90 秒交棒给下一问。

### 2.2 可能的 3 个追问 + 回答要点

**追问 1：200 次迭代不危险吗？不怕它一直转？**

> 要点：① 200 是**上限不是均值**，绝大多数任务 3-10 轮就结束；② 真正的闸门不是迭代数，而是**并发**和**能力**——`max_concurrent_subagents` 默认 4（`config/schema.py:130`），而且子 Agent 的工具注册表里根本没有 `spawn`（`agent/tools/loader.py:100` 按 scope 过滤），它想递归都没有这个语法；③ 成本上界还有 `max_tool_result_chars=16000`（`config/schema.py:131`）兜住单次工具结果。**别把"上限大"讲成"一定跑满"**。

**追问 2：MCP 为什么不在 AgentLoop 里连？**

> 要点：连接是**进程级资源**，回合是**请求级动作**。MCP 要 spawn 子进程、握手、维持 SSE 连接，如果每个回合建一次，会重复握手、泄漏子进程，而且没法做热重载。放在组合根，还能和网关启停顺序对齐（关停时 `gateway_runtime.py:325-327` 里 agent 和 MCP provider 是并列的两项 cleanup）。这是"资源拥有者"和"使用者的分离"，不是随手放的。

**追问 3：你怎么确认这些行号的？**

> 要点：先说方法再给结论——`git describe --tags` 确认是 `v0.3.5` / `1bb712d3`；然后用**函数名/类名去 grep**，而不是背行号（`grep -n "_process_message" nanobot/agent/loop.py`）；最后承认行号会漂移，所以我在笔记里既记行号也记"锚点文本"。**面试里说"在 loop.py 的 `_process_message` 里"永远比背错一个数字安全。**

### 2.3 坏版本长什么样（对照着改）

| 坏版本 | 为什么坏 | 改成 |
|---|---|---|
| "nanobot 是一个轻量级 Agent 框架，有 Channel、MessageBus、AgentLoop、ContextBuilder、ToolRegistry、MCP、Skills、Memory、Subagent、Cron……" | 报目录。面试官听完只知道你背过 README，不知道你懂哪一环 | 只讲**一条链路**，把其余模块留作追问的钩子 |
| "我先介绍一下什么是 AI Agent……" | 面试官比你懂，这 40 秒是纯损耗 | 直接从"这个项目的一条消息怎么走"开始 |
| "这个项目用了 Python、asyncio、pydantic、httpx……" | 技术栈是简历上的事，不是项目介绍 | 换成"它的并发模型是 **队列 + 每会话锁**，不是线程池" |
| "我读了源码，收获很大" | 没有信息量，也没有可追问点 | 换成"我核过三个流行说法，其中两个是错的" |

**一条自检**：讲完 90 秒，如果面试官只能接"哦，然后呢？"，说明你缺钩子；如果他能接"你刚说 200 次迭代，不怕跑飞吗？"，说明钩子生效了——**好的自我介绍是给面试官递问题，不是把话说完。**

---

## 三、话术一展开：标准版项目 STAR

> 面试官："好，那你把这个项目完整讲一下。"（3-5 分钟）

### 3.1 S / T（约 40 秒）

> **S**：我想找一个"能读完全部源码、又有生产级功能"的 Agent 框架。LangChain 那类框架抽象层太多，读两周还在配置层；纯 API Wrapper 又学不到工程问题（并发、持久化、渠道、恢复）。nanobot 的位置在中间：核心链路清晰，但外围功能是真的——17 个渠道、MCP、子 Agent、定时任务、WebUI、Docker 部署都有。
>
> **T**：我的任务是两条：一是建立"源码级"理解，能对任何一条行为说出它在哪个文件哪个函数；二是在不改坏原设计的前提下做一处真实改动，并留下验证方式。

### 3.2 A（约 2 分钟，三个动作）

> **动作一：画主链路，只画一条。** 我从 `channels/*/runtime.py` 出发，沿着 `MessageBus.publish_inbound`（`bus/queue.py:37`）→ `AgentLoop._process_message`（`loop.py:1594`）→ `ContextBuilder.build_system_prompt`（`agent/context.py:101`）→ `AgentRunner.run`（`runner.py:308`）→ 循环（`:435`）→ `_persist_turn`（`loop.py:2014`）→ `OutboundMessage` 走了一遍，把每个环节的输入输出写成表格。这张表后来成了我回答所有追问的索引。
>
> **动作二：核三个"看起来对、其实错"的流行说法。** ① "nanobot 核心 4000 行"——v0.3.5 光 `agent/loop.py` 就 2,388 行、`agent/runner.py` 1,374 行，这个说法不成立（后面第五节的复盘故事就是这个）；② "子 Agent 15 轮"——不存在，子 Agent 继承 `max_tool_iterations`，默认 200（`agent/subagent.py:140-144`）；③ "记忆压缩会写 `MEMORY.md`"——压缩只往 `history.jsonl` 追加摘要，`MEMORY.md` 归低频的 Dream 管（`agent/memory.py:575` 起是 Dream 的受限工具集）。
>
> **动作三：做一处改动，并写清"为什么只能是这个改法"。** 给我自己加的工具声明可见范围，让子 Agent 能看见它——具体见下一节。

### 3.3 R（约 30 秒，替换成你的真实产出）

> 产出是：一张主链路表、一份"常见错误说法 → 源码证据"对照表、一处可回滚的改动。**这里必须写你自己的真实结果**：你跑通了几次实验、写了多少字笔记、改动有没有被上游接受。没有就直说"这是个人学习项目，产出是笔记和一次改动"，比编"上线服务 X 万用户"安全得多。

### 3.4 可能的 3 个追问 + 回答要点

**追问 1：你说 Loop 和 Runner 分工，那锁在哪一层？为什么？**

> 要点：锁在回合层（`AgentLoop._dispatch` 里的 `async with lock, gate`，`loop.py:1418` 附近），因为"同一个会话不能并发跑两个回合"是**会话语义**，不是模型循环的语义。Runner 层只管一次请求内的工具循环，它可以被复用来跑子 Agent（`SubagentManager` 直接 new 一个 Runner），所以不能在那里绑会话锁。**分工标准是"失效原因"**：回合层坏在消息/会话/投递，循环层坏在模型不听话/工具报错/上下文爆。

**追问 2：上下文装不下时到底发生什么？**

> 要点：分三条路，别混。① 显式 `/compact`；② 空闲压缩 `AutoCompact`——按**空闲时长**判断（`agent/autocompact.py:39-55` 的 `_is_expired`，`session_ttl_minutes <= 0` 时永不触发），不是按 token 比例；③ 回合内压力压缩——`ContextGovernor.request_pressure` 在"实测/估算 token ≥ 预算"时返回测量值（`agent/context_governance.py:415`），预算是 `context_window_tokens - max_tokens - 1024`（`:693-706`，`SNIP_SAFETY_BUFFER=1024` 在 `:67`）。默认 200,000 − 8,192 − 1,024 = **190,784** token 才触发。**能报出这个算式，比说"到 70% 就压缩"强得多。**

**追问 3：工具结果太长会怎样？**

> 要点：不是简单截断丢弃。超 `max_tool_result_chars`（默认 16,000）就**落盘成文件 + 回传路径和 1,200 字符预览**（`utils/helpers.py:367` 的 `_TOOL_RESULT_PREVIEW_CHARS`），模型需要细节再 `read_file`；且 `read_file` 本身豁免这套清洗（`context_governance.py:69` 的 `TOOL_RESULT_OFFLOAD_EXEMPT_TOOLS`），因为它天然分页。**"截断"和"搬家"是两种设计**，后者不丢信息。

---

## 四、话术二：一次改动的深挖故事

> 这是技术面最能加分的一节。核心不是"我加了个功能"，而是"我遇到了一个反直觉现象，查清了机制，改法有取舍"。

### 4.1 现象（S，约 20 秒）

> 我给 nanobot 写了一个自定义工具（假设叫 `ticket_query`，查内部工单）。在主 Agent 的终端对话里能正常调用；但我用 `spawn` 起子 Agent 去查同一个库时，子 Agent 说"没有这个工具"。同一个注册表，为什么一个看得见、一个看不见？

### 4.2 任务（T，约 15 秒）

> 目标有两个：让子 Agent 也能用这个工具；同时**确认**主 Agent 专属的 `spawn`/`message`/`cron` 不会被我顺手泄漏给子 Agent（否则会递归）。

### 4.3 动作（A，约 60 秒，四步，每步都有锚点）

**第一步：找到过滤点。** 工具不是"全都塞给模型"，而是注册时按 scope 过滤：

```python
# nanobot/agent/tools/loader.py:92-101（节选，注释为本文所加）
def load(self, ctx: ToolContext, registry: ToolRegistry, *, scope: str = "core") -> list[str]:
    ...
    for tool_cls in source:
        # ★ 白名单过滤：类的 _scopes 里没有当前 scope 就跳过
        if scope not in getattr(tool_cls, "_scopes", {"core"}):
            continue
```

**第二步：找到默认值。** 基类把默认 scope 定成"只有主 Agent"：

```python
# nanobot/agent/tools/base.py:204-208（节选）
# --- Plugin metadata ---
config_key: str = ""
_plugin_discoverable: bool = True
_scopes: set[str] = {"core"}      # ★ 不声明 = 只对主 Agent 可见
```

我的工具没声明 `_scopes`，所以吃默认值 `{"core"}`——子 Agent 当然看不见。参照 `web_search` / `web_fetch` 的写法（`tools/web.py:365`、`:1112` 都是 `{"core", "subagent"}`），我改成了：

```python
# 我的工具类（示意）
class TicketQueryTool(Tool):
    name = "ticket_query"
    _scopes = {"core", "subagent"}   # ★ 一行改动：进入子 Agent 的白名单
```

**第三步：确认子 Agent 真的按这个 scope 加载。** 不是猜的，`subagent.py:224` 就一行：

```python
# nanobot/agent/subagent.py:224
ToolLoader().load(ctx, registry, scope="subagent")
```

子 Agent 拿到的是**独立的注册表**，只装了 `scope="subagent"` 的工具。这解释了为什么 `spawn` 泄漏不出去：它压根没声明 subagent scope，所以子 Agent 的注册表里不存在它——**不是"运行时禁用了"，是"根本不存在"**。

**第四步：发现第二套机制，避免把两件事讲成一件。** 我以为"可见性都由 scope 管"，结果读记忆模块时发现 Dream 的工具有一套**手搓的**注册表：

```python
# nanobot/agent/memory.py:575-614（节选）
def build_dream_tools(self) -> ToolRegistry:
    ...
    editable_files = [self.memory_file, self.soul_file, self.user_file]   # :590
    tools.register(ReadFileTool(workspace=workspace, allowed_dir=workspace, ...))
    tools.register(EditFileTool(workspace=workspace,
                                allowed_dir=skills_dir,               # 写：仅 skills 目录
                                extra_write_allowed_files=editable_files))  # + 三个记忆文件
```

只有四个工具（read_file / edit_file / apply_patch / write_file），且**写权限被收窄到 `skills/` 目录加 `MEMORY.md`/`SOUL.md`/`USER.md` 三个文件**。它既不经过 `ToolLoader`，也不看 `_scopes`。所以正确的说法是：**"可见性"有两套——scope 管 Agent/子 Agent，Dream 用白名单注册表管记忆写入，别混。**

### 4.4 结果（R，约 20 秒）

> 改动本身只有一行；真正的产出是我给自己画了一张三级可见性表（core / subagent / dream-only），以及一条判断准则：**"能做成架构上不存在，就不要做成运行时禁止"**。后来再有人问"子 Agent 能不能递归"，我能答：不能，因为它连这个工具都没有。

### 4.5 可能的 3 个追问 + 回答要点

**追问 1：为什么用类属性声明，而不是加载时传一个白名单参数？**

> 要点：**fail-closed**。类属性是"结构"——注册表在构造时就过滤完，任何新入口（子 Agent、新的 runner、以后加的第三种 scope）都不会因为忘传参数而泄漏。运行时传参是"约定"，多一个入口就多一次忘记的机会。这条在安全设计里是通用经验：**默认拒绝，显式开洞。**

**追问 2：那 `_plugin_discoverable` 和 entry_points 是干什么的？**

> 要点：loader 有两条发现路径，`discover()` 扫包内模块（`pkgutil.iter_modules`，`loader.py:43`），`_discover_plugins()` 读 `entry_points(group="nanobot.tools")`（`:74`）——外部包可以不改 nanobot 源码就注册工具。`_plugin_discoverable = False` 让某个类不被自动发现（比如 `_LegacyErrorPrefixTool` 这种适配器，`loader.py:130`）。同名冲突时插件会让位给内置工具（`:107-117` 有警告日志）。**能说出这两条路径，等于证明我读了整个 loader 而不是只看一行。**

**追问 3：加一个工具最少要做什么？**

> 要点：继承 `Tool`，实现四个抽象成员——`name` / `description` / `parameters` / `execute`（`tools/base.py:171-187`）。其中 `description` 是**给模型看的 prompt**，`parameters` 的 JSON Schema 决定模型会不会填错参数；执行失败要返回 `ToolResult.error`，因为 `ToolRegistry.execute` 会统一追加一句"换个思路再试"的提示（`registry.py:189`）。另外 `read_only`/`concurrency_safe`（`base.py:190-202`）决定它能不能和其他工具并行——写文件类工具必须老实返回 `False`。

---

## 五、话术三：失败与复盘

> 面试官："讲一个你判断错了、后来纠正的事。"这一题考的不是错误本身，而是你有没有**纠错机制**。

### 5.1 S（约 20 秒）

> 我早期整理笔记时，把"nanobot 核心约 4000 行 Python"写成了事实，还写进了简历草稿。理由很充分：我在好几篇中文解读里都看到这句话。

### 5.2 T（约 15 秒）

> 转折点是有人问我："4000 行怎么装下 17 个渠道和一套 WebUI？"我答不上来。所以任务变成：**确认这个数字到底是什么口径，以及我该怎么避免再犯。**

### 5.3 A（约 70 秒，四步）

**第一步：先量，不辩论。**

```powershell
cd nanobot
git describe --tags                 # → v0.3.5（commit 1bb712d3）
(Get-ChildItem nanobot -Recurse -Filter *.py).Count    # → 399
# 排除 tests 目录后 → 329 个 .py（就是常被引用的"329 文件"）
```

单文件量一下就露馅：`nanobot/agent/loop.py` **2,388 行**、`nanobot/agent/runner.py` **1,374 行**——两个文件加起来就接近 4,000，"核心 4000 行"当场不成立。

**第二步：追口径。** 我去 git 历史里找这个数字的来源，在 `v0.1.5` 找到了一个 `core_agent_lines.sh`。脚本自己的 Notes 段写得很清楚：

```text
Notes
-----
  - agent/ only counts top-level Python files under nanobot/agent
  - tools/ is counted separately from nanobot/agent/tools
  - skills/ counts .md, .py, and .sh files
  - not included here: command/, providers/, security/, templates/, nanobot.py, root files
```

也就是说，这个口径只数 6 个目录的**顶层** `.py`，明确不含 providers、channels、cli、tools。而脚本在 v0.3.5 里已经被删掉了（commit `19c3db55`，提交信息就是 `chore: remove core agent line count script`）。**结论：4,000 行是"早期版本 + 窄口径"的宣传数字，被后来的文章当成了现状。**

**第三步：换成可复现的写法。** 我不再写"约 4000 行"，改成一条命令 + 一个日期：

| 说法 | 可复现写法 |
|---|---|
| 规模 | 329 个 `.py`（排除 tests）/ 约 99,398 行 SLOC，`v0.3.5` `1bb712d3` |
| 单个大文件 | `loop.py` 2,388 行、`runner.py` 1,374 行（`(Get-Content x).Count` 即可复现） |
| 工具目录 | `nanobot/agent/tools/` **顶层** 29 个 `.py`、13,440 行（递归还包括各子目录） |

**第四步：把纠错过程沉淀成制度。** 我给自己定了规矩：笔记里**任何数字后面必须跟（命令｜提交｜日期）三件套**；写不出来的数字就删掉，改成定性描述。这条规矩后来直接变成了本书的"附录 D 原文勘误"体例——把"旧说法 / 正确说法 / 证据"三列并排放。

### 5.4 R（约 20 秒）

> 直接结果是简历改了两句话。更重要的收获是：**我学会的不是"记住正确的数字"，而是"让每个论断都带复现路径"**。第二次遇到"37K stars"这种说法时，我的第一反应不是相信，而是"这个数会变、且与我要证明的能力无关，删掉"。

### 5.5 可能的 3 个追问 + 回答要点

**追问 1：那旧文档是不是就没价值了？**

> 要点：分开看。**结构、讲法、类比**是有价值的（章节编排、从类比到代码的顺序），**事实层必须重核**。我在新版里给每章加了两个附录：附录 A 记"版本演进导致的差异"（旧版本当时是对的），附录 D 记"旧文档本身写错的地方"（任何版本都不对）。这个区分让"勘误"不至于变成"否定前人的一切"。

**追问 2：再举一个你自己纠正过的例子？**

> 要点：记忆模块。我原先以为"压缩 = 把摘要写进 `MEMORY.md`"，读代码后发现压缩只把摘要**追加**到 `history.jsonl`（带 cursor），`MEMORY.md` 由低频的 Dream 任务做最小改动更新，而且有 git 版本（可 `/dream-log`、`/dream-restore`）。另外旧文档里的 `save_memory` 工具**根本不存在**，`MemoryConsolidator` 这个类名也不存在——真实的是 `Consolidator` + `MemoryArchiver`。**能说出"我以为 X，实际是 Y，证据在 Z"，比直接背 Y 更有说服力。**

**追问 3：面试里被当场指出错误怎么办？**

> 要点：三步，别硬撑。① 承认——"这条我记错了"；② 给核验方法——"我现在会去 `grep -n` 这个函数名确认"；③ 区分认知等级——"'我记得是这样'和'我核过源码'是两回事，刚才那句属于前者"。**面试官不指望你不出错，他看你出错后会不会继续编。**

---

## 六、高频追问的分类与答法

面试官的追问基本落在五类里。**识别类别比背答案有用**——听到"如果……会怎样"就是场景类，听到"具体在哪"就是证据类。

### 6.1 架构类（考你有没有整体图）

| 追问 | 一句话答法 |
|---|---|
| 一条消息经过哪些环节？ | Channel → `MessageBus.publish_inbound`（`bus/queue.py:37`）→ AgentLoop 七阶段（`loop.py:1679-1686`）→ Runner 循环（`runner.py:435`）→ `_persist_turn`（`:2014`）→ Outbound |
| Loop 和 Runner 怎么分工？ | Loop 管回合（会话键/锁/命令/持久化/投递），Runner 管循环（请求/工具/停止）；子 Agent 复用 Runner 而绕开 Loop |
| 为什么七个阶段顺序不能换？ | `command` 必须在 `build` 前——`/stop` 这类命令不需要构造上下文，先短路省一次完整回合；`save` 必须在 `respond` 前——先落盘再投递，投递失败不丢历史 |

### 6.2 记忆与上下文类（考你有没有抠过细节）

| 追问 | 一句话答法 |
|---|---|
| 压缩的触发条件是什么？ | 三条路：显式 `/compact`、空闲 TTL（`autocompact.py:39-55`）、回合内 token 压力（`context_governance.py:415`，预算 = 窗口 − 输出 − 1024） |
| 压缩产物写到哪？ | 摘要追加到 `history.jsonl`；`MEMORY.md` 只由 Dream 改，且过 git |
| 工具结果超长怎么办？ | 落盘 + 路径 + 1,200 字符预览（`helpers.py:367`），`read_file` 豁免清洗（`context_governance.py:69`） |

### 6.3 工具与 MCP 类（考工程判断）

| 追问 | 一句话答法 |
|---|---|
| MCP 工具叫什么名字？ | `mcp_<server>_<tool>`；资源/提示词分别是 `mcp_<server>_resource_<name>` 和 `mcp_<server>_prompt_<name>` |
| MCP 生命周期归谁？ | 组合根：`gateway_runtime.py:473` 建、`:579` connect、`:325-327` 关停时和 agent 一起 aclose |
| 工具可见性怎么控？ | `Tool._scopes` 声明 + `ToolLoader.load(scope=...)` 过滤（`loader.py:100`）；Dream 另有一套手搓白名单（`memory.py:575`） |
| 工具报错会中断回合吗？ | 不会。`ToolRegistry.execute` 捕获后返回 `ToolResult.error` 并附"换个思路"提示（`registry.py:189`），让模型自我修正 |

### 6.4 渠道类（考你读没读外围）

| 追问 | 一句话答法 |
|---|---|
| 17 个渠道怎么组织？ | 每个渠道一个自包含子包（`manifest.py` / `runtime.py` / `config.py`），`ChannelManager` 统一发现启动（`channels/manager.py:82`） |
| 怎么做访问控制？ | `allowFrom` 白名单 + `*` 通配 + pairing 码；判定在 `BaseChannel.is_allowed`（`channels/base.py:237`），DM 未授权会回一个配对码（`:286`） |
| 飞书要公网 webhook 吗？ | 不要。v0.3.5 的飞书渠道走 lark-oapi 的 **WebSocket 长连接**（`channels/feishu/runtime.py:1` 的模块 docstring 自己写着） |

### 6.5 证据类（考你敢不敢被翻源码）

- "这个行号你确定？"→ 答"我确定的是函数名和文件，行号会随版本漂移，建议按 `_process_message` 搜一下。"
- "你在哪看到的？"→ 答"`nanobot/agent/tools/loader.py:100`，那行是 `if scope not in getattr(tool_cls, '_scopes', {"core"})`。"（能背出整行代码，胜过背行号）
- "这个数字哪来的？"→ 答"我量的：命令是 `(Get-ChildItem nanobot -Recurse -Filter *.py).Count`，日期是 2026-10-27。"——**答不出命令就等于没有数字。**

### 6.6 留三个问题给面试官（也是"证据类"的延伸）

| 问题 | 为什么问这个 |
|---|---|
| "你们现在的 Agent 是'外层编排 + 内层自治'，还是全流程 workflow？" | 一问就知道团队做到哪一层；也顺势展示你懂两种形态的区别 |
| "线上怎么定位一次失败的回合——有 trace，还是只能翻日志？" | nanobot 的 `_run_turn_stage` 就是"按阶段埋点"的思路，你可以顺着聊可观测性 |
| "工具数量到多少了？有没有做过工具检索/裁剪？" | 工具从 20 个涨到 100 个时，"全量注入"会开始吃不消，这是真实的工程拐点 |

**不要问**："你们用什么模型？"（招聘网站上写着）、"加班多吗？"（问法太直白）、"我能学到什么？"（把成长责任推给对方）。

---

## 七、模拟面试片段

> 只保留两段，重点看"追问链"怎么接，不用背。

**面试官**：你说你给工具改了可见范围，为什么原设计要默认"只有主 Agent 能看见"？

**候选人**：因为默认值要服务最危险的场景。工具是能力的集合，`spawn`、`cron`、`message` 这三个一旦被子 Agent 拿到就会出事——子 Agent 能起子 Agent 就递归，能发消息就能绕开主 Agent 的汇总。所以基类 `_scopes = {"core"}`（`tools/base.py:208`）是"默认拒绝"，谁需要子 Agent 用就自己声明，这是典型的 fail-closed。代价是新工具容易"忘了声明、然后疑惑为什么子 Agent 看不见"——我就踩过。

**面试官**：那你觉得这个默认值该改吗？

**候选人**：不该改默认值，该改**反馈**。现在的表现是"工具静默消失"——`loader.py:100` 那个 `continue` 前没有日志，谁都不会意识到自己漏了声明。如果在跳过时打一条 debug 日志，说明"这个类的 `_scopes` 里没有当前 scope"，踩坑成本会低很多。这是个很小但真实的改进点，也是我读 loader 时想到的。

---

**面试官**：如果让你把 nanobot 用到公司内部，你最担心什么？

**候选人**：三个层次，从急到缓。第一是**工作区边界**：`tools.restrict_to_workspace` 默认是 `false`（`config/schema.py:401`），对个人用户合理，对多用户部署就是洞，必须先打开。第二是**渠道准入**：所有渠道必须显式 `"enabled": true` 才启动，而且 `allowFrom` 不填会进 pairing 模式（`channels/manager.py:328` 起的 `_validate_allow_from` 会警告），公网 Bot 一定要填。第三才是**模型行为**（prompt injection）——那部分只能降低概率，真正的边界得靠前两层代码级限制。

---

## 八、面试技巧与表达节奏

### 8.1 一分钟 = 四段

```text
① 结论（10s）  "默认 200 次，子 Agent 直接继承，没有单独上限。"
② 机制（25s）  "配置在 schema.py:129，Loop 构造时传给 SubagentManager，
                每个回合还会同步一次运行时上限。"
③ 为什么（15s）"防烧钱靠并发 4 和 scope 过滤，不靠砍迭代数——
                砍了任务做不完，反而更贵。"
④ 边界（10s）  "更早版本是不是 15 次我没核过，我只对 v0.3.5 负责。"
```

### 8.2 五种翻车场景

| 场景 | 风险 | 应对 |
|---|---|---|
| 简历写"精通 nanobot 源码" | 面试官会挑一个你没读过的模块 | 改成"通读核心链路，外围按需查阅" |
| 数字被追问来源 | 编的数字撑不过第二问 | 每个数字带命令 + 日期，或直接删 |
| 被问"为什么不用 LangChain" | 贬低别人显得不专业 | 讲取舍：抽象层多 → 学习成本高；生态全 → 生产更稳，各有场景 |
| 系统设计题卡住 | 冷场 | 先复述需求 + 画三层（接入/Agent/存储），再逐层展开 |
| 被指出记错 | 硬撑会连带否定前面所有内容 | 承认 + 给核验方法 + 区分"记得"与"核过" |

### 8.3 面试前一天的清单

- [ ] 90 秒版念三遍，**录音回放**，砍掉所有"呃""然后""就是"
- [ ] 三个话术各准备 3 个追问（本章已给），挑一个讲给不懂技术的朋友听
- [ ] 打开 `nanobot` 仓库，随手 `grep` 五个类名确认还在（`AgentLoop`、`AgentRunner`、`ToolRegistry`、`Consolidator`、`MCPProvider`）
- [ ] 把所有"成果数字"过一遍：**每一个都能说出处，说不出的删掉**
- [ ] 准备 2 个问面试官的问题（团队当前最大的技术债 / 新人前三个月的产出预期）

---

## 九、数据诚实与投递前自检

这一节单独写，因为它是本章唯一"讲错就变人品问题"的部分。

1. **示例成果数字不可虚构，投递前替换为真实数据。** 本章 §2.1 的"日均 500 次对话"、旧文档里的"准确率 78% → 95%""成本降低 25%"都是**格式示例**。你要么换成自己的真实数字（并准备好口径、样本、时间段），要么删掉。
2. **没有生产数据就讲机制，不要编场景。** "我在本地跑了 30 次实验，统计了 `stop_reason` 分布"是可信的；"我们线上服务 5 万用户"如果查无实据，一问就穿。
3. **区分"我核过的"和"我记得的"。** 前者给行号，后者给方向。这个区分本身就是能力。
4. **star 数不要当论据。** 它会变，且和你的技术能力无关；换成"329 个文件 / 约 99,398 行 SLOC / 17 个渠道"这种可复现的规模描述。
5. **不确定的版本行为，明确说不确定。** "这是 v0.3.5 的行为"是一句很强的免责，也是一句很强的专业。

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 核心约 4000 行 | 329 个 `.py`（排除 tests）/ 约 99,398 行 SLOC；`loop.py` 2,388 行、`runner.py` 1,374 行 | `pyproject.toml:3`、`nanobot/agent/loop.py:196` | 讲规模用可复现口径，别用宣传数字 |
| 37K+ stars | star 数会变且与技术无关 | —（不应作为论据） | 直接删掉这句 |
| 主 40 / 子 15 次迭代 | 都是 `max_tool_iterations`，默认 200；子 Agent 继承主 Agent 的值 | `config/schema.py:129`、`agent/subagent.py:140-144` | 讲"继承"这个设计意图 |
| `Task` 工具启动子 Agent | 工具名是 `spawn`，参数 `task`/`label`/`temperature`/`wait` | `agent/tools/spawn.py:63` | 说 `spawn`；`Task` 是 asyncio 概念 |
| `MemoryConsolidator` 类 | `Consolidator`（:1072）+ `MemoryArchiver`（:758）+ Dream（Dream 工具集 :575 起） | `nanobot/agent/memory.py` | 拆成"压缩"和"巩固"两条线讲 |
| `save_memory` 虚拟工具 | 不存在；长期记忆由 Dream 用受限工具集低频更新 | `agent/memory.py:575-614` | 别再提；改讲"白名单 + git 版本" |
| `MEMORY.md + HISTORY.md` | `MEMORY.md` + `history.jsonl`；`HISTORY.md` 只是首次迁移源 | `agent/memory.py:107` | 强调 JSONL + cursor |
| `_concurrency_gate(3)` | 容量来自环境变量 `NANOBOT_MAX_CONCURRENT_REQUESTS`，**未设置即不限** | `agent/loop.py:428-430`（读环境变量在 :429） | 说"可配，默认不限" |
| `_TOOL_RESULT_MAX_CHARS = 16000` 常量 | 配置项 `max_tool_result_chars`（默认 16,000）+ 落盘与 1,200 字符预览 | `config/schema.py:131`、`utils/helpers.py:367` | 它是配置不是魔数；是"搬家"不是"丢弃" |
| 飞书适配器要处理 Webhook 加密签名 | v0.3.5 飞书走 lark-oapi **WebSocket 长连接**，不需要公网 webhook | `channels/feishu/runtime.py:1`（模块 docstring） | 这条同时是旧文档的勘误（见 D7） |
| 模拟面试里"MemoryConsolidator 双层记忆 + save_memory"整段 | 机制不存在，讲出来会被追问穿 | 同上面几行 | 换成"三条压缩路径 + history.jsonl + Dream" |

---

## 附录 B：进阶理解

**1. 为什么"我改了什么"比"我读了什么"值钱？**
读源码是被动信息，改动是主动判断。改动的故事里必然包含：现象（反直觉）→ 定位（锚点）→ 取舍（为什么不是另一种改法）→ 验证。这四步是工程能力的最小闭环。而"我读了 8 个模块"只证明你时间多。

**2. 为什么面试官爱问"默认值"？**
默认值是**产品决策的化石**。`restrict_to_workspace` 默认 false，说明 nanobot 定位是"服务自己的个人用户"；`_scopes` 默认只有 core，说明它把"防递归"放在"方便"之上。能解释默认值的来由，说明你理解的是设计意图，不是函数签名。

**3. 为什么"4000 行"这种说法生命力这么强？**
因为它**好用**：短、有对比、能当卖点。它的问题不是数字错，而是把"某个口径下的统计"当成了"项目的属性"。所有"极简""零依赖""几千行"的宣称都该被追问口径——面试里你这么问一句，反而显得专业。

**4. 为什么 nanobot 的 Loop 要写七个阶段而不是一个大函数？**
可观测性。`_run_turn_stage`（`loop.py:1689`）给每个阶段包了耗时日志，出问题能定位到"卡在 build 还是 run"。这是把"调试需求"提前编码进结构，而不是事后加日志。对照旧文档的说法"AgentLoop 就是 ReAct 循环"，其实 Loop 大半代码在处理"回合的杂事"，循环本身在 Runner 里。

**5. 为什么"承认不会"是加分项？**
因为面试官在评估**你的错误会以什么方式暴露**。会承认边界的人，交付时也会说"这块我没验证"；什么都能答的人，交付时会把不确定的东西当确定的东西发出去。前者可管理，后者是风险。

**6. 为什么本章要求"每份话术后面配追问"？**
因为真实面试是**对话**不是演讲。你讲 90 秒，面试官接一句"那 200 次迭代不危险吗"，你的回答质量取决于你有没有预演过这个接续点。话术是线性的，追问是树状的——准备树，不要只背线。

---

## 附录 C：如何把技术细节讲成故事

这一节不讲 Python，讲"口头表达的可复用套路"。五个动作，每个都能立刻用。

### C.1 用"反直觉"开场，而不是用"定义"开场

- ❌ "nanobot 是一个轻量级 Agent 框架，它有 Channel、MessageBus、AgentLoop……"
- ✅ "我一开始以为工具是全都塞给模型的，后来发现子 Agent 拿到的注册表里根本没有某些工具——这是设计，不是 bug。"

**原理**：定义是背景音，反差是钩子。听众的注意力只对"预期被打破"有反应。

### C.2 一句一个锚点，锚点必须是"可被翻到的东西"

三种锚点按强度排序：**整行代码 > 函数名/类名 > 文件名 > 行号**。行号最弱（会漂移），整行代码最强（能背出来说明你真读过）。

- 强："`loader.py` 里那行是 `if scope not in getattr(tool_cls, "_scopes", {"core"})`。"
- 中："在 `ToolLoader.load` 里按 scope 过滤。"
- 弱："大概在 tools 目录某个文件里。"

### C.3 用"我发现 X，因为 Y，所以 Z"的三段式，代替形容词

形容词（"深入""全面""大幅"）没有信息量，因果有。

> "我发现子 Agent 看不见我的工具（X），因为基类 `_scopes` 默认是 `{"core"}` 而我没声明（Y），所以我把声明改成 `{"core","subagent"}`，并且确认 `spawn` 仍然不在里面（Z）。"

### C.4 类比只用一句，然后立刻落回代码

类比是"让外行跟上"，代码是"让内行信服"。两个都要，但顺序不能反。

> "AgentLoop 像前台，AgentRunner 像办事员——前台登记完把单子给办事员跑腿。落到代码：锁在前台（`loop.py:1418` 附近），循环在办事员（`runner.py:435`）。"

**只讲类比不讲代码 = 空；只讲代码不讲类比 = 干。**

### C.5 把"失败"讲成"方法"，而不是讲成"教训"

- ❌ "我学到了做事要严谨。"（无信息）
- ✅ "我改了一个习惯：任何数字后面必须跟（命令｜提交｜日期），写不出来就删。这个习惯让我后来一眼看出'37K stars'不该当论据。"

**原理**：教训是情绪，方法是可复制的行为。面试官招的是能复制的人。

### C.6 练习法：三遍递减

1. 第一遍写 500 字（把所有细节摊开）；
2. 第二遍删到 200 字（只留结论 + 一个锚点 + 一个取舍）；
3. 第三遍**不看稿**讲 90 秒并录音，回放时数"呃"的次数。

能压到 90 秒还不丢信息，说明你真的理解了结构，而不是记住了句子。

---

## 附录 D：原文勘误

下表只列**旧 16 章本身写错**的地方（不是版本演进导致的），按出现顺序排。

| # | 旧文位置 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| D1 | 二、版本 A；三、3.1 S/T；七、追问 13/14；八、模拟面试开场 | 反复使用"GitHub 37K+ stars""核心约 4000 行"作为论据 | v0.3.5 是 329 个 `.py` / 约 99,398 行 SLOC；star 数会变，不应作为技术论据 | `pyproject.toml:3`、`nanobot/agent/loop.py:196` |
| D2 | 三、3.1 A 阶段；四、4.1/4.2；八、MemoryConsolidator 整段 | 把 `MemoryConsolidator` 当成真实类名，并描述成"双层记忆架构" | 真实组件是 `Consolidator`（压缩）+ `MemoryArchiver`（归档）+ Dream（巩固）；压缩只写 `history.jsonl` | `nanobot/agent/memory.py:1072`、`:758`、`:575` |
| D3 | 三、3.1 A；四、4.1 动作二；七、追问 2 | 编造 `save_memory` 虚拟工具，并称"模型自主决定写长期记忆" | 没有这个工具；长期记忆由低频 Dream 任务在受限工具集下改动，且过 git 版本 | `nanobot/agent/memory.py:575-614` |
| D4 | 八、模拟面试"最大迭代次数 15 次"整段 | 说主 Agent 40 轮、子 Agent 15 轮，并解释成"经验平衡点" | 都是 `max_tool_iterations`，默认 200，子 Agent 继承主 Agent 的值 | `nanobot/config/schema.py:129`、`nanobot/agent/subagent.py:140-144` |
| D5 | 八、同一段 | 说主 Agent 用 `Task` 工具启动子 Agent | 工具名是 `spawn`；`Task` 是 asyncio 的并发原语，不是工具 | `nanobot/agent/tools/spawn.py:63` |
| D6 | 七、追问 10 | 说 `_concurrency_gate(3)` 是"全局最多 3 个 AgentLoop 的信号量" | 信号量存在，容量来自 `NANOBOT_MAX_CONCURRENT_REQUESTS`，未设置即不限制 | `nanobot/agent/loop.py:428-430` |
| D7 | 五、5.1 动作"飞书适配器" | 说飞书接入要处理"Webhook 验证的加密签名"、需要公网回调 | v0.3.5 飞书渠道走 lark-oapi 的 WebSocket 长连接，不需要公网 webhook | `nanobot/channels/feishu/runtime.py:1`（模块 docstring） |
| D8 | 七、追问 1/4；四、4.1 动作三 | 把 16000 写成源码常量 `_TOOL_RESULT_MAX_CHARS`，并说"截断丢弃" | 是配置项 `max_tool_result_chars`；超限是落盘 + 路径 + 1,200 字符预览，`read_file` 豁免 | `nanobot/config/schema.py:131`、`nanobot/utils/helpers.py:367`、`nanobot/agent/context_governance.py:69` |
| D9 | 三、3.1 R；四、4.1 R；六、6.1 R | 把"日均 500 次对话""准确率 78% → 95%""成本降低 25%"写成既有事实 | 这些是无口径、无样本的示例数字；模板里必须标注为占位符，投递前替换为真实数据 | —（见本章 §9） |
| D10 | 九、9.1 通用技巧表格"用具体数字"一行 | 只说"用数字有说服力"，没说数字要能复现 | 数字必须带（命令｜提交｜日期），否则是负债不是资产 | 见本章 §5.3 |

---

## 本章总结

这一章真正想教的只有一件事：**把"我读过"升级成"我判断过，而且能当场对账"**。

- 90 秒话术靠的是**一条链路 + 两个反常识细节**，不是把所有模块名报一遍；
- 改动故事靠的是**现象 → 锚点 → 取舍 → 验证**，不是"我优化了性能"；
- 失败故事靠的是**纠错机制**，不是"我认识到了错误"；
- 所有数字必须**带出处**，带不出来就删掉。

> 💡 **下一步读什么**：想补齐话术里的技术细节，去 [17 - 学习资源](../17-learning-resources/README.md) 按"官方文档 → 源码顺序 → 论文"走一遍；想在追问环节更稳，回 [13 - 面试八股文](../13-interview-bagua/README.md) 挑 20 题做"遮住答案讲 90 秒"训练。
> 想直接对账本章的每个断言，打开 `nanobot/agent/tools/loader.py`（第 92-124 行）和 `nanobot/agent/memory.py`（第 575-614 行），这两段是"scope 可见性"和"Dream 受限工具集"的原始证据。
