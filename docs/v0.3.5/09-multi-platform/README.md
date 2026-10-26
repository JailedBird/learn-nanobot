# 09 - 多平台接入

> 🎯 **本章目标**：弄懂 nanobot 如何用「渠道子包 + 双队列总线 + 统一出站调度」把 17 个聊天平台接到同一个 Agent 上，并能独立完成一次真实接入。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）

---

## 目录

- [9.1 多平台支持概述](#91-多平台支持概述)
- [9.2 MessageBus 架构详解](#92-messagebus-架构详解)
- [9.3 ChannelManager 出站策略](#93-channelmanager-出站策略)
- [9.4 BaseChannel 适配器模式](#94-basechannel-适配器模式)
- [9.5 各平台接入指南](#95-各平台接入指南)
- [9.6 session_key 会话隔离机制](#96-session_key-会话隔离机制)
- [9.7 多平台消息流完整链路](#97-多平台消息流完整链路)
- [9.8 实战练习](#98-实战练习)
- [9.9 面试高频题](#99-面试高频题)
- [9.10 本章小结](#910-本章小结)
- [附录 A：与旧版文档的差异](#附录-a与旧版文档的差异)
- [附录 B：进阶理解](#附录-b进阶理解)
- [附录 C：Python 语法加油站](#附录-cpython-语法加油站)
- [附录 D：原文勘误](#附录-d原文勘误)

---

## 9.1 多平台支持概述

### 9.1.1 为什么需要多平台支持

一句话结论：**Agent 只处理"统一消息"，平台差异全部挡在渠道层**。

```
Telegram ─┐
Discord  ─┼──→ nanobot gateway ──→ 同一个 Agent 内核
飞书      ─┤    （常驻后台进程）      （会话按平台隔离）
钉钉      ─┤
WebUI    ─┘
```

启动进程是 **`nanobot gateway`**，不是裸 `nanobot`。裸 `nanobot`（等价 `nanobot agent`）是本地命令行 Agent/TUI，不会连聊天平台；渠道由 gateway 用 `asyncio.create_task(channels.start_all())` 拉起（`nanobot/cli/gateway_runtime.py:941`）。

### 9.1.2 支持的平台列表

`nanobot/channels/` 下有 **17 个渠道子包**，每个至少含 `manifest.py`（静态清单）与 `runtime.py`（联网实现）。目录由 `pkgutil.iter_modules` 自动扫描（`nanobot/channels/registry.py:20`），没有手写注册表。

| 子包 | 凭据 / 接入方式 | 备注 |
|---|---|---|
| `telegram` | `token` | 默认 polling，可切 webhook |
| `discord` | `token` | 需开 Message Content intent |
| `feishu` | `appId` + `appSecret` | WebSocket 长连接，可扫码登录 |
| `dingtalk` | `clientId` + `clientSecret` | Stream 模式，可选依赖 |
| `slack` | `botToken` + `appToken` | Socket Mode |
| `matrix` | `homeserver` + `accessToken` | 也可用账号密码 |
| `mattermost` | `serverUrl` + `token` | |
| `mochat` | `clawToken` | |
| `msteams` | `appId` + `appPassword` | 需公网 HTTPS 回调 |
| `email` | IMAP/SMTP 主机与账号 | 靠 `allowFrom` 控权限 |
| `signal` | `phoneNumber` + signal-cli daemon | |
| `whatsapp` | 扫码登录 | `channels login whatsapp` |
| `weixin` | 扫码登录，token 由登录写入 | `plugins enable` + `channels login` |
| `qq` | `appId` + `secret` | QQ 开放平台，私聊为主 |
| `napcat` | `wsUrl` + `accessToken` | 经 NapCat/OneBot v11 接 QQ |
| `wecom` | `botId` + `secret` | |
| `websocket` | 无凭据 | WebUI 内置渠道，**唯一默认启用** |

两个必踩的坑：**只有 `websocket` 默认启用**（`ChannelPlugin.default_enabled` 默认 `False`，`nanobot/channels/plugin.py:38`；`nanobot/channels/websocket/manifest.py:18` 才改成 `True`），其余渠道必须写 **`"enabled": true`**，否则连实例都不会创建（`nanobot/channels/contracts.py:104` 的 `ChannelActivation.resolve` 把缺失值当 `False`）；**`plugins enable` 管的是可选依赖（pip 包），不是开关**——它装 `dingtalk-stream` 之类，`channels status` 里的 ✓ 仍由 config 的 `enabled` 决定。

> 术语：**渠道（Channel）**＝一个平台的适配器；**消息总线（MessageBus）**＝进程内异步队列；**会话键（session_key）**＝一段对话的身份，形如 `telegram:123456`。

### 9.1.3 核心设计理念

```
第 1 层 Agent 内核（agent/loop.py）：只认 Inbound/OutboundMessage，不 import 平台 SDK
第 2 层 MessageBus（bus/queue.py）：两条 asyncio.Queue，上下行分开
第 3 层 Channel 子包（channels/<name>/）：平台协议 ↔ 统一消息
调度器 ChannelManager：启停渠道、路由、流式合并、重试
```

---

## 9.2 MessageBus 架构详解

### 9.2.1 双队列架构

把 MessageBus 想成**前台的两个文件筐**：收件筐放用户来信，发件筐放回信；前台不关心信从哪来、走哪家快递。

```
┌──────────────────────────────────────────────────────┐
│ MessageBus                                           │
│  inbound  Queue[InboundMessage]   ← 各渠道 publish    │
│           ↓ AgentLoop.consume_inbound()              │
│  outbound Queue[OutboundMessage]  ← Agent publish     │
│           ↓ ChannelManager → Channel.send()          │
└──────────────────────────────────────────────────────┘
```

两条队列在构造时建好，**没有 maxsize**（`nanobot/bus/queue.py:32-33`：`asyncio.Queue()`）。`maxsize=0` 表示无上限，`put()` 不会因"筐满"而阻塞；真正的限流在出站侧（9.3.4）。

### 9.2.2 InboundMessage

```python
# 来源：nanobot/bus/events.py:24-42（节选）
@dataclass
class InboundMessage:
    channel: str       # "telegram"
    sender_id: str     # 发送者 ID（用于鉴权）
    chat_id: str       # 会话 ID
    content: str
    media: list[str] = field(default_factory=list)   # 媒体 URL
    metadata: dict[str, Any] = field(default_factory=dict)
    session_key_override: str | None = None
    input_role: Literal["user", "system"] | None = None

    @property
    def session_key(self) -> str:
        return self.session_key_override or f"{self.channel}:{self.chat_id}"
```

`metadata` 存平台附加信息（`message_id`、线程 id…）；`session_key_override` 用于子 Agent 与定时任务指定会话；`input_role="system"` 表示系统注入。`is_user_input` 属性（`:44-49`）让 `channel == "system"` 的消息不算"用户说过的话"，后台任务不会污染对话历史。

### 9.2.3 OutboundMessage

```python
# 来源：nanobot/bus/events.py:52-68（节选）
@dataclass
class OutboundMessage:
    channel: str
    chat_id: str
    content: str
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    event: "AgentEvent | None" = None
```

`event` 是 v0.3.5 的关键：出站不一定是文本，也可能是 `StreamDeltaEvent`、`StreamEndEvent`、`ProgressEvent` 等类型化事件。ChannelManager 据此决定调 `send_delta()`、`send_reasoning()` 还是普通 `send()`（`nanobot/channels/manager.py:908` 的 `_send_once`）。

### 9.2.4 API 与调用方

```python
# 来源：nanobot/bus/queue.py:37-69（节选）
async def publish_inbound(self, msg):  await self.inbound.put(msg)
async def consume_inbound(self):       return await self.inbound.get()
async def publish_outbound(self, msg): await self.outbound.put(msg)
async def consume_outbound(self):      return await self.outbound.get()
```

| 谁 | 调什么 | 含义 |
|---|---|---|
| 渠道 | `publish_inbound` | 用户消息入收件筐 |
| `AgentLoop.run` | `consume_inbound` | 取一条干活（`nanobot/agent/loop.py:1269`） |
| Agent | `publish_outbound` | 回复入发件筐 |
| ChannelManager | `consume_outbound` | 取一条投递（`nanobot/channels/manager.py:782`） |

### 9.2.5 为什么用队列而不是直接调用

直接调用意味着 Agent 要认识所有渠道类、自己管并发和限流、测试还得 mock 平台 SDK；走 MessageBus 后 Agent 只认识两种消息，加平台只加子包，测试往队列塞一条即可。额外好处：本地订阅（`publish` / `subscribe`，`nanobot/bus/queue.py:92-126`）给 WebUI 这类进程内观察者用，不经过网络发送；`publish_nowait`（`:128`）把分发包成 Task，`drain`（`:145`）在退出前收尾。

---

## 9.3 ChannelManager 出站策略

### 9.3.1 职责

`ChannelManager`（`nanobot/channels/manager.py:82`）是总调度室，docstring 写明三件事：实例化已启用渠道、启停渠道、路由出站消息。落到代码是：**发现+实例化**（`_init_channels`，`:234`）、**启动**（`start_all`，`:598`，先起分发任务 `:606` 再逐渠道 `create_task` `:611`）、**路由**（`_dispatch_outbound_loop`，`:767`）、**投递策略**（合并/过滤/重试/限流）。

### 9.3.2 流式合并：`_coalesce_stream_deltas`

LLM 逐 token 输出，若每个 token 都调一次平台 API，会触发限流、界面闪烁。v0.3.5 不用定时器降频，而是**队列里已经积压几条就合并几条**：

```python
# 来源：nanobot/channels/manager.py:823-828
if isinstance(event, StreamDeltaEvent):
    msg, extra_pending = self._coalesce_stream_deltas(msg)
    pending.extend(extra_pending)
    event = msg.event
```

`_coalesce_stream_deltas`（`:933`）用 `get_nowait()` 吸干发件筐里**同一条流**（同 `channel`、`chat_id`、`stream_id`）的连续 delta，拼成一个 `content`；遇到别的消息立即停手并把那条放回 `pending`，保证顺序。

```
队列已积压 8 个 delta → 合并成 1 条 → 1 次 API 调用
消费够快（队列只有 1 条）→ 不合并 → 逐条 send_delta，打字机效果
```

所以它叫"合并"不叫"降频"：**没有 300ms 定时器，也没有 `_buffer` / `_last_send_time` 字段**。合并只在积压时发生，天然自适应。渠道要支持流式必须重写 `send_delta()`；基类是空操作（`nanobot/channels/base.py:129`），`supports_streaming` 仅在"配置 `streaming: true`"且"子类真重写了"时为真（`:225-235`）。

### 9.3.3 进度与工具提示过滤

```python
# 来源：nanobot/channels/manager.py:803-811（节选）
if progress_event:
    if progress_event.tool_hint and not self._should_send_progress(msg.channel, tool_hint=True):
        continue
    if not progress_event.tool_hint and not self._should_send_progress(msg.channel, tool_hint=False):
        continue
```

`_should_send_progress`（`:347`）读渠道实例的 `send_progress` / `send_tool_hints`。构造渠道时先问渠道的 `progress_transport_defaults()`，没答案再用全局 `channels.sendProgress` / `channels.sendToolHints`（`nanobot/config/schema.py:33-34`，默认 `true`，`nanobot/channels/manager.py:219-228`）。例如微信 manifest 两个都默认 `false`（`nanobot/channels/weixin/manifest.py:20-21`），避免刷屏。

### 9.3.4 重试、并发与顺序

```python
# 来源：nanobot/channels/manager.py:59-62
# Retry delays for message sending (exponential backoff: 1s, 2s, 4s)
_SEND_RETRY_DELAYS = (1, 2, 4)
_OUTBOUND_CONCURRENCY = 32
_OUTBOUND_PENDING_LIMIT = 256
```

重试在 `_send_with_retry`（`:998`）：`max_attempts = max(config.channels.send_max_retries, 1)`（`:1012`）；业务错误由 `should_retry_send_error`（`nanobot/channels/base.py:112`）判定是否值得重试；失败后按 `_SEND_RETRY_DELAYS` 取延迟并 `asyncio.sleep`（`:1043`）。`sendMaxRetries` 默认 **3**（含首次，`nanobot/config/schema.py:37`），退避表 **(1,2,4)** 秒（`nanobot/channels/manager.py:60`），同时发送 32 条、待发 256 条（`:61-62`）；默认 3 次尝试时实际只会等 1s、2s，4s 那档要调大重试次数才用得到。

顺序由 `_queue_outbound`（`:728`）保证：每个 `(channel, chat_id)` 有一条"尾巴任务"，新消息先 `await` 上一条，**同会话严格 FIFO**；不同会话并行；槽位满时调度暂停而不是无限建 Task。

---

## 9.4 BaseChannel 适配器模式

### 9.4.1 适配器模式

各平台协议完全不同（Telegram HTTP API、Discord Gateway、飞书 lark-oapi 长连接、钉钉 Stream），基类把"渠道该长什么样"钉死，子类只填平台细节。

```
        ┌──────────────────────────────┐
        │      BaseChannel(ABC)        │
        │ start() stop() send()        │ ← 三个 @abstractmethod
        │ _handle_message()            │ ← 统一"收消息→发总线"
        │ is_allowed()  login()        │
        └──────────────┬───────────────┘
   ┌──────────────┬────┴─────────┬──────────────┐
Telegram       Discord        Feishu        DingTalk
```

### 9.4.2 抽象基类

```python
# 来源：nanobot/channels/base.py:21-46（节选）
class BaseChannel(ABC):
    name: str = "base"
    display_name: str = "Base"
    send_progress: bool = True
    send_tool_hints: bool = True

    def __init__(self, config: Any, bus: MessageBus):
        self.config = config
        self.logger = logger.bind(channel=self.name)
        self.bus = bus
        self._running = False
```

构造参数只有 `(config, bus)`，**没有 name**——`name` 是类属性。三个抽象方法：`start()`（`nanobot/channels/base.py:75`，建长连接并监听）、`stop()`（`:87`）、`send(msg)`（`nanobot/channels/base.py:92`，失败要抛异常交给 manager 重试）。

收消息走**模板方法** `_handle_message`（`:255`），子类不重写：

```python
# 来源：nanobot/channels/base.py:255-321（大幅节选）
async def _handle_message(self, sender_id, chat_id, content, media=None,
                          metadata=None, session_key=None, is_dm=False, ...):
    permission_id = authorization_id if authorization_id is not None else sender_id
    if not self.is_allowed(permission_id):
        if is_dm:
            code = generate_code(self.name, str(sender_id))
            await self.send(OutboundMessage(..., content=format_pairing_reply(code)))
        return
    msg = InboundMessage(channel=self.name, sender_id=str(sender_id),
                         chat_id=str(chat_id), content=content,
                         media=media or [], metadata=metadata or {},
                         session_key_override=session_key)
    await self.bus.publish_inbound(msg)
```

这一段同时解决四件事：**鉴权**（`is_allowed`，`:237`）、**配对码**（未授权私聊先回配对码而非静默丢弃）、**流式标记**、**统一构造入站消息**。新渠道只要在平台回调里 `await self._handle_message(...)`。

### 9.4.3 一个渠道子包长什么样

| 文件 | 作用 | 例子 |
|---|---|---|
| `manifest.py` | 依赖无关清单：字段、必填项、依赖、runtime 导入路径 | `nanobot/channels/telegram/manifest.py` |
| `runtime.py` | 真正联网的 `BaseChannel` 子类 | `runtime:TelegramChannel` |
| `config.py` / `validation.py` | 配置模型 / 校验 | `nanobot/channels/feishu/config.py` |

manifest 的 `ChannelSetupSpec` 声明字段与必填项，例如 Telegram 的 `token` 是 `field("secret")` 且 `required=(required("token"),)`（`nanobot/channels/telegram/manifest.py:8-30`），飞书是 `required_fields("appId", "appSecret")`（`nanobot/channels/feishu/manifest.py:38`）。`required` 不只给文档看：WebUI 设置页、`onboard`、配置校验都读它。

### 9.4.4 添加新平台的步骤

建目录 `nanobot/channels/line/` 并写 `__init__.py`；写 `manifest.py` 导出 `PLUGIN = ChannelPlugin(name="line", display_name="LINE", runtime="nanobot.channels.line.runtime:LineChannel", setup=SETUP_SPEC, dependencies=(...))`；写 `runtime.py`：

```python
class LineChannel(BaseChannel):
    name = "line"                 # 必须与 manifest 的 name 一致
    async def start(self): ...    # 建 webhook / 长连接
    async def stop(self): ...
    async def send(self, msg: OutboundMessage): ...   # 失败要 raise
    # 平台回调里：await self._handle_message(sender_id=..., chat_id=..., content=...)
```

不用改注册表：`discover_plugins()` 按目录扫描（`nanobot/channels/registry.py:30`），`load_channel_package()` 校验 runtime 路径确实在本子包内（`nanobot/channels/plugin.py:150-172`）。最后在 `channels.line` 写凭据，**记得 `"enabled": true`**。

---

## 9.5 各平台接入指南

通用四步，后面只是换字段：

```bash
nanobot plugins enable dingtalk     # 1. 需要可选依赖的渠道先装依赖
# 2. 编辑 ~/.nanobot/config.json（JSON，不是 YAML），加 channels.<name> 且 enabled=true
nanobot channels status             # 3. 确认读到了这个渠道
nanobot gateway                     # 4. 前台常驻启动网关（渠道只在 gateway 里跑）
```

`channels status` 就是遍历渠道、读配置 `enabled` 打勾（`nanobot/cli/commands.py:522-548`）。

### 9.5.1 Telegram 接入

在 Telegram 找 `@BotFather` → `/newbot` → 拿到 `123456:ABC-DEF...`。

```json
{ "channels": { "telegram": { "enabled": true, "mode": "polling",
  "token": "123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11", "allowFrom": ["你的 Telegram 数字 ID"] } } }
```

字段名是 **`token`**，不是 `bot_token`。`mode` 默认 `polling`；公网 webhook 才需 `webhookUrl` 等（`nanobot/channels/telegram/manifest.py:15-23`）。启动 `nanobot gateway` 后给 bot 发消息；若回配对码，`/pairing approve <code>` 后再发。

### 9.5.2 飞书（Feishu / Lark）接入

飞书最容易被旧资料误导：**它走 WebSocket 长连接，不需要公网 webhook，也不需要 ngrok。**

```python
# 来源：nanobot/channels/feishu/runtime.py:951-953
"""Feishu/Lark channel using WebSocket long connection.
Uses WebSocket to receive events - no public IP or webhook required."""
```

开放平台创建企业自建应用 → 开机器人 → 订阅 `im.message.receive_v1` → 选**长连接** → 拿 App ID / Secret。要流式回复还需 `cardkit:card:write` 权限。

```json
{ "channels": { "feishu": { "enabled": true, "domain": "feishu",
  "appId": "cli_xxxxxxxxxx", "appSecret": "xxxxxxxxxxxxxxxxxxxxxxxx",
  "allowFrom": ["ou_你的open_id"] } } }
```

`appId` / `appSecret` 是落盘字段（snake_case 只做兼容读取，`nanobot/channels/feishu/config.py:17-18`）；`encryptKey`、`verificationToken` 在长连接下可选。也可扫码：`nanobot plugins enable feishu && nanobot channels login feishu`，成功后自动把 `appId`、`appSecret`、`domain`、`enabled` 写回当前 config。

### 9.5.3 钉钉（DingTalk）接入

钉钉用 **Stream 模式**，无需公网 IP，但属于可选依赖渠道（`nanobot plugins enable dingtalk`）。字段是 **`clientId` + `clientSecret`**（对应开放平台 AppKey/AppSecret），不是 `app_key`：

```json
{ "channels": { "dingtalk": { "enabled": true, "groupUserIsolation": false,
  "clientId": "dingxxxxxxxxxx", "clientSecret": "xxxxxxxxxxxxxxxxxxxxxxxx",
  "allowFrom": ["你的 staffId"] } } }
```

字段定义见 `nanobot/channels/dingtalk/manifest.py:8-15`。`groupUserIsolation=true` 时同群每个发言者独立会话，但回复仍回群里。

### 9.5.4 Discord 接入

Developer Portal 建 Application → Bot → 拿 token → 邀请进服务器 → **开 Message Content Intent**（不开则消息内容为空）。同样是 `token`（`nanobot/channels/discord/manifest.py:9`），`intents` 默认 `37377`：

```json
{ "channels": { "discord": { "enabled": true, "intents": 37377,
  "token": "MTIzNDU2Nzg5MDEyMzQ1Njc4OQ.AbCdEf.xxxxxxxx", "allowFrom": ["你的 Discord 用户 ID"] } } }
```

### 9.5.5 微信、QQ 与其他平台

**微信（weixin）** 是可选依赖且必须扫码登录，token 由登录流程写入：

```bash
nanobot plugins enable weixin
nanobot channels login weixin      # 扫码；--force 强制重登
nanobot gateway
```

```json
{ "channels": { "weixin": { "enabled": true, "allowFrom": ["你的微信用户 ID"] } } }
```

**QQ** 走 QQ 开放平台机器人，需先 `nanobot plugins enable qq`，字段是 `appId` + `secret`：

```json
{ "channels": { "qq": { "enabled": true, "msgFormat": "plain",
  "appId": "YOUR_APP_ID", "secret": "YOUR_APP_SECRET", "allowFrom": ["YOUR_OPENID"] } } }
```

> ⚠️ 常见误传：**QQ 不需要 `channels login`**。v0.3.5 只有 `weixin`、`whatsapp`、`feishu` 重写了 `login()`（`nanobot/channels/weixin/runtime.py:979`、`nanobot/channels/whatsapp/runtime.py:383`、`nanobot/channels/feishu/runtime.py:1004`）；`BaseChannel.login()` 默认直接 `return True`（`nanobot/channels/base.py:62-72`），对 QQ 执行 login 不会做任何认证。需要 `plugins enable` 的可选依赖渠道则确实包括钉钉、微信、QQ、NapCat、Slack、Matrix 等。

其他平台最小配置（**全部都要 `"enabled": true`**）：

| 平台 | 关键字段 |
|---|---|
| Slack | `botToken` + `appToken`（Socket Mode） |
| Matrix | `homeserver` + `accessToken`（或 `userId`+`password`） |
| WeCom | `botId` + `secret` |
| Email | `imapHost`/`imapUsername`/`imapPassword` + `smtpHost`… |
| Signal | `phoneNumber` + 本机 signal-cli daemon |
| MS Teams | `appId` + `appPassword`，需公网 HTTPS |
| MoChat | `clawToken` |
| NapCat | `wsUrl` + `accessToken`（OneBot v11，支持 QQ 群） |
| Mattermost | `serverUrl` + `token` |
| WhatsApp | 扫码登录，无 token 字段 |

### 9.5.6 多平台同时接入

```json
{ "channels": {
  "telegram": { "enabled": true, "token": "123456:ABC..." },
  "discord":  { "enabled": true, "token": "MTIz..." },
  "feishu":   { "enabled": true, "appId": "cli_xxx", "appSecret": "xxx" }
} }
```

`nanobot gateway` 启动后三个渠道各跑在独立 Task（`nanobot/channels/manager.py:609-616`），互不阻塞；会话靠 `session_key` 隔离。

### 9.5.7 三个高频故障

漏写 `enabled` / 字段层级放错 / 改的不是生效的 config → `channels status` 显示 ✗。渠道 ✓ 但收不到消息 → 网关没跑、平台权限或事件没配、依赖没装，用 `nanobot gateway --verbose` 查。私聊没反应 → 未授权用户会先收到配对码（`nanobot/channels/base.py:274-297`），需 `/pairing approve <code>`；`allowFrom: ["*"]` 则放开所有人。

---

## 9.6 session_key 会话隔离机制

### 9.6.1 构成

一句话：**`session_key = "{channel}:{chat_id}"`**（`nanobot/bus/events.py:39-42`）。

```python
# 来源：nanobot/session/keys.py:12-16
def session_key_for_channel(channel: str, chat_id: str, *, unified_session: bool = False) -> str:
    if unified_session:
        return UNIFIED_SESSION_KEY        # "unified:default"
    return f"{channel}:{chat_id}"
```

| 场景 | session_key |
|---|---|
| Telegram 私聊 | `telegram:123456789` |
| Telegram 群 | `telegram:-100789` |
| 飞书私聊 | `feishu:ou_xxx` |
| 心跳任务 | `heartbeat` |
| 统一会话模式 | `unified:default` |

### 9.6.2 作用与存储

`session_key` 决定三件事：**历史隔离**（每 key 一份会话）、**并发调度**（同 key 新消息进 pending 队列，不并发跑两个回合）、**回复路由**（OutboundMessage 带 channel + chat_id 原路返回）。

**存储位置要特别注意**：会话不是工作区里的 `sessions/telegram:123456.jsonl`，而是运行目录下按 workspace 分片的 JSONL：

```python
# 来源：nanobot/session/manager.py:1027 与 :1033
return self.sessions_dir / f"{self.storage_key(key)}.jsonl"
...
return self.sessions_dir / f"{safe_filename(key.replace(':', '_'))}.jsonl"
```

形如 `<runtime>/sessions/<workspace-id>/telegram_123456.jsonl`（冒号换成下划线）。会话目录**被强制放在 workspace 之外**：落在 workspace 内直接 `RuntimeError`（`nanobot/session/manager.py:559-563`），防止 Agent 的工具误删/误改自己的历史。

### 9.6.3 session_key_override：线程与子会话

需要"同一个 chat 里再分叉"时，用 `InboundMessage.session_key_override` 字段（`nanobot/bus/events.py:35`），而不是某个 `build_session_key()` 函数。谁在用：**子 Agent 回报**（`nanobot/agent/subagent.py:521-527`）、**定时任务**回原会话（`nanobot/cron/bound_runner.py:120-128`）、**渠道线程**（Discord thread / Slack thread / 飞书话题由渠道算出更细的 key 传进 `_handle_message`）。

### 9.6.4 统一会话模式

一个人多设备时（手机 Telegram + 电脑 WebUI）可开 `{ "agents": { "defaults": { "unifiedSession": true } } }`（`nanobot/config/schema.py:145`）。开启后所有渠道共用一个 `unified:default`，最近说话的渠道记进 metadata（`nanobot/session/keys.py:19-27`）以便回复找对出口；多人场景不要开。

---

## 9.7 多平台消息流完整链路

### 9.7.1 一条消息的旅程

```
Step 1  Telegram 把 Update 推给 Bot
Step 2  TelegramChannel 回调解析
        └── await self._handle_message(...)           channels/base.py:255
Step 3  bus.publish_inbound(msg)                      bus/queue.py:37
        session_key = "telegram:123456789"
Step 4  AgentLoop.run 循环 consume_inbound()           agent/loop.py:1261/1269
        └── 按 session_key 找/建会话，装配上下文
Step 5  AgentRunner 跑"模型 ↔ 工具"循环（如 ToolRegistry.execute）  registry.py:187
Step 6  回复分片 publish_outbound                      bus/queue.py:45
        ├── 普通文本 → OutboundMessage(content=...)
        └── 流式     → StreamDeltaEvent / StreamEndEvent
Step 7  ChannelManager 出站循环                        channels/manager.py:767
        ├── ProgressEvent → 按渠道开关决定发不发
        ├── StreamDeltaEvent → _coalesce_stream_deltas  :933
        └── _queue_outbound → _send_with_retry          :728 / :998
Step 8  TelegramChannel.send(msg) → Bot API → 用户看到回复
```

### 9.7.2 跨平台并发

```
t=0.0s  Telegram A: "你好"        ┐
t=0.5s  Discord  B: "帮我写代码"   ├─ 各自入 inbound 队列
t=0.6s  WebUI    C: "看看文件"     ┘
t=0.7s  AgentLoop 取 A → 建 Task；t=0.8s 取 B → 建 Task（并行）
t=0.9s  取 C：与 A 同会话则排队，否则并行
t=2.0s  A 的回复 → telegram；t=2.3s B 的回复 → discord
```

关键：`AgentLoop.run` 把每条消息**派发成 Task**（`nanobot/agent/loop.py:1261` 的 docstring 写明 "dispatching messages as tasks to stay responsive to /stop"），慢会话不堵别的平台；同一 `session_key` 内部通过 pending 队列串行，避免抢上下文。

---

## 9.8 实战练习

### 练习 1：接入 Telegram 并观察状态

```bash
# 1) 写配置（必须是 ~/.nanobot/config.json，注意 enabled 与 token）
cat > ~/.nanobot/config.json << 'EOF'
{
  "agents": { "defaults": { "model": "anthropic/claude-opus-4-5", "provider": "auto" } },
  "channels": { "telegram": { "enabled": true, "token": "YOUR_BOT_TOKEN" } }
}
EOF

nanobot channels status    # 2) 应看到 Telegram ✓
nanobot gateway            # 3) 启动网关（不是裸 nanobot）
```

### 练习 2：同时接 Telegram + 飞书 + 钉钉

```bash
nanobot plugins enable dingtalk
```

```json
{ "channels": {
  "telegram": { "enabled": true, "token": "TG_TOKEN" },
  "feishu":   { "enabled": true, "appId": "cli_xxx", "appSecret": "xxx" },
  "dingtalk": { "enabled": true, "clientId": "dingxxx", "clientSecret": "xxx" }
} }
```

启动日志应出现 `Channels enabled: telegram, feishu, dingtalk`。验收：三个平台同时在线，历史互不串。

### 练习 3：验证会话隔离与落盘

```bash
# 分别在 Telegram 和 Discord 聊两轮
nanobot sessions list
# 运行目录下会出现（注意下划线与 workspace-id 子目录）：
#   sessions/<workspace-id>/telegram_123456789.jsonl
#   sessions/<workspace-id>/discord_1122334455.jsonl
```

在 workspace 里找 `sessions/` 是找不到的——会话被强制放在 workspace 之外。

---

## 9.9 面试高频题

### 题目 1：如何设计一个支持多平台的 Agent 系统？

> "核心是三层解耦。Agent 内核只处理 `InboundMessage`/`OutboundMessage`，不知道消息来自哪个平台；MessageBus 用两条 `asyncio.Queue` 解耦生产者和消费者；Channel 适配器每个平台一个 `BaseChannel` 子类，只实现 `start`/`stop`/`send`，收消息统一走基类 `_handle_message`。加平台只加子包，manifest 由 `pkgutil` 自动发现。会话隔离靠 `session_key = channel:chat_id`。出站由 ChannelManager 统一做流式合并、进度过滤、1/2/4 秒退避重试，并保证同一会话 FIFO。"

### 题目 2：MessageBus 的双队列解决了什么问题？

> "解耦和削峰。直接调用意味着 Agent 必须持有所有渠道对象，加平台就要改 Agent。双队列让两边只依赖消息结构，慢渠道也不会阻塞其他平台；测试时直接往队列注入消息即可。需要澄清：nanobot 的入站队列是**无界**的 `asyncio.Queue()`，并没有'队列满自动等待'的背压；限流在出站侧——32 个并发发送槽 + 256 个待发槽位。"

### 题目 3：不同平台的消息格式差异怎么处理？

> "适配器模式 + 一个足够通用的消息结构。入站把平台事件映射成 `channel`/`sender_id`/`chat_id`/`content`/`media`/`metadata`，平台特有的 message_id、thread id 放 `metadata`。出站更有意思：`OutboundMessage` 除了文本还能带 `event` 表示类型化事件（流式增量、进度、思考）。渠道按需实现 `send_delta`、`send_reasoning_delta` 等可选钩子；没实现的平台基类默认空操作，流式事件被安全丢弃而不是报错，于是富客户端和纯文本平台共用一套出站流程。"

---

## 9.10 本章小结

```
┌───────────────────────────────────────────────────────┐
│ nanobot gateway：17 个渠道子包（各自独立 Task）         │
│        ↓ inbound          ↑ outbound                  │
│  MessageBus ──→ AgentLoop/AgentRunner ──→ ChannelManager│
│                                        └──→ Channel.send│
│  session_key = "{channel}:{chat_id}"                   │
└───────────────────────────────────────────────────────┘
```

| 考点 | 一句话回答 |
|---|---|
| 渠道数量 | 17 个子包，`pkgutil` 自动发现 |
| 启动命令 | `nanobot gateway`（裸 `nanobot` 是本地 TUI） |
| 消息总线 | `nanobot/bus/queue.py:19`，`inbound` + `outbound` 两条无界队列 |
| 会话键 | `{channel}:{chat_id}`（`nanobot/bus/events.py:40`） |
| 基类 | `BaseChannel`（`nanobot/channels/base.py:21`），抽象 `start/stop/send` |
| 收消息模板 | `_handle_message`（`:255`）：鉴权 → 配对码 → 入队 |
| 流式合并 | `_coalesce_stream_deltas`（`:933`），按积压合并，无定时器 |
| 默认启用 | 只有 `websocket`；其余必须 `"enabled": true` |
| 飞书接入 | WebSocket 长连接，无需公网 webhook |
| 会话存储 | `<runtime>/sessions/<workspace-id>/<key>.jsonl`，在 workspace 之外 |

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 渠道是 `nanobot/channels/telegram.py` 平铺单文件 | 每渠道一个子包（`manifest.py`/`runtime.py`/…） | `nanobot/channels/plugin.py:127-131` | 按"子包 + manifest"理解 |
| 总线在 `nanobot/bus/message_bus.py` | `nanobot/bus/queue.py`，类名仍 `MessageBus` | `nanobot/bus/queue.py:19` | 引用新路径 |
| 约 10 个平台，CLI 也算渠道 | 17 个渠道子包；CLI 是 `nanobot agent` 入口，不是渠道 | `nanobot/cli/commands.py:472` | 平台清单以 manifest 为准 |
| 出站只有文本 | `OutboundMessage` 带 `event`，支持流式/进度/思考事件 | `nanobot/bus/events.py:68`；`nanobot/channels/manager.py:908` | 理解事件驱动出站 |
| 流式靠内部缓冲 + 固定间隔 | 消费队列时按积压合并，渠道实现 `send_delta` | `nanobot/channels/manager.py:933`；`nanobot/channels/base.py:129` | 讲"合并"不是"降频" |
| 渠道开关只有 `enabled` | 另有"可选依赖"：`plugins enable` 装包，`enabled` 才启动 | `nanobot/cli/commands.py:609` | 两个概念分开讲 |
| 配置字段用 snake_case | JSON 落盘以 camelCase 为准，snake_case 兼容读 | `nanobot/channels/feishu/manifest.py:10-11` | 文档统一 camelCase |
| 访问控制只有白名单 | 白名单之外还有 pairing 配对码 | `nanobot/channels/base.py:237-304`；`nanobot/channels/manager.py:328-345` | 讲清 `*`/白名单/配对三条路 |

## 附录 B：进阶理解

**1. 为什么用"子包 + 依赖无关 manifest"？** 渠道依赖很重（Telegram 要 `python-telegram-bot`，飞书要 `lark-oapi`，QQ 要 `qq-botpy`）。若 `import nanobot` 就全导进来，没装可选依赖的用户连 CLI 都起不来。manifest 只放纯数据，真正 import 推迟到 `load_channel_class()`（`nanobot/channels/plugin.py:74`），这是"延迟导入 + 清单驱动"。

**2. 为什么收消息用模板方法？** 若每个渠道自己 `publish_inbound`，鉴权、配对码、流式标记、`session_key_override` 会被复制 17 遍且容易漏。`_handle_message` 把标准动作收敛到基类，安全策略只有一个修改点。

**3. 为什么出站是 per-destination FIFO 而非全局 FIFO？** 全局 FIFO 会让一个慢平台（正在重试的邮件渠道）堵住所有平台。按 `(channel, chat_id)` 串行既保证同对话不乱序，又允许不同对话并行；实现是"任务尾巴" `_outbound_tails`（`nanobot/channels/manager.py:738-749`）。

**4. 为什么流式合并看队列积压而不是 300ms flush？** 固定定时器在模型慢时白等（延迟变差）、模型快时合并过头。"取队列里现有的连续 delta"自适应：消费快就逐条发，消费慢就批量发，不用调参。

**5. 为什么 `send_delta` 默认是空实现而不是 `NotImplementedError`？** 17 个平台不可能都支持流式。空实现让不支持的渠道"安全忽略"流式事件，出站流程不必写 `if` 分支；想支持的渠道重写并用 `supports_streaming`（`nanobot/channels/base.py:225`）声明能力。

**6. 为什么会话存储强制在 workspace 之外？** Agent 有 `write_file`/`exec`。若会话在工作区里，模型可能（无意或恶意）改掉历史、删掉别的会话。`JsonlSessionStore.__init__` 直接拒绝落在 workspace 内的会话根目录（`nanobot/session/manager.py:559-563`），把"记忆"和"可操作文件"物理分开。

## 附录 C：Python 语法加油站

### C.1 `asyncio.Queue`

```python
import asyncio
async def main():
    q = asyncio.Queue()    # maxsize=0 → 无上限
    await q.put("hello")   # 永不阻塞
    print(await q.get())   # 空队列时挂起，直到有数据
asyncio.run(main())
```

用在哪：`MessageBus` 两条队列（`nanobot/bus/queue.py:32-33`）。`await q.get()` 会让出 CPU；主循环外再包 `asyncio.wait_for(..., timeout=1.0)`（`nanobot/agent/loop.py:1269`），是为了每秒醒一次做空闲检查。

### C.2 `@abstractmethod`

```python
from abc import ABC, abstractmethod
class Base(ABC):
    @abstractmethod
    async def send(self, msg): ...
class Impl(Base):
    async def send(self, msg): print(msg)
# Base() → TypeError; Impl() ✅
```

用在哪：`BaseChannel.start/stop/send`（`nanobot/channels/base.py:74/86/91`）。抽象方法＝子类必须实现；`send_delta` 那种非抽象空实现＝可选钩子。

### C.3 `ContextVar`

```python
from contextvars import ContextVar
current = ContextVar("current", default=None)
async def handle():
    current.set("u1")      # 只影响当前协程上下文
    print(current.get())   # u1
```

用在哪：请求上下文（`spawn` 用它拿 `channel`/`chat_id`/`session_key`，`nanobot/agent/tools/spawn.py:90-95`）、Cron 防递归标记（`nanobot/agent/tools/cron.py:62`）。像协程自带的口袋；跨 `create_task` 要显式绑定。

### C.4 `Signal` 与事件循环

```python
import asyncio, signal
async def main():
    stop = asyncio.Event()
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, stop.set)   # 信号 → 协程事件
    await stop.wait()
```

用在哪：网关退出（`nanobot/cli/gateway_runtime.py:107-123`）：第一次 Ctrl+C 只 `set()` 走正常清理，再按一次才强制 cancel。信号处理里只设标志；Windows 支持有限，所以有 `signal.signal` 回退。

## 附录 D：原文勘误

| # | 原文位置 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| D1 | 9.5.1 / 9.8 | 字段写 `bot_token`；权限字段写 `allowed_users` | Telegram/Discord 用 **`token`**；权限字段是 **`allowFrom`** | `nanobot/channels/telegram/manifest.py:11`；`nanobot/channels/discord/manifest.py:9`；`nanobot/channels/base.py:241-243` |
| D2 | 9.5.2 | 飞书写 `app_id`/`app_secret`/`verification_token`/`encrypt_key` | 落盘是 **`appId`/`appSecret`/`verificationToken`/`encryptKey`** | `nanobot/channels/feishu/manifest.py:10-26`；`nanobot/channels/feishu/config.py:17-20` |
| D3 | 9.5.2 Step 3 | 说飞书要公网 Webhook、本地用 `ngrok` | 飞书走 **WebSocket 长连接，不需要公网 IP/webhook/ngrok** | `nanobot/channels/feishu/runtime.py:951-954`、`:1133-1134` |
| D4 | 9.5.1 / 9.8 | 启动写裸 `nanobot`，称"自动启动 Bot 轮询" | 渠道只在 **`nanobot gateway`** 启动 | `nanobot/cli/gateway_runtime.py:941`；`nanobot/cli/commands.py:472` |
| D5 | 9.3.2 | 有 `_stream_delta` 方法、`_buffer`/`_last_send_time`、固定 300ms | 真实方法 **`_coalesce_stream_deltas`**，按队列积压合并，无 300ms、无那两个字段 | `nanobot/channels/manager.py:933`、`:823-828` |
| D6 | 9.3.4 | 退避 1/2/4/8/16 秒，`max_retries=5` | 退避 **(1,2,4)**，`sendMaxRetries` 默认 **3**（含首次） | `nanobot/channels/manager.py:60`、`:1012`、`:1043`；`nanobot/config/schema.py:37` |
| D7 | 9.3.3 | 写了 `_handle_progress`/`_handle_tool_hint`，读 `channel_config["show_progress"]` | 无这两个方法；实际是 `ProgressEvent` + `_should_send_progress()` | `nanobot/channels/manager.py:803-811`、`:347-353` |
| D8 | 9.4.2 | `BaseChannel(name, config, bus)`、`on_message`/`_convert_to_inbound` | 构造是 **`(config, bus)`**，`name` 是类属性；收消息走 **`_handle_message`** | `nanobot/channels/base.py:29-46`、`:255` |
| D9 | 9.6.1 / 9.6.3 | 有 `build_session_key()`，override 放 `metadata` | 无此函数；override 是 **`InboundMessage.session_key_override` 字段** | `nanobot/bus/events.py:35-42` |
| D10 | 9.6.2 / 9.8 | 会话落 `sessions/<session_key>.jsonl`（工作区内、带冒号） | 落 **`<runtime>/sessions/<workspace-id>/<key>.jsonl`**，冒号转 `_`，在 workspace 之外 | `nanobot/session/manager.py:1027`、`:1033`、`:559-563` |
| D11 | 9.5 全节 | 从未提 `"enabled": true` | 除 `websocket` 外必须显式 **`"enabled": true`** | `nanobot/channels/plugin.py:38`；`nanobot/channels/contracts.py:104-109` |
| D12 | 9.2.5 | 说队列"满时自动等待"提供背压 | `asyncio.Queue()` **无 maxsize，入站不会因满阻塞**；限流在出站 | `nanobot/bus/queue.py:32-33`；`nanobot/channels/manager.py:61-62` |
| D13 | 9.5.5 | 微信写成"需第三方框架，较难" | 有 `weixin` 子包：HTTP 长轮询 + 扫码登录 | `nanobot/channels/weixin/manifest.py`；`nanobot/channels/weixin/runtime.py:979` |

---

> **下一步读什么**：继续读 [10 - 子Agent与定时任务](../10-subagent-and-cron/README.md)，看后台任务和定时任务如何复用本章的 `session_key` 与 MessageBus；想直接读源码，从 `nanobot/channels/base.py` 的 `_handle_message` 和 `nanobot/channels/manager.py` 的 `_dispatch_outbound_loop` 入手最省力。
