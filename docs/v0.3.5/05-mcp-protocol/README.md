# 05 - MCP 协议详解

> 🎯 **本章目标**：搞清 MCP 的三大原语、JSON-RPC 消息格式与三种传输方式，并能对照 v0.3.5 源码说清 nanobot 怎样把外部 MCP Server 变成模型可调用的工具。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

---

## 目录

- [5.1 MCP 协议概述](#51-mcp-协议概述)
- [5.2 核心问题：M×N 困境](#52-核心问题mn-困境)
- [5.3 三角架构](#53-三角架构)
- [5.4 三大核心原语](#54-三大核心原语)
- [5.5 通信协议与传输层](#55-通信协议与传输层)
- [5.6 MCP vs Function Calling](#56-mcp-vs-function-calling)
- [5.7 MCP vs Skill vs Function Call 三者关系](#57-mcp-vs-skill-vs-function-call-三者关系)
- [5.8 MCP 在 Nanobot 中的实现](#58-mcp-在-nanobot-中的实现)
- [5.9 MCP 生态发展时间线](#59-mcp-生态发展时间线)
- [5.10 面试高频题与答案](#510-面试高频题与答案)
- [5.11 练习：写一个简单的 MCP Server](#511-练习写一个简单的-mcp-server)
- [5.12 本章总结](#512-本章总结)

---

![MCP 协议漫画](../../../comics/05-mcp-protocol.png)

*MCP 像 AI 世界的 USB-C：把"每个应用 × 每个工具"的专有接线，换成一套标准接口。*

## 5.1 MCP 协议概述

| 项目 | 内容 |
|---|---|
| 全称 | Model Context Protocol（模型上下文协议） |
| 发布方 | Anthropic |
| 首次发布 | 2024 年 11 月（规范修订号 `2024-11-05`） |
| 消息格式 | JSON-RPC 2.0 |
| 传输方式 | stdio / HTTP+SSE / Streamable HTTP |
| 目标 | 让 AI 应用用同一套协议发现并调用外部工具、读取数据 |

一句话定义：

> **MCP 是 AI 应用连接外部工具和数据源的标准化协议，地位类似外设世界的 USB-C。**

### 规范版本要写日期号

这是本章第一个必须纠正的记忆点：**MCP 规范用日期做修订号，不存在语义化版本号。** 截至 v0.3.5 写文档时，公开过四个修订：

| 修订号 | 关键变化 |
|---|---|
| `2024-11-05` | 首个公开修订：JSON-RPC 2.0、stdio 与 HTTP+SSE、Tools/Resources/Prompts、Sampling、Roots |
| `2025-03-26` | 用 **Streamable HTTP** 取代 HTTP+SSE；基于 OAuth 2.1 的授权框架；JSON-RPC batching；tool annotations |
| `2025-06-18` | 移除 batching；结构化工具输出、elicitation、resource links；HTTP 请求要求带 `MCP-Protocol-Version` 头 |
| `2025-11-25` | 实验性 tasks、icons；增强授权发现；Sampling 支持 tools；JSON Schema 2020-12 成为默认方言 |

nanobot 自己不硬编码协议版本，它把协商交给官方 `mcp` SDK：`ClientSession.initialize()`（`nanobot/agent/tools/mcp.py:1138`）完成版本握手。依赖声明是 `mcp>=1.26.0,<2.0.0`（`nanobot/pyproject.toml:44`），本机实测安装的 1.30.0 中 `LATEST_PROTOCOL_VERSION` 为 `2025-11-25`。

---

## 5.2 核心问题：M×N 困境

有 **M 个 AI 应用**（Claude Desktop、Cursor、nanobot……）和 **N 个工具/数据源**（GitHub、数据库、文件系统……）。没有标准协议时，每个应用都要为每个工具写适配代码，总量是 **M × N**；有 MCP 后，应用实现一次 Client、工具实现一次 Server，总量变成 **M + N**。

| 应用数 M | 工具数 N | 无 MCP（M×N） | 有 MCP（M+N） | 节省 |
|---:|---:|---:|---:|---:|
| 3 | 3 | 9 | 6 | 33% |
| 5 | 10 | 50 | 15 | 70% |
| 10 | 20 | 200 | 30 | 85% |
| 20 | 50 | 1000 | 70 | 93% |

```
没有 MCP：应用 ──专用代码──→ 工具        （M×N 条线）
有了 MCP：应用 ──MCP Client──┐
                             ├── MCP 协议 ── MCP Server ── 工具
             应用 ──MCP Client──┘          （M+N 条边）
```

还有一个容易忽略的收益：MCP 统一了**工具发现**。Client 在运行时调用 `tools/list` 就知道 Server 有哪些工具、参数 schema 长什么样；工具升级后应用不用重新编译，重启连接即可。静态 Function Calling 定义做不到这一点。

---

## 5.3 三角架构

| 角色 | 职责 | 类比 | nanobot 中的对应 |
|---|---|---|---|
| **MCP Host** | 运行 AI 模型、管理多个连接的宿主应用 | 电脑主机 | `Nanobot` 门面 / `AgentLoop` |
| **MCP Client** | Host 内部的协议客户端，一个 Client 连一个 Server | USB 控制器 | `mcp` SDK 的 `ClientSession` |
| **MCP Server** | 提供工具、资源、提示模板的进程或服务 | USB 外设 | 外部进程 / 远程服务 |

```
┌─────────────────────── MCP Host（nanobot）───────────────────────┐
│ AgentRunner ── ToolRegistry ── mcp_weather_get_weather          │
│                                      │                           │
│                              MCPToolWrapper                      │
│                                      │                           │
│                         ClientSession（MCP Client）              │
└──────────────────────────────────┬───────────────────────────────┘
                                   │ JSON-RPC 2.0
                     stdio / HTTP+SSE / Streamable HTTP
                    ┌──────────────▼──────────────┐
                    │ MCP Server（weather_server） │
                    │ tools: get_weather           │
                    └─────────────────────────────┘
```

### 重要澄清

- **MCP Client 不是独立应用**，它是 Host 里的协议模块。一个 Host 可以有多个 Client，每个 Client 对应一个 Server 连接。
- nanobot 源码里没有叫 `MCPClient` 的类。它直接用 SDK 的 `ClientSession`（`nanobot/agent/tools/mcp.py:1137`），外面再包一层 `MCPProvider` 管理连接生命周期。

---

## 5.4 三大核心原语

| 维度 | Tools（工具） | Resources（资源） | Prompts（提示模板） |
|---|---|---|---|
| 控制方 | 模型（LLM 决定调用） | 应用（Host 决定读取） | 用户（在 UI 里选择） |
| 用途 | 执行操作 | 提供只读数据 | 复用提示词 |
| 副作用 | 可能有 | 无 | 无 |
| 列举方法 | `tools/list` | `resources/list` | `prompts/list` |
| 使用方法 | `tools/call` | `resources/read` | `prompts/get` |
| 类比 | 动词（做事） | 名词（数据） | 套路（模板） |

### 5.4.1 Tools

工具是 Server 暴露的可执行操作，定义包含 `name`、`description`、`inputSchema`：

```json
{
  "name": "get_weather",
  "description": "Get current weather for a city",
  "inputSchema": {
    "type": "object",
    "properties": { "city": { "type": "string" } },
    "required": ["city"]
  }
}
```

典型流程：Client 调 `tools/list` 拿到工具数组 → Host 把定义转成模型可识别的格式 → 模型决定调用 → Client 发 `tools/call` → Server 执行并返回 `content[]` → Host 把结果回灌模型。

### 5.4.2 Resources

资源是只读数据，用 URI 标识（如 `file:///workspace/README.md`、`db://users/123`），由应用决定何时读取，而不是模型自己决定。可以把它理解成 REST 的 `GET`：拿数据，不改状态。

### 5.4.3 Prompts

提示模板是带参数的、可复用的提示词，由用户在界面上选择。例如 `code_review` 模板接收 `code` 和 `language` 两个参数，一次定义、多次使用。

### 在 nanobot 里，三者都会被包装成"工具"

MCP 的三种原语最终都注册进同一个 `ToolRegistry`，对模型来说都是可调用工具：

- Tools → `mcp_<server>_<tool>`（`nanobot/agent/tools/mcp.py:609`）
- Resources → `mcp_<server>_resource_<name>`（`:771`）
- Prompts → `mcp_<server>_prompt_<name>`（`:875`）

但有个开关：只有 `enabledTools` 含 `"*"`（默认）时，resources 和 prompts 才注册；一旦写了具体工具白名单，nanobot 就认为你想收紧能力，把 resources/prompts 一并跳过（`nanobot/agent/tools/mcp.py:1182-1190`）。

---

## 5.5 通信协议与传输层

### JSON-RPC 2.0

MCP 的每条消息都是一个 JSON 对象。请求带 `id`、`method`、`params`：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": { "name": "get_weather", "arguments": { "city": "Beijing" } }
}
```

成功响应把 `result` 配回同一个 `id`；失败响应则用 `error` 对象（含 `code`、`message`）。`id` 用来配对请求与响应；通知（notification）没有 `id`，也不需要响应。

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { "content": [{ "type": "text", "text": "Beijing: sunny, 22C" }] }
}
```

### 握手：initialize

连接建立后，Client 先发 `initialize`，Server 返回自己支持的协议版本、能力和名称；Client 再发 `notifications/initialized` 表示就绪。从 `2025-06-18` 起，HTTP 传输的后续请求还要带 `MCP-Protocol-Version` 头。nanobot 把这一步交给 SDK：`await session.initialize()`（`nanobot/agent/tools/mcp.py:1138`）。

### 三种传输方式

| 维度 | stdio | HTTP+SSE | Streamable HTTP |
|---|---|---|---|
| 出现时间 | `2024-11-05` | `2024-11-05` | `2025-03-26` 起 |
| 通道 | 父子进程的 stdin/stdout | POST 上行 + SSE 下行，两条通道 | 单个 HTTP 端点，按需升级为 SSE |
| 适用 | 本地工具 | 旧版远程服务 | 现代远程服务 |
| 优点 | 简单、进程隔离、无网络 | 可远程 | 可远程、可无状态、单端点 |
| 缺点 | 只能同机 | 双通道、实现复杂 | 需要正确处理会话头 |

> 记法：**stdio 是本地管道，SSE 是"旧的双车道"，Streamable HTTP 是"一条路按需加一条下行车道"。**

### nanobot 如何自动选择

`connect_mcp_servers` 里有一段很短的判定（`nanobot/agent/tools/mcp.py:1021-1031`）：

```python
# nanobot/agent/tools/mcp.py:1021-1031（节选）
transport_type = cfg.type
if not transport_type:
    if cfg.command:
        transport_type = "stdio"
    elif cfg.url:
        transport_type = "sse" if cfg.url.rstrip("/").endswith("/sse") else "streamableHttp"
    else:
        logger.warning("MCP server '{}': no command or url configured, skipping", name)
        return False
```

规则可以背下来：

1. 配了 `type` 就听 `type` 的（只接受 `stdio` / `sse` / `streamableHttp`，大小写敏感）；
2. 否则有 `command` → `stdio`；
3. 否则有 `url` 且路径以 `/sse` 结尾 → `sse`；
4. 其他 `url` → `streamableHttp`；
5. 两者都没有 → 打日志并跳过这个 Server。

两个工程细节：stdio 启动前会经过 `_normalize_windows_stdio_command`（`:1068`），处理 Windows 上 `npx`/`node` 这类 `.cmd` 包装脚本；HTTP/SSE 连接前会先做 TCP 探测（`:1081`、`:1113`），端口不通就不进入 SDK 的异步上下文，避免清理阶段把事件循环带崩。

---

## 5.6 MCP vs Function Calling

先给结论：**两者不在一个层级，是互补关系。**

| 维度 | MCP | Function Calling |
|---|---|---|
| 本质 | 应用层 Client-Server 协议 | 模型 API 的一项能力 |
| 工具定义在哪 | MCP Server | 调用方代码里 |
| 工具执行在哪 | MCP Server | 调用方代码里 |
| 发现方式 | 运行时 `tools/list` | 开发时写死 |
| 复用范围 | 任何支持 MCP 的 Host | 每个应用各写一份 |
| 传输 | stdio / HTTP | 无（随模型 API 走） |

两者的配合链路：

```
连接 Server → tools/list → 转成 Function Calling 格式 → 注册进 ToolRegistry
→ 模型返回 tool_calls → 找到 mcp_<server>_<tool> → 转成 MCP tools/call
→ Server 执行 → 结果回灌模型
```

对应源码：schema 转换是 `_normalize_schema_for_openai`（`nanobot/agent/tools/mcp.py:496`），注册是 `registry.register(wrapper)`（`:1161`），执行是 `session.call_tool`（`:632-635`）。

> 一句话记忆：**MCP 负责"工具有没有、在哪执行"，Function Calling 负责"模型选不选、选哪个"。**

---

## 5.7 MCP vs Skill vs Function Call 三者关系

| 维度 | MCP Tool | Skill（SKILL.md） | Function Call |
|---|---|---|---|
| 本质 | 外部可执行工具 | 注入上下文的提示与流程 | 模型选择工具的机制 |
| 定义位置 | MCP Server | `skills/` 下的 Markdown | 工具类的 schema |
| 运行位置 | Server 进程/服务 | 模型内部（影响思考） | 运行时执行工具 |
| 跨框架复用 | 是 | 否（nanobot 约定） | 否 |
| 典型例子 | `mcp_github_create_issue` | "代码审查时先读 diff" | 模型返回 `tool_calls` |

三者协作可以压缩成一句话：**Skill 决定"怎么想"，Tool 决定"能做什么"，Function Calling 决定"这一轮做哪件"。**

对应到 v0.3.5：Skill 内容被拼进 system prompt（`nanobot/agent/context.py:132-143`），MCP 工具被注册进 `ToolRegistry`，两者在模型请求里汇合。

---

## 5.8 MCP 在 Nanobot 中的实现

### 5.8.1 配置：`tools.mcpServers`

MCP 配置在 `config.json` 的 `tools.mcpServers` 下，是一个**以 Server 名为 key 的对象**，不是数组（`nanobot/config/schema.py:418`）：

```json
{
  "tools": {
    "mcpServers": {
      "weather": {
        "command": "python",
        "args": ["C:/mcp/weather_server.py"],
        "toolTimeout": 30,
        "enabledTools": ["*"]
      },
      "docs": {
        "url": "https://mcp.example.com/mcp",
        "headers": { "Authorization": "Bearer ${MCP_TOKEN}" }
      },
      "legacy": { "url": "https://mcp.example.com/sse" }
    }
  }
}
```

字段定义在 `MCPServerConfig`（`nanobot/config/schema.py:362-374`）：

| JSON 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `type` | `stdio` / `sse` / `streamableHttp` | 省略（自动判定） | 显式指定传输方式 |
| `command` | string | `""` | stdio：要启动的可执行文件 |
| `args` | string[] | `[]` | stdio：命令行参数 |
| `env` | object | `{}` | stdio：追加环境变量 |
| `cwd` | string | `""` | stdio：Server 的工作目录 |
| `url` | string | `""` | HTTP/SSE：端点地址 |
| `headers` | object | `{}` | HTTP/SSE：自定义请求头 |
| `auth` | `"oauth"` | 省略 | 远程 OAuth；token 不写回 config |
| `toolTimeout` | int | `30` | 单次工具调用超时（秒） |
| `enabledTools` | string[] | `["*"]` | 工具白名单；`["*"]` = 全部（含 resources/prompts） |

两个容易踩的坑：`enabledTools: []` 不是"不限制"，而是**一个都不注册**；白名单里写原始名 `get_weather` 或包装名 `mcp_weather_get_weather` 都行（`nanobot/agent/tools/mcp.py:1151-1152`）。

### 5.8.2 连接与注册：`connect_mcp_servers`

核心函数是 `connect_mcp_servers`（`nanobot/agent/tools/mcp.py:1000`）。它按 Server 逐个处理，失败一个不影响其他：

```
for name, cfg in mcp_servers.items():
    1. 判定传输方式（type / command / url）
    2. 建立传输（stdio_client / sse_client / streamable_http_client）
    3. ClientSession + initialize()
    4. tools/list
    5. 每个工具创建 MCPToolWrapper → registry.register()
    6. enabledTools 允许时，再注册 resources / prompts
```

关键行：建立 stdio 是 `stdio_client(params)`（`:1079`），建立会话是 `ClientSession(read, write)`（`:1137`），列举工具是 `await session.list_tools()`（`:1140`），包装注册在 `:1147-1161`，resources/prompts 在 `:1190-1226`。

### 5.8.3 工具命名与 schema 规范化

工具名由三部分拼成，再做一次清洗：

```python
# nanobot/agent/tools/mcp.py:609（节选）
self._name = _sanitize_mcp_tool_name(f"mcp_{server_name}_{tool_def.name}")
```

`_sanitize_mcp_tool_name`（`:195`）会把非 `[a-zA-Z0-9_-]` 的字符换成 `_`；超过 64 字符时保留前缀并追加 8 位 SHA-1 摘要（`:181-192`）。所以 `weather` + `get_weather` → `mcp_weather_get_weather`，`my-srv` + `read.file` → `mcp_my-srv_read_file`。

schema 也不是原样透传：`_normalize_schema_for_openai`（`:496`）会重写本地 `$ref`、把 `type: ["string","null"]` 这类可空写法归一化，并给 object 补上 `properties` / `required`。它服务于所有兼容 OpenAI 工具格式的 Provider，不只是 OpenAI。

### 5.8.4 生命周期归"组合根"，不归 Loop

这是 v0.3.5 和旧文档差别最大的设计点。`MCPProvider`（`nanobot/agent/tools/mcp.py:1346`）负责持有连接、动态注册工具、热重载和关闭；`AgentLoop` 只拿到一个已经可用的 `ToolRegistry`。创建和关闭都发生在组合根：

```python
# nanobot/nanobot.py:134-142（节选）
tools = ToolRegistry()
mcp_provider = MCPProvider.from_config(config, tools)
loop = AgentLoop.from_config(config, ..., tool_registry=tools)
return cls(loop, config=config, mcp_provider=mcp_provider)
```

- 运行前连接：`await self._mcp_provider.connect()`（`nanobot/nanobot.py:195-196`）
- 关闭：`await self._mcp_provider.aclose()`（`:352-353`）
- CLI `agent`：`nanobot/cli/agent.py:166` 创建、`:194` 关闭
- `serve`（OpenAI 兼容 API）：`nanobot/cli/commands.py:389` 创建、`:428` 关闭

`AgentLoop.from_config` 的 docstring 把这件事说得很直白：tool registry 由调用方持有，方便组合根与 `MCPProvider` 共享（`nanobot/agent/loop.py:468-469`）。

```
组合根（Nanobot / CLI / serve）
  ├── ToolRegistry
  ├── MCPProvider.from_config(config, registry)
  │      └── connect() → connect_mcp_servers() → registry.register(...)
  └── AgentLoop.from_config(..., tool_registry=registry)
         └── AgentRunner 只负责"执行工具"，不负责"连接 MCP"
```

好处是：连接可以在多次 `run()` 之间复用；WebUI/网关可以调用 `reload()` 热更新配置（`:1476`）；测试时可以注入假 registry，不必真的起进程。

`MCPToolWrapper.execute`（`:627`）用 `asyncio.wait_for` 给调用加硬超时（`:632`），瞬态错误重试一次、间隔 1 秒（`:658-667`），会话被 Server 终止时通过重连回调刷新 session（`:651-657`）。结果里的图片会解码保存为本地 artifact，只把路径和元数据回给模型，避免 base64 撑爆上下文（`:704-719`）。

---

## 5.9 MCP 生态发展时间线

```
2024-11-05  首个 MCP 规范修订：stdio + HTTP+SSE，三大原语
2025-03-26  用 Streamable HTTP 取代 HTTP+SSE；OAuth 2.1 授权框架
2025-06-18  移除 batching；结构化工具输出；elicitation；协议版本头
2025-11-25  实验性 tasks；icons；授权发现增强；Sampling 支持 tools
2026-02-02  nanobot 首个公开版本
2026-09-16  nanobot v0.3.5（本章对应版本）
```

> 生态采用方面，不必背具体公司的支持日期，讲清"主流 IDE、桌面 AI 应用和 Agent 框架陆续支持"即可。

---

## 5.10 面试高频题与答案

### Q1：什么是 MCP？一句话解释

> "MCP 是 Anthropic 在 2024 年 11 月发布的模型上下文协议，用 JSON-RPC 2.0 把 AI 应用和外部工具/数据源解耦。它像 AI 世界的 USB-C：工具实现一次 MCP Server，所有支持 MCP 的应用都能用。"

### Q2：MCP 解决的核心问题是什么？

> "解决 M 个应用 × N 个工具的集成碎片化。没有协议时是 M×N 份适配代码，有 MCP 后变成 M 个 Client 加 N 个 Server。规模越大节省越明显，而且工具可以在运行时通过 `tools/list` 动态发现。"

### Q3：三大原语分别由谁控制？

> "Tools 由模型控制，用于执行有副作用的操作；Resources 由应用控制，是只读数据；Prompts 由用户选择，是可复用模板。对应的方法是 `tools/list`、`resources/list`、`prompts/list`。"

### Q4：MCP 和 Function Calling 是什么关系？

> "不同层级、互补。Function Calling 是模型 API 的能力，负责让模型选择工具并生成参数，执行仍在调用方；MCP 是应用层协议，负责工具的定义、发现和执行。nanobot 的做法是：从 MCP Server 拿工具，转成 Function Calling 格式给模型，模型选中后再翻译回 `tools/call`。"

### Q5：nanobot 怎么集成 MCP？

> "配置在 `tools.mcpServers`，每个 Server 一个对象，字段有 `command`/`url`、`toolTimeout`、`enabledTools`。组合根创建 `MCPProvider`，调用 `connect()` 时由 `connect_mcp_servers` 建立连接、`initialize`、`list_tools`，把每个工具包成 `MCPToolWrapper` 注册进 `ToolRegistry`，名字是 `mcp_<server>_<tool>`。关闭由 `aclose()` 负责，`AgentLoop` 不拥有 MCP 生命周期。"

---

## 5.11 练习：写一个简单的 MCP Server

目标：用官方 SDK 写一个 stdio MCP Server，并让 nanobot 把它注册成 `mcp_weather_get_weather`。

### 步骤 1：写 Server

MCP 的 Python SDK 提供 `FastMCP`，不用自己处理 JSON-RPC。新建 `weather_server.py`：

```python
# weather_server.py
# 依赖：pip install "mcp>=1.26"
from mcp.server.fastmcp import FastMCP

# 创建一个名为 weather 的 MCP Server
mcp = FastMCP("weather")


@mcp.tool()
def get_weather(city: str) -> str:
    """返回某个城市的模拟天气（示例数据，不访问网络）。"""
    fake = {
        "Beijing": "Beijing: sunny, 22C",
        "Shanghai": "Shanghai: cloudy, 25C",
    }
    return fake.get(city, f"{city}: unknown, 20C")


if __name__ == "__main__":
    # 不传参数时使用 stdio：读 stdin、写 stdout
    mcp.run()
```

要点：`@mcp.tool()` 把函数名、docstring、参数类型自动转成 `tools/list` 的 schema；stdio Server 的 **stdout 只能写 JSON-RPC**，日志必须走 stderr，FastMCP 默认就把日志写 stderr；`mcp.run()` 默认就是 stdio。想先确认不报错，可以手动 `python weather_server.py`——它会等待 stdin，没有输出是正常的。

### 步骤 2：接进 nanobot

编辑 `~/.nanobot/config.json`，在 `tools` 下加 `mcpServers`（注意是对象，不是数组）：

```json
{
  "tools": {
    "mcpServers": {
      "weather": {
        "command": "python",
        "args": ["C:/mcp/weather_server.py"],
        "toolTimeout": 30
      }
    }
  }
}
```

Windows 下建议把 `command` 写成 Python 的绝对路径，避免 PATH 里是另一个解释器。改完后：

```bash
nanobot status                 # 只检查配置，不会列 MCP 工具
nanobot agent -m "查一下上海天气"
```

模型会调用 `mcp_weather_get_weather`，结果再交回模型组织语言。

### 步骤 3：确认它真的注册了

`nanobot status` 不显示 MCP 工具。想直接验证注册结果，可以跑一个最小脚本：

```python
# verify_nanobot_mcp.py
import asyncio
import sys

from nanobot.agent.tools.mcp import connect_mcp_servers
from nanobot.agent.tools.registry import ToolRegistry
from nanobot.config.schema import MCPServerConfig


async def main() -> None:
    registry = ToolRegistry()
    servers = {
        "weather": MCPServerConfig(
            command=sys.executable,
            args=["C:/mcp/weather_server.py"],
        )
    }
    conns = await connect_mcp_servers(servers, registry)
    print("REGISTERED:", registry.tool_names)
    print("RESULT:", await registry.execute("mcp_weather_get_weather", {"city": "Shanghai"}))
    for conn in conns.values():
        await conn.aclose()


asyncio.run(main())
```

**本机实测结果**（Python 3.13、mcp SDK 1.30.0）：

```
REGISTERED: ['mcp_weather_get_weather']
RESULT: Shanghai: cloudy, 25C
```

这说明代码片段、工具命名规则和调用链都真实可跑。远程 HTTP Server 部分没有在本机验证，接远程服务时请以对方文档为准。

---

## 5.12 本章总结

```
MCP 核心
├── 定位：AI 工具生态的 USB-C，解决 M×N → M+N
├── 规范：日期修订号 2024-11-05 / 2025-03-26 / 2025-06-18 / 2025-11-25
├── 架构：Host（应用）→ Client（会话）→ Server（工具）
├── 原语：Tools（模型控制）/ Resources（应用控制）/ Prompts（用户控制）
├── 传输：stdio（本地）/ HTTP+SSE（旧）/ Streamable HTTP（新）
└── 与 FC 的关系：MCP 供给侧，Function Calling 选择侧

nanobot v0.3.5
├── 配置：tools.mcpServers.<name>（config/schema.py:418）
├── 连接：connect_mcp_servers（agent/tools/mcp.py:1000）
├── 管理：MCPProvider（:1346），组合根创建/connect/aclose
├── 命名：mcp_<server>_<tool>（:609）
└── 生命周期：Loop 不拥有，registry 由组合根共享
```

**自检清单**

- [ ] 能说出四个规范修订号，不再写语义化版本
- [ ] 能画出 Host / Client / Server 三角
- [ ] 能说清三大原语的控制方
- [ ] 能解释 MCP 与 Function Calling 的互补关系
- [ ] 能说出 `tools.mcpServers`、`mcp_<server>_<tool>`、`MCPProvider`
- [ ] 能解释为什么 MCP 生命周期不放在 `AgentLoop`

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 配置写在 `nanobot.yml` | JSON `~/.nanobot/config.json` | `nanobot/config/loader.py:39` | 不要再提 YAML |
| `mcp_servers:` 是数组，每项带 `name` | `tools.mcpServers` 是对象，key 就是 Server 名 | `nanobot/config/schema.py:418` | 按对象写配置 |
| 语义化版本号（v1.x） | 日期修订号 `2024-11-05` 等 | 规范原文；`nanobot/pyproject.toml:44` | 只写日期号 |
| `MCPClient.connect_stdio/connect_http` | 没有 `MCPClient` 类；用 SDK `ClientSession` | `nanobot/agent/tools/mcp.py:1137` | 讲 `ClientSession` + `MCPProvider` |
| `register_mcp_servers` | 函数名是 `connect_mcp_servers` | `nanobot/agent/tools/mcp.py:1000` | 用真实函数名 |
| 工具名 `mcp_filesystem_read_file` | `mcp_<server>_<tool>`，还会清洗/截断 | `nanobot/agent/tools/mcp.py:609`、`:195` | 举例用真实规则 |

---

## 附录 B：进阶理解

**1. 为什么连接生命周期要放在组合根，而不是 AgentLoop？**

`AgentLoop` 的职责是"一条消息怎么变成一次回复"，它不应该知道 MCP 进程怎么启动、怎么重连。把 `MCPProvider` 放在组合根后，CLI、SDK、`serve`、网关可以各自决定何时连接、何时关闭；`AgentLoop` 只依赖一个 `ToolRegistry` 接口。这是典型的"依赖注入 + 生命周期外置"。

**2. 为什么工具名要清洗并限制 64 字符？**

模型 API 对函数名有字符集和长度限制，而 MCP Server 的作者不受这个限制。nanobot 在注册前统一替换非法字符，超长时保留前缀加哈希，既满足 API 约束，又尽量避免不同工具清洗后重名。

**3. 为什么 `enabledTools` 一旦收窄就连 resources/prompts 一起关掉？**

因为白名单的语义是"只允许这些工具"。resources 和 prompts 没有对应的名字过滤字段，如果继续注册，就等于绕过了操作者的收紧意图。安全设计里，**默认拒绝比默认允许更难写错**。

**4. 为什么 HTTP 连接前要先做 TCP 探测？**

`mcp` SDK 的 SSE/Streamable HTTP 客户端内部用 anyio task group。如果端口根本没开，进入上下文后在清理阶段可能抛出 `RuntimeError` 或 `ExceptionGroup`，把调用方的事件循环带崩。先探测一下（3 秒超时）能把"连不上"变成一条 warning，而不是一次崩溃。

**5. 为什么工具结果里的图片要落盘而不是直接回传 base64？**

base64 会迅速占满上下文，而且很多模型对超长 base64 的处理并不可靠。把图片存成 artifact、只回路径和元数据，模型可以用 `message` 工具把文件发给用户。这是"上下文预算"意识在 MCP 层的体现。

---

## 附录 C：Python 语法加油站

### C.1 `async` / `await`：等网络时不堵住别人

```python
import asyncio

async def call_tool(name: str) -> str:
    await asyncio.sleep(0.1)      # 模拟网络等待
    return f"{name} done"

async def main() -> None:
    print(await call_tool("weather"))

asyncio.run(main())
```

- **是什么**：`async def` 定义协程，`await` 表示"这里要等，但事件循环可以去处理别的连接"。
- **在哪用**：`connect_mcp_servers`、`ClientSession.call_tool`、`MCPToolWrapper.execute` 全是异步。
- **怎么读**：看到 `await session.list_tools()`，理解成"发一次请求，等 Server 返回工具列表"。

### C.2 `AsyncExitStack`：把一堆"要关的东西"排好队

```python
from contextlib import AsyncExitStack

async def open_two() -> None:
    async with AsyncExitStack() as stack:
        a = await stack.enter_async_context(open_a())
        b = await stack.enter_async_context(open_b())
        # 退出 with 时按 b → a 的逆序关闭
```

- **是什么**：异步版"退出栈"，可以动态压入多个上下文，退出时逆序清理。
- **在哪用**：`nanobot/agent/tools/mcp.py:1268`，一个 Server 的传输、会话都压在同一个栈里。
- **怎么读**：看到 `enter_async_context`，就想"这里注册了一个退出时要执行的清理动作"。

### C.3 Pydantic 的 `Literal` 与 camelCase 别名

```python
from typing import Literal
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

class MCPServer(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)
    type: Literal["stdio", "sse", "streamableHttp"] | None = None
    tool_timeout: int = 30

print(MCPServer(toolTimeout=60).tool_timeout)  # 60
```

- **是什么**：`Literal` 限定取值只能是列出的字符串；`alias_generator` 让 JSON 里的 `toolTimeout` 映射到 Python 的 `tool_timeout`。
- **在哪用**：`nanobot/config/schema.py:365` 的 `type` 字段；别名规则来自 `nanobot/config_base.py:15`。
- **怎么读**：配置里的 key 和 Python 字段名不一致时，先找 `alias_generator` / `AliasChoices`，不要以为读错了文件。

---

## 附录 D：原文勘误

> 说明：附录 A 记"版本演进导致的差异"，附录 D 记"旧文档本身在任何版本都不成立"的错误。

**D.1（5.8 节 / 5.11 节）"配置写在 `nanobot.yml`，`mcp_servers` 是数组"**

- **错在哪**：v0.3.5 的配置是 JSON，路径 `~/.nanobot/config.json`；MCP 配置位于 `tools.mcpServers`，是对象而不是数组。YAML 示例即使改成 JSON，字段层级也不对。
- **正确说法**：`{"tools": {"mcpServers": {"<name>": {...}}}}`，Server 名就是 key。
- **证据**：`nanobot/config/schema.py:418`；`nanobot/config/loader.py:39`。

**D.2（5.9 节）"2025-05 发布 v1.1，2026-03 发布 v1.2"**

- **错在哪**：MCP 规范没有语义化版本号，只有日期修订号；这两个版本不存在。
- **正确说法**：公开修订为 `2024-11-05`、`2025-03-26`、`2025-06-18`、`2025-11-25`。
- **证据**：MCP 官方 changelog；`mcp` SDK 的 `LATEST_PROTOCOL_VERSION` 实测为 `2025-11-25`。

**D.3（5.8 节）"`MCPClient.connect_stdio` / `connect_http` / `register_mcp_servers`"**

- **错在哪**：这些类名/函数名在 v0.3.5 不存在，是旧版叙述的虚构抽象。
- **正确说法**：SDK 提供 `stdio_client` / `sse_client` / `streamable_http_client` 与 `ClientSession`；nanobot 的入口是 `connect_mcp_servers`，连接管理是 `MCPProvider`。
- **证据**：`nanobot/agent/tools/mcp.py:1000`、`:1012-1015`、`:1346`。

**D.4（5.7 节）"MCP 工具示例如 `mcp_fs_read`、`save_memory`"**

- **错在哪**：`save_memory` 在 v0.3.5 根本不存在；把不存在的工具混进 MCP 示例会误导读者。
- **正确说法**：MCP 工具统一走 `mcp_` 前缀；内置工具清单里没有 `save_memory`。
- **证据**：`nanobot/agent/tools/mcp.py:609`；全仓库搜 `save_memory` 无工具定义。

**D.5（5.8 节）"启动时一次性注册，运行期不再变化"**

- **错在哪**：旧文档只讲了启动注册，没有提到运行期的重连与热重载；读者容易以为改配置必须重启。
- **正确说法**：`MCPProvider` 持有连接，支持会话终止后重连，并提供 `reload()` 在不重启进程的情况下对账配置。
- **证据**：`nanobot/agent/tools/mcp.py:1476`（`reload`）、`:1595` 起的重连处理；组合根创建与关闭见 `nanobot/nanobot.py:134-142`、`:352-353`。

**D.6（5.8 节）"`_normalize_schema_for_openai` 会删掉 `additionalProperties` 和 `$schema`"**

- **错在哪**：v0.3.5 的实现不是"删字段"，而是重写本地 `$ref`、归一化可空类型、补齐 object 的 `properties`/`required`。
- **正确说法**：函数名保留了历史叫法，实际做的是 schema 兼容性归一化。
- **证据**：`nanobot/agent/tools/mcp.py:496-501`、`:445-493`。

---

## 下一章

协议看懂了，下一步把环境跑起来：安装、配置、`status` 自检，再发出第一条消息。

➡️ [06 - 安装与上手](../06-install-and-hands-on/README.md)

想直接读代码，建议从 `nanobot/agent/tools/mcp.py` 的 `connect_mcp_servers`（第 1000 行）开始，对照 5.8 节的六步流程看一遍。

---

> 📝 **本章小结**：MCP 用 JSON-RPC 2.0 和 Host-Client-Server 三角，把工具供给标准化；规范版本只认日期号。落到 nanobot v0.3.5：配置在 `tools.mcpServers`，连接由 `connect_mcp_servers` 完成，工具名是 `mcp_<server>_<tool>`，生命周期由组合根的 `MCPProvider` 管理。记住这四处，比背十页协议细节更有用。
