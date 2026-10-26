# nanobot v0.3.5 学习指南（新版）

> 这是 `docs/` 的 **v0.3.5 版本**，与旧版（`docs/01-…` ~ `docs/17-…`）并存，旧版保留用于对照。
> 本套文档所有机制、字段、行号都按 **HKUDS/nanobot `v0.3.5`（tag `v0.3.5`，2026-09-16）** 源码核实。

---

## 为什么要出一套新版

旧版文档写于 2026 年 2-4 月（nanobot 0.1.x 时期），当时的代码量和结构与现在差别很大：

| 项目 | 旧版文档对应时期 | v0.3.5 |
|---|---|---|
| 代码规模 | 宣传口径"约 4000 行"（只统计 `nanobot/agent/*.py` 顶层） | 约 **99,398 行** / 329 个 Python 文件 |
| 配置格式 | 误写为 YAML | JSON：`~/.nanobot/config.json` |
| 渠道 | 平铺的 `channels/*.py`（9 个） | 17 个渠道**子包** |
| 子 Agent | `spawn` 工具、15 次迭代 | `spawn` 工具仍在，但迭代上限继承主 Agent 的 **200** |
| 记忆 | `MEMORY.md` + `HISTORY.md` | `MEMORY.md` + `history.jsonl` + **Dream** 巩固阶段 |
| 心跳 | `HeartbeatService` 类 | `gateway.heartbeat` + 受保护的 cron 系统任务 + `HEARTBEAT.md` |

新版每章都按同样的结构重写，并在正文之后加了四个附录：

- **附录 A 与旧版文档的差异**：这一章哪些内容因为版本演进而变了；
- **附录 B 进阶理解**：这一章背后的设计取舍（面试加分）；
- **附录 C Python 语法加油站**：本章源码里出现的抽象语法，给新手的最小示例；
- **附录 D 原文勘误**：旧文档里**任何版本都不成立**的错误，逐条给出正确说法与源码证据。

---

## 学习路线图

```
Phase 1 基础          Phase 2 动手            Phase 3 实战          Phase 4 面试
┌────────────┐      ┌────────────┐         ┌────────────┐        ┌────────────┐
│ Agent 概念  │ ──→  │ 安装与配置  │   ──→   │ 安全与部署  │  ──→   │ 面试八股    │
│ 项目概览    │      │ 记忆系统    │         │ 实战项目    │        │ 简历/STAR   │
│ 架构解析    │      │ 技能与工具  │         │            │        │ 学习资源    │
│ 源码走读    │      │ 多平台接入  │         │            │        │            │
│ MCP 协议    │      │ 子Agent/定时 │         │            │        │            │
└────────────┘      └────────────┘         └────────────┘        └────────────┘
```

## 目录

### Phase 1：基础概念（必读）

| 章节 | 内容 | 建议时间 |
|---|---|---|
| [01 - 什么是 AI Agent](01-what-is-agent/README.md) | Agent 定义、三要素、ReAct、框架对比（已更新到 2026 年现状） | 2 小时 |
| [02 - Nanobot 项目概览](02-nanobot-overview/README.md) | 项目背景、v0.3.5 的真实规模与能力、为什么值得学 | 1.5 小时 |
| [03 - 架构深入解析](03-architecture-deep-dive/README.md) | 一条消息的生命周期、Loop/Runner 分工、上下文治理 | 3 小时 |
| [04 - 源码逐行解读](04-source-code-walkthrough/README.md) | 真实目录地图、核心类签名、时序图 | 4 小时 |
| [05 - MCP 协议详解](05-mcp-protocol/README.md) | 三大原语、传输方式、`tools.mcpServers` 集成与自写 Server | 2 小时 |

### Phase 2：动手实践

| 章节 | 内容 | 建议时间 |
|---|---|---|
| [06 - 安装与上手](06-install-and-hands-on/README.md) | 环境、`nanobot onboard`、配置默认值、跑通第一个 Agent | 2 小时 |
| [07 - 记忆系统实战](07-memory-system/README.md) | `MEMORY.md`、`history.jsonl`、Consolidator、Dream | 2.5 小时 |
| [08 - 技能与工具](08-skills-and-tools/README.md) | SKILL.md 规范、Tool 基类、scope 过滤、写自己的工具 | 2.5 小时 |
| [09 - 多平台接入](09-multi-platform/README.md) | 17 个渠道、MessageBus、`gateway` 启动、字段与 `enabled` | 2 小时 |
| [10 - 子 Agent 与定时任务](10-subagent-and-cron/README.md) | `spawn` 工具、SubagentManager、cron 工具、heartbeat | 1.5 小时 |

### Phase 3：项目实战

| 章节 | 内容 | 建议时间 |
|---|---|---|
| [11 - 安全与部署](11-security-and-deploy/README.md) | 工作区边界、shell 沙箱、SSRF、Docker/生产 | 2 小时 |
| [12 - Nanobot 实战项目](12-nanobot-real-projects/README.md) | 运维助手、客服 Bot、MCP Server 三个项目 | 4 小时 |

### Phase 4：面试冲刺

| 章节 | 内容 | 建议时间 |
|---|---|---|
| [13 - 面试八股文](13-interview-bagua/README.md) | 高频题与答案（Nanobot 专项已按 v0.3.5 重写） | 3 小时 |
| [14 - 就业市场分析](14-job-market-analysis/README.md) | 岗位趋势、技能要求（数据均标注来源/未验证） | 1 小时 |
| [15 - 简历模板](15-resume-template/README.md) | 项目经验写法（技术事实已校准） | 1 小时 |
| [16 - STAR 面试话术](16-star-interview-script/README.md) | 项目讲述框架与追问应对 | 1 小时 |
| [17 - 学习资源](17-learning-resources/README.md) | 官方文档、论文、社区（含核对日期） | 0.5 小时 |

---

## 五分钟跑起来

```bash
pip install nanobot-ai        # 或 uv tool install nanobot-ai；需要 Python ≥ 3.11
nanobot onboard --wizard      # 生成 ~/.nanobot/config.json 与 workspace
nanobot status                # 不调用模型，检查配置是否就绪
nanobot -m "Hello!"           # 一次性问答（等价 nanobot agent -m）
nanobot                       # 进入终端对话
nanobot gateway               # 常驻：渠道 + WebUI + cron + heartbeat
```

WebUI：`nanobot webui`（网关健康检查在 `127.0.0.1:18790/health`，WebUI 在 `8765`）。

---

## 怎么读这套文档

1. **顺序读**：01 → 05 打基础，06 → 10 动手，11 → 12 做项目，13 → 17 冲面试。
2. **按需查**：每章开头的"本章目标"写清了"看完能回答什么"，面试前直接跳到附录 B。
3. **对照读**：遇到"这里和旧版不一样"的疑惑，看该章附录 A；想确认某句话是不是旧文档写错了，看附录 D。
4. **动手优先**：06/08/09 三章都有可以直接跑的命令与最小示例，跑一遍比读三遍有用。

---

## 版本与勘误

- 对应源码：`HKUDS/nanobot` tag `v0.3.5`（commit `1bb712d3`，2026-09-16）。
- 文档里的行号会随后续版本漂移；发现对不上时，以函数名/类名为锚点重新搜索（例如 `grep -n "_process_message" nanobot/agent/loop.py`）。
- 本套文档与旧版并存，旧版不删除、不再维护。
