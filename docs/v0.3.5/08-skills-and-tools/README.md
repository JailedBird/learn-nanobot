# 08 - 技能与工具

> 🎯 **本章目标**：分清"给模型看的说明书"（Skill）和"给模型调的函数"（Tool）；能照着模板写出一个 30 行的自定义 Tool 和一个最小 `SKILL.md`；说清工具从被发现到被执行的完整生命周期。
> 📦 **对应版本**：nanobot v0.3.5（tag `v0.3.5`，2026-09-16）
> **前置知识**：[07 - 记忆系统实战](../07-memory-system/README.md)
> **阅读时间**：约 2.5 小时

---

## 目录

- [8.1 Skills 系统概述](#81-skills-系统概述)
- [8.2 SKILL.md 格式规范](#82-skillmd-格式规范)
- [8.3 YAML Frontmatter 字段详解](#83-yaml-frontmatter-字段详解)
- [8.4 技能发现机制](#84-技能发现机制)
- [8.5 渐进披露（Progressive Disclosure）](#85-渐进披露progressive-disclosure)
- [8.6 内置技能列表](#86-内置技能列表)
- [8.7 内置工具完整列表](#87-内置工具完整列表)
- [8.8 ToolRegistry 统一注册与执行机制](#88-toolregistry-统一注册与执行机制)
- [8.9 MCP 工具集成](#89-mcp-工具集成)
- [8.10 自定义 Tool 与 SKILL.md 实战](#810-自定义-tool-与-skillmd-实战)
- [8.11 工具安全机制](#811-工具安全机制)
- [8.12 面试高频题](#812-面试高频题)
- [8.13 本章小结](#813-本章小结)

**术语表**

| 词 | 一句话人话解释 |
|---|---|
| Tool（工具） | 有名字、有参数 schema、能被模型"点名调用"的 Python 代码 |
| Skill（技能） | 一个 `SKILL.md`，用自然语言教模型"这类任务该怎么做" |
| frontmatter | Markdown 顶部 `---` 包起来的 YAML 元数据 |
| 渐进披露 | 先只给模型看技能名和一句话描述，需要时再读全文 |
| scope（作用域） | 工具的可见范围：`core` / `subagent` / `memory` |
| `ToolRegistry` | 工具登记处：注册、生成 schema、执行、统一报错 |

---

## 8.1 Skills 系统概述

### 8.1.1 一句话区别

> **Tool 是给模型"调"的函数；Skill 是给模型"读"的说明书。**

模型决定读文件时输出 `read_file(path="a.py")`——这是 Tool。模型不确定"怎么给一个仓库做 code review"时，先 `read_file("skills/github/SKILL.md")` 把说明书读进上下文——这是 Skill。**Skill 不执行任何代码，它只往上下文里塞知识。**

### 8.1.2 对比表

| 维度 | Tool | Skill |
|---|---|---|
| 载体 | `nanobot/agent/tools/*.py` 里的 `Tool` 子类 | `SKILL.md`（Markdown + YAML） |
| 谁"执行" | Python 代码，确定性 | 模型自己读、自己照做 |
| 怎么进上下文 | `to_schema()` → 请求的 `tools` 字段 | 摘要进 system prompt，全文靠 `read_file` |
| token 成本 | 每轮都发 schema | 摘要每轮发；全文按需 |
| 谁适合写 | 会 Python、要确定性 | 懂业务、要教流程 |

### 8.1.3 它们怎么配合

```
Skill: github（说明书）            它会去调用这些 Tool：
├── 知道创建 PR 的正确顺序           ├── exec       （跑 gh pr create / git push）
└── 知道要先看 diff 再写描述         └── read_file  （看代码）
```

一个 Skill 可以教模型组合任意多个 Tool，一个 Tool 也可以被任意多个 Skill 使用。**Skill 是知识层，Tool 是能力层，两者正交。**

> 💡 记忆口诀：Tool 决定"能不能做"，Skill 决定"做得对不对"。

---

## 8.2 SKILL.md 格式规范

### 8.2.1 目录结构与最小文件

```
<workspace>/skills/github/      # 目录名 = 技能名
├── SKILL.md                    # 必需：说明书本体
├── references/pr-workflow.md   # 可选：更细的参考
└── scripts/create-pr.sh        # 可选：辅助脚本
```

```markdown
---
name: line-count
description: 统计文本文件的行数，用于快速评估文件规模。
---

# Line Count

## 何时使用
用户问"这个文件多大/多少行"，且只需要一个数字时。

## 步骤
1. 用 `read_file` 确认路径存在（或直接用 `exec` 的 `wc -l`）。
2. 报告行数，并说明是否包含空行。
```

### 8.2.3 它是怎么被解析的

```python
# nanobot/agent/skills.py:26-36（解析入口，节选；正则 _STRIP_SKILL_FRONTMATTER 在 :18-21）
def parse_skill_metadata(content: str) -> dict[str, object] | None:
    if not (match := _STRIP_SKILL_FRONTMATTER.match(content)):
        return None                      # 没有 frontmatter
    try:
        parsed = yaml.safe_load(match.group(1))
    except yaml.YAMLError:
        return None                      # YAML 写坏了
    return {str(k): v for k, v in ...}   # 不是 dict 也返回 None
```

三个坑：① 文件**第一个字符**必须是 `---`，前面有空行就匹配不到；② 结束的 `---` 必须**独占一行**；③ YAML 语法错 → 返回 `None`，技能**仍会被发现**但描述退化成目录名（`nanobot/agent/skills.py:301`–`:307`）。也就是说 **frontmatter 写错不会报错，只会静默降级**——这是"技能没生效"最常见的原因。

### 8.2.4 命名与描述校验

安装技能/插件时会走强校验 `valid_skill_metadata`（`nanobot/agent/skills.py:39`–`:48`），规则由 `_SKILL_NAME` 决定：`name` 必须等于目录名、长度 ≤ 64、只允许小写字母/数字/单连字符（不能有 `--`），`description` 必须是非空字符串且 ≤ 1024 字符。调用点在 `nanobot/apps/cli/service.py:1117` 与 `nanobot/agent/plugins.py:479`；手写时不走校验，但照它写才不会被分发流程打回。

---

## 8.3 YAML Frontmatter 字段详解

### 8.3.1 字段总表

| 字段 | 类型 | 必填 | 作用 | 证据 |
|---|---|---|---|---|
| `name` | string | ✅ | 技能名，必须等于目录名 | `skills.py:43` |
| `description` | string | ✅ | 一句话说明，**会进 system prompt** | `skills.py:41`、`:255` |
| `always` | bool | ❌ | `true` = 全文注入，跳过渐进披露 | `skills.py:350`–`:360` |
| `metadata` | 对象或 JSON 字符串 | ❌ | 依赖声明等机器可读信息 | `skills.py:318`–`:336` |
| `homepage` | string | ❌ | 文档链接（内置 weather 用了） | `nanobot/skills/weather/SKILL.md` |

### 8.3.2 `metadata` 必须是**嵌套对象**，不是点分键

✅ 正确写法（与内置技能一致），也可以写成一行 JSON 字符串（YAML 里是字符串，解析器两种都收）：

```yaml
metadata:
  nanobot:
    emoji: "🐙"
    requires:
      bins: ["gh"]           # 需要系统里有 gh 命令
      env: ["GITHUB_TOKEN"]  # 需要这个环境变量

# 等价的单行写法
metadata: '{"nanobot":{"requires":{"bins":["gh"],"env":["GITHUB_TOKEN"]}}}'
```

❌ 错误写法（旧文档的 `nanobot.requires.bins` 点分键）——为什么错，看解析代码就明白了：

```yaml
metadata: '{"nanobot.requires.bins": ["gh"], "nanobot.requires.env": ["GITHUB_TOKEN"]}'
```

```python
# nanobot/agent/skills.py:335-336（节选）
payload = data_object.get("nanobot", data_object.get("openclaw", {}))
return cast(dict[str, Any], payload) if isinstance(payload, dict) else {}
```

`data` 的键是字面量 `"nanobot.requires.bins"`，`data.get("nanobot")` 取不到 → 返回 `{}` → `requires` 为空 → **依赖检查永远通过**。后果不是报错，而是"技能显示可用，一跑就找不到 `gh`"。

依赖的读取（`nanobot/agent/skills.py:265`–`:284` 节选，注意**没有** `nanobot.` 前缀）：

```python
requires = skill_meta.get("requires") or {}
bins_raw = requires.get("bins") or []
return ", ".join([f"CLI: {n}" for n in required_bins if not shutil.which(n)]
                 + [f"ENV: {e}" for e in required_env_vars if not os.environ.get(e)])
```

`_check_requirements`（`:338`）用 `shutil.which` + `os.environ.get` 判定，**缺依赖不阻止加载**，只在摘要后面加 `(unavailable: CLI: gh)`（`:257`–`:259`）。

### 8.3.3 `always` 与内置技能的 metadata

`always` 写在顶层或 `metadata.nanobot.always` 都认（`nanobot/agent/skills.py:356`–`:359`）：`always: true` 会把全文注入 `# Active Skills` 段（`nanobot/agent/context.py:132`–`:136`），适合 1–2 个必须时刻记住的元规则；默认 `false` 只进摘要。

内置 `github` 技能的 frontmatter（`nanobot/skills/github/SKILL.md`）长这样：

```yaml
metadata: {"nanobot":{"emoji":"🐙","requires":{"bins":["gh"]},"install":[
  {"id":"brew","kind":"brew","formula":"gh","bins":["gh"],"label":"Install GitHub CLI (brew)"}]}}
```

`emoji`/`install` 不参与运行时判定，是给 WebUI 和安装引导看的；硬约束只有 `requires.bins`/`requires.env`。

---

## 8.4 技能发现机制

### 8.4.1 三个来源，从高到低

```python
# nanobot/agent/skills.py:103-119（节选，去掉了细节）
skills = self._skill_entries_from_dir(self.workspace_skills, "workspace")   # ① 用户自己的
seen_names = {entry["name"] for entry in skills}
for name, path in plugin_skills:                                            # ② 已启用的插件
    if name in seen_names: continue
    skills.append({"name": name, "path": str(path), "source": "plugin"})
    seen_names.add(name)
skills.extend(self._skill_entries_from_dir(self.builtin_skills, "builtin",   # ③ 包内置
                                           skip_names=seen_names))
```

| 优先级 | 来源 | 路径 | 说明 |
|---|---|---|---|
| ① 最高 | workspace | `<workspace>/skills/<name>/SKILL.md` | 用户自建，可覆盖同名内置 |
| ② | Agent Plugins | 已安装且启用的插件包 | `enabled_agent_plugin_skills`（`skills.py:100`） |
| ③ 最低 | builtin | `nanobot/skills/`（`BUILTIN_SKILLS_DIR`，`skills.py:15`） | 包内预置 11 个 |

`seen_names` 就是"先到先得"：**同名时 workspace 赢**，不需要额外配置。

### 8.4.2 过滤层、入口与读取映射

- `disabled_skills`（`nanobot/config/schema.py:146`）：按名字排除，还会展开 CLI App 的兼容别名（`skills.py:121`–`:126`）。
- `filter_unavailable=True`：把缺 `bins`/`env` 的技能从**摘要**里过滤（`:128`）；`load_skill` 用 `False`（`:142`），按名字直接读仍读得到。
- **两个要记的入口**：`list_skills`（`:90`）列清单、`load_skill`（`:132`）取全文。
- 摘要里给的是相对路径 `skills/github/SKILL.md`，文件却在包目录里，这个翻译由工具完成（`nanobot/agent/tools/filesystem.py:234`–`:248`）：`_builtin_skill_read_path` 把 `skills/<name>/...` 映射到 `BUILTIN_SKILLS_DIR`。模型永远只看到"工作区内的相对路径"。

### 8.4.3 显式点名一个技能

用户在消息里写 `$github`，表示"这一轮请把 github 技能全文放进上下文"。`_SKILL_REFERENCE`（`nanobot/agent/skills.py:23`）匹配 `$name`，`get_explicitly_invoked_skills`（`:165`）解析成技能名，再通过 `RuntimeContextBlock`（`:195`）注入——只对这一轮生效，是手动版渐进披露。

---

## 8.5 渐进披露（Progressive Disclosure）

### 8.5.1 为什么必须这样

20 个技能、每个 500 行，全量注入 ≈ 1 万行 ≈ 十几万 token：费钱、分散注意力、可能直接超窗口。而一个回合**通常只会用到 0～2 个技能**。

### 8.5.2 三层结构

| 层 | 内容 | 何时进上下文 | 成本 |
|---|---|---|---|
| Tier 1 摘要 | 每个技能的 `name` + `description` + 相对路径 | 每轮 | 约 20–40 token/技能 |
| Tier 2 全文 | `SKILL.md` 正文 | 模型 `read_file` 时 | 几百～一千 token/次 |
| Tier 3 深水区 | `references/`、`scripts/` | 模型再读/再执行 | 按需 |

摘要的生成代码（`build_skills_summary`，`nanobot/agent/skills.py:204`）每组一行标题、每技能一行：

```python
# nanobot/agent/skills.py:250-261（节选）
lines = [f"### {label} (`{display_root}`)"]
for entry in entries:
    available = self._check_requirements(meta)
    suffix = f" (unavailable: {missing})" if not available else ""
    lines.append(f"- **{skill_name}** — {desc}{suffix}  `{relative_path}`")
```

渲染出来是"一个分组标题 + 每个技能一行"：每行依次是加粗的技能名、一句描述、反引号包住的相对路径。`# Skills` 外壳来自模板 `nanobot/templates/agent/skills_section.md`，挂载于 `nanobot/agent/context.py:138`–`:143`。

### 8.5.3 `always: true` 是唯一例外

```python
# nanobot/agent/context.py:132-143（节选）
active_skills = self.skills.get_always_skills()
if active_skills:
    parts.append(f"# Active Skills\n\n{active_content}")          # 全文注入
skills_summary = self.skills.build_skills_summary(exclude=set(active_skills), ...)
```

注意 `exclude`：已全文注入的技能**不会**在摘要里重复出现，避免重复占 token。

> 💡 面试加分：渐进披露本质是"把召回决策交给模型"。传统 RAG 用相似度决定召回，nanobot 用"模型读完摘要自己判断"——代价是多一次工具调用，收益是零基础设施。

---

## 8.6 内置技能列表

`nanobot/skills/` 下共 11 个技能（每个一个目录 + `SKILL.md`）：

| 技能 | 做什么 | 依赖（`requires`） |
|---|---|---|
| `github` | 用 `gh` CLI 管 issue / PR / CI | `bins: ["gh"]` |
| `weather` | 查天气与预报（wttr.in，免 API key） | `bins: ["curl"]` |
| `tmux` | 终端多窗口操作 | `bins: ["tmux"]` |
| `cron` | 何时用 `cron` 工具、何时该用 `HEARTBEAT.md` | 无 |
| `memory` | 用 `grep` 搜 `history.jsonl` 查旧对话 | 无 |
| `summarize` / `clawhub` | 长文本/URL/播客摘要 / ClawHub 平台集成 | `bins: ["summarize"]` / 无 |
| `image-generation` / `my` | 图像生成调用规范 / 用 `my` 查看自身运行时配置 | 无 |
| `skill-creator` | **元技能**：教模型写新的 `SKILL.md` | 无 |
| `update-setup` | 升级/安装相关流程 | 无 |

`skill-creator` 特别值得读：Dream 的提示词直接引用它作为格式规范（`nanobot/templates/agent/dream.md` 里的 `Follow {{ skill_creator_path }} for format`）——**Dream 学会写技能，靠的是读这个技能**，一个自举的小设计。

---

## 8.7 内置工具完整列表

### 8.7.1 总览（模型可见名）

| 类别 | 工具名 | 一句话 |
|---|---|---|
| 文件 | `read_file` / `write_file` | 读（文本/图片/PDF/Office，按行分页）、写（自动建父目录） |
| 文件 | `edit_file` / `apply_patch` | 精确替换一段文本 / 结构化补丁（多文件改动首选） |
| 文件 | `list_dir` | 列目录 |
| 搜索 | `find_files` / `grep` | 按路径/glob/类型找文件 / 按正则找内容（支持 PDF/DOCX/XLSX/PPTX） |
| 执行 | `exec` | 跑一条 shell 命令 |
| 执行 | `exec_session` / `list_exec_sessions` | 交互式长进程会话管理 |
| 通信 | `message` | 主动发消息/发文件（跨渠道） |
| 并发 | `spawn` | 起一个子 Agent 干活（v0.3.5 仍叫 `spawn`） |
| 调度 | `cron` | 定时/周期任务 |
| 媒体 | `generate_image` | 生图/改图，落成 artifact |
| 自省 | `my` | 查看/修改自己的运行时配置与 scratchpad |
| 应用 | `run_cli_app` | 跑用户安装的 CLI App |
| 目标 | `create_goal` / `update_goal` | 长任务目标状态机 |
| 会话 | `list_sessions` / `send_session_message` | 列出/给别的会话发消息 |
| 会话 | `search_sessions` / `read_session` | 搜/读别的会话 |
| 网络 | `web_search` | 搜索；后端由配置的 provider 决定 |
| 网络 | `web_fetch` | 抓 URL 转成可读文本 |

`web_search` 的工具名是**固定常量** `"web_search"`（`nanobot/agent/tools/web.py:367`），`web_fetch` 同样固定为 `"web_fetch"`（`:1114`）。会变的是**后端**：`web.py:425`–`:464` 的 `_resolve_provider()` 按 `tools.web.search.provider` 与对应 API key 决定真正用哪家（`brave`/`tavily`/`searxng`/`jina`/`kagi`/`exa`/…），缺 key 就回落 `duckduckgo`。所以面试时说"工具名固定、provider 可切换"才是准确表述。

### 8.7.2 三个最常用的参数细节

**`read_file`**（schema 在 `nanobot/agent/tools/filesystem.py:251`–`:269`）：`path`（必填）、`offset`（1 起，默认 1）、`limit`（默认 2000 行）、`pages`（PDF 页码）、`force`（内容没变也再返回）——例如 `{"path": "src/app.py", "offset": 120, "limit": 80}`。单次最多 128,000 字符（`_MAX_CHARS`，`:274`），文件上限 100 MiB（`:275`）。

**`edit_file`**（`filesystem.py:831`–`:858`）：`path`/`old_text`/`new_text` 必填，可选 `replace_all`、`occurrence`（第几次出现）、`line_hint`（目标行）、`expected_replacements`（替换数量守卫）。

**`grep`**（`nanobot/agent/tools/search.py:514`，参数在 `:527`–`:585`）：`pattern`/`path`/`glob`/`type`/`output_mode`（`content`/`files_with_matches`/`count`）/`head_limit`（默认 250）/`context_before`、`context_after`（默认 5）。

---

## 8.8 ToolRegistry 统一注册与执行机制

### 8.8.1 谁在什么时候建它

```python
# nanobot/agent/loop.py:638-639（主 Agent）
loader = ToolLoader(); registered = loader.load(ctx, self.tools)
# nanobot/agent/subagent.py:224（子 Agent，注意 scope）
ToolLoader().load(ctx, registry, scope="subagent")
```

- **主 Agent**：组合根建一个 `ToolRegistry`，`ToolLoader` 扫包注册一遍。
- **子 Agent**：每次新建**独立** registry（`subagent.py:205`–`:225`），看不到主 Agent 的运行时状态。
- **Dream**：不走 loader，手工注册 4 个文件工具（`memory.py:575`–`:616`）——最小权限清单。

### 8.8.2 注册表本身

```python
# nanobot/agent/tools/registry.py:30-38（节选）
def register(self, tool: Tool) -> None:
    self._tools[tool.name] = tool
    self._cached_definitions = None      # 缓存失效
```

`get_definitions()`（`:86`）把每个工具 `to_schema()` 成 OpenAI function schema，**排序**后缓存：内置在前、`mcp_*` 在后，各自按名字排序（`:99`–`:106`）。稳定顺序是为了命中 provider 的 prompt cache；`register`/`unregister` 会清缓存（`:33`、`:38`），动态加 MCP 工具不会读到旧列表。

### 8.8.3 一次工具调用的四个步骤

```
模型输出 tool_call: {"name": "read_file", "arguments": {...}}
  ① prepare_call（registry.py:110）
     ├─ get(name)         找不到 → 报错 + "Did you mean ..." 建议（:118-124）
     ├─ _coerce_params    拆掉多包一层 "arguments" 的畸形入参（:171-185）
     ├─ cast_params       按 schema 做安全类型转换（"3" → 3）
     └─ validate_params   JSON Schema 校验，失败直接返回错误（:142-146）
  ② await tool.execute(**params)            （:196）
  ③ 结果归一化：错误会追加 "[Analyze the error above and try a different approach.]"（:189）
  ④ 作为 role="tool" 的消息回灌，进入下一轮迭代
```

`ToolResult` 是 `str` 的子类，额外带 `is_error` 标记（`nanobot/agent/tools/base.py:144`–`:156`）——99% 的代码当字符串用，需要区分成败的地方又能区分。

### 8.8.4 scope：三个作用域

```python
_scopes: set[str] = {"core"}                                       # base.py:208 默认值
if scope not in getattr(tool_cls, "_scopes", {"core"}):            # loader.py:100 注册前过滤
    continue
```

| 工具 | `_scopes` | 谁拿得到 |
|---|---|---|
| `read_file`/`write_file`/`edit_file` | `{"core","subagent","memory"}`（`filesystem.py:272` 等） | 主 Agent、子 Agent、Dream |
| `exec`/`grep`/`apply_patch`/`web_fetch` | `{"core","subagent"}` | 主 Agent、子 Agent |
| `spawn`/`message`/`cron`/`generate_image`/`my`/`create_goal`… | 默认 `{"core"}` | **只有主 Agent** |

这解释了一个常见疑问：子 Agent 为什么不能再 `spawn`？不是硬编码禁止递归，而是 `spawn` 没声明 `subagent` 作用域，`load(scope="subagent")` 时被跳过。

### 8.8.5 生命周期全景

```
启动 → ToolLoader.discover()（loader.py:36）→ load(scope) → enabled(ctx) → create(ctx) → register
每回合 → get_definitions()（取缓存）→ 模型返回 tool_calls → execute(name, params) → ToolResult 回灌
关闭 → MCPProvider.aclose()（mcp.py:1672，只有 MCP 工具需要显式关闭）
```

---

## 8.9 MCP 工具集成

MCP（Model Context Protocol）是"外部工具的标准接口"。nanobot 把 MCP server 的工具**包成内置工具的样子**，模型完全感知不到差别。

### 8.9.1 配置在 `tools.mcpServers`

```json
{
  "tools": {
    "mcpServers": {
      "ticket": {
        "command": "npx", "args": ["-y", "@acme/ticket-mcp"],
        "env": { "TICKET_TOKEN": "xxx" }, "toolTimeout": 30, "enabledTools": ["*"]
      },
      "kb": { "url": "https://kb.internal/mcp", "headers": { "X-Key": "..." } }
    }
  }
}
```

- 键名：Python 侧字段是 `mcp_servers`（`nanobot/config/schema.py:418`），落盘用 camelCase `mcpServers`，由 `to_camel` 别名生成器完成（`nanobot/config_base.py:15`）。
- 字段：`MCPServerConfig`（`nanobot/config/schema.py:362`–`:374`）：`type`/`auth`/`command`/`args`/`env`/`cwd`/`url`/`headers`/`toolTimeout`/`enabledTools`。
- 传输自动判定：有 `command` → `stdio`；`url` 以 `/sse` 结尾 → `sse`，否则 `streamableHttp`（`nanobot/agent/tools/mcp.py:1021`–`:1031`）。
- `enabledTools` 默认 `["*"]`（`config/schema.py:374`）；一旦写白名单就只注册列出的工具，连 resources/prompts 也不再暴露（`mcp.py:1170`–`:1177`）。
### 8.9.2 命名与生命周期

```python
# nanobot/agent/tools/mcp.py:609（工具）；:771（resource）/:875（prompt）
self._name = _sanitize_mcp_tool_name(f"mcp_{server_name}_{tool_def.name}")
```

`ticket` server 的 `query` 工具，模型看到的名字是 `mcp_ticket_query`。前缀同时解决冲突（不同 server 可重名）和排序（`mcp_*` 统一排后面，`registry.py:99`）。

生命周期**不属于 AgentLoop**：组合根创建 `MCPProvider`（`mcp.py:1346`），`connect()`（`:1412`）把工具注册进共享 `ToolRegistry`，退出时 `aclose()`（`:1672`）。这样 WebUI/API/gateway 共享同一批 MCP 连接，每个会话的 AgentLoop 只是使用者。

---

## 8.10 自定义 Tool 与 SKILL.md 实战

### 8.10.1 三十行写一个 Tool

新建 `nanobot/agent/tools/linecount.py`：

```python
"""统计文件行数：一个最小的自定义 Tool 示例。"""
from __future__ import annotations
from pathlib import Path
from typing import Any

from nanobot.agent.tools.base import Tool, ToolResult, tool_parameters
from nanobot.agent.tools.context import ToolContext
from nanobot.agent.tools.schema import StringSchema, tool_parameters_schema

@tool_parameters(tool_parameters_schema(                     # ① 声明参数 schema
    path=StringSchema("要统计的文件路径，相对于工作区"), required=["path"]))
class LineCountTool(Tool):                                   # ② 继承 Tool
    _scopes = {"core", "subagent"}                           # ③ 谁能用

    def __init__(self, workspace: Path) -> None:
        self._workspace = workspace

    @classmethod
    def create(cls, ctx: ToolContext) -> Tool:               # ④ 注入依赖
        return cls(workspace=Path(ctx.workspace))

    @property
    def name(self) -> str:                                   # ⑤ 模型看到的名字
        return "line_count"

    @property
    def description(self) -> str:                            # ⑥ 决定用不用
        return "Count the lines of a UTF-8 text file inside the workspace."

    @property
    def read_only(self) -> bool:                             # ⑦ 只读 → 可并发
        return True

    async def execute(self, path: str = "", **kwargs: Any) -> Any:   # ⑧ 干活
        target = (self._workspace / path).resolve()
        if target != self._workspace and self._workspace not in target.parents:
            return ToolResult.error(f"Error: {path} escapes the workspace")
        if not target.is_file():
            return ToolResult.error(f"Error: not a file: {path}")
        return f"{path}: {len(target.read_text(encoding='utf-8').splitlines())} lines"
```

- **① `@tool_parameters`** 是类装饰器（`nanobot/agent/tools/base.py:318`–`:350`）：把 schema 冻进闭包，注入 `parameters` property，并从 `__abstractmethods__` 删掉 `"parameters"`——所以你**不用**手写 `@property def parameters`。
- **② 继承 `Tool`** 是唯一硬要求；抽象成员四个：`name`/`description`/`parameters`/`execute`（`base.py:171`–`:229`）。
- **③ `_scopes`** 决定可见性（见 8.8.4）；**④ `create(ctx)`** 是工厂钩子（`base.py:218`），配置、manager、workspace 都从这里拿，`enabled(ctx)`（`:214`）决定"要不要有这个工具"。
- **⑤ `name`** 必须全局唯一，重名会被覆盖并打印 `Tool name collision`（`loader.py:114`）；**⑥ `description`** 是给模型的唯一线索，"做什么 + 何时用"比实现细节重要。
- **⑦ `read_only`** 决定能否并发（`concurrency_safe` = `read_only and not exclusive`，`base.py:194`）；**⑧ `execute`** 返回字符串或 `ToolResult.error(...)`，抛异常也会被 registry 包成错误（`registry.py:200`），但主动返回更可控。

**它会被自动发现吗？** 会。`ToolLoader.discover()`（`loader.py:36`–`:66`）扫包内所有模块，跳过 `_` 开头和 `_SKIP_MODULES` 里的模块（`:20`、`:44`），再挑出 `Tool` 子类、非抽象、类名不以 `_` 开头、`_plugin_discoverable` 不为 `False` 的类（`:51`–`:63`），按类名排序（`:64`）。重启 Agent 即生效。

> ⚠️ 直接改包内目录只适合开发/自用。要分发请用 entry point 插件（`[project.entry-points."nanobot.tools"]`），loader 从 `entry_points(group="nanobot.tools")` 加载（`loader.py:74`）。

### 8.10.2 写一个最小 SKILL.md

```
<workspace>/skills/line-count/SKILL.md
```

```markdown
---
name: line-count
description: 统计文本文件行数；用户问"这文件多大/多少行"时使用。
metadata:
  nanobot:
    requires:
      bins: []
---

# Line Count

## 何时使用
- 用户问某个文件有多少行、多大。

## 步骤
1. 调用 `line_count(path="...")`。
2. 返回路径越界错误时，改用 `list_dir` 确认工作区内的相对路径。

## 输出格式
`<path>: <N> lines`
```

三件事：① `name` 必须等于目录名 `line-count`（否则安装校验不过，`skills.py:43`）；② `description` 会进 system prompt 摘要，控制在一两句；③ `metadata` 是嵌套对象，不依赖外部 CLI 时写 `bins: []` 或整个省略。

### 8.10.3 验证清单

```bash
nanobot -m "src/app.py 有多少行？"     # 期望：模型调用 line_count
```

排查顺序：① 文件是否叫 `SKILL.md`（全大写）；② 第一行是否 `---`；③ `name` 是否等于目录名；④ 依赖是否装好（`which gh`）；⑤ 工具没生效就检查是否继承 `Tool`、是否被 `_SKIP_MODULES` 跳过。

---

## 8.11 工具安全机制

| 机制 | 默认 | 作用 | 证据 |
|---|---|---|---|
| `tools.restrictToWorkspace` | `false` | 把文件工具限制在工作区内 | `config/schema.py:401` |
| `tools.exec.enable` | `true` | 总开关，关掉则不注册 `exec` | `agent/tools/shell.py`（`ExecToolConfig`） |
| `tools.exec.allowPatterns` / `denyPatterns` / `sandbox` | 空 | 命令白名单/黑名单/沙箱后端（bwrap、seatbelt） | 同上 |
| `tools.maxToolResultChars` | `16_000` | 单工具结果上限，超出转存文件 | `config/schema.py:131` |
| 敏感文件写保护 | 内置 | 拦 `exec` 重定向写 `history.jsonl`/`.dream_cursor` | `agent/tools/shell.py:226` |
| SSRF 防护 | 内置 | `web_fetch` 与 HTTP MCP 拒绝内网地址 | `security/network.py:34` |

提醒一句：`restrictToWorkspace` 默认是 **false**，即"能读工作区外文件"是默认行为。生产部署要显式打开，并在第 11 章逐项确认。

---

## 8.12 面试高频题

### 题目 1：Skill 和 Tool 有什么区别？

> "Tool 是给模型**调**的函数：`Tool` 子类 + `name`/`description`/JSON Schema/`execute`，注册进 `ToolRegistry` 后每轮把 schema 发给模型。Skill 是给模型**读**的说明书：`SKILL.md`，YAML frontmatter 加 Markdown，只有名字和描述进 system prompt，正文要模型自己 `read_file`。打比方：Tool 是锤子和螺丝刀，Skill 是"宜家组装说明"——Skill 不执行任何代码，它教模型怎么组合 Tool。"

### 题目 2：技能的发现顺序和覆盖规则？

> "三个来源，优先级从高到低：`<workspace>/skills/`、已启用的 Agent Plugin、包内置 `nanobot/skills/`。`list_skills` 用 `seen_names` 去重，先到先得，所以同名时 workspace 覆盖内置。之后按 `disabled_skills` 排除、按 `filter_unavailable` 过滤缺依赖的技能——但过滤只影响摘要，按名字 `load_skill` 仍读得到。"

### 题目 3：什么是渐进披露？

> "先给摘要、按需给全文。Tier 1 只把 name、description 和相对路径放进 system prompt，一个技能二三十 token；Tier 2 是模型判断相关后 `read_file` 读全文；Tier 3 再读 `references/` 或跑 `scripts/`。例外是 `always: true` 的技能，全文注入 `# Active Skills` 并从摘要里排除。本质是把召回决策交给模型：不需要向量库，代价是多一次工具调用。"

### 题目 4：工具的 scope 是怎么工作的？

> "`Tool` 类上有 `_scopes` 集合，默认 `{'core'}`。`ToolLoader.load(ctx, registry, scope=...)` 在注册前过滤：`if scope not in tool_cls._scopes: continue`。主 Agent 用 `core`，子 Agent 用 `subagent`，Dream 干脆不用 loader、手工注册 4 个文件工具。子 Agent 不能 `spawn` 不是硬编码禁止递归，而是 `spawn` 只声明了 `core`——加一行 `_scopes = {'core','subagent'}` 就能放开，这是个可以讨论的设计点。"

### 题目 5：给 nanobot 加一个新能力，你会选哪条路？

> "三条路，按成本排序：① 写 Skill——不写代码，教模型用现成工具组合完成；② 写 Tool——继承 `Tool` 实现四个成员，放进 `agent/tools/` 会自动发现，要分发就用 entry point `nanobot.tools`；③ 接 MCP server——跨语言、跨进程，配置在 `tools.mcpServers`，工具名自动变成 `mcp_<server>_<tool>`。判断标准：能用 Skill 解决就别写 Tool（Tool 每轮都花 schema token）；需要确定性、访问本地资源就写 Tool；已有 MCP server 就直接接。"

---

## 8.13 本章小结

```
技能与工具 = 两个正交的扩展层

Skill（知识层，模型读）
├── <workspace>/skills/<name>/SKILL.md  ← 最高优先级
├── Agent Plugins（已启用）
└── nanobot/skills/（内置 11 个）
    摘要进 prompt（name / description / 相对路径），全文靠 read_file
    frontmatter：name / description / always / metadata（嵌套对象）

Tool（能力层，模型调）
├── nanobot/agent/tools/*.py 里的 Tool 子类（自动发现）
├── 四个成员：name / description / parameters / execute
├── scope：core / subagent / memory
└── ToolRegistry：register → get_definitions → execute

MCP（外部能力）→ tools.mcpServers → mcp_<server>_<tool> → 与内置工具同构
```

### 面试记忆清单

| 考点 | 一句话回答 |
|---|---|
| Skill vs Tool | 说明书写给模型读；函数给模型调 |
| frontmatter | `---` 包起来的 YAML；解析失败静默降级 |
| metadata | 嵌套对象 `{"nanobot":{"requires":{"bins":[...]}}}`，不是点分键 |
| 发现顺序 | workspace > Agent Plugins > builtin，同名先到先得 |
| 渐进披露 | 摘要 → 全文 → references/scripts；`always` 例外 |
| scope | `_scopes` 集合，loader 注册前过滤 |
| ToolRegistry | register / get_definitions / execute |
| 加能力 | Skill > Tool > MCP（按成本） |

---

> **下一步读什么**：先读 `nanobot/agent/tools/base.py` 的 `Tool`（:159）和 `nanobot/agent/tools/registry.py` 的 `execute`（:187），再读 `nanobot/agent/skills.py` 的 `list_skills`（:90）；然后进入第 **[09 - 多平台接入](../09-multi-platform/README.md)**——工具和技能最终都要通过渠道把结果送回用户手里。

---

## 附录 A：与旧版文档的差异

| 旧文档说法 | v0.3.5 事实 | 证据 | 建议 |
|---|---|---|---|
| 技能只在 workspace 与包内置两处搜索 | 三段：workspace → Agent Plugins → builtin | `skills.py:103`–`:119` | 补中间一层 |
| 工具定义手写 `parameters` property | 可用 `@tool_parameters` 类装饰器 | `base.py:318`–`:350` | 新写法更省事 |
| 内置工具是平铺的函数 | 是 `Tool` 子类 + 包扫描 + entry point 插件 | `loader.py:36`、`:74` | 讲清"自动发现" |
| 工具无 scope 概念 | 有 `core`/`subagent`/`memory` 三域 | `base.py:208`、`loader.py:100` | 必讲 |
| 内置技能 6 个 | 11 个（含 `memory`、`my`、`update-setup`、`image-generation`） | `nanobot/skills/` | 见附录 D-4 |

## 附录 B：进阶理解（为什么这样设计）

1. **"说明书"和"接口"分离，让非程序员也能扩展 Agent。** 写 Tool 要懂 Python + JSON Schema + 并发；写 Skill 只要会 Markdown。扩展者从"框架开发者"扩大到"业务专家"。
2. **渐进披露是"把检索交给模型"，代价与收益都很明确。** 相比向量检索多一次工具调用，但省掉整套索引基础设施，而且**可解释**——你能从会话记录看到模型到底读了哪个技能。
3. **自动发现 + 稳定排序是为 prompt cache 服务的。** `get_definitions` 排序后缓存，保证工具列表前缀逐字节一致，provider 才能命中缓存。这类"看不见的性能设计"在大项目里很常见。
4. **scope 用"声明式集合"而不是"运行时判断"。** 判断发生在注册前，工具要么在表里要么不在，模型不可能绕过权限——因为它根本看不到那个工具。
5. **MCP 工具被压平成内置工具的形状**，模型端零适配；但 `mcp_` 前缀让人类一眼看出它来自外部进程。**Dream 只拿 4 个文件工具**则是"最小权限"的教科书案例：一个能改长期记忆的自动进程，能力面被压到"读文件 + 改三个指定文件"。

## 附录 C：Python 语法加油站

### C.1 `@dataclass`：给工具传依赖的轻量容器

```python
@dataclass
class ToolContext:
    config: object
    workspace: str
    timezone: str = "UTC"
```

**哪里用**：`agent/tools/context.py:78`（`ToolContext`）、`:29`（`RequestContext`，`frozen=True` 表示不可改）。**读法**：`create(ctx)` 里的 `ctx.xxx` 就是这些字段。

### C.2 `Protocol`：只看"长得像"，不要求继承

```python
class ContextAware(Protocol):
    def set_context(self, ctx: "RequestContext") -> None: ...
```

**哪里用**：`agent/tools/context.py:46`；`ToolRegistry.prepare_call` 用 `isinstance` 判断（`registry.py:128`）。第三方工具只要实现 `set_context`，不继承基类也算"上下文感知"。

### C.3 `@property` + `@abstractmethod`：只读的契约

```python
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...
```

**哪里用**：`agent/tools/base.py:171`–`:187`。用 property 而不是普通字段，是因为 `name` 可能依赖实例状态（`web.py:425` 按 provider 返回不同名字）。子类没实现这三个 property，实例化就 `TypeError`——loader 正是用 `__abstractmethods__` 过滤抽象类的（`loader.py:58`）。

### C.4 类装饰器：`@tool_parameters` 干了什么

```python
def tool_parameters(schema):
    def decorator(cls):
        frozen = deepcopy(schema)
        cls.parameters = property(lambda self: deepcopy(frozen))   # 注入 property
        abstract = getattr(cls, "__abstractmethods__", None)
        if abstract and "parameters" in abstract:
            cls.__abstractmethods__ = frozenset(abstract - {"parameters"})
        return cls
    return decorator
```

**哪里用**：`agent/tools/base.py:318`–`:350`，用法见 `filesystem.py:251`、`spawn.py:24`。两个细节：`deepcopy` 让调用方改返回值不污染全局 schema；改 `__abstractmethods__` 是为了让"不写 `parameters` property"的类也能实例化，否则 `ABC` 直接拒绝创建。

### C.5 `str` 子类：既是字符串又带状态

```python
class ToolResult(str):
    is_error: bool

    def __new__(cls, content: str, *, is_error: bool = False):
        obj = str.__new__(cls, content)
        obj.is_error = is_error
        return obj
```

**哪里用**：`agent/tools/base.py:144`–`:156`；判定函数 `is_tool_error_result`（`registry.py:15`）。老代码把结果当 `str` 拼接/截断全都还能用，新代码需要区分成败时又能拿到 `is_error`。继承内置类型时，构造逻辑写在 `__new__` 而不是 `__init__`。

## 附录 D：原文勘误

| # | 旧文档原文 | 错在哪 | 正确说法 | 证据 |
|---|---|---|---|---|
| 1 | `metadata: '{"nanobot.requires.bins": ["git","gh"]}'` | 点分键不是嵌套结构，解析后取不到 `nanobot`，依赖检查永远通过 | 必须嵌套：`metadata: {"nanobot":{"requires":{"bins":["gh"]}}}` | `agent/skills.py:335`、`:265`–`:284` |
| 2 | `read_file` 参数是 `start_line` / `end_line` | 参数名不存在 | 是 `offset` / `limit`（另有 `pages`、`force`） | `agent/tools/filesystem.py:251`–`:269` |
| 3 | 内置技能 6 个 | 实际 11 个，且缺 `memory`/`my`/`image-generation`/`update-setup`/`cron` | 见 8.6 表格 | `nanobot/skills/` 目录 |
| 4 | MCP 配置在 `mcp.servers` | 字段路径错 | 在 `tools.mcpServers`（Python 字段 `mcp_servers`） | `config/schema.py:418`、`config_base.py:15` |
| 5 | MCP 工具名是 `mcp_<tool>`（如 `mcp_get_weather`） | 少了 server 段，不同 server 同名工具会冲突 | `mcp_<server>_<tool>`，另有 `_resource_`/`_prompt_` 变体 | `agent/tools/mcp.py:609`、`:771`、`:875` |
| 6 | "自定义工具需要手工注册进 `ToolRegistry`" | 不需要手改注册表 | 放进 `agent/tools/` 会被 `ToolLoader` 自动发现；分发用 entry point `nanobot.tools` | `agent/tools/loader.py:36`、`:74` |
| 7 | "工具没有作用域，子 Agent 也能用所有工具" | 有 scope 过滤 | `_scopes` 三域，子 Agent 用 `scope="subagent"` 注册 | `agent/tools/base.py:208`、`loader.py:100`、`subagent.py:224` |
