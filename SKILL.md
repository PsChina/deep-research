---
name: deep-research
description: 生产级多步深度研究。像资深顾问一样工作: 理解问题 → 产出研究计划 → 等你批准 → 假设驱动迭代验证 → 交付专业报告。基于 ReAct + Reflexion + Self-Ask + CRAG + IUI Devil's Advocate + Info Saturation + Toulmin 论文方法的工程组合(非原创新方法)。
---

# Deep Research

> 像资深顾问一样做研究：提出假设 → 派工验证 → 交叉核实 → 形成判断。
> 执行细节见 [`PLAYBOOK`](PLAYBOOK.md)，写作标准见 [`WRITING_STYLE`](WRITING_STYLE.md)，报告骨架见 [`REPORT_TEMPLATES`](REPORT_TEMPLATES.md)，评分标准见 [`RUBRIC`](RUBRIC.md)。

## 🔴 执行前硬门禁（答不出 → 立即停，不进入 Phase 0）

1. 本次 spawn 几个 researcher？（standard ≥2，deep ≥4。**0 = 不是 deep research，停**）
2. Researcher Prompt 模板（含 output schema）是否已准备复制到派工 prompt？
3. 是否已有 ≥1 个搜索计划使用 `freshness:"month"` 或 `freshness:"week"`？

## 触发

- `/deep-research <主题> [--quality=fast|standard|deep] [--eval]`
- 或问题需要跨多个权威源综合（选型/调研/对比/决策）
- **不需要深度研究**：单点事实、不依赖时效数据、"个人项目用 React 还是 Vue" → 降级为快速回答

## 三步工作流

### Step 0: 研究计划（展示后默认 60s 自动开始）

展示以下内容：

- **我理解的问题**: 一句话复述
- **初步判断**: 基于已知信息的直觉（可被推翻）
- **研究角度**: 3-5 个方向，每个一句话
- **预期深度**: fast(5-10min) / standard(15-30min) / deep(40-60min)
- **你会得到的**: 报告类型 + 决策支撑程度
- **你说了算**: 加/删/调什么？

**审批窗口（best-effort，默认行为）**：展示后打一行 `⏱️ 60s 后自动开始；想改现在说，或回「立即开始」直接跑`，跑 `sleep 60`（`run_in_background:true`）**让出回合**，并**记住该后台任务 id**。

- **窗口内用户回复（改 / 批准 / 不做 / 立即开始）→ 第一步先用记住的 id `TaskStop` 掉计时器，再按其意图走**（改→stop 旧 id 后再起新计时器，**任何时刻最多 1 个活跃计时器**；批准 / 立即开始→进 Step 1；不做→停）。⚠️ 漏取消 = 留个幽灵计时器，稍后退出误触发二次开跑（**头号 bug，必须先 kill 再行动**）。
- 用户沉默、计时器退出触发重唤 → 进 Step 1；但**先确认本轮尚未开跑**（上下文里没有本轮 researcher 派工 / Dispatch Gate 记录才算未开跑），已开跑则忽略本次重唤、防重复。未答澄清按模板默认走，TL;DR 末尾标 `clarify_assumed`。
- 环境不自动重唤 → 优雅降级为"等你批准"（= 老行为，绝不卡死成更坏状态）。

**立即开始**（触发时输入已含「立即开始 / 马上开始 / 直接开始 / 不用确认 / 别问了 / skip」）→ 不起计时器，1-2 行压缩陈述计划后直奔 Step 1。

Phase 0 的 ≤3 澄清问合并进计划一次问。批准 / 超时 / 立即开始后自主推进，不再打断（例外仅：密钥泄露 / 武器 / 工具完全失败）。

### Step 1: 派工研究（sub-agent 执行，主 agent 只协调）

**🔴 铁律：研究执行阶段主 agent 不直接搜、不直接读。搜索和阅读 100% 由 researcher sub-agent 完成。**

> 主 agent 自己搜 = 单线程 + 无 Devil's Advocate + 无交叉验证 + 上下文污染。

**🔴 最高优先级反模式**：主 agent 自己搜 = 研究作废。发现即停，改派 sub-agent 重跑。

**执行流程**：

1. **Discovery Bootstrap**：主 agent 按以下**生态维度**批量搜索建图，**禁止凭记忆拟玩家/产品/关键方清单**：
   - 维度 A — 领先者/主流方案（行业头部是谁？）
   - 维度 B — 挑战者/新兴者（谁在快速崛起？）
   - 维度 C — 地区性/独立玩家（非主流市场/开源/替代方案？）
   - 维度 D — 反方/失败案例（谁在唱反调？什么已经失败了？）
   - 维度 E — 开源/独立/非商业生态（哪些重要玩家不在商业产品列表中？开源模型、学术项目、独立开发者工具？）
   - 每个维度 ≥1 条搜索，其中 ≥1 条必须带 `freshness: "month"`（确保生态图有时效基线）
2. **Freshness & Coverage Sweep**（deep tier 必跑）：派独立 sub-agent，任务只有两个——
   - **时效性扫描**：搜过去 90 天的相关新发布/新数据/新事件（不限于"产品发布"，含研究报告、政策变化、市场事件）
   - **覆盖度补盲**：检查 Discovery Bootstrap 结果中是否遗漏了特定地区/语言/阵营的来源（如仅覆盖英文源则补中文/其他语言源）
3. **Dispatch**：显式确认 sub_Q 全部映射到 researcher 后，同帧 spawn 所有 researcher（standard ≥2, deep ≥4），每个独立 context
4. **REFLECT**：所有 researcher 返回后，按下方 REFLECT 强制清单逐项检查 → 判断是否饱和 → 决定是否 Round 2
5. 重复直到不再发现实质性新信息

每轮结束自问：「我本轮是否亲自搜了？」是 → 该轮作废，改派 sub-agent。违反 = `[no-subagent]` degraded。

**进度心跳**：每个 Phase 完成时允许 ≤1 句话通知（如"Phase 2 完成，进入 Phase 3"），非阻塞、不等回复。

#### REFLECT 强制检查清单（每轮 researcher 返回后必过）

| # | 检查项 | 通过标准 | 不通过动作 |
|---|---|---|---|
| 1 | **时效性** | 所有 finding 的 `source_date` 中，最新的一条在 N 天内（快速演进领域 N=30，稳定领域 N=180；主 agent 根据主题自行判断并声明 N 值） | 补搜：`"<topic> latest <current_month> <current_year>"` |
| 2 | **覆盖度** | ≥2 个地理/语言市场的来源、≥2 类 source_type（official_doc / academic / industry / community） | 补搜缺维度的来源 |
| 3 | **对立面** | 每个 sub_Q 的 findings 中 ≥1 条 `challenges_thesis: true` | 该 sub_Q 退回 researcher 补搜反方 |
| 4 | **饱和判断** | 最近一轮新增 finding 数 ≤ 前一轮的 30%，且无全新的实质性信息类别 | 未饱和 → Round N+1 |

#### Researcher Prompt 模板（🔴 必须使用，禁止即兴写）

> v4.9.3 实测：6 个 researcher 中 5 个因未收到 output schema 返回自然语言而非结构化 JSON。

```
你是 deep-research researcher sub-agent。唯一任务是研究以下 sub-question 并返回结构化 JSON。

## Sub-question
<填入 sub_Q>

## Query seeds (每个都搜，含至少1条反方query + 1条时效性query)
1. "<正向搜索query>"
2. "<正向搜索query>"
3. "<反方/批评/失败案例 query>"
4. "<时效性query — 搜过去90天新进展: topic + latest/recent + current_month current_year>"

## Budget
≤15 WebSearch, ≤8 WebFetch。优先官方来源。Query seed 4（时效性）必须使用 freshness 过滤参数。

## Tool Fallback
anysearch 失败 → 重试1次 → 仍失败 → WebSearch → 仍失败 → WebFetch。禁止凭记忆补 finding。**每个 query seed 独立容错**：一条失败不阻塞其他条。所有 query 跑完后，≥50% 成功才继续写 findings，否则标 `status: partial`。

## Extract 后处理（🔴 必须执行，防重复来源污染）
1. **URL 去重**：所有 extract 按 URL 分组，同一 URL 只保留 content 最完整的 extract。记录去重前后数量。
2. **相关性筛选**（🔴 每个 extract 写 ≤3 句为什么与 sub_Q 相关，写不出的丢弃—等同 RAG filter）：每个 extract 标注 `relevance: high|medium|low` + `relevance_reason: "<≤3 句>"`。`low` 的不纳入 findings（记入 exclusion_reasons）。
3. **去重后数量** 如果 < tier_min（std≥2, deep≥3），补搜。
4. **🔴 JSON 校验（代码执行）**：输出 JSON 后主 agent 必须跑 `python3 hooks/validate_researcher_output.py <output.json>`。fail → 退回重出（最多 1 次）。跳过 = `[no-schema-validate]` degraded。（std≥2, deep≥3），补搜。

## Output schema (🔴 严格按此格式输出JSON)
{
  "sub_question": "<sub_Q原文>",
  "findings": [
    {
      "finding_id": "F-<Q编号>-<序号>",
      "claim": "<一句话事实主张，≤200字>",
      "evidence_span": "<来源原文关键句，≥50字>",
      "source_url": "<URL>",
      "source_type": "official_doc|academic|financial_media|industry_blog|community",
      "source_date": "YYYY-MM-DD",
      "confidence": "HIGH|MEDIUM|LOW",
      "challenges_thesis": true/false
    }
  ],
  "source_funnel": {
    "identified": <搜索返回总数>,
    "screened": <去重+标题筛选后>,
    "extracted": <全文extract数>,
    "included": <最终纳入findings>,
    "exclusion_reasons": {"duplicate": N, "irrelevant": N, "paywall": N, "low_quality": N}
  },
  "dedup": {"before": <去重前 extract 总数>, "after": <去重后数,必须 ≤ before>},
  "query_resilience": {"total": <query seed 总数>, "succeeded": <成功条数>, "failed": <失败条数>, "threshold_met": <succeeded ≥ ceil(total×0.5) 则 true,否则 false>},
  "key_insight": "<1句话总结最重要的发现>"
}
```

反方 query 要求：decide/compare → `"regret/reversal/failed <topic>"`；survey/howto → `"criticism/limitation/risk <topic>"`。

时效性 query 要求（seed 4）：格式 `"<topic 核心关键词> <current_month> <current_year> latest"`。快速演进领域（tech/AI/加密等）加 `freshness: "month"`，稳定领域（法律/历史/基础设施等）加 `freshness: "year"`。

### Step 2: 交付报告

顾问级分析，不是 findings dump。有 Central Thesis、有判断、有下一步。
末尾留追问钩子：「对 [具体结论] 想深入？直接说，基于当前研究继续。」

## Tier 定义与硬约束

| Tier | 触发信号 | 时间 | 子问题 | researcher | 总 extract | 总搜索 | 字数 |
|---|---|---|---|---|---|---|---|
| **fast** | "快速/大概/简要" + 子问题≤2 | 5-10min | 1-3 | 0 | ≥3 | ≥5 | 1000-2500 |
| **standard** | 默认 | 15-30min | 3-5 | ≥2 | ≥8 | ≥15 | 2500-5000 |
| **deep** | "深度/系统/全面/对比" 或 子问题≥5 | 40-60min | 5-8 | ≥4 | ≥15 | ≥30 | 5000-12000 |

**deep 额外硬约束**：反方 query ≥2 / 官方一手源 ≥4 / A-tier 源 ≥20% / vendor/个人博客 ≤40% / 选项 ≥2 含 fallback / REFLECT ≥2 轮。

任一未达 → `degraded: true` + TL;DR 前显式告知。

### Phase 完整性（deep tier）

| Phase | deep tier | 跳过后果 |
|---|---|---|
| **Discovery Bootstrap** (Phase 2) | 必跑（按 4 生态维度搜索） | `[no-discovery-bootstrap]` |
| **Freshness & Coverage Sweep** (Phase 2.6) | 必跑（独立 sub-agent） | `[sweep-ignored]` |
| **Dispatch Gate** (Phase 2.8) | 必过（显式输出 checklist） | `[dispatch-gate-skipped]` |
| **REFLECT Round 2** | 必跑（含 4 项强制清单） | `[single-round]` |
| **Logic Self-Check**（6 项） | 必跑（60s，不 spawn sub-agent） | `[no-logic-check]` |
| **QA sub-agent**（fact-check + logic + DA 合并） | 推荐 | 跳过不标 degraded |
| **LLM-judge** | `--eval` 或抽样 20% | 跳过不标 degraded |

### Frontmatter（17 必填字段）

```yaml
quality_audit:
  declared_tier: deep
  actual_tier: deep
  degraded: false
  degradation_reasons: []
  duration_minutes: 47
  researcher_spawned: 5
  total_extracts: 18
  total_searches: 23
  phase_2_6_sweep: ran
  phase_2_8_dispatch: passed
  phase_reflect_rounds: 2
  phase_5_qa: ran
  word_count: 7842
  official_sources: 6
  dissent_sources: 2
  newest_source_date: "2026-05-20"
  geographic_coverage: ["CN", "US", "EU"]
```

## 黄金法则

| # | 法则 | 一句话 |
|---|---|---|
| 1 | **不编造** | 每个数字有可追溯来源。不确定就说"不确定" |
| 2 | **找反方** | 主动搜索反对观点和失败案例。一面倒 = 没做完 |
| 3 | **标缺口** | "我不知道 X → Y 判断置信度只到 Z" |
| 4 | **给判断** | 有 Central Thesis、有置信度标注、有可执行下一步 |
| 5 | **可证伪** | 每个核心结论附带"什么情况下这个结论是错的" |
| 6 | **有时效** | 每个 finding 标注 source_date。报告中最老 source 如超过主题 appropriate 阈值 → 显式标注 `[时效风险]` |

## 反模式

| ❌ | ✅ |
|---|---|
| 主 agent 自己搜"看看背景" | 派 sub-agent |
| Phase 4 outline 又问用户确认 | 批准后自主推进 |
| Researcher prompt 没写 output schema | 使用本文模板 |
| anysearch 报 quota 直接放弃 | fallback WebSearch → WebFetch |
| 推荐默认配置不穷举其他选项 | 穷举所有配置，显式排除论证 |
| 凭记忆拟玩家清单不跑 Sweep | Discovery Bootstrap + Freshness & Coverage Sweep |
| 引用论文名但不实际运用方法 | 用方法，不挂名 |
| 一面倒无反方 | 每条 sub_Q ≥1 反方 query |
| 报告中引用了产品/模型版本但不检查是否有更新版 | 每个具名实体过 Freshness Audit |
| 只搜英文源写"全球"结论 | 至少覆盖 ≥2 个语言/地区市场的来源 |

## 学术基础（理解精神，按需运用）

| 成果 | 洞见 | 应用 |
|---|---|---|
| **Toulmin** (1958) | 好论证 = Claim + Data + Warrant + Qualifier + Rebuttal | 每段满足 ≥4 要素 |
| **ReAct** (2210.03629) | 思考与行动交替优于一次性规划 | 搜一轮 → 反思 → 决定下一步 |
| **Reflexion** (2303.11366) | 自我反思提升下一轮质量 | 每轮结束自问：漏了什么？ |
| **CRAG** (2401.15884) | 检索内容需评估可靠性 | 不搜到什么信什么 |
| **Step-Back** (2310.06117) | 先问本质再拆解 | 不退一步直接搜 = frame blindness |
| **Late-Section Hallucination** (2505.15291) | 后半段幻觉概率显著升高 | 写到后半段重新查证 |

## 工程原创声明

2 项原创工程贡献（其余均为已有学术方法的工程组合）：(1) `inference_chain` finding 字段 — 推理从 dump → 可审计日志；(2) dev-time hook harness — `hooks/verify.sh` + fixture 自测。

## 红线

- ❌ 编造数字或来源 · ❌ 报告含 token/密钥 · ❌ 付费墙/登录页/武器/非法内容 · ❌ 一面倒

## 参考文件

[`PLAYBOOK`](PLAYBOOK.md) 执行 · [`WRITING_STYLE`](WRITING_STYLE.md) 写作 · [`REPORT_TEMPLATES`](REPORT_TEMPLATES.md) 骨架 · [`RUBRIC`](RUBRIC.md) 评分 · [`CHANGELOG`](CHANGELOG.md) 版本
