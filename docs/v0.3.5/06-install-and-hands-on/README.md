# 06 - 安装与上手

> 🎯 **本章目标**：在 30 分钟内装好 nanobot v0.3.5，把配置放到正确的位置，跑通第一条消息，并说清 `config.json` 里每个默认值的含义。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

---

## 目录

- [6.1 为什么需要动手实践](#61-为什么需要动手实践)
- [6.2 环境准备](#62-环境准备)
- [6.3 安装 Nanobot](#63-安装-nanobot)
- [6.4 配置向导 nanobot onboard](#64-配置向导-nanobot-onboard)
- [6.5 config.json 配置详解](#65-configjson-配置详解)
- [6.6 第一次运行：交互模式](#66-第一次运行交互模式)
- [6.7 自定义 AGENTS.md：定义 Agent 身份](#67-自定义-agentsmd定义-agent-身份)
- [6.8 引导文件体系](#68-引导文件体系)
- [6.9 常见问题排错](#69-常见问题排错)
- [6.10 实战练习](#610-实战练习)
- [6.11 面试话术](#611-面试话术)
- [6.12 本章小结](#612-本章小结)

---

## 6.1 为什么需要动手实践

学 Agent 框架，**纸上得来终觉浅**。面试官最想确认三件事：你真的装过吗？能描述配置细节吗？踩过坑吗？

> 💡 一个能说清"`status` 只读配置、不会创建 `~/.nanobot`，所以第一次看到两个 ✗ 是正常的"的候选人，比只会背概念的人可信得多。

本章的路径、默认值和命令输出，都按 v0.3.5 源码与本机实测校正过。**最容易犯的错是把配置写进当前项目目录**——v0.3.5 根本不看 cwd 下的 `config.json`。

---

## 6.2 环境准备

### 6.2.1 系统与 Python

| 项目 | 要求 | 说明 |
|---|---|---|
| Python | **≥ 3.11** | `requires-python = ">=3.11"`（`nanobot/pyproject.toml:6`） |
| 操作系统 | macOS / Linux / Windows | Windows 建议 PowerShell 7 |
| 内存 | 4 GB 起 | 本地模型另算 |
| 磁盘 | 500 MB 起 | 不含模型权重 |
| 网络 | 能访问所选 Provider 的 API | 国内可选 DeepSeek 等 |

旧文档写"Python 3.10+"，在 v0.3.5 会直接安装失败。

### 6.2.2 准备 Python 与 uv

```bash
python --version      # Windows
python3 --version     # macOS / Linux
```

推荐用 `uv`（Rust 写的 Python 包管理器，装包快、自带隔离环境）：

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows PowerShell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

没有 uv 也可以继续，用 `python -m pip` 即可。

### 6.2.3 准备 API Key

| Provider | 适合谁 | 配置字段 |
|---|---|---|
| DeepSeek | 国内网络、性价比 | `providers.deepseek.apiKey` |
| Anthropic | 用 Claude | `providers.anthropic.apiKey` |
| OpenAI | 用 GPT | `providers.openai.apiKey` |
| OpenRouter | 一个 Key 调多家模型 | `providers.openrouter.apiKey` |
| Ollama | 本地免费、离线 | `providers.ollama.apiBase` |

把 Key 放进环境变量，再用 `${VAR}` 引用，避免明文写进配置：

```bash
export DEEPSEEK_API_KEY="sk-xxxxxxxx"          # macOS/Linux
$env:DEEPSEEK_API_KEY = "sk-xxxxxxxx"          # Windows PowerShell
```

nanobot 在内存中解析 `${VAR}`，不会把解析结果写回文件（`nanobot/config/loader.py:193`）。

---

## 6.3 安装 Nanobot

### 6.3.1 两种推荐装法

```bash
# 方式一：pip（建议 python -m pip，避免指向另一个解释器）
python -m pip install nanobot-ai

# 方式二：uv tool（隔离环境，不污染系统 Python）
uv tool install nanobot-ai
```

升级：`python -m pip install -U nanobot-ai` 或 `uv tool upgrade nanobot-ai`。从源码安装则是 `git clone` 后 `python -m pip install -e .`；源码跑 TUI 还需要 Bun，普通学习不必折腾。

### 6.3.2 验证安装

本机实测（v0.3.5）：

```console
$ python -m nanobot --version
🐈 nanobot v0.3.5
```

`nanobot` 不在 PATH 时，用三种兜底方式之一：

```bash
python -m nanobot --version                  # 找到装包的那个 Python
uv tool run --from nanobot-ai nanobot --version
pipx run --spec nanobot-ai nanobot --version
```

> ⚠️ 本机遇到过一个真实故障：虚拟环境目录被移动后，`nanobot.exe` 报 `Failed to canonicalize script path`。这不是 nanobot 的 bug，而是启动器里记录的绝对路径失效了；重建虚拟环境（或 `uv tool install --force`）即可。同一环境下 `python -m nanobot` 仍然可用。

---

## 6.4 配置向导 nanobot onboard

### 6.4.1 先记住配置在哪

```
~/.nanobot/config.json
```

- Linux/macOS：`/home/<you>/.nanobot/config.json`
- Windows：`C:\Users\<you>\.nanobot\config.json`

`get_config_path()` 返回 `Path.home() / ".nanobot" / "config.json"`（`nanobot/config/loader.py:39`）。**当前工作目录不参与查找**；想用别的文件，必须显式 `--config <path>`。

### 6.4.2 三种初始化方式

**A. 最快：`nanobot webui`（v0.3.5 推荐首次使用）**

```bash
nanobot webui
```

准备配置和工作区、启动本地网关，并打开 `http://127.0.0.1:8765`。在浏览器 **Settings → Models** 里选 Provider、填 Key、选模型即可。

**B. 纯 CLI：`nanobot onboard`**

```bash
nanobot onboard
```

不带参数时：配置不存在就写一份默认文件；已存在则询问"覆盖"还是"保留现有值并补全新字段"（`nanobot/cli/commands.py:173-198`）；随后创建 workspace、复制模板，并提示下一步运行 `nanobot webui`。

**C. 交互向导：`nanobot onboard --wizard`**

```bash
nanobot onboard --wizard
```

向导分三步：Provider setup → WebSocket channel → Review（`nanobot/cli/onboard.py:93`）。另外 `--refresh` 不提问、保留已有值，只补新字段（`nanobot/cli/commands.py:152`）。

### 6.4.3 生成的文件

```
~/.nanobot/
├── config.json
├── workspace/
│   ├── AGENTS.md  SOUL.md  USER.md  HEARTBEAT.md
│   ├── memory/MEMORY.md  memory/history.jsonl
│   └── prompts/README.md
└── sessions/            # 会话数据，强制放在 workspace 之外
```

这些模板由 `sync_workspace_templates` 复制（`nanobot/utils/helpers.py:897`），只创建缺失文件，不覆盖你改过的内容。

---

## 6.5 config.json 配置详解

### 6.5.1 一份能跑的配置

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.nanobot/workspace",
      "model": "deepseek/deepseek-chat",
      "provider": "auto",
      "maxTokens": 8192,
      "contextWindowTokens": 200000,
      "temperature": 0.1,
      "maxToolIterations": 200
    }
  },
  "providers": {
    "deepseek": { "apiKey": "${DEEPSEEK_API_KEY}" }
  },
  "channels": {
    "telegram": { "enabled": true, "token": "123456:ABC-DEF..." },
    "feishu": { "enabled": true, "appId": "cli_xxx", "appSecret": "${FEISHU_APP_SECRET}" },
    "dingtalk": { "enabled": true, "clientId": "dingxxx", "clientSecret": "${DINGTALK_SECRET}" }
  },
  "tools": {
    "web": { "search": { "provider": "duckduckgo" } },
    "exec": { "enable": true, "timeout": 60 },
    "restrictToWorkspace": false
  }
}
```

> 这是**示例**，不是要你覆盖整份配置。实际使用时只改需要的片段；`onboard` 已经准备好了完整文件。

### 6.5.2 agents.defaults 默认值

| 字段（JSON） | 默认值 | 含义 | 证据 |
|---|---|---|---|
| `workspace` | `~/.nanobot/workspace` | Agent 工作目录 | `config/schema.py:119` |
| `model` | `anthropic/claude-opus-4-5` | 默认模型标识 | `:121` |
| `provider` | `auto` | 按模型名前缀/关键词推断 | `:122` |
| `maxTokens` | `8192` | 单次回复上限 | `:125` |
| `contextWindowTokens` | `200000` | 上下文预算 | `:126` |
| `temperature` | `0.1` | 生成随机性 | `:127` |
| `maxToolIterations` | `200` | 单回合工具循环上限 | `:129` |
| `maxConcurrentSubagents` | `4` | 并发子 Agent 上限 | `:130` |
| `maxToolResultChars` | `16000` | 单个工具结果截断长度 | `:131` |

**模型写法**：v0.3.5 默认值带 Provider 前缀（`anthropic/claude-opus-4-5`）。两种写法都合法：

```json
{ "model": "deepseek/deepseek-chat", "provider": "auto" }
{ "model": "deepseek-chat", "provider": "deepseek" }
```

`provider: "auto"` 时模型名前缀优先匹配（`nanobot/config/schema.py:538`）；写了 `provider` 就强制用指定 Provider（`:518`）。

**`contextWindowTokens` 不是越大越好**：上下文治理用它算 token 预算（`nanobot/agent/context_governance.py:705`）。设得比模型真实窗口大，请求会超限；设得太小，历史会被更早裁剪。

### 6.5.3 providers

```json
{
  "providers": {
    "deepseek": { "apiKey": "${DEEPSEEK_API_KEY}" },
    "ollama": { "apiBase": "http://localhost:11434/v1" },
    "my-proxy": { "apiKey": "sk-xxx", "apiBase": "https://proxy.example.com/v1" }
  }
}
```

`apiKey`/`apiBase` 是 camelCase，snake_case 也接受。自定义 Provider 直接写新 key，再用 `provider: "my-proxy"` 或 `model: "my-proxy/模型名"`。OAuth Provider 用 `nanobot provider login <name>`，token 存在独立存储，不写进 `config.json`。

### 6.5.4 channels

关键不是字段多，而是**每个渠道都要 `"enabled": true`**。绝大多数渠道默认 `enabled=false`（`nanobot/channels/plugin.py:38`），不显式打开就不会启动。

| 渠道 | 必填字段 | 获取位置 |
|---|---|---|
| Telegram | `token` | @BotFather |
| Discord | `token` | Discord Developer Portal |
| 飞书 | `appId` + `appSecret` | 飞书开放平台 |
| 钉钉 | `clientId` + `clientSecret` | 钉钉开放平台 |
| Slack | `botToken` + `appToken` | Slack App 配置 |

字段名来自各渠道 manifest：Telegram `token`（`nanobot/channels/telegram/manifest.py:10`）、飞书 `appId`/`appSecret`（`nanobot/channels/feishu/manifest.py:11-12`）、钉钉 `clientId`/`clientSecret`（`nanobot/channels/dingtalk/manifest.py:9-10`）。渠道要生效必须跑常驻网关 `nanobot gateway`；`nanobot agent` 只开本地终端。

### 6.5.5 tools 与查找顺序

| 字段 | 默认值 | 说明 |
|---|---|---|
| `tools.restrictToWorkspace` | `false` | 为 true 时限制文件工具在工作区内（`config/schema.py:401`） |
| `tools.exec.enable` | `true` | 是否允许 shell（`agent/tools/shell.py:96`） |
| `tools.exec.timeout` | `60` | 单条命令硬超时秒数（`:97`） |
| `tools.web.search.provider` | `duckduckgo` | 搜索后端（`agent/tools/web.py:65`） |
| `tools.mcpServers` | `{}` | MCP Server 列表（`config/schema.py:418`） |

> `restrictToWorkspace` 默认是 **false**。旧文档说"默认会限制"，正好说反了。

配置只有两条来源：

```
1. 命令行 --config / -c <path>     （最高）
2. ~/.nanobot/config.json          （默认）
```

**没有**"当前目录 config.json"，也没有 `~/.config/nanobot/`。`nanobot status` 会打印实际读取的路径（`nanobot/cli/commands.py:677`）。

---

## 6.6 第一次运行：交互模式

### 6.6.1 启动方式

```bash
nanobot                       # 等价于 nanobot agent，进入终端 UI
nanobot agent --classic       # 经典 Python 提示符，兼容性最好
nanobot agent -m "你好"        # 发一条就退出，适合脚本/排错
```

v0.3.5 裸 `nanobot` 默认尝试 native TUI；平台不支持时用 `--classic` 回退。`-m` 单次模式最适合验证 API Key。

### 6.6.2 退出与观察

退出：`/quit`、`/exit`、`exit`、`quit`、`:q` 或 Ctrl+C（`nanobot/cli/terminal.py:51`）；`/help` 查看斜杠命令（`nanobot/command/builtin.py:1098`）。

| 位置 | 内容 |
|---|---|
| `~/.nanobot/config.json` | 配置 |
| `~/.nanobot/workspace/AGENTS.md` | 项目/工作区指令 |
| `~/.nanobot/workspace/memory/MEMORY.md` | 长期记忆 |
| `~/.nanobot/workspace/memory/history.jsonl` | 追加式历史摘要 |
| `~/.nanobot/sessions/<workspace-id>/*.jsonl` | 会话记录，**不在 workspace 内** |

会话目录不是随手定的：`JsonlSessionStore` 把根目录设为配置目录下的 `sessions/`，并显式拒绝落在 workspace 里（`nanobot/session/manager.py:557-563`），防止 Agent 误删或泄露自己的历史。

### 6.6.3 命令速查

| 命令 | 用途 |
|---|---|
| `nanobot --version` | 版本（`🐈 nanobot v0.3.5`） |
| `nanobot agent` / `nanobot` | 终端对话；`-m` 单次 |
| `nanobot gateway` | 常驻网关：渠道 + WebUI + cron |
| `nanobot webui` | 启动/加入网关并打开浏览器 |
| `nanobot serve` | OpenAI 兼容 API，默认 `127.0.0.1:8900` |
| `nanobot onboard` | 初始化/刷新配置（`--wizard` 交互） |
| `nanobot status` | 只读自检，不调模型 |
| `nanobot channels status\|login` | 渠道状态/登录 |
| `nanobot plugins list\|enable\|disable` | 可选功能/渠道依赖 |
| `nanobot provider login\|logout` | OAuth Provider |
| `nanobot sessions restore-workspace` | 降级前把会话拷回 workspace |
| `nanobot trigger <id> "msg"` | 向触发点投递消息 |

---

## 6.7 自定义 AGENTS.md：定义 Agent 身份

`AGENTS.md` 是这个工作区的"常驻指令"：项目约定、常用命令、代码风格、不要做的事。它会被拼进 system prompt，让 Agent 不用每次听你重复。v0.3.5 在构建上下文时读取三个引导文件（`nanobot/agent/context.py:92`），其中 `AGENTS.md` 从当前项目根读，`SOUL.md`/`USER.md` 从 Agent 全局 workspace 读（`:198-202`）。

### 6.7.1 一个可直接用的例子

```markdown
# 我的编程助手

你是一个 Python 后端助手，服务于本项目。

## 工作约定
- 改代码前先用 `read_file` 看现状，不要凭记忆改。
- 新增依赖必须说明理由，并同步更新 `pyproject.toml`。
- 跑测试用 `pytest -q`；失败时先贴最小复现。

## 禁止
- 不要执行 `rm -rf`、`git push --force`。
- 不要提交任何 `.env` 或密钥。
```

写到 `~/.nanobot/workspace/AGENTS.md`（或对应项目的 `AGENTS.md`），下次启动自动生效。

### 6.7.2 为什么改了模板却没生效

nanobot 会把你没改过的默认模板当作"没配置"跳过：`_SKIPPABLE_DEFAULTS = {"AGENTS.md", "USER.md"}`（`nanobot/agent/context.py:93`），内容与内置模板完全一致就不注入（`:215-218`）。所以默认内容不浪费 token；一旦你加了字，它就生效。

---

## 6.8 引导文件体系

| 文件 | 位置 | 作用 |
|---|---|---|
| `AGENTS.md` | 项目根 / workspace | 项目级指令、工作约定 |
| `SOUL.md` | Agent workspace | 人格、语气、价值观 |
| `USER.md` | Agent workspace | 用户画像、偏好 |
| `HEARTBEAT.md` | workspace | 心跳任务清单，由 cron 系统任务读取 |
| `memory/MEMORY.md` | workspace/memory | 长期事实，始终注入 |
| `prompts/README.md` | workspace/prompts | 提示模板说明，不直接注入 |

### 6.8.1 拼接顺序不是"优先级"

旧文档画过"SOUL > AGENTS > USER > TOOLS"的优先级图。v0.3.5 没有覆盖关系，而是按固定顺序拼接（`nanobot/agent/context.py:111-152`）：

```
identity → AGENTS/SOUL/USER → tool contract → 当前项目路径
→ MEMORY.md → Active Skills + Skills 摘要 → 会话摘要
```

同名内容不会互相覆盖；后出现的段落只是排在后面。想让某条规则更"强势"，靠写得明确，而不是靠文件名排序。

### 6.8.2 SOUL.md、USER.md、HEARTBEAT.md

```markdown
# SOUL.md
我是一个简洁、诚实的个人助理。
- 先给结论，再给理由。
- 不确定就说不确定，不编造。
```

```markdown
# USER.md
- 称呼：小明
- 时区：Asia/Shanghai
- 偏好：中文、代码示例、先结论后细节
```

心跳不是单独的 `HeartbeatService` 类，而是 `gateway.heartbeat` 配置 + 一个受保护的 cron 系统任务，任务内容来自 `HEARTBEAT.md`。默认 `enabled=true`、`interval_s=1800`（`nanobot/config/schema.py:330`）。把"每次都要检查、没事别打扰"的巡检项写进 `HEARTBEAT.md`；一次性提醒用 `cron` 工具。

### 6.8.3 没有 TOOLS.md

v0.3.5 的 workspace 模板只有 `AGENTS.md`、`SOUL.md`、`USER.md`、`HEARTBEAT.md`、`memory/MEMORY.md`、`prompts/README.md`（`nanobot/utils/helpers.py:918-923`）。旧文档里的 `TOOLS.md` 不存在。

---

## 6.9 常见问题排错

**1. `nanobot: command not found`**

```bash
python -m nanobot --version
uv tool run --from nanobot-ai nanobot --version
```

确认装包的解释器与 PATH 里的 `nanobot` 是同一个；`pip show nanobot-ai` 可看安装位置。

**2. Python 版本过低**

报错通常含 `requires-python >=3.11`。升级 Python，或在 uv 里指定：`uv venv --python 3.12`。

**3. `Failed to canonicalize script path`**

本机实测：虚拟环境被移动/复制后，`nanobot.exe` 里的绝对路径失效。重建环境（`uv tool install --force` 或重新 `python -m venv`），临时绕过用 `python -m nanobot`。

**4. 配置文件带 BOM**

本机实测报错：`JSON syntax error at line 1, column 1: Unexpected UTF-8 BOM`。用编辑器把编码切到 **UTF-8**（不是 UTF-8 with BOM），或：

```python
from pathlib import Path
p = Path.home() / ".nanobot" / "config.json"
p.write_text(p.read_text(encoding="utf-8-sig"), encoding="utf-8")
```

**5. API Key 或模型不匹配**

`status` 说没有 Key 时，检查 `providers.deepseek.apiKey`、当前 shell 的环境变量、`model` 前缀是否指向了另一个 Provider。`provider: "deepseek"` 配 `model: "gpt-4o"` 会找不到模型；改成 `deepseek/deepseek-chat` + `provider: "auto"`，或 `deepseek-chat` + `provider: "deepseek"`。

**6. 配置改了没生效**

先看 `nanobot status` 打印的 `Config:` 路径。放在项目目录的 `config.json` 不会被读取；移动到 `~/.nanobot/`，或每次加 `--config`。

**7. 渠道不启动**

检查 `"enabled": true`、必填字段（Telegram `token`，飞书 `appId`+`appSecret`）、依赖（`nanobot plugins enable <name>`），并确认跑的是 `nanobot gateway`。

**8. MCP Server 连不上**

先在 `nanobot agent` 日志里找 Server 名；stdio Server 的 stdout 只能走 JSON-RPC，不能打日志。完整排错见第 05 章 5.8 节。

---

## 6.10 实战练习

### 练习 1：把自检跑一遍

```bash
python -m nanobot --version        # 应输出 🐈 nanobot v0.3.5
nanobot status                     # 看 Config / Workspace / Agent 三行
```

`status` 是只读的：不创建 `~/.nanobot`，也不调用模型。第一次看到 `Config: ... ✗` 很正常，说明还没跑 `onboard`。

### 练习 2：做一个翻译助手

1. 打开 `~/.nanobot/workspace/AGENTS.md`，写入：

```markdown
# 翻译助手
- 中文输入翻译成英文，英文输入翻译成中文。
- 保留原文语气，专业术语后面加括号注释。
- 只输出译文，不要解释。
```

2. 启动 `nanobot agent --classic`；
3. 输入"今天天气不错，适合跑步"，看它是否只输出英文译文。

### 练习 3：观察记忆与会话

1. 和 Agent 聊几轮，告诉它你的名字和偏好；
2. 退出后看：

```bash
ls ~/.nanobot/workspace/memory/
# MEMORY.md  history.jsonl
ls ~/.nanobot/sessions/
# <workspace-id>/  里面是 *.jsonl
```

3. 重新启动，问"我叫什么"，验证跨会话记忆。

长期事实在 `MEMORY.md`，追加流水在 `history.jsonl`；**没有** `HISTORY.md`（它只是旧版本的迁移来源）。

---

## 6.11 面试话术

### 话术 1：描述你如何搭建 nanobot

> "我用 `uv tool install nanobot-ai` 装，要求 Python ≥3.11。然后 `nanobot onboard --wizard` 选 Provider、填 Key、选模型，配置写到 `~/.nanobot/config.json`。我用 `nanobot status` 自检——它只读配置、不调模型，能快速区分'配置没就绪'和'网络/Key 有问题'。最后 `nanobot agent -m "Hello"` 验证链路。"

### 话术 2：配置设计有什么特点

> "JSON 管机器参数（Provider、模型、渠道、工具），Markdown 管行为（AGENTS.md 项目指令、SOUL.md 人格、USER.md 用户画像）。JSON 由 WebUI 写入、Pydantic 校验，落盘 camelCase，同时接受 snake_case；Markdown 由人维护，可用 Git 追踪。配置固定放 `~/.nanobot`，不跟项目走，所以换项目不用重配 Provider。"

### 话术 3：你踩过什么坑

> "两个。一是把 `config.json` 放在项目目录，结果 `nanobot status` 一直读 `~/.nanobot/config.json`——查找顺序只有 `--config` 和 home，cwd 不参与。二是编辑器把配置存成带 BOM 的 UTF-8，启动直接报 `Unexpected UTF-8 BOM`；改成无 BOM 就好了。"

---

## 6.12 本章小结

| 知识点 | 要点 |
|---|---|
| 安装 | `pip install nanobot-ai` 或 `uv tool install nanobot-ai`，Python ≥3.11 |
| 配置路径 | `~/.nanobot/config.json`；cwd 不参与，`--config` 可覆盖 |
| 初始化 | `nanobot webui`（推荐）/ `nanobot onboard` / `--wizard` / `--refresh` |
| 默认值 | 模型 `anthropic/claude-opus-4-5`、200 轮、200k 上下文、temperature 0.1 |
| 渠道 | `token` / `appId`+`appSecret` / `clientId`+`clientSecret`，且 `enabled: true` |
| 引导文件 | AGENTS / SOUL / USER / HEARTBEAT / MEMORY，没有 TOOLS.md |
| 运行 | `nanobot`、`nanobot agent --classic`、`nanobot agent -m` |
| 自检 | `nanobot status` 只读，不调模型 |

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| Python 3.10+ | 要求 **≥3.11** | `nanobot/pyproject.toml:6` | 面试说 3.11 |
| 配置在项目目录的 `config.json` | `~/.nanobot/config.json`，cwd 不参与 | `config/loader.py:39` | 强调 home 路径 |
| `workspace: "."` | 默认 `~/.nanobot/workspace` | `config/schema.py:119` | 别再说当前目录 |
| `max_tool_iterations=40` | `maxToolIterations=200` | `config/schema.py:129` | 改成 200 |
| `max_tokens=16384`、`context=128000`、`temperature=0.7` | `8192` / `200000` / `0.1` | `:125-127` | 按默认值表 |
| `bot_token` / `app_id` | `token` / `appId`，且要 `enabled: true` | 各渠道 manifest | 用 camelCase |
| `tools.web_search` | `tools.web.search` | `config/schema.py:392` | 改层级 |
| `AGENTS/SOUL/USER/TOOLS.md` 四层 | 没有 `TOOLS.md`，多一个 `HEARTBEAT.md` | `utils/helpers.py:918-923` | 按真实模板讲 |
| 会话在 workspace 的 `sessions/*.jsonl` | 在配置目录的 `sessions/<workspace-id>/` | `session/manager.py:557-563` | 讲"强制外置" |

---

## 附录 B：进阶理解

**1. 为什么配置放在 home，而不是项目目录？**

nanobot 是"个人助理"而不是"项目依赖"：同一个 Provider、同一套渠道、同一份记忆要服务多个项目。配置放 `~/.nanobot` 后，换项目只换 workspace，不用重配 Key。需要多实例时，用 `--config` 指向另一份文件即可。

**2. 为什么用 JSON 而不是 YAML？**

配置主要由 WebUI/CLI 写入，JSON 序列化没有歧义，Pydantic 也能直接校验类型。YAML 对人更友好，但缩进、锚点、隐式类型转换会带来"看起来对、解析出来不对"的问题。代价是 JSON 不能写注释，所以"解释"被移到了 Markdown 和文档里。

**3. 为什么落盘用 camelCase，却接受 snake_case？**

前端（TS/JS）习惯 camelCase，Python 字段习惯 snake_case。`config_base.py` 用 `alias_generator=to_camel` + `populate_by_name=True` 同时接受两种写法（`nanobot/config_base.py:15`），保存时统一写 camelCase（`config/loader.py:157`）。两边都不用迁就对方。

**4. 为什么会话数据强制放在 workspace 之外？**

Agent 会大量读写 workspace。如果会话历史也在里面，一次误删、一次 `git clean`、一次打包就可能把历史带走；更糟的是，Agent 可能读到自己的原始会话记录。`JsonlSessionStore` 因此在初始化时就拒绝 root 落在 workspace 内（`nanobot/session/manager.py:559-563`）。

**5. 为什么引导文件用 Markdown 而不是数据库？**

个人 Agent 的"人格/偏好/项目约定"量很小，但需要人随时改、可 diff、可回滚。Markdown 文件满足这三点，还能被 Git 追踪；数据库反而增加迁移和备份成本。

---

## 附录 C：Python 语法加油站

### C.1 Pydantic 的 `Base` 与 camelCase 别名

```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel

class Base(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)

class Defaults(Base):
    max_tokens: int = 8192

print(Defaults(maxTokens=4096).max_tokens)   # 4096
print(Defaults(max_tokens=8192).max_tokens)  # 8192
```

- **是什么**：`alias_generator` 自动生成 camelCase 别名；`populate_by_name=True` 让 Python 字段名也能直接传。
- **在哪用**：`nanobot/config_base.py:15`；所有配置类都继承它。
- **怎么读**：看到 JSON 里的 `maxTokens`、`toolTimeout`，就到 Python 类里找 `max_tokens`、`tool_timeout`。

### C.2 `str | None` 与 `Path.expanduser()`

```python
from pathlib import Path

def config_path(raw: str | None) -> Path:
    if raw:
        return Path(raw).expanduser().resolve()
    return Path.home() / ".nanobot" / "config.json"
```

- **是什么**：`str | None` 表示"字符串或空"；`expanduser()` 把 `~` 展开成 home。
- **在哪用**：`nanobot/config/loader.py:39` 的默认路径；`config/schema.py:493` 的 `workspace_path`。
- **怎么读**：看到 `~` 不要以为它一定被展开，找 `expanduser()`。

### C.3 上下文管理器：`with` 与 `async with`

```python
from pathlib import Path

with Path("config.json").open(encoding="utf-8") as handle:
    data = handle.read()          # 退出 with 自动关闭文件
```

```python
async with Nanobot.from_config() as bot:
    result = await bot.run("Hello")   # 退出时自动 aclose()
```

- **是什么**：`with` 保证进入/退出成对执行；`async with` 是异步版本。
- **在哪用**：`nanobot/nanobot.py:355-359` 的 `__aenter__`/`__aexit__` 让 `Nanobot` 支持 `async with`，退出时释放 MCP 连接。
- **怎么读**：看到 `async with`，先找类的 `__aenter__`/`__aexit__`，就知道它申请/释放了什么资源。

---

## 附录 D：原文勘误

> 说明：附录 A 记"版本演进导致的差异"，附录 D 记"旧文档本身在任何版本都不成立"的错误。

**D.1（6.4.3 / 6.5.6 节）"在 workspace 目录运行 onboard，会生成当前目录的 config.json"**

- **错在哪**：`get_config_path()` 固定返回 `~/.nanobot/config.json`，与 cwd 无关；`onboard` 没有"把配置写到当前目录"的分支。
- **正确说法**：配置永远写到 home；要换位置只能 `--config`。
- **证据**：`nanobot/config/loader.py:39`；`nanobot/cli/commands.py:164`。

**D.2（6.2.1 节）"Python 3.10+"**

- **错在哪**：`requires-python` 是 `>=3.11`，3.10 会在安装阶段被拒绝。
- **正确说法**：Python ≥3.11，推荐 3.12/3.13。
- **证据**：`nanobot/pyproject.toml:6`。

**D.3（6.5.2 节）默认值表**

- **错在哪**：`model=gpt-4o`、`workspace="."`、`max_tokens=16384`、`context_window_tokens=128000`、`temperature=0.7`、`max_tool_iterations=40` 全部与 v0.3.5 不符。
- **正确说法**：`anthropic/claude-opus-4-5`、`~/.nanobot/workspace`、`8192`、`200000`、`0.1`、`200`。
- **证据**：`nanobot/config/schema.py:119-131`。

**D.4（6.5.4 节）渠道字段 `bot_token` / `app_id`**

- **错在哪**：v0.3.5 的字段是 camelCase（`token`、`appId`、`appSecret`、`clientId`），而且必须 `enabled: true` 才启动。`bot_token` 这类写法不会被识别。
- **正确说法**：按各渠道 manifest 的字段名写，并显式启用；访问控制用 `allowFrom`。
- **证据**：`nanobot/channels/telegram/manifest.py:10`；`nanobot/channels/feishu/manifest.py:11-12`；`nanobot/channels/dingtalk/manifest.py:9-10`。

**D.5（6.8 节）"SOUL > AGENTS > USER > TOOLS 四层优先级"**

- **错在哪**：既没有 `TOOLS.md`，也不是覆盖式优先级；v0.3.5 是按顺序拼接，且 `HEARTBEAT.md` 也是引导文件之一。
- **正确说法**：AGENTS / SOUL / USER 三个引导文件 + HEARTBEAT + MEMORY，按 `build_system_prompt` 的顺序合并。
- **证据**：`nanobot/agent/context.py:92`、`:198-202`、`:111-152`；`nanobot/utils/helpers.py:918-923`。

**D.6（6.5.6 节）"配置查找顺序：当前目录 → ~/.config/nanobot → 环境变量"**

- **错在哪**：v0.3.5 只有两级：`--config` 和 `~/.nanobot/config.json`。没有 cwd，也没有 XDG 路径。
- **正确说法**：默认 home；命令行显式覆盖；cwd 完全不参与。
- **证据**：`nanobot/config/loader.py:36-39`；`nanobot/cli/commands.py:164`。

**D.7（6.4.1 节）"onboard 默认就是交互式向导"**

- **错在哪**：不带参数的 `onboard` 主要是创建/刷新配置；交互向导要显式 `--wizard`。v0.3.5 的推荐首次入口其实是 `nanobot webui`。
- **正确说法**：`nanobot onboard` 非交互；`--wizard` 才是三步问答。
- **证据**：`nanobot/cli/commands.py:151-152`；`nanobot/cli/onboard.py:93`。

**D.8（6.6.4 节）"会话文件是 workspace 下的 `sessions/cli:default.jsonl`"**

- **错在哪**：会话存储被强制放在 workspace 之外，并且按 workspace 分成独立命名空间；文件名也不是 `cli:default.jsonl` 这种形式。
- **正确说法**：`<配置目录>/sessions/<workspace-id>/*.jsonl`。
- **证据**：`nanobot/session/manager.py:557-563`、`:579`。

**D.9（6.11 话术 3）"`restrict_to_workspace` 默认会限制 Agent"**

- **错在哪**：默认值是 `false`，即默认不限制在工作区内；旧文档把安全选项说反了。
- **正确说法**：需要收紧时显式设 `tools.restrictToWorkspace: true`。
- **证据**：`nanobot/config/schema.py:401`。

---

## 下一章

装好、跑通之后，下一步看 Agent 怎么"记住事"：`MEMORY.md`、`history.jsonl`、Consolidator 与 Dream 两阶段。

➡️ 第 07 章「记忆系统实战」（对应源码：`nanobot/agent/memory.py`）

如果现在就想读代码，建议先打开 `nanobot/config/loader.py` 的 `load_config`（第 43 行）和 `nanobot/agent/context.py` 的 `build_system_prompt`（第 101 行），对照本章 6.5 与 6.8 节看一遍。

---

> 📝 **本章小结**：v0.3.5 的安装是 `pip install nanobot-ai` 或 `uv tool install nanobot-ai`，门槛是 Python ≥3.11；配置只认 `~/.nanobot/config.json`（或 `--config`），不认 cwd。默认模型、200 轮迭代、200k 上下文这些数字要记准；渠道字段是 `token`/`appId`/`clientId` 且必须 `enabled: true`。最后，`status` 只读、不调模型，是排错的第一步。
