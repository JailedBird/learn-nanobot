# 17 - 学习资源（v0.3.5 版）

> 🎯 **本章目标**：给你一张"读完能自己往下走"的资源地图——官方文档怎么读、源码按什么顺序读、哪些论文值得花时间、哪些链接别信。看完你应该能回答："这个机制官方在哪写的？源码在哪？论文依据是什么？"
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，commit `1bb712d3`，2026-09-16）
> **前置知识**：[04 - 源码逐行解读](../04-source-code-walkthrough/README.md)、[13 - 面试八股文](../13-interview-bagua/README.md)
> **阅读时间**：约 40 分钟（当清单用，不用一次读完）

> 🔖 **链接核对日期：2026-10-27**。本章所有外链都是这一天打开确认过的；**链接可能变动，失效时以官方仓库为准**（`github.com/HKUDS/nanobot` 的 `docs/` 目录永远最新）。
> 本章**不写 star 数、下载量、社区规模**——这些数字每天都在变，写了就是给读者埋雷。

---

## 目录

- [一、官方资源：先看这四个](#一官方资源先看这四个)
- [二、官方文档地图](#二官方文档地图)
- [三、源码阅读顺序（配合第 04 章）](#三源码阅读顺序配合第-04-章)
- [四、论文与经典](#四论文与经典)
- [五、MCP 生态与模型供应商文档](#五mcp-生态与模型供应商文档)
- [六、社区与交流](#六社区与交流)
- [七、工具与平台](#七工具与平台)
- [八、学习路线图](#八学习路线图)
- [九、书籍与核对习惯](#九书籍与核对习惯)
- [附录 A：与旧版文档的差异](#附录-a与旧版文档的差异)
- [附录 B：进阶理解](#附录-b进阶理解)
- [附录 C：如何高效读官方文档](#附录-c如何高效读官方文档)
- [附录 D：原文勘误](#附录-d原文勘误)

---

![nanobot 架构漫画](../../../comics/02-nanobot-architecture.png)

*先有地图再读细节：官方文档讲"是什么"，源码讲"怎么实现"，论文讲"为什么这样设计"*

## 一、官方资源：先看这四个

新手最容易犯的错，是先搜"nanobot 教程"，结果读到三个月前的二手解读。**顺序要反过来**：官方文档 → 源码 → 二手文章。

### 1.1 nanobot.wiki（官方文档站，含简体中文）

| 入口 | 地址 | 说明 |
|---|---|---|
| 文档站首页 | https://nanobot.wiki | 官网 + 文档入口 |
| 英文（默认） | https://nanobot.wiki/docs/latest/getting-started/nanobot-overview | `latest` 跟着最新发布走 |
| **简体中文** | https://nanobot.wiki/cn/docs/latest/getting-started/nanobot-overview | 中文前缀是 `/cn/`（`/zh-Hant/` 是繁体，别混） |
| 指定版本 | https://nanobot.wiki/docs/0.3.5/getting-started/nanobot-overview | 学 v0.3.5 就用它，别用 `latest` |
| 快速开始 | https://nanobot.wiki/docs/latest/getting-started/quick-start | 装完先跑通这条 |
| 单文件索引 | https://nanobot.wiki/llms.txt | 给模型/爬虫看的全站目录，人也能用 |

界面语言（2026-10-27 实测）：English、简体中文（`/cn/`）、繁體中文（`/zh-Hant/`）、日本語、한국어、Español、Français、Bahasa Indonesia、Tiếng Việt、Русский。**中文翻译可能滞后于英文**，术语对不上时以英文页 + 源码为准。

> 链接核对：2026-10-27；可能变动，失效时以官方仓库为准。

### 1.2 GitHub 仓库（唯一权威）

| 入口 | 地址 | 什么时候用 |
|---|---|---|
| 仓库主页 | https://github.com/HKUDS/nanobot | 一切从这里出发 |
| Releases | https://github.com/HKUDS/nanobot/releases | 看版本与更新日志；**对账用 tag，不用 main** |
| Issues / Discussions | https://github.com/HKUDS/nanobot/issues · https://github.com/HKUDS/nanobot/discussions | 先搜再问；报 bug 带版本 + 系统 + 复现步骤 |
| 贡献指南 | https://github.com/HKUDS/nanobot/blob/main/CONTRIBUTING.md | 想提 PR 先读这个 |

读源码前先对齐版本，本书所有行号都以这个 tag 为准：

```bash
git clone https://github.com/HKUDS/nanobot.git && cd nanobot
git describe --tags        # 看当前在哪个版本
git checkout v0.3.5        # 对齐本书所有行号
```

> 链接核对：2026-10-27；可能变动，失效时以官方仓库为准。

### 1.3 PyPI 包名是 `nanobot-ai`

| 项 | 值 |
|---|---|
| 包名 | `nanobot-ai`（**不是** `nanobot`） |
| 地址 | https://pypi.org/project/nanobot-ai/ |
| 安装 | `pip install nanobot-ai` 或 `uv tool install nanobot-ai` |
| 版本（核对时）/ Python | 0.3.5 / ≥ 3.11（见 `pyproject.toml` 的 `requires-python`） |

**为什么单独记包名**：`pip install nanobot` 装到的是另一个同名项目，报错非常莫名。这个坑每年都有人踩。

### 1.4 半小时上手（把四个资源串起来）

`pip install nanobot-ai` → `nanobot onboard --wizard`（生成 `~/.nanobot/config.json`）→ `nanobot status`（不调模型，先看配置就绪没）→ `nanobot -m "Hello!"`（验证 provider 通）→ `nanobot`（进终端对话）。跑不通先去 `docs/troubleshooting.md`，别先怀疑自己。

> 链接核对：2026-10-27；可能变动，失效时以官方仓库为准。

---

## 二、官方文档地图

仓库里的 `docs/` 比 wiki 更新（wiki 是发布版文档，`docs/` 跟着 `main` 走）。下表按"你现在想解决什么"组织，**不是**按文件名排序。

### 2.1 入门与概念（先读这四篇）

| 文件 | 解决什么问题 |
|---|---|
| `docs/README.md` | 全站路由表：按"你的处境"给入口，先定位再读 |
| `docs/quick-start.md` | 装 + 配 provider + 拿到第一条回复 |
| `docs/start-without-technical-background.md` | 完全不懂终端/API Key 的零基础走法 |
| `docs/concepts.md` | 配置、工作区、网关、会话、工具、记忆的"人话解释"；**读源码前必读**，它先给你正确的名词表 |

### 2.2 架构与扩展（本书第 03/04 章的官方对照版）

| 文件 | 解决什么问题 |
|---|---|
| `docs/architecture.md` | 源码层面的"谁拥有什么、一条消息怎么流"——读完第 03 章来这里对账 |
| `docs/development.md` | 开发环境、测试、CI；想提 PR 时读 |
| `docs/channel-package-guide.md` | 怎么新增一个渠道子包（最完整的扩展示例） |
| `docs/my-tool.md` | 运行时自检与调优 |

### 2.3 配置与供应商（当字典查，不要通读）

| 文件 | 解决什么问题 | 注意 |
|---|---|---|
| `docs/configuration.md` | **每个配置字段与默认值**（10 万字符级别） | 用 Ctrl+F 搜字段名，别从第一行读 |
| `docs/providers.md` | 供应商/模型如何匹配与选择 | 配 `provider: "auto"` 时看它 |
| `docs/provider-cookbook.md` | 可直接粘贴的供应商配置配方 | 抄配置最快 |
| `docs/chat-apps.md` | 各聊天渠道的前置条件与手写 JSON | 接 Telegram/飞书/钉钉时读 |
| `docs/memory.md` | 长期记忆 `MEMORY.md` 与 Dream 的行为 | 配合本书第 07 章 |
| `docs/automations.md` | 定时任务与本地触发器 | 想做"每天早报"时读 |

### 2.4 运行与集成

| 文件 | 解决什么问题 |
|---|---|
| `docs/cli-reference.md` | 所有命令与参数（`agent`/`gateway`/`webui`/`serve`/`channels`/`plugins`…） |
| `docs/chat-commands.md` | 聊天里的斜杠命令（`/compact`、`/model`、`/sessions`…） |
| `docs/webui.md` | WebUI 全部功能：工作台、活动面板、Apps、Skills、Automations |
| `docs/openai-api.md`、`docs/python-sdk.md` | OpenAI 兼容 API 的路由与载荷；把 nanobot 当库用（类、事件、会话、hook） |
| `docs/deployment.md`、`docs/troubleshooting.md` | Docker/Compose/服务化；按"装/配/模型/网关/渠道/工具"分层的排查表 |
| `docs/websocket.md`、`docs/multiple-instances.md` | 自己写客户端时的鉴权与线协议；多实例/多工作区隔离 |

### 2.5 任务导向的 `docs/guides/`（新手最该先看这里）

`guides/` 是"照着做就出结果"的短文，比上面的大文件友好。挑和新版章节直接对应的几个：

- `guides/build-a-personal-ai-agent.md`（第 06 章）、`guides/ai-agent-memory.md`（第 07 章）
- `guides/configure-mcp-tools.md`、`guides/mcp-tools-for-ai-agents.md`（第 05、08 章）
- `guides/feishu-ai-agent.md`、`telegram-ai-agent.md`、`slack-ai-agent.md`、`discord-ai-agent.md`（第 09 章）
- `guides/configure-web-search.md`（`web_search`/`web_fetch`）、`guides/configure-model-fallback.md`（第 03、06 章）
- `guides/deploy-nanobot-gateway.md`、`secure-local-ai-agent.md`、`configure-langfuse-observability.md`（第 11 章）

> ⚠️ **一个坑**：网上流传的 `docs/mcp-presets.md` 在 v0.3.5 仓库里**不存在**。MCP 文档的真实位置是 `docs/guides/configure-mcp-tools.md`（怎么配）和 `docs/webui.md#apps`（Apps 预设面板）。详见附录 D 的 D1。

> 链接核对：2026-10-27；可能变动，失效时以官方仓库为准（`docs/` 里文件会改名，以 `git ls-tree` 为准）。

---

## 三、源码阅读顺序（配合第 04 章）

**不要从 `loop.py` 第 1 行开始读**——2388 行的文件，前 200 行全是 import 和常量，很容易第三天放弃。按"数据 → 编排 → 循环 → 上下文 → 工具 → 记忆 → 外围"读，每步带一个具体问题。

| 步 | 读什么 | 带着什么问题 | 读完能回答 |
|---|---|---|---|
| 1 | `pyproject.toml` + `config/schema.py` | 默认值是多少？配置的唯一真相在哪？ | `max_tool_iterations=200`、`context_window_tokens=200000`、`max_tool_result_chars=16000` |
| 2 | `bus/events.py` + `bus/queue.py` | 消息长什么样？两个队列各管哪个方向？ | `session_key = {channel}:{chat_id}`；`publish_inbound` 在 `nanobot/bus/queue.py:37` |
| 3 | `agent/loop.py`（只读 `_process_message`） | 一个回合几步？哪步能短路？ | 七阶段 `restore→compact→command→build→run→save→respond`（`:1679-1686`） |
| 4 | `agent/runner.py`（`run`/`_run_core`） | 循环的终止条件有几个？ | `for iteration in range(spec.max_iterations)`（`:435`）+ 结果回灌 |
| 5 | `agent/context.py` + `context_governance.py` | 每次请求塞了什么？装不下怎么办？ | `build_system_prompt`（`nanobot/agent/context.py:101`）；预算 = 窗口 − 输出 − 1024 |
| 6 | `agent/tools/`（`loader`→`base`→`registry`→任选一工具） | 工具怎么被发现、过滤、执行、报错？ | `_scopes` 过滤（`nanobot/agent/tools/loader.py:100`）、`ToolRegistry.execute`（`nanobot/agent/tools/registry.py:187`） |
| 7 | `agent/memory.py` | 压缩和"巩固"是一件事吗？谁有写权限？ | `Consolidator`（`:1072`）写 `history.jsonl`；Dream 受限工具集（`:575`）改 `MEMORY.md` |
| 8 | `channels/base.py` + 一个渠道子包 + `cli/gateway_runtime.py` | 渠道怎么接入？谁组合整个系统？ | `BaseChannel._handle_message`（`nanobot/channels/base.py:255`）；MCP 在组合根 `:473` 建、`:579` 连 |
| 9 | `agent/tools/mcp.py` | 外部工具怎么变成内置工具的？ | `MCPToolWrapper`（`:595`）、`MCPProvider`（`:1346`），工具名 `mcp_<server>_<tool>` |

**三条读法建议**：① 按**类名 grep**，别背行号（`grep -rn "class AgentRunner" nanobot/`）；② 只读**调用链**——先读"谁调它"、再读"它调谁"，最后才看实现；③ 每读完一个模块**画一张图**，只写"输入 → 处理 → 输出 + 一个关键判断"，画不出来说明没读懂。

> 💡 配套练习：读完第 6 步，写一个 `Tool` 子类并声明 `_scopes = {"core", "subagent"}`，再用 `spawn` 起子 Agent 验证它被看见——这一步做通，你就真懂了 scope。

---

## 四、论文与经典

不是所有论文都要读。**工程岗只要"知道它解决什么问题、代价是什么"**，公式推导可以跳过。

### 4.1 Agent 与推理（和本书最相关）

| 论文 | 年份 / arXiv | 为什么值得读 |
|---|---|---|
| **ReAct: Synergizing Reasoning and Acting** | 2022 / 2210.03629 | `AgentRunner` 循环就是它的工程版：想一步、做一步、看结果再想。读完就懂"为什么工具结果要回灌进对话" |
| **Reflexion** | 2023 / 2303.11366 | 失败后写反思、下次带着反思重试。对照 nanobot：它只在工具报错时附一句"换个思路"，成本低一个量级 |
| **Toolformer** | 2023 / 2302.04761 | 模型自学"何时该调工具"。理解它，你就明白工具**描述**为什么比工具**实现**更影响效果 |
| **Lost in the Middle** | 2023 / 2307.03172 | 长上下文里"中间的信息最容易被忽略"。直接支撑 nanobot 两个设计：工具结果落盘、摘要放稳定前缀 |
| **Self-RAG** | 2023 / 2310.11511 | 让模型自己判断"要不要检索、结果可不可信"。做 RAG 工具前先读，能省掉很多想当然 |
| **Chain-of-Thought Prompting** | 2022 / 2201.11903 | "让模型先想再答"的起点，也是理解 reasoning content 字段的背景 |

### 4.2 训练与对齐（面试常问，理解思想即可）

| 论文 | 年份 / arXiv | 为什么值得读 |
|---|---|---|
| **Attention Is All You Need** | 2017 / 1706.03762 | Transformer 原典。不用会推导，但要知道"注意力是 Q·K 相似度加权 V" |
| **InstructGPT（RLHF 三阶段）** | 2022 / 2203.02155 | SFT → 奖励模型 → PPO 的经典流程；理解"对齐"到底在做什么 |
| **LoRA** | 2021 / 2106.09685 | 冻结原权重、只训低秩矩阵，"微调便宜"的起点。面试高频 |
| **QLoRA** | 2023 / 2305.14314 | 4-bit 量化 + LoRA，把微调塞进单卡；理解"量化换显存"的代价 |
| **DPO** | 2023 / 2305.18290 | 把"奖励模型 + RL"合并成一个分类损失，不用 PPO 也能做偏好对齐 |

### 4.3 协议与规范

| 规范 | 为什么值得读 |
|---|---|
| **MCP 规范**（https://modelcontextprotocol.io/specification） | 读懂 Tools/Resources/Prompts 三个原语归谁控制，Host–Client–Server 三层怎么分工 |
| **MCP 版本页**（https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning） | 解释"为什么 MCP 用日期号版本"，顺手纠正"v1.1/v1.2"这种说法 |
| **JSON-RPC 2.0** | MCP 的报文底座；看懂 request/response/notification 三种消息，调试时能直接读日志 |

> **读法**：① 先读摘要 + 图建立直觉；② 再看实验/消融，知道结论的边界；③ 最后才看方法细节。一篇 40 分钟够了，读不完先看中文解读建立框架，再回原文核对。
> arXiv 编号核对：2026-10-27；以标题为准，编号打不开时按标题搜索。链接可能变动，失效时以官方仓库为准。

---

## 五、MCP 生态与模型供应商文档

### 5.1 MCP 官方

| 入口 | 地址 | 用途 |
|---|---|---|
| 文档首页 / 规范 | https://modelcontextprotocol.io · https://modelcontextprotocol.io/specification | 概念、架构、SDK 索引；逐条读协议（左侧有版本切换器） |
| 扩展 / Registry / SEPs | https://modelcontextprotocol.io/extensions/overview · /registry/about · /seps | 官方扩展机制、服务注册表、协议演进提案（想知道下版会变什么看 SEPs） |
| 示例 | https://modelcontextprotocol.io/examples | 官方示例清单 |

**版本号注意**：MCP 规范用**日期号**（截至 2026-10-27 最新为 `2026-07-28`），没有 `v1.1` 这种语义化版本。看到"v1.1 规范"基本可判断作者没读原文。

### 5.2 MCP 工具链（写 Server / 调试必备）

| 资源 | 地址 | 为什么用 |
|---|---|---|
| Python SDK | https://github.com/modelcontextprotocol/python-sdk | 写第一个 MCP Server 用它，比手写 JSON-RPC 快十倍 |
| 官方 servers 集合 | https://github.com/modelcontextprotocol/servers | 参考实现，学别人的工具描述怎么写 |
| Inspector | https://github.com/modelcontextprotocol/inspector | 图形化调试：列工具、传参数、看返回 |
| 社区精选 | https://github.com/punkpeye/awesome-mcp-servers | 找现成 Server；**质量参差，先看最后更新时间** |

在 nanobot 里用起来：配置写在 `config.json` 的 `tools.mcpServers.<name>`（有 `command` 就是 stdio，`url` 以 `/sse` 结尾是 sse，否则 streamableHttp），工具会以 `mcp_<server>_<tool>` 出现。官方说明见 `docs/guides/configure-mcp-tools.md`，本书第 05、08 章有逐行解析。

### 5.3 模型供应商文档（配 provider 的第一手资料）

| 供应商 | 文档入口 | nanobot 侧怎么配 |
|---|---|---|
| Anthropic / OpenAI | https://docs.anthropic.com · https://platform.openai.com/docs | `providers.anthropic.apiKey` / `providers.openai.apiKey`；默认模型是 `anthropic/claude-opus-4-5` |
| DeepSeek / 阿里云百炼 | https://api-docs.deepseek.com · https://help.aliyun.com/zh/model-studio/ | 典型 OpenAI 兼容用法；国内网络友好 |
| 硅基流动 / OpenRouter | https://docs.siliconflow.cn · https://openrouter.ai/docs | 一个 key 调多家模型；OpenRouter 适合做 fallback |
| Ollama / vLLM | https://docs.ollama.com · https://docs.vllm.ai | 本地模型（`http://localhost:11434/v1`）/ 自建高性能 OpenAI 兼容服务 |

**对照关系**：`docs/providers.md` 讲概念 → `docs/provider-cookbook.md` 给配方 → `docs/configuration.md` 是字典。三者按这个顺序查，别一上来翻字典。

> 链接核对：2026-10-27；可能变动，失效时以官方仓库为准。

---

## 六、社区与交流

| 平台 | 地址 | 适合问什么 |
|---|---|---|
| GitHub Issues / Discussions | https://github.com/HKUDS/nanobot/issues · /discussions | 确定是 bug 的（附版本 + 系统 + 复现）；用法与方案讨论 |
| Discord / X | https://discord.gg/MnCvHqpUGB · https://x.com/nanobot_project | 实时交流（英文为主）；版本更新与演示 |
| 微信 / 飞书群 | 仓库根目录 `COMMUNICATION.md` | 中文用户交流 |
| MCP Discord | https://discord.gg/6CSzBmMkjX | 协议本身的讨论（不是 nanobot 的） |

**参与开源的四个台阶**（从易到难）：① 改文档——命令跑不通、术语不一致，这类 PR 最容易合；② 报高质量 Issue——`nanobot --version` + 系统 + 配置（**去掉密钥**）+ 期望 vs 实际 + 日志片段；③ 补测试或加一个小工具；④ 写一个能跑通的最小示例项目。
**一条纪律**：提 Issue 前先 `git log --oneline -20`，很多问题在 `main` 上已经修了。

> 链接核对：2026-10-27；可能变动，失效时以官方仓库为准。

---

## 七、工具与平台

| 类别 | 工具 | 用途 |
|---|---|---|
| 读码 | Git（含 `git log -S` 搜代码变更）、ripgrep、VS Code/Cursor | 切 tag、按类名定位、跳转调用层级 |
| 环境 | uv、ruff、basedpyright、pytest | 与 CI 一致；提 PR 前跑一遍 |
| 调试 | MCP Inspector、Docker Desktop、Jupyter | 调试 MCP Server、跑部署与沙箱实验、拆函数做小实验 |
| 观测 | Langfuse、OpenTelemetry、Prometheus + Grafana | 官方有 Langfuse 指南；`_run_turn_stage` 是天然埋点位置；网关健康检查在 `127.0.0.1:18790/health` |

> 链接核对：2026-10-27；可能变动，失效时以官方仓库为准。（本节只给类别，具体产品请按官网核对。）

---

## 八、学习路线图

### 8.1 两周速通（每天 2-3 小时）

| 天 | 内容 | 产出 |
|---|---|---|
| 1-2 | 第 01、02 章 + `docs/concepts.md` | 一句话说清 Agent 与普通调用的区别 |
| 3-4 | 第 03 章 + `docs/architecture.md` | 画出主链路图（Channel → Bus → Loop → Runner → Tool） |
| 5-7 | 第 06 章 + 本文 §1.4；第 04 章按第三节顺序读第 1-5 步 | 本地跑通对话；说出七阶段与 200 次迭代在哪定义 |
| 8-9 | 第 05、08 章 + 写一个 `Tool` 子类 | 自己的工具能被主 Agent 和子 Agent 调用 |
| 10-12 | 第 07 章 + `docs/memory.md`；第 09 章 + 接一个渠道 | 解释"压缩"与"Dream"分工；手机上和自己的 Agent 对话 |
| 13-14 | 第 13 章抽 20 题 + 第 16 章话术 | 90 秒讲清项目 |

### 8.2 六周深入（每天 1-2 小时）

- **第 1-2 周**：01-06 章 + 通读 `docs/configuration.md` 的 `agents`/`tools`/`channels` 三节。
- **第 3 周**：第 04 章全流程 + 手画 `loop.py`/`runner.py` 两张图，给 Tool 机制写一篇笔记。
- **第 4 周**：第 05、08 章 + 用 Python SDK 写一个 MCP Server 接进 nanobot。
- **第 5 周**：第 07、10、11 章 + 用 Docker 部署一次，打开 `restrict_to_workspace` 与渠道 `allowFrom`。
- **第 6 周**：第 12 章挑一个项目做完整实现 + 第 13/16 章面试准备。

### 8.3 持续学习（三个月一次"体检"）

① 每月看一次 Releases，读带 `breaking` 的条目；② 每季重新 `git ls-tree` 一遍 `docs/`，确认本清单里的文件名还在（附录 C 有命令）；③ 每季挑一个笔记里写过的"事实"重核一遍——**行号会漂移，结论要重验**；④ 长期维护一份"证据库"（文件名 + 类名 + 一行代码），比收藏夹有用一百倍。

---

## 九、书籍与核对习惯

### 9.1 书（少而精）

| 书 | 为什么读 |
|---|---|
| 《动手学深度学习》（李沐等） | 免费、有可运行代码，补基础最省时间 |
| 《Speech and Language Processing》（Jurafsky & Martin） | NLP 经典，免费在线版，查概念用 |
| 《Designing Data-Intensive Applications》 | 会话持久化、队列、幂等——读懂 `session/` 与 `bus/` 会轻松很多 |
| 《The Pragmatic Programmer》 | 讲"怎么验证自己的假设"，和本章"每个数字带出处"是一套思路 |

### 9.2 三条可执行的自检

```bash
git ls-tree -r v0.3.5 --name-only docs/ | head -40   # 1. 文件还在吗
grep -rn "class MCPProvider" nanobot/                # 2. 类还在吗（行号会漂移，类名不会）
grep -n "max_tool_iterations" nanobot/config/schema.py  # 3. 默认值还是 200 吗
```

**核对日期：2026-10-27**。如果你在半年后看到这一章，请把上面三条跑一遍再相信任何数字；**链接可能变动，失效时以官方仓库为准。**

### 9.3 本章的三条"不要"

① 不要用 star 数判断项目质量，也别写进简历——它每天在变，且和你的能力无关；② 不要相信"某某行数"这种宣传口径，除非它给了统计脚本和版本（第 16 章 §5 就是一个真实的反例复盘）；③ 不要读三个月前的二手教程当权威，`docs/configuration.md` 和 `nanobot --help` 才是第一手。

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 建议 |
|---|---|---|
| 用 `Stars` 列给仓库排名（37K+/95K+/…） | star 数随时变，与技术判断无关 | 删掉该列，改"定位 + 一句话价值" |
| "Nanobot 4000 行代码涵盖完整 Agent 能力" | 329 个 `.py` / 约 99,398 行 SLOC；`loop.py` 2,388 行 | 换可复现口径，或只作定性描述 |
| 以 DeepWiki 具体页面为主要文档来源 | 第三方 AI 生成，可能与源码不一致，子页面 URL 未核实 | 主用官方 `docs/` 与源码，DeepWiki 只作补充 |
| `docs/mcp-presets.md` 作 MCP 文档入口 | v0.3.5 无此文件 | 改指 `docs/guides/configure-mcp-tools.md` 与 `docs/webui.md#apps` |
| 资源以"博客文章/视频"为主 | 二手内容更新慢、易失真 | 顺序改为：官方文档 → 源码 → 论文 → 社区 |
| 四周速成 / 写死"134 道面试题" | 新版 17 章塞不进四周；题目数会调整 | 两周速通 + 六周深入；不写死数量 |

---

## 附录 B：进阶理解

**1. 为什么"官方文档优先"不是官僚主义？**
因为文档和代码在同一个仓库、同一个 PR 里改，博客不会。一份三个月前的教程可能还在教 `bot_token`，而 v0.3.5 的字段是 `token`（且必须 `"enabled": true` 才启动）。**腐坏速度：博客 > wiki > docs/ > 源码**，越靠右越可信。

**2. 为什么本清单坚持不写字数、star、下载量？**
它们是**衰减最快的指标**，还会诱导错误判断。"37K stars"不能告诉你并发模型是否合理；"200 次迭代、子 Agent 继承"能。选资源时先问"这份材料会不会过期"。

**3. 为什么源码要按"数据 → 编排 → 循环 → 上下文"读？**
因为依赖方向就是这么走的：`AgentRunSpec`（`runner.py:89`）是数据契约，`AgentLoop` 组装它，`AgentRunner` 消费它，`ContextGovernor` 在消费中改它。**先读数据的人读编排不会迷路；先读编排的人每遇一个字段就得跳回去查。**

**4. 为什么三类文档要用三种读法？**
`concepts.md` 通读（建名词表）、`architecture.md` 精读（对照源码）、`configuration.md` 检索（Ctrl+F 查字段）。用同一种方式读三类文档，结果要么读不完，要么记不住。

**5. 为什么引用要给版本，不给 `latest`？**
`latest` 会跟着发布变，`0.3.5` 不会。同理 MCP 用日期号版本，表达的是"这一版何时冻结"，避免"v1.1 兼容 v1.0 吗"的歧义。

**6. 为什么"能自己核对"比"记得多"重要？**
半年后行号大概率全变，但 `grep -rn "class MCPProvider"` 永远有效。本章真正想给的不是清单，是**核对清单的方法**（附录 C）。

---

## 附录 C：如何高效读官方文档

### C.1 先看"路由表"，不要从头读

`docs/README.md` 是"你的处境 → 读哪篇"的表格。**先定位再读**，比按字母顺序快十倍；找不到入口就看文件名，英文命名很直白（`configuration.md` 就是配置）。

### C.2 三类文档三种读法

| 类型 | 例子 | 读法 |
|---|---|---|
| 概念型 | `concepts.md`、`architecture.md` | 通读一遍，画出名词表 |
| 配方型 | `guides/*.md`、`provider-cookbook.md` | 照着抄，跑通再改 |
| 字典型 | `configuration.md`、`cli-reference.md` | 只搜不读（Ctrl+F 字段名） |

**反例**：把 `configuration.md`（10 万字符级别）从第一行读到最后一行——99% 的人会在第三节放弃。

### C.3 边读边对账

读到一个结论，立刻用 grep 验证。例如读到"工具名是固定的"：

```bash
grep -n 'name = "web_search"' nanobot/agent/tools/web.py
# → 367:    name = "web_search"
```

**对得上 → 记进笔记（带文件:行号）；对不上 → 以源码为准，并把差异记下来**（这就是"勘误"的原料）。

### C.4 用 wiki 补"发布版视角"，用 `docs/` 补"最新视角"

写文章/做分享用 `nanobot.wiki/docs/0.3.5/...`（与发布版一致）；读源码/提 PR 用仓库 `docs/`（跟着 `main` 走，可能已修）。两者冲突时，先看 `git log` 里该文件的最后修改时间，再决定信谁。

### C.5 建立自己的"三行笔记"

```text
结论：工具按 scope 过滤，默认只有 core
证据：agent/tools/loader.py:100 + tools/base.py:208
边界：v0.3.5；Dream 不走这套（memory.py:575 起）
```

**为什么三行**：结论会被复用，证据可被复核，边界防止过度概括。三行写不出来，说明这节还没读懂——回去读，别急着往下走。

---

## 附录 D：原文勘误

下表只列**旧 17 章本身有问题**的地方（不是版本演进导致的）。

| # | 旧文位置 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| D1 | 三、DeepWiki 文档链接 | 把 `docs/mcp-presets.md` 当官方 MCP 文档入口（该文件在 v0.3.5 中不存在） | MCP 配置文档是 `docs/guides/configure-mcp-tools.md`；预设面板在 `docs/webui.md#apps` | `git ls-tree -r v0.3.5 --name-only docs/`（只有 `guides/configure-mcp-tools.md`、`guides/mcp-tools-for-ai-agents.md`） |
| D2 | 一、核心 GitHub 仓库推荐表 | 用 `Stars` 列（37K+/95K+/135K+…）当推荐依据 | star 数随时变、与技术判断无关；应给"定位 + 一句话价值" | —（见本章开头声明） |
| D3 | 一、1.1 行 | "4000 行代码涵盖完整 Agent 能力" | 329 个 `.py` / 约 99,398 行 SLOC；`loop.py` 2,388 行、`runner.py` 1,374 行 | `pyproject.toml:3`、`nanobot/agent/loop.py:196` |
| D4 | 三、3.1 / 3.2 | 把 DeepWiki 子页面 URL（如 `deepwiki.com/HKUDS/nanobot/agent-loop`）当可引用文档 | DeepWiki 是第三方 AI 解读，路径未核实且可能与源码不一致；应以官方 `docs/` 与源码为准 | 官方入口：`https://nanobot.wiki/docs/latest/getting-started/nanobot-overview`（核对 2026-10-27） |
| D5 | 二、博客文章推荐表 | 列出具体文章标题与平台（如"深入解析 Nanobot AgentLoop 源码 / CSDN"），无法核实是否存在 | 二手文章标题不可作为资源清单；改为"检索建议 + 官方一手来源" | —（该类标题无法核实，已删除） |
| D6 | 五、视频教程推荐表 | 列出具体视频与讲师，无法核实链接与时效 | 只保留可长期检索的关键词（如"Transformer 论文精读""MCP 入门"），不给具体视频 | — |
| D7 | 四、4.4 MCP 与协议 | 用 `v1.1`/语义化版本理解 MCP 规范 | MCP 规范用**日期号**（截至 2026-10-27 最新 `2026-07-28`），没有 `v1.1` | `https://modelcontextprotocol.io/specification`（核对 2026-10-27） |
| D8 | 八、8.1 四周计划 | 第 3 周就排"模型训练 + RAG 实践"，与章节顺序脱节 | 训练/RAG 属可选深入方向；主线是"概念 → 装机 → 源码 → 工具/记忆 → 渠道 → 安全部署 → 面试" | 见本章第八节 |
| D9 | 总结 | "4000 行源码是理解 Agent 原理的最佳教材" | 教材价值在**结构与链路清晰**，不在行数；行数说法本身不成立 | `nanobot/agent/loop.py:196`、`nanobot/agent/runner.py:141` |
| D10 | 全章 | 未给任何链接的核对日期 | 所有外链必须带核对日期，并声明"链接可能变动，失效时以官方仓库为准" | 本章开头声明 |

---

## 本章总结

资源清单的价值不在"多"，而在**能不能被验证**。这一章给你的其实是一套优先级：

1. **官方文档**（`nanobot.wiki` + 仓库 `docs/`）解决"它是什么、怎么配"——第一手，会随版本更新；
2. **源码**解决"它到底怎么实现"——唯一的最终裁决者，按"数据 → 编排 → 循环 → 上下文 → 工具 → 记忆 → 外围"读；
3. **论文**解决"为什么这样设计、代价是什么"——工程岗理解思想即可，别陷进公式；
4. **社区与工具**解决"踩坑了找谁、调试用什么"——Issues 先搜再提，Inspector 先跑再看。

> 💡 **下一步读什么**：这份清单是本书的最后一章，接下来该你自己走了——回 [03 - 架构深潜](../03-architecture-deep-dive/README.md) 把主链路图默画一遍，然后打开 `nanobot/agent/tools/loader.py` 读第 92-124 行，那是"官方文档没写、但你必须知道"的那类细节。想练表达，去 [16 - STAR 面试话术](../16-star-interview-script/README.md)。
