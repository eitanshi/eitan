---
title: gbrain × 光启 集成架构
type: framework
tags: [framework, system-design, integration, pkm, second-brain, tech-debt]
lineage: [[concepts/第二大脑系统的设计原则]]
created: 2026-07-16
updated: 2026-07-16
confidence: low
status: seedling
sources: [[raw/观点-2026-07-16-gbrain集成探索]]
contested: false
---

## 一句话定义

把 gbrain（Garry Tan 的 Postgres-原生第二大脑）作为光启的**机器查询后端**——保留 markdown 作为人类真相源，把检索/综合/图谱推理委托给 gbrain over MCP，让 Hermes 在 .hermes.md 召回链路中多一层 `gbrain think` 调用。

## 为什么需要这层

- 光启现在：**markdown + wikilink（手工） + .hermes.md（静态注入）**——大脑能记但不会"想"。
- gbrain 提供：**混合检索 + 自动图谱 + LLM 综合**（带引用 + gap analysis）。
- 关键观察：搜索引擎给你页面，脑给你答案。

## 与现有架构的关系

```
                ┌──── 真理源层 (人类可读, git 控制) ────┐
                │                                        │
                │   ~/knowledge/wiki/*.md               │
                │   ~/knowledge/raw/*.md                │
                │                                        │
                └───────── ▲          ▼ ────────────────┘
                          │          │
                          │  write   │  read (gbrain import + put_page)
                          │          │
                ┌─────────┴──────────┴───────────────────┐
                │                                          │
                │       gbrain                            │
                │   (Postgres + PGLite + 向量 + 图谱)     │
                │                                          │
                └─────── ▲            ▼ ──────────────────┘
                          │  think    │  search / graph-query / capture
                          │ query     │
                ┌─────────┴────────────┴───────────────────┐
                │                                          │
                │   Hermes + .hermes.md                   │
                │   (Hindsight → gbrain think → wiki 文件 │
                │    搜索 fallback)                         │
                │                                          │
                └─────── ▲                                ┘
                          │ 回答
                          │
                       用户
```

## 集成矩阵（核心决策）

| 集成点 | 现状 | 集成后 |
|--------|------|--------|
| 检索 | `[[wikilink]]` + 文件名 search | 既有 wikilink **保留**，但被 `gbrain think` 先行查询 |
| 综合 | 人工串联卡片 | gbrain 自动综合 + 引用 + gap analysis |
| 图谱 | 隐式（卡片标题暗示） | 自动抽取 typed edges |
| 通路 | Hermes 每次重召回 | Hermes 在 .hermes.md 召回后，多一次 gbrain think |
| 写入 | 手动写 markdown | 仍然手动写（gbrain 的 `put_page` 同步进库） |

## 我的角色分工

**人工层 负责**：
- 决定何时升级 skill、何时分叉步骤、何时合并观点
- 卡片命名 / 链接 / lineage
- 写"reasoning path"，不是写"页面"

**gbrain 层 负责**：
- 高速召回 + 综合
- 检测事实间矛盾 + 标注 gap
- 维护 typed-edge 图谱 (works_at / invested_in / founded)
- Dream cycle：回填引用错误、合并冗余、横切新发现的关联

## 关键工程难点（来自烟雾测试）

1. **安装路径**：Windows 上 Bun 不走 `~/.bun/bin/`，走 `~/AppData/Roaming/npm/`——`bun link` 要重设 PATH。
2. **git clone 完整性**：gbrain 300+ 分支，按 git 协议 unpack 在 Windows 会截断——准备 codeload zip fallback。
3. **bun registry `ConnectionRefused`**：默认 registry 在受限网络拉不到 `@types/node`——需配 bunfig.toml mirror。
4. **zod v3/v4 ABI mismatch**：`@modelcontextprotocol/sdk` 硬编码 `require('zod/v3')` 但 zod v4 不暴露该子路径——pin `zod@3` 或走 bun 官方装路径。

→ 详见 [[raw/观点-2026-07-16-gbrain集成探索]] 的 4.x 节详尽记录。

## 当前状态（2026-07-16）

- **Phase**: seedling
- **完成度**: 调研 + 烟雾测试失败采集；
- **未做**: 真正跑通 CLI、未做 import 评估、未做 dream cycle 配置；
- **价值**: 即使集成失败，这条记录已经产生认知收益——揭示了「为什么光启还没集成 gbrain」的具体技术债。

## 核心论点（苏格拉底提炼）

1. **「光启 vs gbrain 二选一」是伪命题**：前者是**知识的人类可读形态**，后者是**机器查询接口**。两者服务对象不同。误把它们当成竞争关系是一个认知陷阱。
2. **集成真正解决的问题不是「更聪明的 AI」**，而是「让 Hermes 不再每次失忆」——这正是「降熵」在工具层的体现。
3. **失败也是知识**：烟雾测试的 Bash Install Path / Git Clone 完整性 / 邮件 ABI 三个失败，写入 raw 的同时让我重新意识到 *Windows + 非传统 toolchain 本身的失败成本*——这条经验下一次会节省时间。
4. **暂停 vs 推进 的 准则**：在 ROI 不明朗时停下记录 > 死磕推进。光启 v2 的「可溯源」原则正是为此而设。

## 辩论过程

- **反方**：
  - "全部用 gbrain 不就完了？双轨没价值。"
  - 答：那是「工具替代」而非「能力叠加」。人类读 markdown 的认知时延极低（30 秒）；让 ag 跑 think 多数时候是 over-engineering。

- **正方**：
  - "rownload-de-gbrain 会让 Hermes 体验发生本质变化——从「会忘记的助手」变成「周明的活档案」。"
  - 答：是的，但前提是「以查询接口」而非「接管真相」。

- **我的回应**：
  - 应该把 gbrain 的"思维部分"拿过来，而不该把它的"存储部分"换走。**真相留在 git，多层检索服务在临时层**。

## 最终结论（共识）

**集成形态**：markdown 是真相、PGLite 是索引、gbrain think 是查询入口、Hermes 是一线交互。
**当前任务**：先把烟雾测试的下游修法方案写进 todo list，**不在本次会话强推**。

## 缘起溯源

| 节点 | 时间 | 来源 | 做了什么 |
|:----|:-----|:-----|:---------|
| 萌芽 | 2026-07-16 | 用户："配置 gbrain" | 目标是「让 Hermes 调用 gbrain 替代手动 wiki」 |
| 展开 | 2026-07-16 | gbrain README + INSTALL | 获得全貌 —— 是 OpenClaw/Hermes 的官方 PKM 后端 |
| 成型 | 2026-07-16 | 烟雾测试 失败 | 决定暂停 Integration 实施，记录当前真相 |

## 开放问题

- gbrain 的默认 search mode 是否为『tokenmax』？需要用户决策。
- 我们现有 markdown 结构 与『gbrain 推荐 schema』差别多大？
- import 后怎么避免双重存储、确保以 markdown 为 single source of truth？
- Windows 上 `dream cycle` (cron-adjacent) 是不是唯一不能 依赖的环节？
- 如果 gbrain 起不来，Hermes	top-of-mind 是降级走传统 wiki 还是 报告错误？

## 交叉引用

- [[第二大脑系统的设计原则]] — 设计哲学的前提
- [[concepts/降熵-收敛与确定性]] — v2.3 元目的如何在这个集成中体现
- [[工作流-四层工作法]] — Phase 0 召回环节多一条 gbrain think
- [[raw/观点-2026-07-16-gbrain集成探索]] — 本次全面原始记录

## 原始对话

> 来自 session: 2026-07-16 Hermes (deepseek-v4-pro + minimax-m3) — 用户告知 "配置 https://github.com/garrytan/gbrain"，期间调查 README / INSTALL_FOR_AGENTS.md / AGENTS.md / gbrain.yml。
>
> 烟雾测试涉及：Bun 1.3.14 安装（npm route）→ gbrain 仓库 zip（14.44MB）→ npm install（--legacy-peer-deps）→ bun run CLI 遇到 zod v3/v4 mismatch。
>
> 终止决定同时间：选项③ "探索性记录"获得用户明确锁定。
