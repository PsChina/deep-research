---
name: deep-research
description: 四阶段递归深度研究。好奇循环广泛探索 → 深度推理检测盲区 → 同行评审对抗验证 → 采纳优化后交付顾问级报告。适用于选型 / 竞品对比 / 技术调研 / 可行性评估 / 趋势综合等需跨多权威源交叉验证的开放问题；单点事实、写代码、纯计算、简单 how-to 不要用。

> 研究不是流水线，是四个可递归组合的阶段。deep-research 是可调用工具，不是全流程 orchestrator。
> 架构设计见 [`ARCHITECTURE_v6.md`](ARCHITECTURE_v6.md)，操作手册见 [`PLAYBOOK`](PLAYBOOK.md)，写作标准见 [`WRITING_STYLE`](WRITING_STYLE.md)。

---

## 触发

- `/deep-research <主题> [--quality=fast|standard|deep] [--eval]`
- 或问题需要跨多个权威源综合（选型/调研/对比/决策）
- **不需要深度研究**：单点事实、不依赖时效数据、"个人项目用 React 还是 Vue" → 降级为快速回答

---

## 架构：四阶段递归研究 + 两道覆盖率闸门 + 方向确认

```
Phase 0: 方向确认 🔴      → 展示研究计划 + 60s 窗口；方向已锁则跳过
Phase 1: 好奇循环        → 广泛探索，建立生态图，发现分类维度
Phase 1.25: 新鲜度扫描 🔴→ 强制闸门①：搜当月新发布实体，防榜单滞后遗漏（增量）
Phase 1.5: 时效性校验 🔴  → 强制闸门②：核心数据是否最新 + 厂商覆盖是否完整（存量+增量）
Phase 2: 深度推理循环    → 换框自审（含厂商覆盖） + 盲区检测 + 形成可证伪判断
Phase 3: 同行评审        → 独立 sub-agent 对抗验证
Phase 4: 采纳与优化      → 逐条修正评审意见
Phase 5: 输出报告        → 顾问级分析报告
```

**递归关系**：Phase 2/3/4 可 spawn 子 Phase 1（好奇循环）和子 Phase 2（深度推理）。递归深度硬上限：fast ≤1, standard ≤2, deep ≤3。

**Phase 1.25 和 Phase 1.5 永远不可跳过**——这是从 Phase 1 进入 Phase 2 的两道硬闸门。Phase 1.25 防「新生事物未入榜单」（增量盲区），Phase 1.5 防「已知数据过期 + 厂商覆盖不全」（存量盲区）。

---

## Phase 0：方向确认（Direction Gate）🔴

**目标**：研究开始前展示计划 + 60s 窗口，让用户纠正方向。避免「跑完发现不是用户要的」。

### 先自判方向

一个问题：*有没有只有用户能拍板、且会改变「我研究什么」的方向选择？*

- **方向已锁 → 不展示计划清单、不起计时器**：1-2 行压缩陈述「我理解的问题 + 切入角度」后直奔 Phase 1。命中任一即锁：
  ① 主题 + 要查的具体现象/主张已给全、无实质分叉、无缺会改变全局的约束（诊断/核实/查现状类，如「X 为什么 Y」「X 是不是真的」「X 现状如何」；⚠️「现象/主张唯一」≠「归因框架/核实口径唯一」——归因角度、核实口径或覆盖范围多义、选哪个会改写「研究什么」→ 判待定）；
  ② 输入含「立即开始 / 马上开始 / 直接开始 / 不用确认 / 别问了 / skip」。

- **方向待定 → 展示计划 + 走 60s 窗口**。命中任一即待定：survey/选型类（含「现状如何 / 有哪些 / 盘点」式伪装句）、研究范围或口径需用户定、问题有多种解读、缺会改变全局的约束。

判不准默认「待定」——错锁方向的代价远大于多等 60s。

### 计划展示（仅待定时）

AI 输出：我理解的问题 / 初步判断（可推翻）/ 3-5 研究角度 / 预期深度（fast·standard·deep）/ 你会得到什么 / 你能改什么。如有澄清问题，合并提出（≤3 个）。

### 60s 窗口

打 `⏱️ 60s 后自动开始；想改现在说，或回「立即开始」直接跑`，跑 `sleep 60`（`run_in_background:true`）让出回合，记住其 task id。

- **窗内用户回复** → **先用该 id `TaskStop` 杀计时器**（漏杀 = 幽灵计时器稍后二次开跑，**头号 bug，先 kill 再动**），再按意图走：改 → 重规划（最多 1 个活跃计时器）/ 批准·立即开始 → Phase 1 / 不做 → 停。
- **计时器到点或沉默重唤** → Phase 1；先确认本轮未开跑（无本轮 researcher 派工 / Dispatch Gate 记录），已派则忽略防重复。未答澄清按模板默认走，报告末尾标 `clarify_assumed`。
- **环境不自动重唤** → 降级为「等你批准」（= 老行为，绝不卡死）。

方向已锁 / 批准 / 超时后自主推进，不再打断（例外仅：密钥泄露 / 武器 / 工具完全失败）。

---

## Phase 1：好奇循环（Curiosity Loop）

**目标**：广泛探索，不设预设分类框架。让搜索结果揭示「这个领域怎么分」。

**角色分工**：
- 主 agent = 协调者：发问、派工、综合、判断
- sub-agent = 执行者：搜、读、摘录（不带判断）

**每轮流程**：
1. 发问：生成 3-6 个 curiosity_questions（无产品名/公司名）
2. 标注类型：[core]（影响中央判断）→ 派 2 个独立 sub-agent；[fill]/[verify] → 派 1 个
3. 🔴 同帧并行 spawn 所有 deep-research 工具调用
4. 综合返回 → 去重、交叉验证、更新框架
5. 判断边际收益 → 继续/收敛

**🔴 curiosity_questions 必须覆盖的维度（每轮至少各 1 条）**：
- **无偏探索**：描述性语言，不带产品名/公司名（如「行业头部」「新兴挑战者」）
- **维度穷举**：按 Phase 1 已发现的**每个分类维度**，穷举该维度下的实体，逐个搜其最新状态。如研究「编码大模型」→ 维度含「厂商」（OpenAI/Anthropic/DeepSeek/…）、「模型代际」（GPT-5.5/Claude Opus 4.8/…）、「开源vs闭源」等；研究「数据库选型」→ 维度含「类型」（SQL/NoSQL/图/时序）、「部署模型」（自托管/云/Serverless）等。🔴 **维度清单从当期研究动态发现，不预设、不硬编码、不跨研究复用**。目的：防榜单只覆盖部分维度/实体导致系统性遗漏
- **反方/失败案例**：主动搜索反对观点和负面评价

**收敛条件**：连续 2 轮未发现新分类维度 / 新实体全 filler / 达硬上限（fast=1, std=3, deep=5 轮）。

---

## Phase 1.25：新鲜度扫描（Freshness Sweep）🔴

**目标**：Phase 1 收敛后，强制搜索「当月/近期新发布」的实体。**防榜单滞后——静态榜单从更新到收录新产品有 2-4 周延迟，刚发布的模型/产品会系统性遗漏。**

**为什么需要这道闸**：Phase 1 的 curiosity_questions 偏总结性（「best X 2026」「top Y comparison」），搜索引擎返回的是已有排名的榜单页面，不是新闻式结果。Phase 1.5 只检查已知数据的时效，不探测未知实体的存在。二者都防不住「6 月 20 日发布的模型，6 月 30 日做研究」。

### 扫描 query（必须包含）

1. `"<主题领域> new release <current_month> <current_year>"` （如 `"coding model new release June 2026"`）
2. `"<主题领域> latest launch just announced <current_year>"`
3. `"<主题领域> <current_month> <current_year> 最新发布"` （中文源）
4. 🔴 **逐维度穷举 + 无偏搜索**：基于 Phase 1 发现的分类维度，逐维度搜该维度下实体的最新状态（如研究编码模型 → 搜各厂商最新模型；研究数据库 → 搜各类型最新版本）。同时加一条无偏 query 捕获 Phase 1 完全未覆盖的新维度/新实体：`"<主题领域> new entrant emerging <current_year>"`

### 判定

- **发现 Phase 1 未覆盖的新实体** → spawn 子 Phase 1 深挖该实体，然后重新判定收敛
- **未发现新实体** → 通过，进入 Phase 1.5
- **硬上限**：≤2 轮补搜

### 输出

`{new_entities_discovered[], sub_phase1_spawned: true/false, sweep_passed: true/false}`

---

## Phase 1.5：时效性校验（Freshness Gate）🔴

**目标**：Phase 1 收敛后、进入深度推理前，强制验证所有核心数据的时效性。**这是硬闸门，Phase 1.5 不通过不得进入 Phase 2。**

**闸门判定（逐条检查每个 [core] finding）**：

| 检查项 | 判定标准 | 不通过动作 |
|---|---|---|
| **日期锚定** | `source_date` 在 freshness 窗口内（见 Tier 映射表） | spawn 子 Phase 1，query 强制带 `current_month current_year` |
| **代际检查** | 结论引用的产品/模型/版本是否为**当前最新代际**？（如：还在引用 o3 但 o4 已发布 → 不通过） | 搜索「X latest version/release 2026」确认最新代际，替换过时引用 |
| **竞品对称性** | 所有核心竞品的基准数据是否来自**同一时期**？（如：A 公司用 2026 Q2 数据，B 公司用 2025 Q1 数据 → 不通过） | 统一拉齐到同一时间窗口重新搜索 |
| **反方时效** | 反方/批评观点是否也覆盖了**最新版本**？（如：批评的是 2025 版产品，2026 版已修复 → 不通过） | 搜索「X criticism/issues 2026 latest」 |
| **🔴 维度穷举完整度** | Phase 1 发现的**每个分类维度**下的实体是否已穷举？是否存在某维度下漏了重要实体？（如研究编码模型，「厂商」维度有 OpenAI 但漏了 GPT-5.5，「代际」维度有 Opus 4.8 但漏了 Fable 5 → 不通过；研究数据库，「类型」维度有 SQL 和 NoSQL 但漏了图数据库 → 不通过） | 从 Phase 1 findings 提取分类维度清单（动态构建），逐维度检查实体穷举是否完整；缺则补搜 |

**通过标准**：所有 [core] finding 通过上述 5 项检查。最多 2 轮补搜；2 轮后仍有不通过的，标记为 `[stale-gap]` 并在报告中显式声明「以下结论基于旧数据，置信度降级」。

**输出**：`{stale_findings[], replacement_findings[], gaps_marked_stale[], dimension_coverage: {discovered_dimensions[], entities_per_dimension{}, missing_entities[]}, gate_passed: true/false}`

---

## Phase 2：深度推理循环（Deep Reasoning Loop）

**目标**：带着 Phase 1 的生态图，做深度分析——形成判断、检测盲区、交叉验证。

**🔴 换框自审（强制首步，不可跳过，必须输出以下 checklist 结果）**：

**框架自审**：
1. 我目前用什么分类框架理解这个领域？
2. 如果换一个学科/市场/视角，会用什么不同框架？
3. 有没有「我根本不知道存在的类别」？
4. 我的分类框架本身有哪些盲区？

**🔴 维度穷举自审（强制，防「榜单里没有就以为不存在」。分类维度 + 实体清单均从当期研究动态构建，不预设）**：
5. 从 Phase 1 + Phase 1.25 的 findings 中**提取所有已发现的分类维度**（如研究编码模型 → 厂商、代际、产品形态等；研究数据库 → 类型、部署模型等 → 什么维度由研究主题决定）→ 逐维度检查：该维度下的实体是否已穷举？
6. 每个维度下，有没有重要实体在 Phase 1 榜单中完全缺席？（可能原因：搜索遗漏 / 榜单未收录 / 该实体确实不存在。前两种 → spawn 子 Phase 1 补搜）
7. 有没有已发现实体的最新代际/版本缺失？（如找到了 GPT-5.4，但 GPT-5.5 已发布 2 个月 → 盲区）
8. 有没有新维度/新实体未在任何 Phase 1 search result 中出现？（Phase 1.25 的无偏搜索应捕获）→ 若有，spawn 子 Phase 1 深挖
9. 对于所有缺失，Phase 1.25 和 Phase 1.5 是否已补全？若未 → spawn 子 Phase 1 补搜

**输出格式**（必须显式写在推理中，不可跳过）：
```
换框自审结果:
- 当前框架: <框架名>
- 替代框架: <框架名>
- 未知类别风险: <有/无，描述>
- 已发现分类维度: [维度1 (实体数: N), 维度2 (实体数: M), ...]
- 各维度缺失实体: [{维度, 缺失实体, 原因: <搜索遗漏/榜单未收录/确实不存在>}]
- 代际/版本缺失: [{实体, 已有版本, 最新版本, 发布日期}] → 补搜: <是/否>
- 全新维度/实体（Phase 1.25 无偏搜索发现）: [...]
- 盲区汇总: <一句话>
```

**每轮流程**：
1. 换框自审（含维度穷举） → 产出 {盲区, 矛盾, 弱支撑, 维度缺失, 代际缺失}
2. 标注类型：[core] → 2 sub-agent，其余 → 1
3. 🔴 同帧并行 spawn 子好奇循环 + 子深度推理 + 验证搜索
4. 综合 → 更新 Central Thesis + 反事实 + 置信度
5. 自我拷问 → 继续/收敛

**收敛条件**：盲区填补完毕 + 判断可证伪 + 每个核心结论 ≥2 独立来源 + 非 echo chamber。

---

## Phase 3：同行评审（Peer Review）

**目标**：对抗性验证。🔴 必须派独立 sub-agent（fresh context），主 agent 不得自审。

**推荐并行模式**（std/deep）：同帧 spawn 2-3 个 reviewer——
- Reviewer A：证据可靠性
- Reviewer B：逻辑与框架
- Reviewer C：覆盖度与遗漏

**输出**：`{attacks[], echo_chamber_detected, missing_perspectives[], overall_verdict}`

---

## Phase 4：采纳与优化（Adopt & Optimize）

逐条处理评审意见。critical+severe 必须处理，moderate 建议处理或标 known-limitation。
需补证据/重新论证的 → 🔴 同帧并行 spawn 子循环。

---

## Phase 5：输出报告

从研究者切换到写作者。不是 findings dump——有 Central Thesis、反事实、置信度、下一步。末尾留追问钩子。

---

## 停止条件

### 各阶段收敛

| 阶段 | 收敛条件 | 硬上限 |
|---|---|---|
| Phase 0 | 方向已锁 或 60s 到期 或 用户批准 | — |
| Phase 1 | 连续 2 轮无新维度 或 新实体全 filler | fast=1, std=3, deep=5 |
| Phase 1.25 | 当月新发布实体已扫描 | ≤2 轮补搜 |
| Phase 1.5 | 所有 [core] finding 通过 5 项检查（含厂商覆盖完整度） | ≤2 轮补搜 |
| Phase 2 | 盲区填补 + 判断可证伪 + 证据 ≥2 源 | ≤4 子循环 |
| Phase 3 | overall_verdict: pass 或 conditionally_pass | 2 轮 |
| Phase 4 | 所有 critical/severe 已处理 | — |

### 反停止信号（任意阶段命中 → 继续）

| 信号 | 直觉 |
|---|---|
| 发现新分类维度 | 框架在进化 |
| 新证据推翻已有结论 | 结论在变化 |
| 某方向全同向（echo chamber） | 缺反方 |
| 意外发现重要未知实体 | serendipity |
| 核心结论只有 1 个来源 | 需验证 |
| Phase 1.25 检出未收录新实体 | 榜单滞后，需补搜 |
| Phase 1.5 检出 stale finding 或维度穷举不全 | 数据过时或某维度实体遗漏，需补搜 |

---

## Workflows 编排铁律

```
🔴 每个 Phase 内，所有独立子任务必须同帧并行 spawn，不串行。

🔴 冗余策略：
  [core] 方向（影响 Central Thesis）→ 2 个独立 sub-agent 交叉验证
  [fill]/[verify] 方向 → 1 个 sub-agent
  每 Phase 每轮 [core] 方向 ≤2 个（避免冗余爆炸）

🔴 主 agent 只亲自做 3 件事：发问、换框自审、综合判断。
  其余一切搜/读/摘录/验证 → sub-agent。
```

---

## deep-research 工具契约

```yaml
deep-research:
  description: 对指定主题执行一轮聚焦搜索+阅读，返回结构化发现。
  input:
    topic: string
    scope: broad | focused
    phase: curiosity | deep_reasoning | peer_review | optimize
    curiosity_questions: [string]     # 🔴 必填
    known_dimensions: [{name, entities}]
    known_entities: [{name, type, needs_deep_dive, reason}]
    max_rounds: number
    freshness: "month" | "week" | "year"
  output:
    findings: [finding]
    discovered_dimensions: [{name, description, confidence, entity_count}]
    discovered_entities: [{name, type, confidence, impact, needs_deep_dive}]
    answered_questions: [string]
    new_questions: [string]
    framework_stable: boolean
    marginal_value: {new_dimensions, decision_affecting, filler}
    should_continue: boolean
```

---

## Tier 映射与并行度

| Tier | 触发 | freshness 窗口 | P1 轮数 | P1 并行 | P2 子循环 | P3 | 总搜索 | 字数 |
|---|---|---|---|---|---|---|---|---|
| **fast** | "快速/简要" | **1 年内** | 1 | 1-2 | 0 | 跳过 | ≤5 | 1000-2500 |
| **standard** | 默认 | **3 个月内** | 3 | 3-5 | ≤2 并行 | 1-2 reviewer | ≤20 | 2500-5000 |
| **deep** | "深度/全面" | **1 个月内** | 5 | 5-8 | ≤4 并行 | 2-3 reviewer | ≤40 | 5000-12000 |

> 🔴 **freshness 窗口是 Phase 1.5 闸门的判定基准**。所有 [core] finding 的 `source_date` 必须在此窗口内，否则不通过。

---

## 黄金法则

| # | 法则 | 一句话 |
|---|---|---|
| 1 | **不编造** | 每个数字有可追溯来源。不确定就说"不确定" |
| 2 | **找反方** | 主动搜索反对观点和失败案例。一面倒 = 没做完 |
| 3 | **标缺口** | "我不知道 X → Y 判断置信度只到 Z" |
| 4 | **给判断** | 有 Central Thesis、有置信度标注、有可执行下一步 |
| 5 | **可证伪** | 每个核心结论附带"什么情况下这个结论是错的" |
| 6 | **有时效** | 每个 finding 标注 source_date |
| 7 | **并行优先** | 所有独立子任务同帧发出——不等不串行 |
| 8 | **核心才冗余** | [core] 方向 2 个交叉验证，其余 1 个——不浪费 |
| 9 | **主 agent 不搜** | 搜/读/摘录全部走 sub-agent。主 agent 只发问、换框、综合 |
| 10 | **防榜单滞后 + 维度穷举** | Phase 1.25 强制搜当月新发布；Phase 1.5 + Phase 2 逐维度穷举实体。分类维度从研究动态构建，不预设。不假设「榜单没有就不存在」 |

---

## 反模式

| ❌ | ✅ |
|---|---|
| 主 agent 自己搜/读 ≥3 个来源 | 调 deep-research 或 spawn sub-agent |
| 主 agent 串行一个一个发 sub-agent | 同帧并行 spawn——所有独立子任务一起发出 |
| 所有子方向无差别派 2 个 | [core] 2 个交叉验证，其余 1 个 |
| 搜索 query 含具体产品/公司名 | 描述性语言：「行业头部」「新兴挑战者」——但 curiosity_questions 必须同时包含厂商穷举 query |
| Phase 2 跳过换框自审 | 换框自审是强制首步，必须输出 checklist（含厂商覆盖检查） |
| Phase 2 换框自审只问框架不问维度穷举 | 换框自审必须逐维度检查实体穷举——「榜单里漏了某个重要实体，是没搜还是不存在？」 |
| Phase 1.25 新鲜度扫描被跳过 | Phase 1.25 和 Phase 1.5 同为硬闸门，不可跳过 |
| Phase 3 主 agent 自审 | 独立 sub-agent（fresh context） |
| 发现盲区标 `[gap]` 跳过 | spawn 子好奇循环填补 |
| deep-research 一键生成报告 | 它返回发现，综合和判断在主 agent |
| 一面倒无反方 | 每条核心方向 ≥1 反方 |
| Sub-agent 返回 raw dump 原样转用户 | 主 agent 提炼 → 综合 → 判断 → 呈现 |
| 搜索 query 不限制时效 / 用裸年份 `2025` | 所有 query 带 `current_month current_year`；Phase 1.5 闸门强制校验 |
| Phase 1 收敛后直接进入 Phase 2 | 必须先过 Phase 1.25（新鲜度扫描）+ Phase 1.5（时效性校验）两道闸门 |
| 信任单一榜单覆盖全维度 | 每个榜单都有收录延迟和选择性偏差；Phase 1.5 + Phase 2 逐维度验证穷举完整度 |

---

## Researcher Prompt 模板（deep-research 工具内部使用）

> 主 agent 调用 deep-research 工具时，工具内部自动使用此模板派 researcher sub-agent。

```
你是 deep-research researcher sub-agent。唯一任务是研究以下 sub-question 并返回结构化 JSON。

## Sub-question
<填入 sub_Q>

## Query seeds (每个都搜，含至少1条反方query + 🔴 所有 query 强制带时效性约束)
1. "<正向搜索query> + latest + current_month current_year"
2. "<正向搜索query> + latest + current_month current_year"
3. "<反方/批评/失败案例 query> + current_month current_year"
4. "<时效性query: topic + latest/recent + current_month current_year>"

🔴 **时效性约束不是可选的**——所有 query 必须包含 `current_month current_year` 或等价时间锚点。
禁止使用裸年份（如 `2025`）作为唯一时效性约束——必须精确到月。

## Budget
≤20 WebSearch, ≤12 WebFetch。优先官方来源。

## Tool Fallback
anysearch 失败 → 重试1次 → WebSearch → WebFetch。禁止凭记忆补 finding。

## Extract 后处理
1. URL 去重
2. 相关性筛选（每个 extract ≤3 句 why relevant，写不出的丢弃）
3. 去重后 < tier_min 则补搜

## Output schema (🔴 严格按此格式输出JSON)
{
  "sub_question": "<sub_Q原文>",
  "findings": [{
    "finding_id": "F-<编号>-<序号>",
    "claim": "<一句话事实主张，≤200字>",
    "evidence_span": "<来源原文关键句，≥50字>",
    "source_url": "<URL>",
    "source_type": "official_doc|academic|financial_media|industry_blog|community",
    "source_date": "YYYY-MM-DD",
    "confidence": "HIGH|MEDIUM|LOW",
    "challenges_thesis": true/false
  }],
  "source_funnel": {"identified": N, "screened": N, "extracted": N, "included": N, "exclusion_reasons": {}},
  "dedup": {"before": N, "after": N},
  "query_resilience": {"total": N, "succeeded": N, "failed": N, "threshold_met": true},
  "key_insight": "<1句话总结>"
}
```

---

## 红线

- ❌ 编造数字或来源 · ❌ 报告含 token/密钥 · ❌ 付费墙/登录页/武器/非法内容 · ❌ 一面倒

## 参考文件

[`ARCHITECTURE_v6.md`](ARCHITECTURE_v6.md) 架构 · [`PLAYBOOK`](PLAYBOOK.md) 操作手册 · [`WRITING_STYLE`](WRITING_STYLE.md) 写作 · [`REPORT_TEMPLATES`](REPORT_TEMPLATES.md) 骨架 · [`RUBRIC`](RUBRIC.md) 评分 · [`CHANGELOG`](CHANGELOG.md) 版本
