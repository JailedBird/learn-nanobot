# 02 - Nanobot 项目概览

> 🎯 **本章目标**：用一组可核实的事实说清 nanobot v0.3.5 是什么、有多大、能接什么、为什么这样设计，并能纠正旧文档里"YAML / 4000 行 / 11+ 供应商 / 8+ 平台 / save_memory"等过时说法。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

---

## 目录

- [2.1 项目背景](#21-项目背景)
- [2.2 为什么叫"超轻量级"](#22-为什么叫超轻量级)
- [2.3 核心特性详解](#23-核心特性详解)
- [2.4 项目数据与里程碑](#24-项目数据与里程碑)
- [2.5 为什么选 Nanobot 学习](#25-为什么选-nanobot-学习)
- [2.6 与其他框架的差异化对比](#26-与其他框架的差异化对比)
- [2.7 面试话术](#27-面试话术)
- [2.8 本章总结](#28-本章总结)

---

## 2.1 项目背景

### 谁做的、什么时候发布的

**HKUDS** 是香港大学数据科学实验室（The University of Hong Kong Data Science Lab）。nanobot 由该实验室开源，**首次发布于 2026 年 2 月 2 日**，本套文档对应 **v0.3.5（2026-09-16）**。

它的自我定位是：

> 一个**可自托管（self-hosted）的个人 AI Agent 运行时**：能在 WebUI、终端和聊天软件里跑，内置工具、长期记忆、MCP、模型路由、子 Agent 委派、定时任务，以及一个 OpenAI 兼容 API。

注意这里的用词是 **runtime（运行时）**，不是 framework 或 library。这个区别很重要：

- 框架（framework）通常要你写代码把它嵌进自己的应用；
- 运行时（runtime）是**一个能独立跑起来的进程**，你通过配置和渠道去用它。

nanobot 两种用法都支持：`nanobot gateway` 起一个常驻进程，或者用 Python SDK / HTTP API 把它当库调用。但它的主线体验是"跑起来就能用"。

### 它想解决什么问题

```
问题 1：Agent 框架学起来太重
  LangChain 抽象层多，容易"会调 API 但不懂系统"；AutoGPT 类项目偏实验

问题 2：个人 Agent 缺"最后一公里"
  有模型 API，但没有渠道、记忆、定时、权限；部署又要数据库和容器

问题 3：工具生态碎片化
  每个框架一套私有工具协议，接一个外部工具要写一堆适配
```

nanobot 的答案不是"做一个更全的框架"，而是"**做一个能自托管、能读穿、开箱即用的 Agent 运行时**"：单进程、JSON 配置、workspace 目录、内置渠道和工具。这个取舍决定了它后来所有的设计。

### 项目定位

> 一个装完就能在聊天软件里用的个人 Agent；核心链路足够小，小到可以一个周末读完主干。

它不追求"什么都能编排"（那是 LangGraph 的领域），也不追求"完全自主"（那是 AutoGPT 的叙事）。它追求的是**可理解、可部署、可修改**。

### 术语速查（本章会反复出现）

| 术语 | 一句人话 |
|---|---|
| Provider | 模型供应商适配层，把不同厂商的 API 包装成同一个接口 |
| Channel | 聊天渠道适配器，比如 Telegram、飞书、WebUI 的 WebSocket |
| Gateway | 常驻进程，负责把渠道、Agent、定时任务、WebUI 组装起来 |
| MessageBus | 进程内的异步消息总线，解耦"收消息"和"处理消息" |
| Session | 一段连续对话，键是 `{channel}:{chat_id}`（`nanobot/bus/events.py:40`） |
| Workspace | 工作目录，记忆、技能、产物、cron 状态都在这里 |
| MCP | Model Context Protocol，工具接入的标准化协议 |
| Skill | 一份 `SKILL.md`，用自然语言教 Agent 怎么用某类工具/做某类任务 |
| SLOC | 去掉空行、注释、docstring 后的有效代码行数 |

---

## 2.2 为什么叫"超轻量级"

这一节要诚实地回答一个问题：**README 明明写着 ultra-lightweight，为什么源码有近十万行？**

### 先把数字摆出来

| 口径 | 数字 |
|---|---|
| 旧文档宣传的"核心代码" | 约 4,000 行 |
| v0.3.5 实现代码（SLOC） | **99,398 行** |
| Python 文件数（非测试） | **329 个** |
| 其中 `channels/` 子包 | 21,848 行 |
| 其中 `agent/`（含 tools） | 19,465 行 |
| 其中 `webui/`（Python 侧） | 18,022 行 |
| 其中 `providers/` | 13,483 行 |
| 其中 `cli/` | 7,206 行 |
| WebUI 前端（TS/TSX/CSS） | 约 100,668 行（另一套技术栈） |

### "4000 行"是怎么来的

旧文档的数字不是凭空编的，它来自仓库根目录早期的一个统计脚本 `core_agent_lines.sh`：这个脚本只统计 `nanobot/agent/*.py` **顶层文件**，不包含 `agent/tools/`、`providers/`、`channels/`、`cli/`、`session/`、`webui/`。

在 v0.1.5 时期，这个口径算出来是 **4,042 行**。所以旧文档写"约 4000 行"在当时是"有来源的"，只是**口径极窄**，而且被当成了整个项目的规模。v0.3.5 已经不在这个量级，仓库里也没有这个脚本了。

### 那它到底"轻"在哪

"轻"不是总行数少，而是**理解成本和部署成本低**：

**1. 部署轻：一个进程 + 一个 JSON + 一个目录**

```bash
pip install nanobot-ai
nanobot onboard --wizard     # 生成 ~/.nanobot/config.json 和 workspace
nanobot gateway              # 一个进程跑起渠道、Agent、cron、WebUI
```

不需要 Postgres、Redis、消息队列、容器编排。配置就是一个 JSON 文件（`~/.nanobot/config.json`，`nanobot/config/loader.py:39`），状态都在 workspace 目录里。

**2. 核心链路轻：主干约 6,500 行**

把"一条消息从进来到出去"真正会用到的文件加起来：

| 文件 | 行数 | 作用 |
|---|---|---|
| `nanobot/agent/loop.py` | 2,388 | 回合编排 |
| `nanobot/agent/runner.py` | 1,374 | 模型/工具循环 |
| `nanobot/agent/memory.py` | 1,297 | 记忆读写与 Dream |
| `nanobot/config/schema.py` | 707 | 配置结构 |
| `nanobot/agent/context.py` | 363 | 上下文拼装 |
| `nanobot/agent/tools/registry.py` | 212 | 工具注册表 |
| `nanobot/bus/queue.py` | 148 | 消息总线 |

合计约 **6,500 行**。这才是一个周末能读穿的部分；剩下的九万多行主要是 17 个渠道、46 个 Provider 条目、WebUI、TUI、CLI、安全策略这些"外围适配"。

**3. 概念轻：没有自研编排引擎**

很多框架会实现 Planner、TaskGraph、状态机、DAG 执行器。nanobot 的核心洞察是：**LLM 本身就是规划器**。它只实现一个 ReAct 循环，让模型自己决定下一步（第 01 章 1.4 节）。这砍掉了大量"编排框架"代码。

**4. 扩展轻：工具和渠道都是"放一个文件/目录就能被发现"**

- 工具：在 `nanobot/agent/tools/` 放一个 `Tool` 子类，`ToolLoader` 用 `pkgutil` 自动扫描（`nanobot/agent/tools/loader.py:36`）。
- 渠道：在 `nanobot/channels/` 放一个子包，实现 `manifest.py` + `runtime.py`，`ChannelManager` 自动发现（`nanobot/channels/manager.py:82`）。

### 结论：怎么讲"轻量"才不心虚

不要说"它只有 4000 行"。正确的说法是：

> "nanobot 的**部署形态**很轻：单进程、JSON 配置、文件系统记忆，不需要外部基础设施。它的**核心链路**也很集中，`loop.py` + `runner.py` 两个文件就能讲清 Agent 怎么跑。但它的**总量不小**，v0.3.5 约 99,398 行，增长主要来自 17 个渠道和 46 个 Provider 条目——这些是适配成本，不是核心复杂度。"

这个说法既尊重事实，也能体现你真的读过代码。

---

## 2.3 核心特性详解

### 特性一：46 条 Provider 记录，覆盖主流模型供应商

Provider 层解决的是"不同厂商 API 长得不一样"的问题。nanobot 把所有供应商的元信息集中在一张表里：`PROVIDERS`（`nanobot/providers/registry.py:146`）的每一项 `ProviderSpec` 记录名字、模型关键字、环境变量、后端实现、默认 base URL 等字段，加一家供应商通常只要加一条记录 + 一个配置字段。

v0.3.5 的 `PROVIDERS` 里有 **46 条**记录（`nanobot/providers/registry.py:146` 起），包括：

- 直连供应商：Anthropic、OpenAI、Gemini、DeepSeek、DashScope、Moonshot、MiniMax、Mistral、Groq、xAI、Zhipu、StepFun、Qianfan……
- 网关：OpenRouter、AiHubMix、SiliconFlow、Hugging Face、Eden AI……
- 本地/自托管：vLLM、Ollama、LM Studio、OVMS
- 特殊后端：Azure OpenAI、AWS Bedrock、GitHub Copilot、OpenAI Codex

注意"46 条"不等于"46 家厂商"：里面包含 `custom`（自定义 OpenAI 兼容端点）、网关、以及 `opencode_zen` 这类兼容别名。面试时说"40 多家/40 多个 Provider 条目"比说"精确 46 家"更稳妥。

**配置长这样**（JSON，不是 YAML）：

```json
{
  "agents": {
    "defaults": {
      "model": "deepseek/deepseek-v4-pro",
      "maxToolIterations": 200
    }
  },
  "providers": {
    "deepseek": { "apiKey": "${DEEPSEEK_API_KEY}" }
  }
}
```

几个要点：

- 根字段是 `providers`，每家一个键；
- 密钥可以写 `${ENV_VAR}`，启动时在内存里解析，**不会回写**到配置文件；
- 键名用 **camelCase**（`apiKey`、`maxToolIterations`），snake_case 也兼容，因为 Pydantic 基类同时开了别名生成和 `populate_by_name`（`nanobot/config_base.py:12-15`）；
- `provider` 默认是 `"auto"`，会根据模型名前缀、API key 前缀、base URL 关键字自动匹配（`nanobot/config/schema.py:122`）。

### 特性二：17 个聊天渠道，每个都是一个子包

这是 nanobot 最有辨识度的能力：它不只是一个库，而是一个能直接接进聊天软件的系统。

| 分类 | 渠道 |
|---|---|
| 国际 IM | `telegram`、`discord`、`slack`、`signal`、`whatsapp`、`matrix`、`mattermost`、`msteams`、`email` |
| 国内 IM | `feishu`（飞书）、`dingtalk`（钉钉）、`wecom`（企业微信）、`weixin`（微信）、`qq`、`mochat`、`napcat` |
| 通用 | `websocket`（WebUI / 自定义客户端） |

**渠道是子包，不是单文件。** 以飞书为例，`nanobot/channels/feishu/` 下有 `manifest.py`（元信息、配置字段、依赖）、`config.py`（配置模型）、`runtime.py`（收发实现）、`connect.py`、`websocket.py`（长连接）和 `webui/`（设置界面）。

所有渠道都实现同一个基类 `BaseChannel`（`nanobot/channels/base.py:21`），把平台消息转成统一的 `InboundMessage`，再丢进 `MessageBus`（`nanobot/bus/queue.py:37`）。Agent 侧完全不知道消息来自哪个平台——这是标准的**适配器模式**。

**配置示例（Telegram）**：

```json
{
  "channels": {
    "telegram": { "enabled": true, "token": "${TELEGRAM_TOKEN}", "allowFrom": ["123456789"] }
  }
}
```

飞书的字段换成 `"enabled": true` + `appId` + `appSecret` 即可（字段名以 `nanobot/channels/feishu/config.py` 为准）。

**两个必须记住的坑**：

1. **所有渠道默认 `enabled=false`。** 只有 `websocket` 例外，它的 `default_enabled=True`（`nanobot/channels/websocket/manifest.py:18`）。不写 `"enabled": true`，配置再全也不会启动。
2. **字段名按渠道不同。** Telegram/Discord 用 `token`；飞书用 `appId` + `appSecret`；钉钉用 `clientId` + `clientSecret`；QQ 用 `appId` + `secret`。不要背"统一字段"，看各渠道的 `manifest.py` 最准。

缺依赖时，用 `nanobot plugins enable <渠道名>` 安装（例如 `weixin`、`dingtalk`、`matrix`）；微信这类需要扫码授权的渠道，还要走 `nanobot channels login <渠道名>`。

### 特性三：MCP 协议原生支持

MCP（Model Context Protocol）是 Anthropic 在 2024 年底提出的工具接入标准。nanobot 不是"后来加了个适配器"，而是把 MCP 当成一等公民：

- 配置在 `tools.mcpServers.<名字>`（`nanobot/config/schema.py:418`）；
- 支持 `stdio`、`sse`、`streamableHttp` 三种传输，`type` 可以省略并自动判定；
- 工具注册后名字统一为 `mcp_<server>_<tool>`（`nanobot/agent/tools/mcp.py:609`）；
- 连接生命周期由组合根管理：`MCPProvider`（`:1346`）创建 → `connect()`（`:1412`）→ 工具进 `ToolRegistry` → 退出时 `aclose()`。

```json
{
  "tools": {
    "mcpServers": {
      "github": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-github"],
        "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
      },
      "remote": {
        "url": "https://example.com/mcp/",
        "headers": { "Authorization": "Bearer ${REMOTE_MCP_TOKEN}" }
      }
    }
  }
}
```

传输判定规则很朴素：有 `command` 就走 stdio；`url` 以 `/sse` 结尾就走 sse；否则走 streamableHttp。

MCP 的价值在于**工具生态复用**：同一个 GitHub MCP Server，Claude Desktop、Cursor、nanobot 都能用；nanobot 不需要为每个工具写一遍适配。第 05 章会拆协议细节，第 08 章会讲怎么接自己的 Server。

### 特性四：记忆系统 = `MEMORY.md` + `history.jsonl` + Dream

旧文档写的是"`MEMORY.md` + `HISTORY.md`，由 `MemoryConsolidator` 压缩，`save_memory` 工具触发写入"。这三处都不对。v0.3.5 的真实结构是：

```
workspace/
├── SOUL.md                 # Agent 的人格/行为设定
├── USER.md                 # 用户画像
└── memory/
    ├── MEMORY.md           # 长期事实，始终注入 system prompt
    ├── history.jsonl       # 追加式历史日志，一行一条，带 cursor
    └── .dream_cursor       # Dream 处理到哪一条
```

- **`MEMORY.md`**：长期事实。`ContextBuilder.build_system_prompt`（`nanobot/agent/context.py:101`）每次都会把它拼进 system prompt——注意是"每次"，不是"检索命中才注入"。
- **`history.jsonl`**：会话摘要的追加日志。回合结束后，`Consolidator`（`nanobot/agent/memory.py:1072`）把摘要写进去。
- **`HISTORY.md`**：只在首次发现时迁移成 `history.jsonl`（`nanobot/agent/memory.py:106`），之后不再使用。
- **Dream**：由 gateway 注册的系统 cron 任务（`nanobot/cli/gateway_runtime.py:822`），默认每 2 小时跑一次（`nanobot/config/schema.py:58-59`）。它读取"还没处理过"的历史增量，连同 `SOUL.md`、`USER.md`、`MEMORY.md` 一起交给模型，让模型对长期文件做**最小改动**，并交给 `GitStore` 提交版本。可以用 `/dream-log` 看历史版本，用 `/dream-restore` 回滚。

**没有 `save_memory` 工具。** v0.3.5 的内置工具清单里没有它，源码里也搜不到对应的工具类。想"记住"什么，是在 Dream 阶段通过文件工具完成的。

**压缩 ≠ 写 `MEMORY.md`。** `Consolidator` 只写 `history.jsonl`；只有 Dream 会改 `MEMORY.md`。这条区分在面试里很加分，因为它说明你理解"流水"和"事实"的分层。

### 特性五：Skills 技能系统

Skill 是"用自然语言写的操作手册"，不是可执行插件。目录结构：

```
workspace/skills/
└── code-review/
    └── SKILL.md
```

`SKILL.md` 的头部是一段 YAML frontmatter，正文是给模型看的指令：

```markdown
---
name: code-review
description: 按团队规范审查一个 diff，输出 blocking / non-blocking 两类问题
---

# Code Review

1. 先读 diff，确认改动范围。
2. 检查错误处理、边界条件、并发安全。
3. 输出格式：...
```

- 加载器：`SkillsLoader`（`nanobot/agent/skills.py:51`）；
- 来源优先级：workspace 技能 > 插件技能 > 内置技能（`nanobot/skills/` 下有 cron、github、memory、summarize、weather 等）；
- `list_skills`（`:90`）会把技能清单放进 system prompt，模型按需用 `read_file` 读具体的 `SKILL.md`；
- 配置里可以用 `disabledSkills` 禁用某些技能（`nanobot/config/schema.py:146`）。

> **别被 YAML 迷惑**：配置是 JSON；`SKILL.md` 头部的 frontmatter 是 YAML，但那是"文档元信息"，不是运行时配置。两者不冲突。

### 特性六：子 Agent 与定时任务

**子 Agent（SubAgent）**：

- 主 Agent 通过 `spawn` 工具（`nanobot/agent/tools/spawn.py:63`）派一个后台任务；
- 子 Agent 有独立的会话和工具集，主 Agent 可以继续处理别的消息；
- **迭代上限继承主 Agent**：`SubagentManager` 的 `max_iterations` 默认取 `AgentDefaults().max_tool_iterations`，也就是 200（`nanobot/agent/subagent.py:140-144`），不是旧文档说的 15；
- 并发上限默认 4（`max_concurrent_subagents`，`nanobot/config/schema.py:130`），由信号量控制；
- 工具按 scope 过滤，子 Agent 拿不到 `spawn`/`message`/`cron` 这类 core-only 工具（`nanobot/agent/tools/loader.py:100`），从代码上杜绝"子 Agent 再 spawn 子 Agent"的递归。

**定时任务（Cron）**：

旧文档给了一段 `cron:` 的 YAML 列表，这在 v0.3.5 不存在。真实机制是：

- 配置根字段只有 `agents`、`channels`、`transcription`、`providers`、`api`、`gateway`、`tools`、`modelPresets`（`nanobot/config/schema.py:422-438`），**没有 `cron`**；
- 定时任务由 `cron` 工具（`nanobot/agent/tools/cron.py:118`）在运行时增删查，支持 `add` / `list` / `remove`；
- 任务持久化在 `<workspace>/cron/jobs.json`（`nanobot/cli/gateway_runtime.py:463`），由 `CronService`（`nanobot/cron/service.py:168`）调度；
- 支持一次性、固定间隔、cron 表达式三种模式，时间带时区；
- CLI 可以手动触发：`nanobot trigger <job-id> "消息"`。

**心跳（Heartbeat）**不是独立的 `HeartbeatService` 类，而是：

- 配置在 `gateway.heartbeat`（`nanobot/config/schema.py:326/359`），默认 `enabled=true`、`interval_s=1800`；
- 由受保护的 cron 系统任务执行，读取 `<workspace>/HEARTBEAT.md`；
- 只有 `HEARTBEAT.md` 里有活动任务时才真正干活，且结果还要过一道"是否值得通知"的判断（`nanobot/cli/gateway_runtime.py:621-633`）。

---

## 2.4 项目数据与里程碑

### 关键数据（v0.3.5）

| 指标 | 数据 | 口径 |
|---|---|---|
| 版本 | 0.3.5 | tag `v0.3.5` |
| 发布日期 | 2026-09-16 | commit `1bb712d3` |
| 首次发布 | 2026-02-02 | 0.1.x |
| 实现代码 | 99,398 行 | SLOC，`nanobot/` 包，排除测试 |
| Python 文件 | 329 个 | 非测试 |
| 渠道 | 17 个 | `nanobot/channels/` 子包 |
| Provider 记录 | 46 条 | `PROVIDERS` 元组 |
| 内置工具名 | 24 个 | 不含 MCP 动态工具 |
| 运行时依赖 | 30+ 条 | `pyproject.toml` 的 `dependencies` |
| WebUI 前端 | 约 100,668 行 | TS/TSX/CSS |
| Python 版本 | ≥ 3.11 | `pyproject.toml` |
| 开源协议 | MIT | `LICENSE` |

### 模块规模（SLOC）

```
channels   ██████████████████████ 21,848
agent      ███████████████████ 19,465   （含 tools）
webui      ██████████████████ 18,022    （Python 侧）
providers  █████████████ 13,483
cli        ███████ 7,206
```

这张图解释了一个常见误解：**"Agent 核心"不是代码量最大的部分**。渠道和 Provider 这些"适配层"才是大头。这也是为什么读源码要先读 `agent/`，不要从 `channels/` 开始。

### 里程碑

| 时间 | 事件 |
|---|---|
| 2026-02-02 | 项目首次发布，定位"超轻量级个人 Agent" |
| 2026 年上半年 | 渠道从 9 个扩展到 17 个；引入 WebUI、TUI |
| 2026-09-16 | v0.3.5：Loop/Runner 分层、Dream 记忆、46 个 Provider、MCP 完整生命周期 |

### 关于 Star 数

旧文档写"37K+ Stars"。Star 数是**随时间变化的快照**，也无法从源码核实，本套文档不写死。面试时如果一定要提，说"GitHub 上热度很高，具体数字建议看仓库首页"，比背一个可能过期的数字安全。

---

## 2.5 为什么选 Nanobot 学习

### 学习价值

| 维度 | nanobot 的优势 | 其他框架的常见问题 |
|---|---|---|
| 链路完整性 | 一条消息从渠道到回复全程可追 | 只看到"链式调用"，看不到部署形态 |
| 核心可读性 | 主干约 6,500 行，两个类讲清循环 | 抽象层多，读着读着迷路 |
| 工程细节 | 上下文压缩、scope、沙箱、重试都有 | Demo 级实现，缺生产细节 |
| 技术热点 | MCP、子 Agent、Dream、多渠道 | 只覆盖其中一两项 |
| 差异化 | 面试中研究 nanobot 的人少 | "我学过 LangChain"太同质 |
| 可动手 | 装完就能接 Telegram/飞书 | 搭一个能用的 Demo 成本高 |

### 一条可行的阅读路线

```
第 1 天：跑起来
  nanobot onboard --wizard → nanobot status → nanobot agent -m "Hello"

第 2-3 天：读主干
  agent/loop.py（2,388 行）→ agent/runner.py（1,374 行）
  → agent/context.py（363 行）→ bus/queue.py（148 行）

第 4-5 天：工具、记忆、一个渠道
  tools/registry.py → tools/loader.py → tools/spawn.py
  → memory.py（1,297 行）→ channels/telegram/ → config/schema.py
```

### 和"读 LangChain"的对比

| 阶段 | nanobot | LangChain |
|---|---|---|
| 跑通 | 半天（安装 + 配 key） | 1-2 天（概念多） |
| 抓住主干 | 2-3 天（6,500 行） | 1-2 周（抽象层多） |
| 讲清"一条消息怎么走" | 可以画完整时序图 | 容易停在"调了某个 Chain" |
| 面试差异化 | 高 | 低（同质化） |

**结论**：nanobot 的优势不是"小"，而是"**闭环**"。你能看到消息、上下文、模型、工具、记忆、渠道如何咬合，这比背 API 有用得多。

---

## 2.6 与其他框架的差异化对比

| 维度 | Nanobot | LangChain / LangGraph | CrewAI | AutoGPT |
|---|---|---|---|---|
| 形态 | 可自托管运行时 | 开发框架/编排库 | 多 Agent 框架 | 自主 Agent 实验 |
| 代码规模 | 99,398 行 SLOC | 数十万行量级 | 数万行量级 | 十万行量级 |
| 配置 | JSON `config.json` | 代码为主 | 代码 + 装饰器 | 代码 + 环境变量 |
| 渠道 | 17 个内置 | 需自行集成 | 无 | Web UI |
| Provider | 46 条记录 | 集成包极多 | 依赖 LangChain | 主要 OpenAI |
| 记忆 | 文件 + Dream | 多后端可选 | 内置 | 内置 |
| MCP | 原生 | 通过集成包 | 有限 | 无原生 |
| 定时任务 | `cron` 工具 + 服务 | 需自行实现 | 无 | 无 |
| 部署 | 单进程 + workspace | 视应用而定 | 中等 | 较重 |
| 适合人群 | 个人/小团队、想读懂系统的人 | 企业应用、需要大量集成 | 多 Agent 协作 | 研究者 |

### 核心差异

**Nanobot vs LangChain**：LangChain 像一个大超市，什么都有、集成齐全，但抽象层多、找东西费劲、版本变化快；nanobot 像一家自营小馆，从采购到上菜全流程可见、部署简单，但生态不如 LangChain、复杂编排能力有限。

**Nanobot vs AutoGPT**

AutoGPT 追求"给一个目标就全自主"，但长程可靠性、成本、安全问题一直没有完全解决。nanobot 不追求完全自主：它有明确的迭代上限（200）、工具 scope、workspace 边界、channel 访问控制。**它更像"可控的半自主"，而不是"放养"。**

**Nanobot vs CrewAI**

CrewAI 的核心是"角色 + 任务 + 协作"，适合多 Agent 分工。nanobot 以单 Agent 为主，多 Agent 只做到"主 Agent 派子 Agent"这一层，且子 Agent 不能再派。**两者解决的问题不同：一个偏协作编排，一个偏单机运行时。**

### 讲差异时的三个关键词

1. **形态**：runtime vs framework vs library。
2. **闭环**：能不能从消息入口追到回复出口。
3. **成本**：部署成本、理解成本、运行成本，三者要分开说。

---

## 2.7 面试话术

### 话术一：介绍你学的项目

> "我系统读过 HKUDS/nanobot，一个可自托管的个人 Agent 运行时。它 v0.3.5 大约 99,398 行 Python（SLOC，329 个非测试文件），但核心链路很集中：`AgentLoop` 负责回合编排，`AgentRunner` 负责模型/工具循环，工具通过注册表和 scope 管理，记忆用 `MEMORY.md` + `history.jsonl` + Dream 两阶段，渠道有 17 个，Provider 表里有 46 条记录。它最吸引我的是'闭环'——一条消息从 Telegram 进来，到 Agent 处理，再到回复出去，每一层都能在源码里找到。"

### 话术二：为什么选它而不是 LangChain

> "有三个原因。第一，nanobot 是运行时，不是库，我能看到完整的部署形态：渠道怎么启动、gateway 怎么组装、cron 怎么调度。第二，它的核心链路只有约 6,500 行，我能读穿并讲清每个设计决策，而不是停留在'会调 API'。第三，它把很多生产细节做进去了：上下文压缩、工具 scope、workspace 边界、MCP 生命周期。这些细节在面试追问时比'我用了什么链'更有说服力。"

### 话术三：它最大的技术亮点

> "我认为是'把复杂度放在适配层，把简单留给核心'。核心只有一个 ReAct 循环，规划交给模型；复杂的东西——17 个渠道、46 个 Provider、WebUI、MCP——都被隔离在适配层。所以它的总代码量不小，但概念模型很简单。这个取舍让个人开发者能读懂主干，同时又能真的把 Agent 接到微信、飞书、Telegram 上。"

---

## 2.8 本章总结

```
1. 定位：可自托管的个人 Agent 运行时（不是"又一个 LangChain"）
2. 规模：99,398 行 SLOC / 329 个非测试 py 文件；核心链路约 6,500 行
3. 能力：17 渠道 / 46 条 Provider / 24 个内置工具；配置是 JSON
4. 记忆：MEMORY.md + history.jsonl + Dream；没有 save_memory
5. 子 Agent：spawn 工具，继承 200 轮上限（不是 15 轮）
```

**面试 Checklist**

- [ ] 能用 30 秒说清"运行时"和"框架"的区别
- [ ] 能说出 99,398 / 329 这两个数字，并解释"4000 行"的来源
- [ ] 能列举 17 个渠道中的至少 6 个，并说出 `enabled: true` 的坑
- [ ] 能说清 46 条 Provider 记录意味着什么（不是 46 家厂商）
- [ ] 能解释 `MEMORY.md` 与 `history.jsonl` 的分工，以及 Dream 干什么
- [ ] 能纠正"YAML 配置 / save_memory / 子 Agent 15 轮 / cron YAML 列表"

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 核心代码约 4,000 行 | 99,398 行 SLOC / 329 个非测试 py 文件 | `_work/facts-v035.md` §1；`nanobot/` 包统计 | 讲"核心 6,500 行、总量近十万" |
| YAML / `nanobot.yml` | JSON `~/.nanobot/config.json` | `nanobot/config/loader.py:39` | 第 06 章展开 |
| 11+ LLM 供应商 | 46 条 `ProviderSpec` | `nanobot/providers/registry.py:146` | 说"40 多个 Provider 条目" |
| 8+ 聊天平台 | 17 个渠道子包 | `nanobot/channels/` 下 17 个目录 | 第 09 章展开 |
| 渠道是平铺 `.py` 文件 | 每个渠道是子包（manifest/runtime/…） | 如 `nanobot/channels/telegram/manifest.py:36` | 讲适配器模式 |
| 心跳用 `HeartbeatService` 类 | `gateway.heartbeat` + cron 系统任务 + `HEARTBEAT.md` | `nanobot/config/schema.py:326`；`nanobot/cli/gateway_runtime.py:621` | 不要把心跳讲成独立服务 |
| 记忆是 `MEMORY.md` + `HISTORY.md` | `MEMORY.md` + `history.jsonl` + Dream | `nanobot/agent/memory.py:76-78` | 第 07 章展开 |
| 子 Agent 15 次迭代 | 继承主 Agent 的 200 | `nanobot/agent/subagent.py:140-144` | 讲"继承 + scope 限制" |
| cron 用 YAML 列表 | 运行时 `cron` 工具 + `<workspace>/cron/jobs.json` | `nanobot/agent/tools/cron.py:118`；`nanobot/cli/gateway_runtime.py:463` | 第 10 章展开 |
| GitHub 37K+ Stars | 随时间变化的快照，未核实 | 无法从源码验证 | 不写死具体数字 |

---

## 附录 B：进阶理解

**1. 为什么渠道要经过 MessageBus，而不是直接调 AgentLoop？**

因为"收消息"和"处理消息"的节奏不一样：一个群可能同时来十条消息，而 Agent 一次只能处理一个回合。MessageBus 把两者解耦，渠道只管把消息塞进队列，AgentLoop 按自己的节奏消费。这样新增渠道不需要改 Agent，新增 Agent 逻辑也不需要改渠道。

**2. 为什么 Provider 元信息要集中在一张表里？**

`PROVIDERS` 同时驱动四件事：环境变量名、模型名匹配、状态展示、请求参数差异。集中之后，加一家供应商只需要"加一条记录 + 加一个配置字段"，而不是在五个文件里改代码。这是"单一事实来源"的典型应用。

**3. 为什么配置用 JSON，而不是 YAML？**

JSON 更严格：没有缩进歧义、没有隐式类型转换、解析器到处都有。代价是不能写注释。对一个"由程序生成、由 WebUI 编辑"的配置来说，严格比好写更重要——用户的配置大多不是手写的，而是向导和 WebUI 写出来的。

**4. 为什么 workspace 是核心边界？**

记忆、技能、cron 状态、产物、Git 版本都在 workspace 里。这让"隔离"变成一个目录问题：换一个 workspace 就是换一个 Agent 的身份和数据；备份就是打包一个目录；权限控制也可以围绕这个目录做。后面第 11 章的工作区边界、沙箱，都是这个设计的延伸。

**5. 为什么 Dream 要单独做一个阶段，而不是每轮都写 `MEMORY.md`？**

每轮都写长期记忆会有三个问题：写得太频繁（成本高）、容易写进噪音（质量差）、无法回滚（危险）。Dream 把"记忆固化"变成低频、批量、可审计的操作：有 cursor 记录处理进度，有 Git 记录版本，有 `/dream-log` 和 `/dream-restore` 兜底。

**6. 为什么子 Agent 不能再 spawn、gateway 要单独做组合根？** 递归 spawn 会让资源消耗指数级增长且难定位，所以 nanobot 用工具 scope 从代码层面禁掉（`nanobot/agent/tools/loader.py:100`），用"能力缺失"代替"运行时判断"，更难被绕过。而 `nanobot/cli/gateway_runtime.py:340` 的 `_run_gateway` 把 config、bus、session、agent、channels、cron、MCP、WebUI 组装起来并在退出时按顺序关闭：依赖关系一眼可见、生命周期集中管理。Loop 本身不负责创建 MCP 连接，就是这个原则的体现。

---

## 附录 C：Python 语法加油站

### C.1 Pydantic `BaseSettings` + 别名：一份 schema 同时吃两种命名

```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

class Base(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)

class AgentDefaults(Base):
    max_tool_iterations: int = 200

print(AgentDefaults(maxToolIterations=300).max_tool_iterations)  # 300
print(AgentDefaults(max_tool_iterations=300).max_tool_iterations) # 300
```

- **是什么**：`alias_generator` 自动把 `max_tool_iterations` 映射成 `maxToolIterations`；`populate_by_name=True` 让两种写法都能读。
- **在哪用**：`nanobot/config_base.py:12-15`；所有配置类都继承它。
- **怎么读**：看到 JSON 里的 `apiKey`，就去 schema 里找 `api_key`，它们是同一个字段。

### C.2 `pkgutil.iter_modules`：自动发现工具和渠道

```python
import pkgutil, importlib

for _finder, name, _ispkg in pkgutil.iter_modules(["nanobot/agent/tools"]):
    module = importlib.import_module(f"nanobot.agent.tools.{name}")
    print(module.__name__)
```

- **是什么**：扫描一个包目录下的所有模块，不需要手写 import 清单。
- **在哪用**：`nanobot/agent/tools/loader.py:43`（工具发现）、渠道发现同理。
- **怎么读**：新增工具"只要放文件"，靠的就是它；调试"为什么我的工具没被加载"时，先看文件名是否以 `_` 开头、是否在跳过名单里。

### C.3 `ABC` + `@classmethod create`：工具的统一生命周期

```python
from abc import ABC, abstractmethod

class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @classmethod
    def enabled(cls, ctx) -> bool:
        return True

    @classmethod
    def create(cls, ctx) -> "Tool":
        return cls()
```

- **是什么**：抽象基类规定接口，`create` 负责"从上下文里拿依赖再实例化"。
- **在哪用**：`nanobot/agent/tools/base.py:159` 的 `Tool`；`spawn.py:55` 附近的 `create` 会从 `ctx.subagent_manager` 取依赖。
- **怎么读**：想知道一个工具怎么被造出来，先看它的 `create`；想知道它什么时候被跳过，看 `enabled` 和 `_scopes`。

### C.4 `Literal` + 类型别名：把"枚举"写进类型系统

```python
from typing import Literal

Backend = Literal["openai_compat", "anthropic", "azure_openai", "bedrock"]

def load(backend: Backend) -> None: ...
```

- **是什么**：限定字符串只能取几个值，IDE 和类型检查器能提前报错。
- **在哪用**：`ProviderSpec.backend`、`MCPServerConfig.type`、`CronSchedule.kind`。
- **怎么读**：看到 `Literal[...]`，就知道这个字段的合法取值就是括号里那几个，不用去别处找常量表。

---

## 附录 D：原文勘误

> 说明：附录 A 记"版本演进导致的差异"，附录 D 记"旧文档本身写错了"。以下按旧文档小节列出。

**D.1（2.2 / 2.3 / 2.6 节）"Nanobot 用 YAML 配置文件（`nanobot.yml`）"**

- **错在哪**：v0.1.x 到 v0.3.5 都不是 YAML 配置，也不存在 `nanobot.yml` 这个文件名；旧文档的 YAML 示例与任何版本的实际配置格式都不符。
- **正确说法**：配置是 JSON，路径 `~/.nanobot/config.json`；键名以 camelCase 落盘，snake_case 兼容。
- **证据**：`nanobot/config/loader.py:39`；`nanobot/config_base.py:12-15`；`docs/configuration.md` 开头。

**D.2（2.3 节特性一）"11+ LLM 供应商"，并用 `provider.type` 配置**

- **错在哪**：供应商数量严重低估；配置结构也不对，没有 `provider.type` 这个字段。
- **正确说法**：`PROVIDERS` 里有 46 条 `ProviderSpec`；配置在 `providers.<名字>.apiKey`，模型在 `agents.defaults.model`，`provider` 默认 `"auto"`。
- **证据**：`nanobot/providers/registry.py:146`；`nanobot/config/schema.py:122`。

**D.3（2.3 节特性二）"8+ 聊天平台"，表里写 `wechat`、`web`**

- **错在哪**：渠道数量和命名都不对。微信渠道的注册名是 `weixin`（显示名是 WeChat）；"Web" 对应的渠道是 `websocket`。
- **正确说法**：17 个渠道子包：dingtalk、discord、email、feishu、matrix、mattermost、mochat、msteams、napcat、qq、signal、slack、telegram、websocket、wecom、weixin、whatsapp。
- **证据**：`nanobot/channels/weixin/manifest.py:35`；`nanobot/channels/` 目录列表。

**D.4（2.3 节特性四）"`save_memory` 虚拟工具触发写入 `MEMORY.md`"**

- **错在哪**：v0.3.5 没有 `save_memory` 工具，也没有"虚拟工具"这层机制。
- **正确说法**：回合后 `Consolidator` 写 `history.jsonl`；Dream 阶段用普通文件工具修改 `MEMORY.md`，并由 Git 提交。
- **证据**：`nanobot/agent/memory.py:1072`、`:543`、`:76`；`nanobot/agent/tools/` 下无对应工具类。

**D.5（2.3 节特性四）"`MemoryConsolidator` 定期压缩 `HISTORY.md`"**

- **错在哪**：类名不存在，文件名也不对。v0.3.5 是三个角色：`Consolidator`（回合摘要）、`MemoryArchiver`（归档）、`Dream`（长期记忆固化）。
- **正确说法**：`Consolidator` → `history.jsonl`；`MemoryArchiver` 负责归档；Dream 负责 `MEMORY.md`/`SOUL.md`/`USER.md` 的最小改动。
- **证据**：`nanobot/agent/memory.py:1072`、`:758`、`:543`。

**D.6（2.3 节特性六）"子 Agent 有独立的 15 次迭代限制"**

- **错在哪**：15 是早期版本的数字，v0.3.5 已经改为继承主 Agent 的配置。
- **正确说法**：`SubagentManager.max_iterations` 默认取 `AgentDefaults().max_tool_iterations`，即 200；调用方可以显式传入覆盖。
- **证据**：`nanobot/agent/subagent.py:140-144`；`nanobot/config/schema.py:129`。

**D.7（2.3 节特性六）"cron 任务写在 YAML 配置里"**

- **错在哪**：配置 schema 的根字段里没有 `cron`；旧文档的 `cron:` 列表是虚构的。
- **正确说法**：定时任务由 `cron` 工具在运行时增删，持久化到 `<workspace>/cron/jobs.json`，由 `CronService` 调度；心跳由 `gateway.heartbeat` + `HEARTBEAT.md` 驱动。
- **证据**：`nanobot/config/schema.py:422-438`（根字段无 cron）；`nanobot/agent/tools/cron.py:118`；`nanobot/cli/gateway_runtime.py:463`。

**D.8（2.2 节 / 2.4 节）"核心代码 ~4,000 行"**

- **错在哪**：把"只统计 `agent/*.py` 顶层"的窄口径当成了全项目规模，且与 LangChain 等框架的全量代码直接比较。
- **正确说法**：v0.3.5 是 99,398 行 SLOC / 329 个非测试 py 文件；核心链路约 6,500 行。
- **证据**：`_work/facts-v035.md` §1；对 `nanobot/` 包按 SLOC 口径统计；v0.3.5 仓库已无 `core_agent_lines.sh`。

**D.9（2.3 节特性三）"MCP Server 用列表配置（`mcp_servers: [...]`）"**

- **错在哪**：v0.3.5 的 MCP 配置是一个对象（按名字索引），不是列表；字段名是 `mcpServers`（snake_case `mcp_servers` 兼容）。
- **正确说法**：`tools.mcpServers.<name>`，每项支持 `command`/`args`/`env`/`cwd` 或 `url`/`headers`/`auth`，工具名变成 `mcp_<server>_<tool>`。
- **证据**：`nanobot/config/schema.py:362-374`、`:418`；`nanobot/agent/tools/mcp.py:609`。

**D.10（2.3 节特性二）"渠道示例只写 token，没提 `enabled`"**

- **错在哪**：旧文档的渠道示例只有凭据字段，漏了 `enabled` 开关，也把字段名统一化了；照着写会得到一个"配置齐全但不启动"的渠道。
- **正确说法**：除 `websocket` 外所有渠道默认 `enabled=false`，必须显式 `"enabled": true`；字段名按渠道不同（`token` / `appId`+`appSecret` / `clientId`+`clientSecret`）。
- **证据**：`nanobot/channels/telegram/manifest.py:30`（`required("token")`）；`nanobot/channels/feishu/config.py:16-18`；`nanobot/channels/plugin.py:38`（`default_enabled=False`）。

---

## 下一章

现在你已经知道 nanobot"有多大、能做什么、为什么这样设计"。下一步进入架构层：一条消息从渠道进来后，`AgentLoop` 和 `AgentRunner` 到底怎么分工，上下文治理又是在哪一层做的。

➡️ [03 - 架构深入解析](../03-architecture-deep-dive/README.md)

想先动手的，可以直接跑：

```bash
nanobot status
nanobot gateway --verbose
```

观察启动日志里渠道、MCP、cron 的注册顺序，再回来看 2.3 节，会更有体感。

---

> 📝 **本章小结**：nanobot v0.3.5 是一个"部署轻、核心链路轻、但适配层很重"的个人 Agent 运行时：99,398 行 SLOC、329 个非测试文件、17 个渠道、46 条 Provider 记录、24 个内置工具、JSON 配置、`MEMORY.md` + `history.jsonl` + Dream 记忆。记住"轻在形态、重在适配"这八个字，就不会再被"4000 行"带偏。
