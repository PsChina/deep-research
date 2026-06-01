# PLAYBOOK (v5.5)

执行手册。硬约束以 [`SKILL.md`](SKILL.md) 为准。版本演进见 [`CHANGELOG.md`](CHANGELOG.md)。

---

## 执行概览（三 Tier）

### Fast Tier (5-10min, 主自跑)

```
Phase 0(15s): 子问题 1-3, 默认跳 clarify
Phase 1(1m): step-back 1 句 + 1-3 sub_Q
Phase 2(3-5m): 主 agent batch_search ≤5 并行, 每 sub_Q ≥2 extract, 总 ≥5 extract
Phase 3(2-3m): 写报告 1000-2500 字, 每段 cite + so-what
Phase 4/5: 跳过 (无 DA / 无 QA / 无 judge)
```

### Standard Tier (15-30min, ≥2 researcher)

```
Phase 0(30s): 3-4 sub_Q, decide→问 tech stack, howto→问 runtime, 没答记 clarify_assumed
Phase 1(2m): step-back + Discovery Bootstrap + 审批窗口 60s 自动 / "立即开始"跳(见 SKILL Step 0)
Phase 2(5-10m): spawn ≥2 researcher, 每 prompt 含 sub_Q + seeds + 反方 + schema
Phase 3(2-3m): REFLECT → 决定 Round 2
Phase 4(5-8m): 综合 findings → outline → 写正文 (Toulmin ≥4), 每 2000 字 refresh evidence
Phase 5(2-3m): QA sub-agent (fact-check + logic + DA 合并, 可选)
Phase 6(3-5m): 渲染 + frontmatter + verify.sh scan
Phase 7: 抽样 30% LLM-judge
```

### Deep Tier (40-60min, ≥4 researcher, 强推理模型)

```
Phase 0(30s): ≥5 sub_Q, "深度/系统/全面/对比" 信号
Phase 1(2m): step-back + Discovery Bootstrap (5 维度) + 审批窗口 60s 自动(可改/立即开始)后自主推进
Phase 2.6(2m): Freshness & Coverage Sweep (独立 sub-agent)
Phase 2.8(30s): Dispatch Gate — 显式输出 checklist 确认派工
Phase 3(10-15m): spawn ≥4 researcher (强推理), max_searches=20, max_extracts=12
Phase 3.7(3-5m): REFLECT 必跑, ≥2 round, Saturation Checklist 5 项全 ✅ 才放行
Phase 4(10-15m): 综合 + outline + Toulmin 6 ≥4 + Rebuttal ≥3 + Logic Self-Check 6 项
Phase 5(3-5m): QA sub-agent (推荐)
Phase 6: frontmatter + verify.sh scan
Phase 7: 抽样 20% 或 --eval LLM-judge
```

---

## Phase 1 — 计划与研究设计

### Step-Back

先退一步理解核心矛盾，再拆解子问题 (arXiv:2310.06117)。

### Discovery Bootstrap（生态/市场 topic 必跑）

按 SKILL.md 的 A-E 五维度批量搜索建图。**禁止凭记忆枚举玩家**。跳过 = `[no-discovery-bootstrap]` degraded。

### 模板特定默认问

| template | 必问 | 默认假设 |
|---|---|---|
| decide | "你的技术栈是什么?" | JS/TS 全栈 |
| howto | "运行时 / 部署目标 / 团队规模?" | local dev / 1 人 |
| compare | "成本 / 性能 / 学习曲线 哪条优先?" | 综合权衡 |
| survey | "时间窗口 / 行业 / 受众层次?" | 2024-2026 / 通用工程 |

用户没答 → 走默认 + TL;DR 末尾显式标假设。

### 中英双语

topic 含 [中国/国内/A股/港股/政策] → 中英各一轮 batch_search。

---

## Phase 2.6 — Freshness & Coverage Sweep

> **触发**: deep tier。详见 SKILL.md Step 1 §2。

派独立 sub-agent，双任务：(a) 过去 90 天时效性扫描 (b) 覆盖度补盲（语言/地区缺口）。跳过 = `[sweep-ignored]` degraded。

---

## Phase 2.8 — Dispatch Gate

Phase 3 搜索开始**前**，主 agent 显式输出：

```
🔴 Dispatch Gate:
[ ] sub_Q 已全部映射到 researcher? 预期 spawn ___ 个 (std≥2, deep≥4)
[ ] 每个 researcher 有独立 query_seeds (含反方 query)?
[ ] 主 agent 确认 Phase 3-5 不亲自搜索?
[ ] 所有 researcher 同帧并行 spawn?
```

未过此 gate → 禁止进入 Phase 3。

---

## Phase 3 — 派工研究

### Researcher 派工

- 同帧 spawn 所有 researcher，每个独立 context
- 使用 SKILL.md 中的 Researcher Prompt 模板（含 output schema + Tool Fallback + Extract 后处理）
- 每个 researcher: ≤20 search, ≤12 extract (deep)

### Extract Gate

每 sub_Q extract 数: fast ≥1, standard ≥2, deep ≥3。不达标 → 补派 follow-up extract。

### 🔴 Researcher Output Validator（v5.2，代码执行）

每个 researcher 返回 JSON 后，主 agent 立即跑 `python3 hooks/validate_researcher_output.py <output.json> --json`。
- **fail** → 退回 researcher 重出（最多 1 次，仍 fail 标 `[schema-fail]` partial）
- **warn**（resilience 不达标）→ 放行但标 warning
- **pass** → 放行

校验三级：Schema（必填字段）/ Dedup（before ≥ after）/ Resilience（≥50% 成功）。跳过 = `[no-schema-validate]` degraded。

## Phase 3.7 — REFLECT & Saturation

### REFLECT 流程

所有 researcher 返回后：
1. 交叉验证 findings，标记冲突
2. 评估信息饱和：是否还在发现实质性新信息？
3. 生成 Round 2 子问题（比 Round 1 更窄更深）
4. deep tier 必跑 ≥2 轮

### Saturation Checklist（5 项全 ✅ 才停止）

| 项 | 判据 |
|---|---|
| **Coverage** | 所有 sub_Q 有 ≥tier_min extract，无空白 |
| **Diversity** | ≥3 独立域名，≥1 异见源 |
| **Dissent** | 每条 sub_Q 至少 1 条反方 finding |
| **Recency** | 最新来源在 6 个月内（快变化领域）/ 12 个月（慢变化） |
| **Gap Close** | Round 2 的新发现 ≤ Round 1 的 20% |

### False-Saturation 防护

以下信号说明未真正饱和：
- 所有 finding 同向（echo chamber）
- 只搜了 1 轮就停 → 强制再搜 1 轮
- 来源全来自同一生态位（如全 vendor 博客）

---

## Phase 4 — 综合与写作

### 综合流程

1. **Schema Drift**: 统一所有 researcher 的 source_type/confidence 口径
2. **Source Dedup (v5.1)**：跨 researcher 按 source_url 分组。同一 URL 被 ≥2 researcher 引用 → 合并（保留最完整 evidence_span + 标注 `cross_researcher: true`）。
3. **Aggregate**: 合并同主题 findings
4. **Cross-Validate**: 矛盾 findings → 仲裁（优先 HIGH confidence + official_doc）
5. **Outline**: 按论证逻辑链组织，不按 sub_Q 编号罗列
6. **写作**: 遵循 [`WRITING_STYLE.md`](WRITING_STYLE.md)

### Logic Self-Check（6 项，60s 主 agent 自检）

| # | 检查项 | 判据 |
|---|---|---|
| 1 | **默认值陷阱** | 推荐"X+Y 最优"时穷举了 X 所有配置选项？ |
| 2 | **选项完备性** | decide 模板有 ≥2 选项含 fallback？ |
| 3 | **假二分** | 是否错误地将连续谱系简化为二选一？ |
| 4 | **范畴错误** | 是否将不同层面的概念放在同一维度比较？ |
| 5 | **循环论证** | 结论是否依赖自身？ |
| 6 | **未检验假设** | 是否有隐含前提未在 clarify_assumed 中声明？ |

任一项 fail → 修正后复检。跳过 = `[no-logic-check]` degraded。

### Late-Section Refresh

每 2000 字重新查证 evidence，优先查后半段（arXiv:2505.15291）。

---

## Phase 5 — QA Sub-Agent（推荐）

合并 fact-check + logic audit + Devil's Advocate 为一个 sub-agent。评分标准见 [`RUBRIC.md`](RUBRIC.md)。

**派工**: fresh context，输入草稿 + 所有 researcher findings。

```
你是 Quality Assurance sub-agent。同时做三件事：
1. Fact Check: 抽样 5 条 claim，验证 source 真支持
2. Logic Audit: 6 项检查（见 Phase 4 Logic Self-Check）
3. Devil's Advocate: 攻击最弱 claim + echo chamber + omission + reasoning leap

输出: {severity, attacks[], missing_topics[], overall_verdict}
```

severe → 修正后重跑相应 sub_Q。moderate → 加 Qualifier/Rebuttal。minor → 微调。

---

## Phase 6 — 渲染

1. 按 [`REPORT_TEMPLATES.md`](REPORT_TEMPLATES.md) 选模板渲染
2. 填 frontmatter 17 必填字段（见 SKILL.md）
3. 跑 `hooks/verify.sh scan <report.md>`
4. 不得谎报 quality_audit 字段

---

## 参考

- 硬约束: [`SKILL.md`](SKILL.md)
- 写作标准: [`WRITING_STYLE.md`](WRITING_STYLE.md)
- 模板骨架: [`REPORT_TEMPLATES.md`](REPORT_TEMPLATES.md)
- 评分标准: [`RUBRIC.md`](RUBRIC.md)
- 版本演进: [`CHANGELOG.md`](CHANGELOG.md)

---

**版本**: v5.5。硬约束以 SKILL.md 为准。**最后更新**: 2026-06-01
