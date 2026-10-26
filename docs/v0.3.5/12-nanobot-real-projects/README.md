# 12 - Nanobot 实战项目

> 🎯 **本章目标**：把前 11 章的零件装成三个能跑的系统——运维助手、客服 Bot、数据库 MCP Server，每个都用 v0.3.5 的真实配置、命令和落盘位置复现。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

---

## 目录

- [导读](#导读) · [项目一：智能运维助手](#项目一智能运维助手难度中级)
  - [1.1 需求](#11-项目需求分析) · [1.2 架构](#12-架构设计) · [1.3 实现](#13-核心实现) · [1.4 部署](#14-部署与运行) · [1.5 话术](#15-面试话术star-法) · [1.6 验收](#16-验收标准) · [1.7 坑](#17-常见坑)
- [项目二：多平台智能客服 Bot](#项目二多平台智能客服-bot难度中高级)
  - [2.1 需求](#21-项目需求分析) · [2.2 架构](#22-架构设计) · [2.3 实现](#23-核心实现) · [2.4 部署](#24-部署与运行) · [2.5 话术](#25-面试话术star-法) · [2.6 验收](#26-验收标准) · [2.7 坑](#27-常见坑)
- [项目三：数据库 MCP Server](#项目三数据库-mcp-server难度高级)
  - [3.1 需求](#31-项目需求分析) · [3.2 架构](#32-架构设计) · [3.3 实现](#33-核心实现) · [3.4 部署](#34-部署与运行) · [3.5 话术](#35-面试话术star-法) · [3.6 验收](#36-验收标准) · [3.7 坑](#37-常见坑)
- [面试高频题](#面试高频题10-道) · [练习](#练习题与思考题) · [总结](#本章总结)
- [附录 A](#附录-a与旧版文档的差异) · [B](#附录-b进阶理解) · [C](#附录-cpython-语法加油站) · [D](#附录-d原文勘误)

---

## 导读

三个项目对应三种能力：**用工具**（单机 + 定时）、**接渠道**（对外服务）、**写协议**（自己产出工具）。

| 项目 | 一句话 | 动到的 v0.3.5 部件 | 难度 |
|---|---|---|---|
| 一、运维助手 | 每天自动巡检、写报告、拦危险命令 | `exec`、`cron` 工具、`SOUL.md`、`jobs.json` | ⭐⭐⭐ |
| 二、客服 Bot | 在 Telegram 答疑，答不了转工单 | `channels.telegram`、`skills/`、workspace 文件 | ⭐⭐⭐⭐ |
| 三、数据库 MCP | 让 Agent 安全地只读查库 | MCP SDK（FastMCP）、`tools.mcpServers`、`mcp_<server>_<tool>` | ⭐⭐⭐⭐⭐ |

### 先背四个命令

| 命令 | 干什么 | 会跑 cron 吗 |
|---|---|---|
| `nanobot -m "..."` | 发一条消息就退出（脚本、冒烟） | 不会 |
| `nanobot agent` | 交互终端（裸 `nanobot` 等价） | 不会，只建/查任务 |
| `nanobot gateway` | 常驻：渠道 + cron + WebUI | **会** |
| `nanobot status` | 体检配置、workspace、provider | 不会 |

证据：根命令把 `-m` 转发给 `agent`（`nanobot/cli/entry.py:23-31`，参数在 `nanobot/cli/agent.py:54-58`）；`CronService.start()` 只在网关调用（`nanobot/cli/gateway_runtime.py:910`）。

> ⚠️ 不存在的命令：`start`、`chat`、`pip install nanobot`。对应写法是 `nanobot gateway`、`nanobot -m "..."`、`pip install nanobot-ai`（包名见 `pyproject.toml:2`）。

### 一份配置，四个抽屉

```json
{
  "agents": { "defaults": { "workspace": "~/.nanobot/workspace", "model": "anthropic/claude-opus-4-5", "provider": "anthropic" } },
  "providers": { "anthropic": { "apiKey": "${ANTHROPIC_API_KEY}" } },
  "tools": { "exec": { "enable": true }, "mcpServers": {} },
  "channels": { "telegram": { "enabled": false, "token": "", "allowFrom": [] } }
}
```

| 抽屉 | 管什么 | 哪个项目改 |
|---|---|---|
| `agents.defaults` | 模型、workspace、时区、迭代上限 | 三个都改 |
| `providers.<name>.apiKey` | 模型服务的钥匙 | 三个都改 |
| `tools` | 内置工具与 MCP server | 项目一（`exec`）、项目三（`mcpServers`） |
| `channels.<name>` | 对外渠道，必须 `"enabled": true` | 项目二（`telegram`） |

根字段见 `nanobot/config/schema.py:427-433`；默认值从 `:116` 起（`maxToolIterations=200` 在 `:129`）。

术语：**Workspace** 是 Agent 的家目录（默认 `~/.nanobot/workspace`，放 `SOUL.md`、`skills/`、`memory/`、`cron/`）；**Session** 是一段对话，键为 `{channel}:{chat_id}`（如 `telegram:123456789`、`cli:direct`）；**Tool** 是模型能调的函数；**Skill** 是写给模型看的 Markdown 说明书，不执行代码；**Channel** 是消息出入口；**Cron job** 是落盘的定时任务，到点作为原会话的一轮输入执行。

### 本章代码的验证状态

本机实际跑过（Windows 11 + Python 3.13 + v0.3.5 源码）：`cron` 工具建任务 → 先写 `action.jsonl`、启动调度器后合并进 `jobs.json`；项目二 config 通过 pydantic 校验且 `nanobot channels status` 显示 Telegram ✓；`SKILL.md` 被 `SkillsLoader` 发现；`SOUL.md` 被拼进 system prompt；`allowPatterns` 的放行/拦截；项目三的 `db_server.py` 被 `MCPProvider` 连上并成功查询、`DROP` 被拒；`nanobot status` 的正常与缺环境变量两种输出。

**未在本机验证**：真实模型对话内容（无 API key）、真实 Telegram 收发（无 bot token）、网关长时间常驻。文中逐处标注。

---

## 项目一：智能运维助手（难度：中级）

### 1.1 项目需求分析

你负责 3 台 Linux 服务器，每天要看负载、磁盘、内存，找出 CPU 大户，整理成报告。这些事不需要"智能"，但需要**每天同一时间、不漏项、有记录**。

需求四条：采集 `uptime`/`df -h`/`free -m`/`ps aux`；磁盘 > 85% 标「严重」；写 `reports/ops-YYYY-MM-DD.md`，每天 09:00（Asia/Shanghai）；只读优先，结论后附命令。

### 1.2 架构设计

```
每天 09:00 → cron 任务（会话里创建，落盘 jobs.json）
   → 作为「原会话的一轮输入」执行
   → AgentLoop/ContextBuilder（SOUL.md + MEMORY.md + 技能摘要）
   → 模型选工具：exec(uptime/df/free/ps) → write_file(reports/…)
   → 报告落盘；若会话绑定渠道，回复也发回那个聊天
```

| 设计点 | v0.3.5 做法 | 为什么 |
|---|---|---|
| 执行命令 | `exec` + `tools.exec.allowPatterns`/`denyPatterns` | 白名单"只允许"，黑名单兜底（`nanobot/agent/tools/shell.py:824-834`） |
| 定时 | 会话里让模型调 `cron` 工具 | 配置里没有 cron 数组 |
| 人设 | `<workspace>/SOUL.md` | 从 Agent 全局 workspace 读（`nanobot/agent/context.py:198-202`） |
| 报告/记忆 | `write_file` 写 `reports/`；`memory/MEMORY.md` 自动注入 | 文件最稳，不依赖渠道（`context.py:128-130`） |

旧文档给过 `"cron": [{...}]` 配置。v0.3.5 没有这个字段：`Config` 根只认 `agents`/`channels`/`transcription`/`providers`/`api`/`gateway`/`tools`/`modelPresets`（`nanobot/config/schema.py:427-433`），多写根级键会直接让加载失败（本机实测 `Unknown setting`）。定时只有一条路：会话里用工具创建，落盘 `<workspace>/cron/jobs.json`（`nanobot/cron/service.py:380-430`）。

### 1.3 核心实现

#### 1.3.1 config.json

写到 `~/.nanobot/config.json`（Windows：`C:\Users\<你>\.nanobot\config.json`）：

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.nanobot/workspace", "model": "anthropic/claude-opus-4-5",
      "provider": "anthropic", "timezone": "Asia/Shanghai", "timezoneMode": "manual",
      "maxToolIterations": 60
    }
  },
  "providers": { "anthropic": { "apiKey": "${ANTHROPIC_API_KEY}" } },
  "tools": {
    "restrictToWorkspace": false,
    "exec": {
      "enable": true,
      "timeout": 60,
      "allowPatterns": ["uptime", "df -h( .*)?", "free -m", "ps aux( .*)?", "systemctl status [a-z0-9_.@-]+"],
      "denyPatterns": ["rm -rf.*", ".*shutdown.*", ".*reboot.*", ".*mkfs.*", ".*dd if=.*"]
    }
  }
}
```

- `timezoneMode: "manual"`：不写时自动探测系统时区（`nanobot/config/schema.py:159-173`），服务器建议写死。
- `maxToolIterations: 60`：最多 60 轮工具调用（默认 200，`schema.py:129`）。
- `allowPatterns` 是**整条命令 fullmatch**，`;` 串联时每段都要命中（`nanobot/agent/tools/shell.py:824-827`）；`denyPatterns` 是正则 `search`（`:829-831`）。
- `${VAR}` 只在内存解析、不回写（`nanobot/config/loader.py:196-215`）；环境变量没导出时配置不 ready。

本机实测：`uptime`、`df -h /var`、`ps aux --sort=-%cpu`、`systemctl status nginx` 放行；`service nginx restart`、`cat /etc/passwd` 被白名单拦；`shutdown -h now`、`df -h; rm -rf /` 被 deny 拦。

#### 1.3.2 SOUL.md（人设）

写到 `~/.nanobot/workspace/SOUL.md`：

```markdown
# 小运

我是一名值班运维工程师，名字叫「小运」。

## 我的原则

- 先取证据，再下结论：每条结论后面都附上产生它的命令。
- 只读优先：能用 uptime、df -h、free -m 说清的事，绝不做写操作。
- 不确定就说不确定，并给出下一条排查命令。

## 我的巡检顺序

1. uptime（负载） 2. df -h（磁盘） 3. free -m（内存） 4. ps aux --sort=-%cpu（大户）

## 我的输出格式

1. 一句话结论（正常 / 警告 / 严重）
2. 指标表格（指标、当前值、阈值、状态）
3. 异常项 + 建议命令
```

- `SOUL.md` 从 Agent 全局 workspace 读，`AGENTS.md` 从当前项目根读（`nanobot/agent/context.py:198-202`）：人设写 `SOUL.md`，项目约定写 `AGENTS.md`。
- 内容与内置模板一字不差会被当成"没配置"跳过（`context.py:93`、`:215-218`）。加一句自己的话就生效。

#### 1.3.3 一个 cron 任务

不要在配置文件里写 cron。启动交互终端，用人话说清楚：

```
$ nanobot agent
你 > 帮我建个定时任务：名字 ops-daily，每天 09:00（Asia/Shanghai）巡检一次。
     按 SOUL.md 的顺序跑 uptime、df -h、free -m、ps aux，报告写到
     reports/ops-<当天日期>.md；磁盘超过 85% 时在开头标「严重」。
```

模型会发出这样一次工具调用（参数名来自 `cron` 的 schema，`nanobot/agent/tools/cron.py:22-52`）：

```json
{
  "action": "add",
  "name": "ops-daily",
  "message": "执行一次巡检：按 SOUL.md 跑 uptime、df -h、free -m、ps aux，写入 reports/ops-<当天日期>.md；磁盘超过 85% 时在开头标「严重」。",
  "cron_expr": "0 9 * * *",
  "tz": "Asia/Shanghai"
}
```

返回 `Created job 'ops-daily' (id: 5716c026)`。三个硬约束：

1. **任务要绑定会话**：工具从当前请求取 `channel`/`chat_id`，拿不到就报 "must be created from a chat session"（`cron.py:172-176`）；cron 任务内部也不能再建（`:148-149`）。
2. **`message` 不能空**，它是触发时真正执行的指令（`:130-134`、`:166-171`）。
3. **`tz` 只能配 `cron_expr`**（`:177-205`）。

在 `nanobot agent` 里建时上下文是 `channel="cli"`、`chat_id="direct"`（`nanobot/agent/loop.py:1107-1112`），所以 `sessionKey` 是 `cli:direct`。

#### 1.3.4 落盘长什么样

调度器没启动时，任务先写同目录的 `action.jsonl`，`jobs.json` 还不存在（本机实测）：

```
add -> Created job 'ops-daily' (id: 5716c026)
jobs.json exists: False
action.jsonl exists: True
{"action": "add", "params": {"id": "5716c026", ...}}
```

只有 `_running` 为真才直接写 `jobs.json`（`nanobot/cron/service.py:732-738`）；`nanobot gateway` 启动后 `_load_store` 会把 action 合并并落盘（`:277-314`、`:343-345`）。合并后的 `~/.nanobot/workspace/cron/jobs.json`（节选，省略时间戳与 `runHistory`）：

```json
{
  "version": 1,
  "jobs": [{
    "id": "5716c026", "name": "ops-daily", "enabled": true,
    "schedule": { "kind": "cron", "expr": "0 9 * * *", "tz": "Asia/Shanghai" },
    "payload": { "kind": "agent_turn", "message": "执行一次巡检：……", "deliver": false,
                 "sessionKey": "cli:direct", "originChannel": "cli", "originChatId": "direct" },
    "state": { "nextRunAtMs": 1793062800000 }
  }]
}
```

`originChannel` 就是"结果发回哪里"。`cli` 在网关里不是真实渠道，所以 CLI 里建的任务**报告必须落盘**，否则回复被丢弃（日志 `Unknown channel: cli`，`nanobot/channels/manager.py:845`）。想推手机就把同一个任务建在 Telegram 会话（项目二）。

### 1.4 部署与运行

```bash
uv tool install nanobot-ai        # 或 python -m pip install nanobot-ai
nanobot onboard --wizard          # 生成默认配置与 workspace
# 合并 1.3.1 的 config.json，把 1.3.2 的 SOUL.md 放到 ~/.nanobot/workspace/
export ANTHROPIC_API_KEY=sk-ant-xxxx     # PowerShell: $env:ANTHROPIC_API_KEY="..."
nanobot status
nanobot -m "按 SOUL.md 跑一次巡检，写到 reports/ops-manual.md，只执行只读命令"
nanobot agent                     # 建 1.3.3 的 cron 任务
nanobot gateway --background      # 调度器开始跑
nanobot gateway status && nanobot gateway logs
cat ~/.nanobot/workspace/cron/jobs.json
```

`status` 正常时：`Config ✓`、`Workspace ✓`、`Agent: ✓ provider/model configuration is ready`；忘了导出环境变量时是 `Agent: ✗ configuration is not ready` + `Found 1 missing environment variable reference(s).`（均本机实测）。`--background` 是"关掉终端也不停"的官方方式（仓库根 `README.md:190-201`）。

### 1.5 面试话术（STAR 法）

> **面试官**：介绍一个你做过的 Agent 项目。

**S**：3 台服务器，每天手工巡检，漏项、忘记录、磁盘快满没人发现。
**T**：做每天 9 点自动巡检、能写报告、不能执行危险命令的助手。
**A**：① 人设写 `SOUL.md`，固定顺序和输出格式；② 命令安全两层——`allowPatterns` 白名单 + `denyPatterns` 兜底；③ 定时用 `cron` 工具，落盘 `jobs.json`，网关调度；④ 报告 `write_file` 落盘，不依赖渠道。
**R**：巡检时间从 20 分钟降到 0；报告可回溯；危险命令在工具层被拦，与模型"听不听话"无关。

### 1.6 验收标准

- [ ] `nanobot status` 的 Config / Workspace / Agent 都是 ✓。
- [ ] `nanobot -m "..."` 能在 `reports/` 下生成报告。
- [ ] 让它执行 `rm -rf /tmp/x` → `Command blocked by deny pattern filter`；`cat /etc/passwd` → `... by allowlist filter`。
- [ ] 网关启动日志出现 `Cron service started with 1 jobs`。
- [ ] `jobs.json` 里有 `ops-daily`、`expr`、`tz`、`sessionKey`。

### 1.7 常见坑

| 坑 | 现象 | 原因与解法 |
|---|---|---|
| config 写 `"cron": [...]` | `Unknown setting`，加载失败 | 没有这个字段；用 `cron` 工具（`schema.py:427-433`） |
| 用 `"enabled": true` 配 exec | 看似生效，其实被忽略 | 字段名是 `enable`（`shell.py:96`） |
| 白名单写 `"df"` | `df -h` 被拦 | fullmatch，要写 `"df -h( .*)?"` |
| 建完任务就等 | 到点没动静 | `nanobot agent` 不启动调度器，要跑 `nanobot gateway` |
| 只靠聊天回复 | 日志 `Unknown channel: cli` | CLI 在网关里没有渠道；报告落盘或改建在 Telegram |
| 只有 `action.jsonl` | 找不到 `jobs.json` | 调度器没启动过，网关启动后合并 |
| 审计写进 `MEMORY.md` | 长期记忆越来越脏 | 审计写 `reports/`；记忆由 `MEMORY.md`+`history.jsonl`+Dream 管 |

---

## 项目二：多平台智能客服 Bot（难度：中高级）

### 2.1 项目需求分析

一个 SaaS 产品，客户在 Telegram 提问。FAQ 能答的按知识库答、不编造；答不了/投诉/退款/账号安全 → 转人工并留工单；只服务白名单客户；话术统一，不承诺赔偿和上线时间。

功能三块：FAQ 与升级（读 workspace 知识库、按规则建工单 JSON）、访问控制与话术（`channels.telegram.allowFrom`、`skills/customer-service/SKILL.md`）、会话记忆（每个 chat 一个 session）。

### 2.2 架构设计

```
Telegram 用户 → TelegramChannel（channels/telegram/runtime.py）
   → is_allowed(sender_id) ← allowFrom
   → MessageBus.publish_inbound（nanobot/bus/queue.py:37）
   → AgentLoop._process_message（nanobot/agent/loop.py:1594）
        会话键 telegram:<chat_id>（nanobot/bus/events.py:40）
   → ContextBuilder：SOUL/USER + MEMORY + 技能摘要（agent/context.py:101）
   → 模型：读 SKILL.md → 读 knowledge/faq.md → 必要时 write_file 工单
   → OutboundMessage → MessageBus → TelegramChannel.send
```

| 设计点 | 做法 | 为什么 |
|---|---|---|
| 接平台 | `channels.telegram` + `token`/`allowFrom`/`enabled` | 字段来自渠道配置类（`channels/telegram/runtime.py:413-419`） |
| 谁能用 | `allowFrom` 白名单 | 默认拒绝；陌生人只拿到配对码（`channels/base.py:237-304`） |
| 话术/知识库 | `SKILL.md` + `knowledge/faq.md` | 改内容不用改代码 |
| 工单 | `<workspace>/tickets/*.json` | 一个目录就是队列 |

技能是**渐进披露**的：system prompt 里只有一行 `description`，需要时模型用 `read_file` 读正文（`nanobot/agent/skills.py:204-215`）。要每轮注入就写 `always: true`（费 token）。

### 2.3 核心实现

#### 2.3.1 config.json（Telegram）

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.nanobot/workspace", "model": "anthropic/claude-opus-4-5",
      "provider": "anthropic", "timezone": "Asia/Shanghai", "timezoneMode": "manual"
    }
  },
  "providers": { "anthropic": { "apiKey": "${ANTHROPIC_API_KEY}" } },
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "${TELEGRAM_BOT_TOKEN}",
      "allowFrom": ["123456789"],
      "mode": "polling",
      "groupPolicy": "mention"
    }
  },
  "tools": { "restrictToWorkspace": true }
}
```

- `enabled` **必须显式 `true`**：配置类里默认 `False`（`channels/telegram/runtime.py:416`）。
- `token`：BotFather 给的 `123456:ABC-DEF...`。旧文档的 `bot_token` 不是字段名，写了会被忽略，启动时报 "bot token not configured"（`:605-606`）。
- `allowFrom`：**字符串数组**；写数字过不了 pydantic 校验（`list[str]`，本机实测 `Input should be a valid string`），渠道不启动。
- `mode: "polling"` 不用公网 IP（`webhook` 需要 HTTPS 地址，`:447-459`）；`restrictToWorkspace: true` 把文件工具关在工作区内。
- 本机已用 v0.3.5 schema 校验通过，`nanobot channels status` 显示 `Telegram ✓`。

#### 2.3.2 SKILL.md（话术与升级规则）

写到 `~/.nanobot/workspace/skills/customer-service/SKILL.md`（目录名必须等于 frontmatter 的 `name`）：

````markdown
---
name: customer-service
description: 产品客服话术与升级规则。回答前先读本技能，按模板回复；遇到投诉、退款、账号安全或用户要求人工时，走升级流程建工单。
---

# 客服话术与升级规则

## 第一步：查知识库

1. 用 `read_file` 读 `~/.nanobot/workspace/knowledge/faq.md`。
2. 命中 → 用「直接回答」；没命中 → 不要猜，用「转人工」并建工单。

## 话术模板

### A. 直接回答

> 您好，关于「{用户问题}」：{FAQ 原文答案}
> 还有疑问直接回复我；需要人工跟进可以说「转人工」。

### B. 转人工 / 建工单

> 您好，这个问题我请同事跟进。已为您登记工单 {工单号}，
> 我们会在 1 个工作日内通过当前聊天联系您。

工单：用 `write_file` 写到 `tickets/<YYYYMMDD-HHMMSS>-<chat_id>.json`：

```json
{"chat_id": "<chat_id>", "question": "<用户原话>", "summary": "<一句话>",
 "priority": "normal", "created_at": "<ISO 时间>", "status": "open"}
```

## 升级规则（命中任意一条必须转人工）

- 提到「投诉」「12315」「曝光」「律师」「退款」；
- 账号安全：改绑手机、注销、找回密码失败；
- 同一用户连续 2 轮说「没解决」「还是不行」；
- 明确说「转人工」「找客服」。

## 绝对不要做

- 承诺赔偿、退款金额、上线时间；
- 编造 FAQ 里没有的功能；
- 暴露内部工单号之外的任何 ID。

## 回复风格

中文用户说中文、英文用户说英文；一段不超过 3 行；结尾只问一个跟进问题。
````

只需 `name` + `description`：`name` 必须等于目录名、`description` 会进 system prompt（`nanobot/agent/skills.py:39-48`）。本机用临时 workspace 验证过：技能以 `workspace` 来源被发现，摘要里能看到名字和描述。

#### 2.3.3 knowledge/faq.md

写到 `~/.nanobot/workspace/knowledge/faq.md`：

```markdown
# 产品 FAQ

## Q：怎么导出数据？

1. 进入「设置 → 数据」；2. 选时间范围，点「导出 CSV」；
3. 导出完成后发到注册邮箱，链接 24 小时内有效。

## Q：API 限流是多少？

默认每分钟 60 次。需要提高配额请提交工单，注明用途和预估 QPS。

## Q：支持哪些付款方式？

对公转账、支付宝、微信支付。发票在「设置 → 账单」里申请。
```

知识库的关键不是"大"，而是**和 SKILL 里的引用路径对得上**。工单起步用 `write_file` 落 JSON 就够了；要进真正的系统（查状态、改状态、按用户过滤）时，换成项目三那种 MCP Server：`create_ticket`/`get_ticket` 注册成 `mcp_ticket_*`，Agent 只负责"什么时候调"。

### 2.4 部署与运行

```bash
nanobot plugins enable telegram     # 装渠道依赖并打开 enabled（也可手写配置）
export TELEGRAM_BOT_TOKEN=123456:ABC-DEF...
export ANTHROPIC_API_KEY=sk-ant-xxxx
nanobot channels status              # 确认 Telegram ✓
nanobot gateway --background
nanobot gateway logs
```

四步验收：① 自己的账号发"你好"能收到回复；② 问"怎么导出数据"能答出 `faq.md` 里的步骤；③ 说"我要退款"回复工单号且 `tickets/` 多一个 JSON；④ 不在 `allowFrom` 里的账号收不到正常回答。

加平台只改 `channels` 段：飞书用 `appId`+`appSecret`（+`encryptKey`、`verificationToken`），走 WebSocket 长连接，不用公网 webhook（`channels/feishu/manifest.py:11-38`）；钉钉用 `clientId`+`clientSecret`，先 `nanobot plugins enable dingtalk`（`channels/dingtalk/manifest.py:9-14`）。

### 2.5 面试话术（STAR 法）

> **面试官**：说说你怎么做一个多平台客服 Bot。

**S**：客户在 Telegram 咨询，之前靠人工，回答不一致，投诉没留痕。
**T**：做只服务白名单客户、按 FAQ 回答、超纲自动转工单的 Bot。
**A**：① `channels.telegram` + `allowFrom` 白名单，默认拒绝；② 话术和升级规则写成 `SKILL.md`，渐进披露；③ 知识库放 workspace，`restrictToWorkspace` 收紧文件工具；④ 转人工先落 `tickets/*.json`，要接真系统时换 MCP。
**R**：回答口径统一；投诉 100% 留痕；新增平台只改 `channels` 段。

### 2.6 验收标准

- [ ] `nanobot channels status` 里 Telegram 是 ✓。
- [ ] 白名单账号能收到 FAQ 答案，且能在 `faq.md` 找到原文。
- [ ] 说"我要退款"时给出工单号，不再解释政策。
- [ ] `tickets/` 下的 JSON 含 `chat_id`、`question`、`status`。
- [ ] 非白名单账号拿不到正常回答。

### 2.7 常见坑

| 坑 | 现象 | 原因与解法 |
|---|---|---|
| `allowFrom: [123456789]` | 渠道起不来 | `list[str]` 不收数字，写 `"123456789"` |
| 忘了 `"enabled": true` | 一直是 ✗ | 渠道默认关闭（`telegram/runtime.py:416`） |
| 用 `bot_token` | "bot token not configured" | 字段是 `token`（`telegram/manifest.py:10`） |
| `allowFrom: ["*"]` | 任何人可用 | `*` 是通配（`channels/base.py:246`），生产别用 |
| frontmatter 缺失/写错 | 技能"没生效" | `name`/`description` 硬要求（`skills.py:39-48`），目录名=技能名 |
| 相对路径读知识库 | WebUI 里找不到 | Telegram/CLI 相对 workspace，WebUI 可能换 project root；关键路径写绝对 |
| token 写进 JSON | 泄露 | 用 `${TELEGRAM_BOT_TOKEN}`，只在内存解析（`config/loader.py:196-215`） |

---

## 项目三：数据库 MCP Server（难度：高级）

### 3.1 项目需求分析

数据团队要查业务库，但不想给每个人开账号，需要"只读、可限流、可审计"的查询入口：`list_tables`（列表）、`describe_table`（字段）、`query_readonly`（一条 SELECT，限行数）。

安全五层：**SQL 检查**（只允许 SELECT/WITH、单语句）→ **只读连接**（`mode=ro`）→ **结果限制**（最多 500 行）→ **工具收敛**（`enabledTools`、`toolTimeout`）→ **系统权限**（只读账号/挂载、非 root）。

### 3.2 架构设计

```
nanobot（CLI 或 gateway） → MCPProvider.connect() ← tools.mcpServers
   → stdio 子进程 db_server.py（FastMCP）
   → list_tables / describe_table / query_readonly
   → SQLite（mode=ro）
```

配置里的 server 名会变成工具前缀：`businessdb` + `query_readonly` → 模型看到 **`mcp_businessdb_query_readonly`**（`nanobot/agent/tools/mcp.py:609`）。本机实测三个名字：`mcp_businessdb_{list_tables,describe_table,query_readonly}`。

**生命周期在组合根**：`MCPProvider`（`mcp.py:1346`）由 CLI 或 gateway 创建，`connect()` 后把工具注册进 `ToolRegistry`，退出时 `aclose()`（`nanobot/cli/agent.py:166`、`:190-194`）；`AgentLoop` 只跑回合，不拥有连接。

**为什么不手写 JSON-RPC**：旧文档用 `sys.stdin.readline()` 自己处理 `initialize`/`tools/list`/`tools/call`。它"能跑"，但少了 `notifications/initialized`、参数 schema 校验、取消/进度/批量分页、resources/prompts，stdio 分帧和错误码也得自己写。`mcp` 已是 nanobot 的运行时依赖（`pyproject.toml:44`），直接用官方 `FastMCP` 更省事。

### 3.3 核心实现

#### 3.3.1 db_server.py（40 行左右）

```python
"""db_server.py —— 只读数据库 MCP Server（MCP 官方 SDK 的 FastMCP）。"""
import json, os, re, sqlite3
from contextlib import closing
from pathlib import Path
from mcp.server.fastmcp import FastMCP

DB = Path(os.environ.get("BUSINESS_DB_PATH", "business.db")).expanduser()
mcp = FastMCP("business-db")  # 进程内服务名；nanobot 侧的工具前缀来自配置 key businessdb

def _connect():  # mode=ro：就算 SQL 检查被绕过，SQLite 也拒绝写入
    return closing(sqlite3.connect(f"file:{DB.as_posix()}?mode=ro", uri=True))

@mcp.tool()
def list_tables() -> str:
    """列出库里所有表和视图。"""
    with _connect() as conn:
        return "\n".join(f"- {n} ({t})" for n, t in conn.execute("SELECT name, type FROM sqlite_master WHERE type IN ('table','view')").fetchall()) or "(空库)"

@mcp.tool()
def describe_table(table: str) -> str:
    """查看一张表的字段名、类型与主键。"""
    if not re.fullmatch(r"[A-Za-z_][A-Za-z0-9_]*", table):
        return "表名不合法"
    with _connect() as conn:
        return "\n".join(f"- {r[1]} {r[2]}{' PK' if r[5] else ''}" for r in conn.execute(f'PRAGMA table_info("{table}")').fetchall()) or "没有这张表"

@mcp.tool()
def query_readonly(sql: str, max_rows: int = 100) -> str:
    """执行一条只读 SELECT；max_rows 上限 500。"""
    if not re.match(r"(?is)^(select|with)\s", sql.strip().rstrip(";")) or ";" in sql.strip().rstrip(";"):
        return "已拒绝：只允许单条 SELECT / WITH"
    try:
        with _connect() as conn:
            cur = conn.execute(sql)
            rows = cur.fetchmany(max(1, min(int(max_rows), 500)))
    except sqlite3.Error as exc:
        return f"查询失败：{exc}"
    return json.dumps({"columns": [c[0] for c in cur.description or []], "rows": [list(r) for r in rows]}, ensure_ascii=False, default=str)

if __name__ == "__main__":
    mcp.run(transport="stdio")  # nanobot 通过 stdio 拉起这个进程
```

- `@mcp.tool()` 注册工具，**docstring 就是工具描述**；`mode=ro` 是最后一道保险。
- 表名正则是因为 `PRAGMA table_info` 不能参数化；`fetchmany(500)` 保护上下文也保护数据库。

本机用 MCP 官方 stdio 客户端验证：工具列表正确；`SELECT ... LIMIT 3` 返回 `{"columns": [...], "rows": ...}`；`DROP TABLE users` 和多语句都返回"已拒绝"。

#### 3.3.2 config.json 的 MCP 片段

放进 `~/.nanobot/config.json` 的 `tools` 里：

```json
{
  "tools": {
    "mcpServers": {
      "businessdb": {
        "type": "stdio",
        "command": "/opt/mcp-db/.venv/bin/python",
        "args": ["/opt/mcp-db/db_server.py"],
        "cwd": "/opt/mcp-db",
        "env": { "BUSINESS_DB_PATH": "/srv/data/business.db" },
        "toolTimeout": 30,
        "enabledTools": ["list_tables", "describe_table", "query_readonly"]
      }
    }
  }
}
```

字段定义在 `nanobot/config/schema.py:362-374`：

| 字段 | 作用 | 备注 |
|---|---|---|
| `type` | `stdio`/`sse`/`streamableHttp` | 可省；有 `command` 判 stdio，`url` 以 `/sse` 结尾判 sse（`mcp.py:1021-1031`） |
| `command`+`args` | 拉起子进程 | 必须是网关能执行的路径，别依赖 shell 别名 |
| `cwd` | 子进程工作目录 | 相对路径的 env 会相对它 |
| `env` | 额外环境变量 | 放 `BUSINESS_DB_PATH` 这类配置 |
| `toolTimeout` | 单次调用超时秒数（默认 30） | 超时 nanobot 取消等待（`mcp.py:636-642`） |
| `enabledTools` | 只注册这些工具，默认 `["*"]` | 收窄后 resources/prompts 也不注册（`schema.py:374` 注释） |

不需要模型 key 的自检脚本（存成 `check_mcp.py`）：

```python
import asyncio
from pathlib import Path
from nanobot.agent.tools.mcp import MCPProvider
from nanobot.agent.tools.registry import ToolRegistry
from nanobot.config.loader import load_config

async def main() -> None:
    registry = ToolRegistry()
    provider = MCPProvider.from_config(load_config(Path.home() / ".nanobot" / "config.json"), registry)
    await provider.connect()
    try:
        print(sorted(registry.tool_names))
    finally:
        await provider.aclose()

asyncio.run(main())
```

本机实测输出：`['mcp_businessdb_describe_table', 'mcp_businessdb_list_tables', 'mcp_businessdb_query_readonly']`；网关日志里是 `MCP server 'businessdb': connected, 3 capabilities registered`（`mcp.py:1234`）。

### 3.4 部署与运行

```bash
mkdir -p /srv/data && cp business.db /srv/data/business.db   # 只读副本
python3 -m venv /opt/mcp-db/.venv
/opt/mcp-db/.venv/bin/pip install "mcp>=1.26,<2"
# 放好 db_server.py、改 config.json，然后：
python3 check_mcp.py
nanobot -m "先列出数据库里的表，再统计每个部门的订单金额，用 Markdown 表格给我"
```

期望：先 `list_tables`，再 `describe_table`，最后 `query_readonly`；返回 JSON，模型组织成表格。没发生时先看日志有没有 `connected`，再跑自检脚本——把"模型问题"和"连接问题"分开。Windows 两处注意：JSON 里反斜杠写 `\\`（或全用 `/`）；`command` 写 Python 的**绝对路径**，不要写 `python`。

### 3.5 面试话术（STAR 法）

> **面试官**：你开发过 MCP Server 吗？

**S**：数据团队要查库，直接给账号风险大，出过误 UPDATE。
**T**：做只读、限流、可审计的查询 MCP Server，接到 nanobot 上。
**A**：① 用官方 `mcp` 的 `FastMCP`，40 行拿到握手、schema、错误处理；② 三层防护——SQL 只允许 SELECT/WITH 且单语句、连接 `mode=ro`、结果最多 500 行；③ `enabledTools` 收工具、`toolTimeout=30` 防挂死；④ 用 `tools.mcpServers` 接入，工具名自动变 `mcp_businessdb_*`。
**R**：自然语言查数；写操作在数据库层就不可能成功；有超时和行数上限，不会拖垮 Agent。

### 3.6 验收标准

- [ ] `check_mcp.py` 打印 3 个 `mcp_businessdb_*` 工具。
- [ ] `nanobot -m "列出所有表"` 能触发 `list_tables`。
- [ ] `SELECT` 正常；`DROP`/`DELETE`/多语句被拒。
- [ ] `max_rows=1` 只返回 1 行。
- [ ] `enabledTools` 只留 `query_readonly` 后工具数变成 1。
- [ ] 数据库文件不可写时仍能查询（证明真的只读）。

### 3.7 常见坑

| 坑 | 现象 | 原因与解法 |
|---|---|---|
| `command` 写 `python` | 连接失败 | 网关 PATH 里可能没有；写绝对路径 |
| `cwd` 不设、`args` 相对 | 找不到脚本 | `args` 用绝对路径或设 `cwd`（`mcp.py:1067-1078`） |
| DB 路径写错 | "unable to open database file" | 先跑自检脚本确认能连 |
| `enabledTools` 名字写错 | 工具数变 0，只有 warning | 用原始名或 `mcp_<server>_<tool>`（`mcp.py:1141-1153`） |
| 把 `toolTimeout` 当数据库超时 | 调用被杀，库还在跑 | 慢查询要在 SQL/数据库层治理 |
| 只读连接=绝对安全 | 大表全扫照样拖垮库 | 加 `LIMIT`、索引、只读账号、独立副本 |
| 返回"已拒绝" | 模型当普通结果继续 | 这是 `isError=false`；要硬失败就 `raise`（本机验证会变 `isError=True`） |

---

## 面试高频题（10 道）

**1. 怎么设计运维 Agent？** 人设（`SOUL.md` 固定顺序）、权限（`allowPatterns` + `denyPatterns`）、调度（`cron` 工具，落盘 `jobs.json`）、产物（报告写文件）。安全边界在工具层，不在 prompt。

**2. 多平台怎么保证一致？** 平台之上只有一套：消息进 `MessageBus`（`InboundMessage` 带 channel/chat_id/sender_id），会话键 `{channel}:{chat_id}`（`bus/events.py:40`），知识库和技能共用 workspace。

**3. MCP 安全怎么做？** 输入（只允许 SELECT/WITH、单语句）、连接（`mode=ro`）、结果（行数上限）、工具（`enabledTools`、`toolTimeout`）、系统（只读账号、非 root），再加审计。正则黑名单是第一道门，不是最后一道。

**4. 记忆怎么设计？** `MEMORY.md`（长期事实，始终注入）+ `history.jsonl`（压缩后的追加日志）+ `SOUL.md`/`USER.md`。**没有 `save_memory` 工具**；压缩只写 `history.jsonl`，Dream 再挑重点更新长期文件。

**5. 遇到过幻觉吗？** 三件事压住：知识库优先（SKILL 写死先读 faq）、没命中就转人工、话术禁止承诺；再加"回答里的关键事实必须能在工具返回原文里找到"。

**6. 怎么保证工具调用可靠？** 超时（`toolTimeout`、`exec.timeout`）、失败可见（`ToolResult.error`）、上限（`maxToolIterations=200` 防死循环）、幂等（写文件覆盖而非追加）。

**7. 为什么选 nanobot 而不是 LangChain？** 配置驱动（`config.json` + Markdown）、MCP 原生、渠道与记忆开箱即用。代价是深度定制要读源码。

**8. 怎么测试 Agent 系统？** 单元（工具函数）、集成（MCP 连接 + 工具注册）、端到端（真实渠道）、安全（prompt 注入、越权命令、恶意 SQL）。每层都要能独立复现。

**9. 响应太慢怎么优化？** 减少工具轮次、控制上下文（`maxToolResultChars=16000`）、选对模型、流式输出、慢查询加索引而不是调大超时。

**10. Skill 和 RAG 的区别？** Skill 是过程性知识（怎么做），RAG 是事实性知识（知道什么）。Skill 用渐进披露：摘要进 prompt，正文按需读。FAQ 小的时候 `read_file` 就够。

---

## 练习题与思考题

1. **改阈值**：把项目一的 85% 改成 80%，只改 `SOUL.md` 和 cron 的 `message`，确认报告开头出现「严重」。
2. **换落点**：把项目一的 cron 任务改建在 Telegram 会话，对比 `jobs.json` 里 `originChannel` 的变化。
3. **加平台**：给项目二加飞书（`appId`+`appSecret`），只改 `channels` 段，验证技能和知识库不用动。
4. **收权限**：把项目三的 `enabledTools` 只留 `query_readonly`，用自检脚本确认工具数从 3 变 1。
5. **写技能**：给项目三写 `skills/data-analyst/SKILL.md`，规定"先 `describe_table` 再写 SQL、金额保留两位小数"。

---

## 本章总结

```
项目一：cron → exec → 报告        （工具 + 调度）
项目二：渠道 → 技能 → 工单        （渠道 + 知识）
项目三：配置 → MCP → 只读查询      （协议 + 安全）
```

共同点：**能用配置和 Markdown 解决的，不要写代码；必须写代码的，就写成工具或 MCP Server。**

最容易记错的三件事：配置是 `~/.nanobot/config.json`（四个抽屉）；定时任务不是配置项而是会话里的 `cron` 工具，落盘 `<workspace>/cron/jobs.json`；包名 `nanobot-ai`，命令是 `nanobot -m` / `agent` / `gateway` / `status`。

**下一步读什么**：想深挖渠道读第 09 章 + `nanobot/channels/base.py`；想深挖定时读第 10 章 + `nanobot/cron/service.py`；想练面试表达读第 13 章。

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| `AGENTS.md` 当人设 | 人设是 `SOUL.md`；`AGENTS.md` 是项目指令 | `agent/context.py:198-202` | 人设写 `SOUL.md` |
| 顶层 `model`/`provider`/`system_prompt_file` | 在 `agents.defaults`；没有 `system_prompt_file` | `config/schema.py:116-131`、`:427-433` | 迁移字段 |
| `"cron": [...]` | 无此字段，cron 由工具创建 | `agent/tools/cron.py:209-218` | 删掉 |
| `nanobot start --config` | 没有 `start`；用 `nanobot gateway --config` | `cli/commands.py:451-464` | 改命令 |
| `nanobot chat "..."` | 没有 `chat`；用 `nanobot -m "..."` | `cli/entry.py:23-31` | 改命令 |
| `pip install nanobot` | 包名 `nanobot-ai` | `pyproject.toml:2` | 改包名 |
| `platforms.*.bot_token` | `channels.*.token`，且要 `enabled: true` | `channels/telegram/runtime.py:413-419` | 改字段 |
| 顶层 `mcp_servers` | `tools.mcpServers` | `config/schema.py:418` | 挪进 `tools` |
| 手写 JSON-RPC server | 用官方 `mcp` 的 `FastMCP` | `agent/tools/mcp.py:1012-1015` | 换实现 |
| `tools.read_file.allowed_paths` | 无此字段；`FileToolsConfig` 只有 `enable` | `agent/tools/filesystem.py:28-31` | 用 `restrictToWorkspace` |
| `tools.exec.enabled` | 字段名是 `enable` | `agent/tools/shell.py:96` | 改字段 |
| 所有操作记 `MEMORY.md` | 记忆由 `MEMORY.md`+`history.jsonl`+Dream 管 | `agent/memory.py:58`、`:1072` | 审计写文件 |

## 附录 B：进阶理解

**1. 为什么配置用 JSON + pydantic？** 根级模型对未知键零容忍：多写一个 `cron` 直接 `Unknown setting` 并拒绝启动——配置错在启动时炸，而不是半夜静默跑错。`alias_generator=to_camel` 让磁盘写 `apiKey`、代码写 `api_key`（`config_base.py:12-15`）。

**2. 为什么 cron 任务绑定会话？** 它要"有地方交付"：每条任务记 `session_key` + `origin_channel` + `origin_chat_id`（`cron/bound_runner.py:65-129`）。代价是 CLI 里建的任务在网关里没有 `cli` 渠道，回复被丢弃（`channels/manager.py:845`）。"在哪儿建"决定"结果去哪儿"。

**3. 为什么 Skill 只给摘要？** 上下文稀缺。全文注入每轮多几千 token；v0.3.5 让 `description` 进 prompt、正文留在磁盘（`agent/skills.py:204-215`），模型自己决定读不读。`always: true` 是给"每轮必须遵守"的规则留的口子。

**4. 为什么 MCP 生命周期在组合根？** 连接有状态（stdio 子进程、SSE 会话），而 `AgentLoop` 是一次回合的编排者。CLI/gateway 各自持有 `MCPProvider`，才能在退出时干净 `aclose()`，也能运行期重连（`agent/tools/mcp.py:1346`、`:1412-1459`）。

**5. 为什么工具名加前缀？** 两个 server 可能都有 `search`，前缀就是命名空间；名字还会被清洗并截断到 64 字符（`agent/tools/mcp.py:181-197`）。

## 附录 C：Python 语法加油站

### C.1 pydantic：配置为什么能提前报错

```python
from pydantic import BaseModel

class MCPServerConfig(BaseModel):
    command: str = ""
    tool_timeout: int = 30

cfg = MCPServerConfig(command="python", tool_timeout="30")  # "30" 转成 int
print(cfg.tool_timeout + 1)                                  # 31
MCPServerConfig(tool_timeout="abc")                          # ValidationError
```

用在哪：`config/schema.py:362-374`、`config_base.py:12-15`。读法：先看字段名和默认值，再看 `field_validator`/`model_validator`；报错里的 `type=string_type`、`input_value=123` 就是类型不对。真实坑：`allowFrom: [123456789]` → `Input should be a valid string`。

### C.2 `async def`：工具函数为什么是协程

```python
import asyncio

async def fetch() -> str:      # 定义协程函数
    await asyncio.sleep(0.1)   # 遇到 IO 让出控制权
    return "done"

print(fetch())                 # <coroutine ...>，还没执行
print(asyncio.run(fetch()))    # done
```

用在哪：`CronTool.execute`（`agent/tools/cron.py:136`）、`MCPToolWrapper.execute`（`agent/tools/mcp.py:627`）。读法：`async def` 返回协程，必须有人 `await` 或 `asyncio.run`；工具里可能等网络、等子进程。

### C.3 asyncio 子进程 + 异步上下文管理器

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def open_mcp() -> None:
    params = StdioServerParameters(command="python", args=["db_server.py"])
    async with stdio_client(params) as (read, write):   # 拉起子进程，拿两条流
        async with ClientSession(read, write) as session:
            await session.initialize()
            print(await session.list_tools())
```

用在哪：`agent/tools/mcp.py:1012-1015`、`:1067-1079`。读法：`async with` = 进入时建资源、退出时自动关；MCP 连接活在这两个 `async with` 里，所以生命周期必须由创建它的任务负责。

### C.4 pathlib：路径别用字符串拼

```python
from pathlib import Path

db = Path("~/data/business.db").expanduser()
print(db.as_posix())                 # /home/you/data/business.db
cfg = db.parent / "config.json"      # 用 / 拼
db.parent.mkdir(parents=True, exist_ok=True)
```

用在哪：`db_server.py` 的 `as_posix()`、`cron/service.py:378`（建 `cron/`）、`agent/tools/filesystem.py:564`（写文件前建父目录）。读法：`~` 不会自动展开，要 `expanduser()`；Windows 下 `as_posix()` 把 `\` 变 `/`，拼 URI 特别有用。

### C.5 contextlib.closing：`with sqlite3.connect()` 不会关连接
```python
import sqlite3
from contextlib import closing

with closing(sqlite3.connect("business.db")) as conn:   # 退出时一定 close
    print(conn.execute("SELECT 1").fetchall())
```

用在哪：`db_server.py` 的 `_connect()`。读法：`sqlite3.Connection` 的 `with` 只管事务（提交/回滚），不管关闭；长跑的服务要用 `closing`，否则攒文件句柄。

## 附录 D：原文勘误

| # | 原文位置 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| 1 | §1.3 `"cron": [...]` | v0.3.5 没有该配置项，写了配置加载失败 | 会话里用 `cron` 工具创建，落盘 `cron/jobs.json` | `config/schema.py:427-433`、`agent/tools/cron.py:209-218` |
| 2 | §1.3 `"system_prompt_file"` | 不是字段；引导文件自动加载 | `AGENTS.md`/`SOUL.md`/`USER.md` 由 `ContextBuilder` 读取 | `agent/context.py:194-221` |
| 3 | §1.4 `pip install nanobot` | 包名错 | `pip install nanobot-ai` | `pyproject.toml:2` |
| 4 | §1.4 `CMD ["nanobot","start",...]` | 没有 `start` 子命令 | `nanobot gateway --config ...` | `cli/commands.py:451-464` |
| 5 | §1.4 `nanobot chat "..."` | 没有 `chat` 子命令 | `nanobot -m "..."`（或 `nanobot agent -m`） | `cli/entry.py:23-31` |
| 6 | §2.3 `"platforms"` + `"bot_token"` | 根键是 `channels`，字段是 `token`；`bot_token` 被忽略 | `channels.telegram.{enabled,token,allowFrom}` | `channels/telegram/runtime.py:413-419` |
| 7 | §2.3/§3.3 顶层 `"mcp_servers"` | 位置错 | `tools.mcpServers` | `config/schema.py:418` |
| 8 | §1.3 `tools.read_file.allowed_paths` | 没有该字段，也不会生效 | 用 `tools.restrictToWorkspace` + `tools.exec.sandbox` 管边界 | `agent/tools/filesystem.py:28-31`、`config/schema.py:401` |
| 9 | §2.3/§3.3 手写 JSON-RPC | 缺 `notifications/initialized`、schema 校验、取消/进度/批量 | 用 `mcp` 的 `FastMCP` | `agent/tools/mcp.py:1012-1015`、`pyproject.toml:44` |
