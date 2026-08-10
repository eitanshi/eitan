---
title: gbrain × 光启 集成架构
type: framework
tags: [framework, system-design, integration, pkm, second-brain, tech-debt]
lineage: [[concepts/第二大脑系统的设计原则]]
created: 2026-07-16
updated: 2026-08-10
confidence: medium
status: growing
sources: [[raw/观点-2026-07-16-gbrain集成探索]]
contested: false
---

## 一句话定义

把 gbrain（Garry Tan 的 Postgres-原生第二大脑）作为光启的**机器查询后端**——保留 markdown 作为人类真相源，把检索/综合/图谱推理委托给 gbrain，让 Hermes 在召回链路中多一层 `gbrain think` 调用。

## 为什么需要这层

- 光启：**markdown + wikilink（手工）+ git 备份**——大脑能记但不会"想"。
- gbrain 提供：**混合检索 + 自动图谱 + LLM 综合**（带引用 + gap analysis）。
- 关键观察：搜索引擎给你页面，脑给你答案。

## 与现有架构的关系（2026-08-10 实测版）

```
                ┌──── 真理源层 (人类可读, git 控制) ────┐
                │                                        │
                │   /root/brain/*.md                    │
                │   concepts/  raw/  (光启 vault)       │
                │   git: github.com/eitanshi/eitan      │
                │   每日 21:00 cron 自动 commit+push     │
                └───────── ▲          ▼ ────────────────┘
                          │          │
                          │  write   │  read (gbrain sync --repo)
                          │          │
                ┌─────────┴──────────┴───────────────────┐
                │                                          │
                │       gbrain v0.42.75.0                 │
                │   PGLite (/root/.gbrain/brain.pglite)   │
                │   embedding: SiliconFlow Qwen3-Embed-0.6B│
                │   chat: SiliconFlow DeepSeek-V4-Flash    │
                │                                          │
                └─────── ▲            ▼ ──────────────────┘
                          │  think    │  search / query
                          │ query     │
                ┌─────────┴────────────┴───────────────────┐
                │                                          │
                │   Hermes CLI + cron                      │
                │   每日 03:00 cron: sync+extract+embed    │
                │                                          │
                └─────── ▲                                ┘
                          │ 回答
                          │
                       用户
```

## 集成矩阵（2026-08-10 实际状态）

| 集成点 | 状态 | 说明 |
|--------|------|------|
| 检索 | ✅ 已通 | `gbrain search`（tsvector）+ `gbrain query`（混合 RRF）均可用 |
| 综合 | ✅ 已通 | `gbrain think` 带引用回答（DeepSeek-V4-Flash via SiliconFlow） |
| 图谱 | ✅ 已通 | `gbrain extract` 抽取 126 links |
| 向量化 | ✅ 已通 | 31 pages / 176 chunks 全部嵌入 |
| 写入 | 手动写 markdown | gbrain 只读索引，真相源在 git |
| 自动同步 | ✅ cron 03:00 | sync → extract → embed（替代常驻 autopilot，避免锁冲突） |
| Hermes 通路 | ⏳ 待接 | CLI 已可用，MCP/Hermes 原生调用未接 |

## 部署记录（2026-08-03 ~ 08-10）

- **安装**：`bun install -g github:garrytan/gbrain`（不要装 npm 的 gbrain，无关包）
- **初始化**：`gbrain init --pglite --embedding-model openai:Qwen/Qwen3-Embedding-0.6B --embedding-dimensions 1024`
- **配置**：`~/.gbrain/config.json` — SiliconFlow 单端点管 embedding + chat（config.json 见下）
- **内容导入**：`gbrain sync --repo /root/brain` → 31 pages / 176 chunks
- **升级**：v0.42.72.1 → v0.42.75.0（`bun update gbrain`，GitHub 直连慢，约 10 分钟）

## ⚠️ 关键工程坑（2026-08-10 实战记录）

### 坑 1：embedding 只认环境变量，不认 config.json 的 provider_base_urls
`~/.gbrain/config.json` 里的 `provider_base_urls.openai` 不会驱动 embedding 的 base URL。
**必须** export：
```bash
export OPENAI_API_KEY="sk-..."       # SiliconFlow key
export OPENAI_BASE_URL="https://api.siliconflow.cn/v1"
```
已固化到 `/root/.gbrain-env.sh`（.bashrc 自动 source）。

### 坑 2：AI SDK 默认走 Responses API → SiliconFlow 404
`createOpenAI().languageModel()` 走 `/v1/responses`（OpenAI 新 API），SiliconFlow 只支持
`/v1/chat/completions`。表现为 `chat(...) Not Found`。
**修复**：改了 gbrain 源码 `src/core/ai/gateway.ts` 两处 `languageModel()` → `.chat()`。
⚠️ 升级 gbrain 会被覆盖，需重打补丁。

### 坑 3：PGLite 单写者锁 → autopilot 与 CLI 不能并存
autopilot 常驻进程永久持有 PGLite data-dir 锁，所有 CLI 查询（search/think）等锁 30s 后超时。
**解决**：停用 systemd autopilot，改 cron 定时跑 sync（每次跑完即释放锁）。
```bash
systemctl --user stop gbrain-autopilot && systemctl --user disable gbrain-autopilot
```
cron: 每日 03:00 `~/.hermes/scripts/sync_guangqi_gbrain.sh`（sync+extract+embed）。

### 坑 4：openai recipe 模型白名单
`src/core/ai/recipes/openai.ts` 的 chat.models 只有 gpt-5.2/gpt-4o-mini，deepseek 模型被拒
（unknown_model）。**修复**：把 `deepseek-ai/DeepSeek-V4-Flash` 加进 chat.models 数组。
⚠️ 同坑 2，升级会被覆盖。

### 坑 5：startup update-check 卡死主流程
`gbrain sync` 每次启动检查更新，网络差时（GitHub 直连 ~24s）可能长时间阻塞。
**缓解**：`export GBRAIN_SKIP_STARTUP_HOOKS=1`（已写进同步脚本）。

### 坑 6：think 的 Anthropic 硬依赖（已绕过）
`gbrain think` 设计上要求 ANTHROPIC_API_KEY。通过坑 2/4 的补丁 + `gbrain config set models.think openai:deepseek-ai/DeepSeek-V4-Flash` 绕过，实测可出带引用回答。

## 当前状态（2026-08-10）

- **Phase**: growing（从 seedling 升级）
- **完成度**：CLI 全链路通（import/extract/embed/search/query/think）；自动同步 cron 已配
- **未做**：Hermes MCP 集成（gbrain serve + MCP 挂到 Hermes）；dream cycle 配置
- **价值**：光启现在可被机器检索+综合，Hermes 召回链路可加 gbrain think

## 核心论点（苏格拉底提炼）

1. **「光启 vs gbrain 二选一」是伪命题**：前者是**知识的人类可读形态**，后者是**机器查询接口**。两者服务对象不同。误把它们当成竞争关系是一个认知陷阱。
2. **集成真正解决的问题不是「更聪明的 AI」**，而是「让 Hermes 不再每次失忆」——这正是「降熵」在工具层的体现。
3. **失败也是知识**：Windows 烟雾测试的 4 个失败 → Linux 上换装路径全部绕过；今天的 6 个新坑同样写进本卡，下次节省时间。
4. **暂停 vs 推进 的 准则**：在 ROI 不明朗时停下记录 > 死磕推进。光启 v2 的「可溯源」原则正是为此而设。（2026-08-10 验证：推进到 CLI 全通后停下，MCP 集成留待 ROI 明确时再做。）

## 辩论过程

- **反方**：
  - "全部用 gbrain 不就完了？双轨没价值。"
  - 答：那是「工具替代」而非「能力叠加」。人类读 markdown 的认知时延极低（30 秒）；让 ag 跑 think 多数时候是 over-engineering。
- **正方**：
  - "gbrain 会让 Hermes 体验发生本质变化——从「会忘记的助手」变成「周明的活档案」。"
  - 答：是的，但前提是「以查询接口」而非「接管真相」。
- **我的回应**：
  - 应该把 gbrain 的"思维部分"拿过来，而不该把它的"存储部分"换走。**真相留在 git，多层检索服务在临时层**。

## 最终结论（共识）

**集成形态**：markdown 是真相、PGLite 是索引、gbrain think 是查询入口、Hermes 是一线交互。
**当前任务**：CLI 通路已完成；MCP 集成（Hermes 原生调用 gbrain）待 ROI 明确时推进。

## 缘起溯源

| 节点 | 时间 | 来源 | 做了什么 | 认知状态跃迁 |
|:----|:-----|:-----|:---------|:-------------|
| 萌芽 | 2026-07-16 | 用户："配置 gbrain" | 目标是「让 Hermes 调用 gbrain 替代手动 wiki」 | 梦 → 觉 |
| 展开 | 2026-07-16 | gbrain README + INSTALL | 获得全貌 —— 是 Hermes 的官方 PKM 后端 | 觉 → 醒 |
| 成型 | 2026-07-16 | 烟雾测试 失败 | 决定暂停 Integration 实施，记录当前真相 | 醒 → 悟 |
| 部署 | 2026-08-03 | Linux 服务器 | bun 安装成功 + PGLite 初始化 + SiliconFlow 配置 | 悟 → 道 |
| 打通 | 2026-08-10 | Hermes 会话 | 导入光启 31 页 / 176 chunks，search/query/think 全通 | 道（稳态） |

## 开放问题

- [x] ~~gbrain 默认 search mode 是否 tokenmax？~~ → 实测 query 走混合检索，无需决策
- [x] ~~markdown 结构 vs gbrain schema 差别？~~ → 直接 sync，零适配
- [x] ~~双重存储如何避免？~~ → gbrain 只读索引，真相在 git，`gbrain capture` 不用
- [ ] Hermes MCP 集成（gbrain serve + MCP）是否值得做？ROI 待评估
- [ ] dream cycle（自动补链/合并冗余）需要 ANTHROPIC_API_KEY，暂不可用——用 SiliconFlow 的 think 已够用？

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
> 2026-08-10 续：Linux 部署完成，CLI 全链路打通（见本卡「部署记录」与「关键工程坑」）。
