# CHANGELOG

## v5.5 — 2026-06-01 (计划 gate：60s best-effort 自动推进 + 立即开始)

**触发**: Step 0 是"必须等批准"的无限等待，人走开就永久卡住。要 60s 窗口（超时自动跑 / 期间可改 / 可即时开始），及 `立即开始` 一词直接跳过窗口。

**核心洞察**: Claude Code 用户输入驱动、非时钟驱动，无原生"空闲超时"（AskUserQuestion 无超时 / 无 UserIdle hook / Cron 不精确）。唯一能做"沉默自动推进"的是后台计时进程退出触发 re-invoke——标准 build 不保证。故设计为 **best-effort + 优雅降级**：自动推进是增强，缺失时退化为"等批准"（老行为），任何环境都不丢功能、不卡死。

**修改**:

1. **SKILL.md Step 0**：「必须等批准」→ 审批窗口（展示计划 → `sleep 60`（run_in_background）让出 → 超时重唤自动跑）。三条正确性：① 用户窗口内插话**先 `TaskStop` 掉计时器再行动**（防幽灵计时器二次开跑，头号 bug）② 重唤先查研究是否已开始、防重复 ③ 不重唤则降级为等待。
2. **立即开始**：commands/deep-research.md 解析 `$ARGUMENTS` 含「立即开始/马上开始/直接开始/不用确认/skip」→ 跳过窗口直奔执行；argument-hint 加 `[立即开始]`。
3. **PLAYBOOK** Phase 1（Standard/Deep）措辞对齐；header v5.4→v5.5。

**不做**: 不加 PreToolUse 硬拦（防错方向）；不引入 hook/Python（timer 即 sleep，立即开始命令级解析够）；description frontmatter 不动（避免臃肿）。

## v5.4 — 2026-05-27 (执行纪律强化：硬门禁 + 砍仪式)

**触发**: AI 编程工具深度研究真实执行（2026-05-27）。deep tier 任务中主 agent 0 个 sub-agent spawn、全自己搜、跳 REFLECT → 漏掉 Opus 4.7 / GPT-5.3-Codex / DeepSeek V4 等关键最新数据。事后诊断：v5.3 的机制本身够用，但 agent 进门后第一件事不是遵守规则，而是按自己的理解开干。

**核心洞察**: 加更多 checklist 治不了「不读规则」的病。Agent 在顶部最可能停的位置放最少、最硬的规则才可能被遵守。

**修改**:

1. **新增「执行前硬门禁」**（SKILL.md 顶部，触发条件之前）
   - 3 行：spawn 几个 researcher？/ 模板准备了？/ freshness 参数用了？
   - 任一答不出 → 立即停，不进入 Phase 0
   - 位置优先于所有其他内容（agent 进门第一眼撞上）

2. **删除 Pre-Report Freshness Audit JSON**（~32 行）
   - 放在 REFLECT 之后 → agent 已经跳过 REFLECT → 不可能执行
   - 时效性检查保留在 REFLECT 第 1 项（执行时机正确）
   - 覆盖度检查保留在 REFLECT 第 2 项 + Discovery Bootstrap 维度 C/E

3. **Discovery Bootstrap 增加维度 E** — 开源/独立/非商业生态
   - 触发：R02 实测Latest Releases Sweep 抓到 17 个主 agent 完全不知道的新玩家（多来自开源/独立生态）
   - 生态图从 A-D 四维度 → A-E 五维度

4. **新增「最高优先级反模式」**（铁律下方，执行流程之前）
   - 一行：主 agent 自己搜 = 研究作废。发现即停，改派 sub-agent 重跑。

5. **同步清理**
   - Frontmatter 18→17 字段（移除 phase_freshness_audit）
   - 黄金法则 #6 去掉「交付前过 Freshness Audit」引用
   - 反模式表移除 Pre-Report Self-Check 条目
   - Phase 完整性表移除 Pre-Report Freshness Audit 行

**设计原则**:
- 不加水，只改结构：信息顺序从「参考文档式」→「执行门禁式」
- 最小可行规则放在最大可见位置
- 砍掉放错位置的检查（及时性 > 完备性）

**预期效果**:
- 执行前自检完成率：0% → 目标 >90%（3 行门禁在顶部，跳过难度 > 遵守难度）
- 总行数：2337 → 1674（-28%）。CHANGELOG 691→93（历史归档为摘要表），PLAYBOOK 272→218（去重+修复过期引用），SKILL.md 243→232

**已知风险**:
- 硬门禁对强推理模型（Opus 4.7）效果待验证（对话记录中的违规者就是 Opus 4.7）
- 如果 agent 连顶部 3 行也跳过，需要更根本的机制（如 hook-level enforcement）

## v5.0.1 — 2026-05-27 (post-release: L1 regression 问题集 + A 股实测评估)

**触发**: 用户要求评估 deep-research skill 价值，对比普通对话 vs deep-research 同 prompt（A股基金投资研究）产出。随后要求优化并提交。

**A 股实测发现**（degraded 状态仍全面碾压）:
- 即使 anysearch 限流导致 3 个 researcher 失败（degraded: true），deep-research 输出在估值分析、风险覆盖、来源可追溯、自我认知等维度仍远超普通对话
- 核心差异：Central Thesis 驱动 vs 数据罗列、反方视角 vs 无风险分析、信息缺口诚实 vs 无自我认知
- 详见评估：用户桌面 `desktop/research/` 两份对比报告

**修改**:
1. **L1 regression 问题集建成**（`eval/questions/INDEX.md`）
   - 12 题：4 模板（survey/compare/decide/howto）× 3 难度（easy/medium/hard）
   - 每题固定 pass criteria，禁止回调
   - 套件通过条件：≥10/12 overall ≥0.75，0 题 <0.55，平均 factuality ≥0.80
   - baseline 尚未运行
2. **EVAL.md 更新**: L1 状态从"未提供完整题包"→"问题集已建成，待 baseline"

**未改**:
- SKILL.md/PLAYBOOK.md 不做修改（v5.0 Pareto 简化后已足够紧凑，triage 新增内容与简化哲学冲突）

---

## 历史版本摘要（v1.0 — v4.9.3，2026-05-26）

| 版本 | 关键改动 | 教训/结果 |
|---|---|---|
| v4.9.3 | 真实执行数据驱动：researcher prompt + output schema；frontmatter ~100→15 字段 | Phase 完成率 45%，5/6 researcher 返回自然语言而非 JSON |
| v4.9.1 | 独立 sub-agent 审计后降级 v4.8/v4.9；撤 PRISMA/Cochrane/Kappa 宣称 | 5:1 抽审 N 远不达 Kappa 推断要求，「合规」宣称是学术装饰 |
| v4.9 | Evidence Quality 借鉴（后被降级） | 预防性升级，无具体 bug 驱动 — over-engineering 信号 |
| v4.8 | Stage-Gate 10 闸门系统（后被降级） | 复合通过率 19.7%，audit-of-audit 嫌疑 |
| v4.7 | Phase 4.8 Logic Audit 6 项 | 真实 bug：推荐 Claude Code 时漏了 Opus 4.7 |
| v4.5 | 知识过时假设 + 不打断纪律 + Phase 2.6 Latest Releases Sweep | R02 实测：Sweep 抓到 17 个全新玩家，含 DeepSeek V4 |
| v4.4 | Dispatch Gate 事前拦截 | DeepSeek spawn 0 researcher 而 Claude spawn 5 — 需要事前拦截 |
| v4.3 | Discovery Bootstrap + Dependency Mining（数据驱动防遗漏） | 「谁被漏了」应由数据回答，不由模型记忆 |
| v4.2 | Invisible Layer Check 防 Frame Blindness | 生态题因按产品形态分解遗漏 DeepSeek（基础设施层） |
| v3.2 | HDIR 五轮循环（后被 v2.7.3 撤） | LLM judge：HDIR 仍是 ReAct+Reflexion 包装，原创性 0.78 |
| v3.1 | 顾问级输出：Central Thesis + 三幕结构 + 顾问语气 | 从 findings dump 升级为顾问分析 |
| v3.0 | Info Saturation Protocol：停止条件从「量」→「质」 | 5 项 Saturation Checklist |
| v2.9 | 搜索量从「配额制」→「按需+饱和制」 | deep ≥80 搜索是配额制错误，修正为 ≥30 |
| v2.8 | 搜索量对齐 GPT DR（200-800 次） | 移除 URL 硬上限，用 info saturation 替代 |
| v2.7 | dev-time hook harness + fixture 自测 | 从「逻辑正确」→「实测正确」 |
| v2.4 | R03 实测后系统补强：反方搜索/Extract Gate/AI confabulation lint | 主 agent 4 项 metrics 50%+ 谎报 |
| v2.3 | Research-Informed Quality：10 项论文依据 enhancement | 修 5 处错误 attribution（CHI→IUI, DeepHalluBench 不存在等） |
| v2.0 | Museum Grade：八阶段 + supervisor-researcher 双层 | 首个完整工作流 |
| v1.0 | 初始六阶段：Plan→Search→Read→Reflect→Synthesize→Report | 基于 9 篇行业资源 |

完整细节见 git history `git log -- skills/deep-research/CHANGELOG.md`。
