# 07 - 记忆系统实战

> 🎯 **本章目标**：看完你能一口气回答三个问题——nanobot 的记忆分成哪几个文件？每次对话结束后到底是谁、在什么时候、往哪个文件写了什么？为什么"压缩"这一步永远不碰 `MEMORY.md`？
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）
> **前置知识**：[06 - 安装与上手](../06-install-and-hands-on/README.md)
> **阅读时间**：约 2.5 小时

---

![记忆系统漫画](../../../comics/04-memory-system.png)

*从"两层记忆"升级为"四文件 + 两阶段"：回合里压缩写日志，定时任务 Dream 才提炼长期记忆*

## 目录

- [7.1 为什么 Agent 需要记忆](#71-为什么-agent-需要记忆)
- [7.2 记忆系统的挑战](#72-记忆系统的挑战)
- [7.3 四个文件与两个阶段](#73-四个文件与两个阶段)
- [7.4 MEMORY.md —— 长期记忆](#74-memorymd--长期记忆)
- [7.5 history.jsonl —— 追加日志](#75-historyjsonl--追加日志)
- [7.6 Consolidator 压缩机制](#76-consolidator-压缩机制)
- [7.7 短期记忆：Session 会话历史](#77-短期记忆session-会话历史)
- [7.8 记忆系统完整数据流](#78-记忆系统完整数据流)
- [7.9 记忆系统与其他框架对比](#79-记忆系统与其他框架对比)
- [7.10 实战练习](#710-实战练习)
- [7.11 面试高频题](#711-面试高频题)
- [7.12 本章小结](#712-本章小结)

**术语表（第一次出现就在这里查）**

| 词 | 一句话人话解释 |
|---|---|
| Session（会话） | 一段连续对话的持久化记录，键是 `{channel}:{chat_id}`，比如 `cli:default` |
| turn（回合） | 用户发一句 → Agent 思考/调工具 → 回一句，这一整个来回叫一个回合 |
| token | 模型计费与容量单位，约等于 0.7 个汉字；上下文窗口按 token 算 |
| cursor（游标） | 日志里的自增序号，用来表示"读到第几条了" |
| checkpoint（检查点） | 一段对话被压成的一小段摘要，用来替换原来的长历史 |
| 压缩（compact / consolidate） | 把长对话压成摘要，腾出上下文空间 |
| 巩固（consolidation / Dream） | 把零散日志提炼成长期事实，写进 `MEMORY.md` 等文件 |

---

## 7.1 为什么 Agent 需要记忆

想象一个没有记忆的助手：

```
对话 1:
你：我叫张三，做后端，最近在准备 Agent 方向面试
Agent：你好张三！……

对话 2（第二天，新会话）:
你：上次说的面试准备，帮我列个计划
Agent：抱歉，我们之前聊过吗？
```

纯 LLM 每次请求都是"失忆"的：它只看得见这次请求里塞进去的内容。所谓"记忆系统"，本质就是**回答一个问题：这次请求里，到底该塞什么进去？**

| 能力 | 无记忆 | 有记忆 |
|---|---|---|
| 跨会话延续 | 每次从零开始 | 记得上次聊到哪 |
| 用户偏好 | 每次重新问 | 直接按你的习惯来 |
| 长期项目 | 无法承接 | 能接着做 |
| 成本 | 每次都塞全量 | 只塞提炼过的事实 |

> 💡 关键认知：记忆系统不是"数据库"，而是**上下文预算管理器**。它是为了在有限 token 里塞进最有用的信息。

---

## 7.2 记忆系统的挑战

设计记忆时要同时满足三件互相打架的事：

```
        完整性（什么都记住）
             /        \
            /          \
       成本（省 token）—— 实时性（马上能查到）
```

1. **完整性 vs 成本**：全塞进上下文 → 贵，而且模型会被无关信息干扰（"lost in the middle"）。
2. **完整性 vs 实时性**：全存进向量库 → 每次都要检索，链路变长。
3. **成本 vs 实时性**：激进压缩省钱，但可能把关键约束压没了。

业界常见五种做法：

| 方案 | 优点 | 缺点 |
|---|---|---|
| 滑动窗口截断 | 实现最简单 | 早期信息直接消失 |
| 向量检索（RAG） | 语义召回强 | 需要向量库，多一套运维 |
| LLM 摘要压缩 | 保留语义、省钱 | 有信息损失，要处理好失败 |
| 知识图谱 | 结构化推理强 | 抽取和建模成本高 |
| **纯文件（nanobot）** | 零依赖、人类可读可改、git 可审计 | 只能全文/关键词检索 |

nanobot 选的是最后一种，并且把它拆成**两个阶段**来弥补"文件不擅长自动整理"的短板：先廉价地把对话压成日志，再由一个定时的、有 git 审计的"整理工"把日志提炼成长期事实。

---

## 7.3 四个文件与两个阶段

### 7.3.1 文件地图

```
<workspace>/                      # 默认 ~/.nanobot/workspace/
├── AGENTS.md                     # 项目级说明（可选，按项目目录读）
├── SOUL.md                       # Agent 自己的行为准则：身份、护栏、交互风格
├── USER.md                       # 用户画像：称呼、语言、偏好、习惯
├── memory/
│   ├── MEMORY.md                 # 长期事实：项目目标、架构决策、基础设施
│   ├── history.jsonl             # 追加日志：每次压缩产生的一条摘要
│   ├── .cursor                   # 写指针：history.jsonl 最后分配到的 cursor
│   └── .dream_cursor             # 读指针：Dream 已经处理到哪个 cursor
└── skills/                       # 技能目录（见第 08 章）
```

四个"记忆文件"由 `MemoryStore` 统一管理，构造函数里就把路径全定好了：`nanobot/agent/memory.py:58`（类），`:76`–`:82`（`memory_file` / `history_file` / `legacy_history_file` / `soul_file` / `user_file` / `_cursor_file` / `_dream_cursor_file`）。

### 7.3.2 谁写、何时写、是否注入

这是本章最重要的一张表，记住它就不会再搞混：

| 文件 | 谁写 | 什么时候写 | 是否进 system prompt |
|---|---|---|---|
| `MEMORY.md` | **Dream**（用文件工具改） | 每 2 小时一次的定时任务 | ✅ 每轮都注入 |
| `SOUL.md` | **Dream** | 同上 | ✅ 每轮都注入 |
| `USER.md` | **Dream** | 同上 | ✅ 每轮都注入 |
| `history.jsonl` | **Consolidator / MemoryArchiver** | 每回合上下文压力大时、空闲时、`/compact` | ❌ 不注入，按需 `grep` |
| `Session`（会话文件） | `AgentLoop._persist_turn` | 每回合结束 | ✅ 会被回放（截断后） |

注入侧的证据在 `nanobot/agent/context.py:127`–`:136`：`MEMORY.md` 被拼成 `# Memory / ## Long-term Memory`，`always` 技能被拼成 `# Active Skills`；而 `history.jsonl` 从头到尾没有出现在 `build_system_prompt` 里。

> 🧠 类比：`MEMORY.md` 是**贴在办公桌前的便签**（每次抬头都看得见），`history.jsonl` 是**档案柜里的流水账**（要查的时候自己翻）。

### 7.3.3 两个阶段

```
                 ┌──────────────────────── 阶段一：压缩（回合内/回合边界）────────────────────────┐
用户消息 → 回合 → │ 上下文快满了？ idle 超时了？ 用户敲了 /compact？                                  │
                 │        ↓ 是                                                                      │
                 │ Consolidator（memory.py:1072）                                                    │
                 │   └─ MemoryArchiver.archive（memory.py:758 / :826）                                │
                 │        ├─ 让模型写一段"工作记忆 checkpoint"                                        │
                 │        ├─ append 到 memory/history.jsonl（memory.py:993）   ← 唯一落盘动作        │
                 │        └─ 摘要写回 session，替换旧前缀（memory.py:1261）                            │
                 └──────────────────────────────────────────────────────────────────────────────────┘
                                                    │
                                      （历史日志越积越多）
                                                    ▼
                 ┌──────────────────── 阶段二：巩固（定时，默认每 2 小时）───────────────────────────┐
                 │ Dream（gateway 的 cron 系统任务，gateway_runtime.py:562）                          │
                 │   ├─ 读 .dream_cursor 之后的增量（memory.py:543 build_dream_prompt）              │
                 │   ├─ 用受限文件工具对 SOUL/USER/MEMORY 做最小改动（memory.py:575）                │
                 │   ├─ 成功 → 推进 .dream_cursor（memory.py:504）                                   │
                 │   └─ 无论成败 → GitStore 提交一次版本（gitstore.py:125）                          │
                 └──────────────────────────────────────────────────────────────────────────────────┘
```

一句话总结两阶段的分工：**阶段一负责"别把上下文撑爆"，阶段二负责"别让日志烂在硬盘里"。**

---

## 7.4 MEMORY.md —— 长期记忆

### 7.4.1 它是怎么进到模型眼前的

```python
# nanobot/agent/context.py:127-130（节选，保留原缩进）
if include_memory:
    memory = self.memory.read_memory()                       # 读 memory/MEMORY.md
    if memory and not self._is_template_content(memory, "memory/MEMORY.md"):
        parts.append(f"# Memory\n\n## Long-term Memory\n{memory}")
```

- `read_memory()` 只是 `read_text`，文件不存在就返回空串（`nanobot/agent/memory.py:229`，底层 `read_file` 在 `:99` 捕获 `FileNotFoundError`）。
- 如果内容还和内置模板一模一样，就**不注入**——避免把空模板当知识喂给模型。
- 因此 `MEMORY.md` 每轮都花 token：它必须短、必须是"事实"，不能是"叙事"。

### 7.4.2 它长什么样

Dream 的提示词要求写成**原子事实**，而不是"讨论过养猫"这种描述（`nanobot/templates/agent/dream.md`）：

```markdown
# Memory

## 项目
- 目标：把内部工单机器人迁到 nanobot，2026 Q4 上线
- 架构：nanobot gateway + 自建 MCP server（工单查询）
- 基础设施：Postgres 16 在 db.internal:5432

## 决策
- 2026-09-20：放弃向量库，改用 grep + history.jsonl
```

### 7.4.3 谁有权写它

这是本章的核心红线，三处证据：

1. **提示词层面**：内置 `identity.md` 里写着 "Only Dream memory-consolidation tasks may edit the profile and long-term memory files listed above."（`nanobot/templates/agent/identity.md`）。
2. **工具层面**：Dream 跑在一个**白名单工具注册表**里，只有 `read_file` / `edit_file` / `apply_patch` / `write_file`，且写权限被限制在 `skills/` 加三个可编辑文件（`nanobot/agent/memory.py:575`–`:616`，`editable_files = [self.memory_file, self.soul_file, self.user_file]` 在 `:590`）。
3. **代码层面**：`write_memory()` 确实存在（`nanobot/agent/memory.py:232`），但整包里只有 SDK 调用它（`nanobot/sdk/clients.py:167`）。**压缩链路一次都没调用过。**

### 7.4.4 为什么用 Markdown 而不是数据库

| 维度 | Markdown 文件 | 向量库 |
|---|---|---|
| 依赖 | 无 | 需要服务 + 索引 |
| 可读 | 人类直接读 | 需要查询接口 |
| 可改 | 用户手改立刻生效 | 要重新 embedding |
| 审计 | `git log` 直接看 | 难 |
| 检索 | 关键词 | 语义 |

代价也很明确：不支持语义检索、全量注入吃 token、不适合上万条记忆。nanobot 的定位是"个人助手"，这个取舍是划算的。

---

## 7.5 history.jsonl —— 追加日志

### 7.5.1 一行一条，带自增 cursor

```jsonl
{"cursor": 41, "timestamp": "2026-10-26 21:03", "content": "- [durable] 工单 MCP server 的 tool 名统一为 mcp_ticket_query", "session_key": "cli:default"}
{"cursor": 42, "timestamp": "2026-10-26 21:40", "content": "- [ephemeral] 正在改 context.py 的注入顺序"}
```

字段就四个（`nanobot/agent/memory.py:316`–`:319`）：`cursor`（必填，自增）、`timestamp`（`%Y-%m-%d %H:%M`，见 `:303`）、`content`（一段文本）、`session_key`（可选，谁产生的）。

### 7.5.2 两个游标，一个写一个读

| 文件 | 含义 | 谁维护 | 代码 |
|---|---|---|---|
| `memory/.cursor` | 最后**分配**出去的 cursor | `append_history` 每次写完就更新 | `:81`、`:321` |
| `memory/.dream_cursor` | Dream 已经**消费**到哪一条 | Dream 成功后推进 | `:82`、`:498`、`:504` |

为什么要有两个？因为"写日志"和"消化日志"是**两个独立节奏**：写日志几乎每回合都可能发生，消化是每 2 小时一次。两个指针分开，Dream 才能稳定地"只读增量、读完再前移"，天然是**至少一次（at-least-once）投递**：

```python
# nanobot/agent/memory.py:399-401（读增量的唯一入口）
def read_unprocessed_history(self, since_cursor: int) -> list[dict[str, Any]]:
    """Return history entries with a valid cursor > *since_cursor*."""
    return [e for e, c in self._iter_valid_entries() if c > since_cursor]
```

### 7.5.3 并发安全与容量治理

- **追加是原子的**：`_append_lock` 把"分配 cursor + 写一行"锁在一起（`:87`、`:308`），否则两个协程可能算出同一个 cursor。
- **有上限**：默认保留 1000 条（`_DEFAULT_MAX_HISTORY`，`:61`），`compact_history()` 在 Dream 结束后调用（`gateway_runtime.py:617`），且**不会删掉还没被 Dream 处理的行**（`:403`–`:431`，关键在 `first_unprocessed` 那段）。
- **有硬上限**：单条摘要最多 64000 字符（`_HISTORY_ENTRY_HARD_CAP`，`:751`），raw 兜底最多 16000 字符（`_RAW_ARCHIVE_MAX_CHARS`，`:750`）。
- **模型不能直接写**：`exec` 工具明确拦截对 `history.jsonl` / `.dream_cursor` 的重定向和 `sed -i`（`nanobot/agent/tools/shell.py:226`–`:232`），注释里写得很直白——"直接写会破坏 cursor 格式并让 `/dream` 崩掉"。

### 7.5.4 HISTORY.md 只是"搬迁来源"

旧版本的记忆是一个 Markdown 文件 `memory/HISTORY.md`。v0.3.5 里它只在一个地方出现：**首次启动时的迁移**（`nanobot/agent/memory.py:106`–`:144`）：

```python
# nanobot/agent/memory.py:112-115（节选）
if not self.legacy_history_file.exists():
    return
if self.history_file.exists() and self.history_file.stat().st_size > 0:
    return
```

迁移细节值得记一笔：迁移完把 `HISTORY.md` 改名成 `HISTORY.md.bak`（`:137`），并且**把 `.dream_cursor` 直接设成最后一条**（`:132`–`:134`），注释解释了原因——"升级时不要把用户的全部历史重新灌进 Dream"。这是个很工程化的细节：**迁移要做幂等，更要做"别惊动下游"**。

---

## 7.6 Consolidator 压缩机制

`Consolidator`（`nanobot/agent/memory.py:1072`）是压缩的**编排者**：它管锁、管预算、管把摘要写回会话；真正调模型、真正写 `history.jsonl` 的是它内部的 `MemoryArchiver`（`:758`）。面试时把这两个名字说清楚，比笼统说"压缩器"专业得多。

### 7.6.1 四个触发条件

| 触发 | 条件 | 入口 |
|---|---|---|
| ① token 预算 | 估算输入 ≥ `context_window − max_tokens − 1024` | `context_governance.py:384` → `:521` |
| ② 空闲超时 | 会话静默超过 `session_ttl_minutes`（默认 **15** 分钟） | `autocompact.py:68` → `memory.py:1222` |
| ③ 手动 | 用户敲 `/compact` | `command/builtin.py:356` |
| ④ 供应商原生压缩 | provider 自己做了压缩，本地补一份可移植摘要 | `memory.py:1144` |

细节：

- ① 的预算常量是 `_SAFETY_BUFFER = 1024`（`memory.py:1075`），为了吸收 tokenizer 估算误差；`input_budget` 的通用版本在 `context_governance.py:693`。**注意预算不是 `context_window_tokens` 本身**，要减掉输出预留和安全垫。
- ① 的调用链：`AgentLoop` 把 `consolidate_history` 作为回调传给 runner（`nanobot/agent/loop.py:1184`–`:1193`），runner 里的 `ContextGovernor.prepare_request` 发现压力后才回调它。
- ② 的阈值与扫描间隔来自配置：`session_ttl_minutes` 默认 15、`idle_compact_check_interval_seconds` 默认 60（`nanobot/config/schema.py:147`、`:153`），主循环在 `asyncio.wait_for` 超时时顺手扫一次（`loop.py:1249`–`:1259`）。
- ③ 之外还有一个 `/new`：它会把旧会话整体归档再开新会话（`command/builtin.py:313`–`:340`）。

### 7.6.2 一次压缩做了什么

```
① Construct：取 session.messages[last_archived:archive_end]
   （memory.py:1005-1008，跳过 _command 与 summary checkpoint）
② Prompt  ：agent/consolidator_archive.md（"给下一轮用的工作交接"）
③ Call    ：provider.chat_stream_with_retry（:913）
④ Append  ：self.store.append_history(summary, session_key=...)  ← 唯一落盘（:993）
⑤ Checkpoint：session.commit_summary_checkpoint(summary, insert_at=archive_end)
   （:1261 → session/manager.py:323）
```

第 ⑤ 步很关键：压缩**不是**把旧消息删掉，而是在 `archive_end` 位置插一条**隐藏的续接标记**，同时把 `last_archived` 水位推到 `archive_end`：

```python
# nanobot/session/manager.py:332-342（节选）
self.messages.insert(boundary, {
    "role": "user",
    "content": SUMMARY_CONTINUATION_TEXT,     # "Continue the active task from ..."
    HIDDEN_HISTORY_META: True,                # 回放时隐藏
    ...
})
self.metadata["_last_summary"] = {"text": summary, "last_active": ...}
self.last_archived = boundary
```

之后 `get_history()` 只从 `last_archived` 之后开始回放（`nanobot/session/manager.py:358`），所以旧消息**还在文件里**（可审计、可恢复），但**不再进 prompt**。这比"截断即删除"安全得多。

### 7.6.3 两级容错：2 次尝试之后直接 raw

这是旧文档错得最厉害的一处，我们按代码重讲：

```python
# nanobot/agent/memory.py:909-930（节选）
response: LLMResponse | None = None
for attempt in range(2):                    # ← 只尝试两次
    try:
        with llm_usage_source("dream"):
            response = await runtime.provider.chat_stream_with_retry(...)
    except Exception:
        phase = "provider call" if attempt == 0 else "tool-call recovery"
        logger.warning("Memory archive {} failed, raw-dumping to history", phase)
        return raw_fallback()               # ← 直接兜底，不再重试
    if response.should_execute_tools is not True or attempt == 1:
        break
    ...  # 把模型的 tool_call 塞回去，附一句"归档不执行工具"，再试一次
```

- **第 1 次**：正常请求。如果模型不调工具、返回了文本，直接成功。
- **第 2 次**：只有在模型"非要调工具"时才发生。代码把它请求的工具调用补上假结果 `_ARCHIVE_TOOL_RESULT`（`:752`，内容是"归档阶段不执行工具，请现在直接返回 checkpoint"），再问一次。
- **失败即兜底**：任何异常、任何不满意，都走 `raw_fallback()`（`:843`），最终由 `raw_archive()`（`:665`）写出一条 `[RAW] N messages` 的原文快照（`:689`）。

哪些情况会触发 raw 兜底（都在 `:975`–`:991`）：

| 情况 | 代码位置 |
|---|---|
| provider 抛异常 | `:922` |
| `finish_reason` 是 `error` / `length` | `:975` |
| 第二次仍然返回 tool_calls | `:981` |
| 返回空内容 | `:985` |
| 内容被 `strip_think` 清成空 | `:989` |
| 输入本身塞不进预算 | `:899` |

> ✅ **正确说法**：nanobot 的压缩容错是**两级**——"2 次尝试（含一次工具恢复）"→"raw 原文兜底"。代码里没有任何 `tool_choice` 强制（`tool_choice` 只出现在 `nanobot/providers/*.py` 的传输层参数里），也没有"连续 3 次失败"这回事。

### 7.6.4 为什么压缩不写 MEMORY.md

初学者最容易问："既然都调模型总结了，顺手更新 `MEMORY.md` 不是更省事？" nanobot 故意不这么干，理由有五条，每条都能在代码里找到落点：

1. **频率不匹配**。压缩可能每回合发生一次，而长期记忆的写入需要"跨会话视角"。让高频路径写低频文件，等于让流水账污染档案。
2. **内容粒度不匹配**。压缩产出的是"[ephemeral] 正在改注入顺序"这种**会话工作状态**（`consolidator_archive.md` 明确要求打 `[ephemeral]`），而 `MEMORY.md` 要的是"工单系统迁 nanobot，Q4 上线"这种**稳定事实**。
3. **审计不可用**。`MEMORY.md` 的每次变化都必须能 `git log` 追溯、能用 `/dream-restore` 回滚（`command/builtin.py:785`）。压缩路径没有提交动作，写了就查不到是谁改的。
4. **权限面要小**。只有 Dream 的受限工具集能写这三个文件（`memory.py:575`–`:616`）。压缩链路的写入面被压到**只有一个 `append_history`**（`:993`），这让"谁可能改坏我的长期记忆"这个问题有一个极短的答案。
5. **失败要可恢复**。压缩失败时它必须能"什么都不改地退回去"；如果它顺手改过 `MEMORY.md`，"退回去"就不可能了。raw 兜底之所以敢直接 dump 原文，正是因为它只碰日志。

一句话：**压缩是"降噪"，Dream 才是"下结论"。** 把两者混在一起，就同时失去了速度和可信度。

---

## 7.7 短期记忆：Session 会话历史

### 7.7.1 存在哪、叫什么

会话**不在 workspace 里**，而在实例数据目录下，一个 workspace 一个命名空间：

```
~/.nanobot/sessions/<workspace-id>/<base64url(session_key)>.jsonl
例：~/.nanobot/sessions/9f3c…/Y2xpOmRlZmF1bHQ.jsonl      # key = "cli:default"
```

- 目录选择在 `nanobot/session/manager.py:551`–`:563`；如果算出来的目录落在 workspace 内部，会**直接抛错**（`:559`），注释说得很清楚：会话存储必须在 agent workspace 之外。
- 文件名是 key 的 **base64url（去掉 `=`）** 编码（`:1006`、`:1026`）。原因：`cli:default` 里的冒号在 Windows 上是非法文件名字符，base64 还顺带解决了大小写不敏感文件系统的碰撞。
- 为什么要放到 workspace 外：workspace 里的文件是**模型能用工具读写**的，会话不该被模型（或 Dream 的 git 提交）碰到。

### 7.7.2 文件格式：第一行是元数据

```jsonl
{"_type":"metadata","key":"cli:default","created_at":"...","updated_at":"...","metadata":{...},"last_archived":42,"last_consolidated":42}
{"_type":"provider_state","state":{...}}          （可选，只有存在 provider 续接状态时才有）
{"role":"user","content":"你好","timestamp":"..."}
{"role":"assistant","content":"你好！","timestamp":"..."}
```

写入逻辑在 `nanobot/session/manager.py:1314`–`:1339`：先写 metadata 行，再（可选）写 provider state 行，然后一条消息一行；`os.replace` 原子替换（`:1344`），文件权限 `0o600`（`:610`）。

### 7.7.3 `last_archived` 水位线

`Session` 里保存的是 `last_consolidated`，但通过 `last_archived` 这个 property 暴露（`nanobot/session/manager.py:285`、`:303`–`:310`），注释解释了原因：**保留旧字段名，等持久化数据和 SDK 调用方都迁完再改**。这是"兼容性优先"的典型写法。

- `last_archived = N` 的含义：前 N 条已经被压缩过，不再回放进 prompt。
- `_persist_turn` 在每回合结束时保存会话（`nanobot/agent/loop.py:2014`、`:2037`），压缩产生的 checkpoint 也在这一步落盘。
- Dream 自己也是一个会话（key 形如 `dream:20261026-100000`，`memory.py:699`），但只保留最近 10 个（`prune_dream_sessions`，`:722`），避免 Dream 把会话目录撑爆。

---

## 7.8 记忆系统完整数据流

```
用户消息
   │
   ├─ 拼 system prompt：SOUL.md + USER.md + MEMORY.md + 技能摘要
   │   （context.py:101 build_system_prompt）
   ├─ 拼 messages：session.get_history()，从 last_archived 起
   │   （context.py:231 build_messages）
   │
   ├── 估算 ≤ 预算 ──────────────► 正常请求模型
   └── 估算 > 预算 ──► ContextGovernor 压缩
                         ├─ Consolidator.summarize_transcript
                         │    └─ MemoryArchiver.archive
                         │         ├─ history.jsonl ← append（memory.py:993）
                         │         └─ session 插入 checkpoint（:1261）
                         └─ 用 checkpoint 重建请求，继续本回合
   │
   ▼
回合结束：AgentLoop._persist_turn（loop.py:2014）→ 保存 session
   │
   ▼
（每 2 小时）Dream：读 .dream_cursor 之后 → 改 MEMORY/SOUL/USER → 提交 git → 推进 cursor
```

---

## 7.9 记忆系统与其他框架对比

| 框架/方案 | 存储 | 检索 | 巩固方式 | 透明度 |
|---|---|---|---|---|
| **nanobot** | Markdown + JSONL | 全量注入 + `grep` | 两阶段（压缩 + Dream） | 高 |
| LangChain / LlamaIndex | 向量库 | 语义相似度 | 手动/自定义 | 低 |
| MemGPT | 分层"内存页" | 分页换入换出 | LLM 自编辑 | 中 |
| AutoGPT | JSON 文件 | 向量 | 无 | 中 |
| CrewAI | 进程内共享 | 直接访问 | 无 | 低 |

**nanobot 方案的甜点区**：单用户、记忆量在几百条、要求"我能手改、能 diff、能回滚"。
**不适用的场景**：上千页文档的知识库、多租户共享知识、需要多跳推理。

对比 MemGPT 特别有面试价值：MemGPT 把上下文当"内存"、把外部存储当"磁盘"，靠模型自己决定换页；nanobot 把"换页"拆成**两个不同频率、不同权限的进程**（压缩高频低权限，Dream 低频高权限）。前者灵活但不可预测，后者笨但可审计。

---

## 7.10 实战练习

### 练习 1：看一眼真实的记忆目录

```bash
nanobot onboard --wizard        # 生成 ~/.nanobot/config.json + workspace
ls -la ~/.nanobot/workspace ~/.nanobot/workspace/memory
cat ~/.nanobot/workspace/memory/MEMORY.md
cat ~/.nanobot/workspace/memory/.cursor          # 可能还不存在
```

### 练习 2：手动触发一次 Dream

```bash
nanobot -m "记住：我的项目用 Postgres 16，部署在 k8s 上"   # 先制造一点对话
# 在交互式会话里：
/dream                          # 立刻跑一次巩固（command/builtin.py:451）
/dream-log                      # 看这次改了什么（:722）
/dream-restore                  # 看可回滚的版本列表（:785）
git -C ~/.nanobot/workspace log --oneline
```

观察点：`MEMORY.md` 应该被**局部修改**（只加一两行），而不是整篇重写。

### 练习 3：把空闲阈值调到 0 分钟，看压缩怎么触发

```json
{ "agents": { "defaults": { "idleCompactAfterMinutes": 1 } } }
```

聊几轮后静置 1 分钟，再 `wc -l ~/.nanobot/workspace/memory/history.jsonl`——应该多了一条（或多条）。然后把值改回 `0`（关闭空闲压缩）。

### 练习 4：读一条日志，反推它来自哪次会话

```bash
tail -n 3 ~/.nanobot/workspace/memory/history.jsonl | python -m json.tool
```

每条都有 `cursor`、`timestamp`、`content`，压缩产生的还带 `session_key`。

### 练习 5（危险但值得）：破坏一行，看它怎么自愈

把 `history.jsonl` 最后一行的 `"cursor"` 改成 `"abc"`，再跑 `/dream`。你会看到一条 warning（"history.jsonl contains an invalid cursor ... dropping it"，`memory.py:349`）并且**程序不崩**——非法条目被跳过，cursor 仍能从 `.cursor` 或全量扫描里恢复。

---

## 7.11 面试高频题

### 题目 1：nanobot 的记忆是怎么分层的？

> "它把记忆分成**四个持久文件 + 一个会话库**。`SOUL.md`、`USER.md`、`memory/MEMORY.md` 是长期记忆，每轮都注入 system prompt；`memory/history.jsonl` 是追加日志，不进 prompt，需要时模型自己 `grep`；会话则是每个 session 一个 JSONL 文件，存在实例目录下、workspace 之外，按 `last_archived` 水位回放。
>
> 关键在于它不是一开始就让模型"自由写记忆"，而是**两个阶段**：回合里由 `Consolidator` 把上下文压成 checkpoint 追加进日志；每 2 小时由 Dream 读增量、用受限文件工具对三个长期文件做最小改动，并用 GitStore 提交版本。压缩只写日志，Dream 才写长期记忆。"

### 题目 2：压缩的触发条件是什么？

> "四条：① token 预算——估算输入超过 `context_window − max_tokens − 1024`，由 `ContextGovernor` 在准备请求时判定；② 空闲——会话静默超过 `session_ttl_minutes`（默认 15 分钟），由 `AutoCompact` 扫描；③ 手动 `/compact`；④ provider 自己做了原生压缩，本地补一份可移植摘要。
>
> 压缩后的旧消息不会被删，而是在边界处插入一条隐藏的续接标记，把 `last_archived` 推到边界，之后回放从那里开始。"

### 题目 3：压缩失败怎么办？

> "两级：先试 2 次——第一次正常请求，如果模型非要调工具，就把工具调用补上'归档不执行工具'的结果再问一次；两次之后任何不满意（异常、finish_reason 是 error/length、空内容、仍然返回 tool_calls）都直接走 `raw_fallback`，把原文按 `[RAW] N messages` 追加进 `history.jsonl`，单条上限 16000 字符。
>
> 也就是说它宁可存一段难看的原文，也不让记忆丢失；而且它只碰日志，不动长期记忆，所以兜底是**可恢复**的。"

### 题目 4：为什么要两个 cursor？

> "`.cursor` 是写入侧的分配指针，`.dream_cursor` 是消费侧的读指针。写日志每回合都可能发生，巩固每 2 小时一次，两者频率差几个数量级。分开之后 Dream 只需要读 `cursor > .dream_cursor` 的增量，读成功才前移，天然是 at-least-once；失败时指针不动，下一轮会重试同一批数据，不会丢。"

### 题目 5：为什么不用向量数据库？

> "取舍问题。文件方案的收益是零依赖、人类可读可改、`git log` 可审计、`/dream-restore` 可回滚；代价是没有语义检索、全量注入吃 token、规模上限低。
>
> 对'个人助手、几百条记忆'这个定位是划算的。如果要做大规模知识库，我会在保留文件是**唯一真源**的前提下，加一层可重建的索引（embedding 或 FTS），索引坏了直接重建，不影响真源。"

---

## 7.12 本章小结

```
nanobot 记忆 = 4 个持久文件 + 1 个会话库 + 2 个阶段

长期记忆（每轮注入）
├── SOUL.md          行为准则        ┐
├── USER.md          用户画像        ├─ 只允许 Dream 编辑
└── memory/MEMORY.md 长期事实        ┘

追加日志（不注入）
├── memory/history.jsonl  {"cursor","timestamp","content"[,"session_key"]}
├── memory/.cursor        写入指针
└── memory/.dream_cursor  消费指针

会话（短期）
├── 位置：<config-dir>/sessions/<workspace-id>/<base64url>.jsonl
├── 格式：metadata 行 + 消息行
└── 水位：last_archived（别名，真字段是 last_consolidated）

阶段一 Consolidator（压缩）
├── 触发：token 预算 / 空闲 15min / /compact / provider 原生
├── 产物：history.jsonl 追加 + session checkpoint
└── 容错：2 次尝试 → raw_fallback（16000 字符上限）

阶段二 Dream（巩固）
├── 触发：cron 系统任务，默认每 2 小时（dream.interval_h=2）
├── 输入：.dream_cursor 之后的增量（每批最多 20 条）
├── 动作：最小改动 SOUL/USER/MEMORY，成功后推进 cursor
└── 审计：GitStore 提交，可 /dream-log、/dream-restore
```

### 面试记忆清单

| 考点 | 一句话回答 |
|---|---|
| 记忆架构 | 4 文件 + 会话库，压缩与巩固两阶段 |
| MEMORY.md | 长期事实，每轮注入，只有 Dream 能改 |
| history.jsonl | 追加日志，不注入，靠 cursor 增量消费 |
| `.cursor` / `.dream_cursor` | 写指针 / 读指针，实现 at-least-once |
| 压缩触发 | 预算、空闲 15min、`/compact`、provider 原生 |
| 压缩容错 | 2 次尝试 → raw 原文兜底（两级） |
| 为什么压缩不写 MEMORY.md | 频率、粒度、审计、权限、可恢复性 |
| 会话存哪 | 实例目录下、workspace 之外，base64url 文件名 |
| 版本化 | GitStore + `/dream-log` + `/dream-restore` |

---

> **下一步读什么**：先读 `nanobot/agent/memory.py` 的 `MemoryStore`（:58）→ `MemoryArchiver.archive`（:826）→ `Consolidator`（:1072）这条线；然后读第 **[08 - 技能与工具](../08-skills-and-tools/README.md)**——Dream 之所以能"改文件"，靠的正是第 08 章的 Tool 体系。

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| `MEMORY.md` + `HISTORY.md` 双层 | `MEMORY.md` + `history.jsonl`，另有 `SOUL.md`/`USER.md` | `memory.py:76`–`:82` | 按"四文件 + 两阶段"重讲 |
| `MemoryConsolidator` 压缩并重写 `MEMORY.md` | `Consolidator` 编排、`MemoryArchiver` 落盘，只写 `history.jsonl`；`MEMORY.md` 由 Dream 编辑 | `memory.py:993`、`:575` | 把"压缩"和"巩固"分开讲 |
| `save_memory` 虚拟工具 + `tool_choice` 强制 | 无此工具；归档是一次普通 chat 调用 | 全包 `save_memory` 命中 0 次 | 删掉整节 |
| 会话存 `<workspace>/sessions/<key>.jsonl` | 存 `<config-dir>/sessions/<workspace-id>/<base64url>.jsonl`，且**禁止**放在 workspace 内 | `manager.py:551`–`:563`、`:1006`、`:1026` | 重写路径与命名 |
| 压缩触发 = `estimate > context_window_tokens` | 预算是 `context_window − max_tokens − 1024`；另有空闲/手动/provider 原生三条 | `memory.py:1075`、`:1199`；`context_governance.py:384` | 改成"四个触发条件" |
| 无 Dream / 无 git | Dream 是 cron 系统任务，默认每 2 小时；文件由 GitStore 版本化 | `config/schema.py:53`–`:80`；`gitstore.py:47` | 补一整节 |

## 附录 B：进阶理解（为什么这样设计）

1. **把"高频写"和"低频决策"拆开，是整个设计的灵魂。** 压缩链路权限极小（只能 append 日志），Dream 权限大但频率低、有审计。这样"模型胡说八道"的最坏后果被限制在"多一条烂日志"，而不是"长期记忆被改烂"。
2. **追加日志（WAL 思路）比"就地更新"更抗错。** `history.jsonl` 只追加、cursor 单调递增、读取方自己维护读指针——这是数据库 WAL 的思路搬到文件上：写者不用知道读者进度，读者不用锁写者。
3. **`[ephemeral]`/`[durable]`/`[permanent]` 这些标记是"给下一轮模型看的元数据"。** 它们不进最终记忆（Dream 要求 strip），但能让整理阶段的模型知道"这条两周后可能就没用了"。用文本标记代替 schema 字段，好处是模型天然理解。
4. **checkpoint 代替删除，是"可恢复性"的底线。** 旧消息留在文件里，只是不回放；`/dream-restore`、git、`HISTORY.md.bak` 都是同一个思路的不同层次。
5. **迁移时把 Dream 指针设到队尾，而不是队首。** 这是"升级不该产生副作用"的具体体现：新版本首次启动不应该突然把用户一年的历史灌给模型跑一遍。
6. **对比 MemGPT**：MemGPT 让模型自己管理分页，灵活但行为不可预测、成本不可控；nanobot 用两个固定节奏的后台流程换取可预测性与可审计性。取舍点在于"你是要 agent 自主，还是要运维能看懂"。

## 附录 C：Python 语法加油站

### C.1 `@dataclass`：自动生成 `__init__` 的类

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Session:
    key: str                                        # 必填
    messages: list[dict] = field(default_factory=list)   # 可变默认值必须用 factory
    updated_at: datetime = field(default_factory=datetime.now)
```

- **在本项目哪里用**：`nanobot/session/manager.py:275`–`:287`（`Session`）、`:29`（`CommitInfo`）、`nanobot/agent/tools/context.py:78`（`ToolContext`）。
- **为什么用 `default_factory`**：`messages: list = []` 会让所有实例共享同一个列表——Python 里最经典的坑。
- **读法建议**：看到 `@dataclass` 就先去数字段，字段名通常就是持久化 JSON 的键名。

### C.2 `TypedDict`：给 dict 加"类型注解"，运行时仍是 dict

```python
from typing import TypedDict

class SessionPayload(TypedDict):
    key: str
    created_at: str
    updated_at: str
    metadata: dict
    messages: list
```

- **在本项目哪里用**：`nanobot/session/manager.py:485`（`SessionPayload`）、`nanobot/session/summary.py:24`（`SessionSummary`）。
- **和 dataclass 的区别**：`TypedDict` 只是给静态检查器看的，运行时对象还是普通 `dict`，所以能直接 `json.dumps`；dataclass 是真类，要自己转换。
- **读法建议**：项目里凡是"要落盘成 JSON"的结构，多半是 `TypedDict`。

### C.3 `Protocol`：结构化接口（"长得像就算实现"）

```python
from typing import Protocol

class SessionStore(Protocol):
    def load(self, key: str) -> "Session | None": ...
    def save(self, session: "Session", *, fsync: bool = False) -> None: ...

class JsonlSessionStore:      # 不需要显式继承 SessionStore
    def load(self, key: str): ...
    def save(self, session, *, fsync: bool = False): ...
```

- **在本项目哪里用**：`nanobot/session/manager.py:526`（`SessionStore`），实现是 `:548` 的 `JsonlSessionStore`；`SessionManager.__init__` 允许注入 `store=`（`:1651`）。
- **好处**：想换存储后端（比如换成 sqlite）不用改 `SessionManager`，只要新类方法签名对得上。
- **读法建议**：看到 `Protocol` 就去找"谁实现了它"，通常在同目录或注入点。

### C.4 `@property` + setter：给旧字段名留后路

```python
class Session:
    last_consolidated: int = 0

    @property
    def last_archived(self) -> int:
        return self.last_consolidated

    @last_archived.setter
    def last_archived(self, value: int) -> None:
        self.last_consolidated = value
```

- **在本项目哪里用**：`nanobot/session/manager.py:303`–`:310`；注释写着"保留旧存储名，等持久化数据和 SDK 调用方迁移完"。
- **好处**：读代码的人用新名字，磁盘上的老字段名继续能用，不需要一次性数据迁移。
- **读法建议**：看到 property 返回另一个字段，八成是**兼容层**，别当成新状态。

### C.5 `with suppress(...)`：只忽略指定的异常

```python
from contextlib import suppress

with suppress(FileNotFoundError):
    with open(self.history_file, "r", encoding="utf-8") as f:
        ...          # 文件不存在就静默跳过
```

- **在本项目哪里用**：`nanobot/agent/memory.py:438`（读日志）、`:500`（读 cursor）、`:486`（Windows 上 fsync 目录会 `PermissionError`）。
- **注意**：`suppress` 只吞列出来的异常，别的照样抛；它比 `try/except: pass` 更安全，因为"忽略哪些"是明写的。
- **读法建议**：日志/游标/缓存这类"读不到就当空"的地方，几乎都是 `suppress`。

## 附录 D：原文勘误

| # | 旧文档原文 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| 1 | "第二层是 `HISTORY.md` 历史时间线，仅追加" | 文件名与格式都变了 | `memory/history.jsonl`，一行一个 JSON 对象，带自增 `cursor` | `nanobot/agent/memory.py:77`、`:316` |
| 2 | "`save_memory` 是一个虚拟工具，通过 `tool_choice` 强制调用" | 全包没有 `save_memory`，压缩路径也没有 `tool_choice` | 归档是一次普通模型调用；`tool_choice` 只出现在 `providers/` 的传输层 | `memory.py:909`–`:930`；全包 `save_memory` 命中 0 次 |
| 3 | "`MemoryConsolidator` 类" | 类名不存在 | `Consolidator`（编排）+ `MemoryArchiver`（落盘）+ `Dream`（巩固） | `memory.py:1072`、`:758`、`gateway_runtime.py:562` |
| 4 | "三级容错：tool_choice → auto → 连续 3 次失败后 Raw Archive" | 机制、次数都不对 | **两级**：`for attempt in range(2)`（含一次工具恢复）→ `raw_fallback()`；单条上限 16000 字符 | `memory.py:910`、`:843`、`:750` |
| 5 | "压缩后 `MEMORY.md` 覆盖为 `memory_update`" | 压缩根本不写 `MEMORY.md` | 压缩只 `append_history`；`MEMORY.md` 仅 Dream 可编辑 | `memory.py:993`、`:590`；`templates/agent/identity.md` |
| 6 | "Session 存 `<workspace>/sessions/<session_key>.jsonl`" | 路径、命名都不对 | `<config-dir>/sessions/<workspace-id>/<base64url(key)>.jsonl`，且禁止放在 workspace 内 | `session/manager.py:551`–`:563`、`:1006`、`:1026` |
| 7 | "`estimate_prompt_tokens_chain > context_window_tokens` 就触发压缩" | 少了输出预留与安全垫 | 预算是 `context_window − max_tokens − 1024` | `memory.py:1075`、`:1199`；`context_governance.py:693` |
| 8 | "会话每行是一个对话回合" | 第一行是元数据记录，不是消息 | 首行 `{"_type":"metadata",...,"last_archived":N}`，之后才是消息行 | `session/manager.py:1320`–`:1339` |
