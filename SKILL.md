---
name: orion-research
description: >
  When the user needs cross-source investigation, sub-question decomposition,
  conclusion verification, or source conflict handling. Trigger words: 猎真
  (unconditional deep-research pass), 深度调研, 调研, 研究, investigate, research.
  Use when model knowledge alone is unreliable, or when source timeliness,
  traceability, cross-validation, and evidence quality materially affect the answer.
  For simple factual queries or single-authoritative-source questions, use plain
  web search instead — do NOT trigger this skill.
license: MIT
metadata:
  version: 3.4.0
---

# Orion Research (猎真) — Deep Research Skill

## Overview

深度调研技能，基于 **SaC（Search as Code）范式**：用代码编排完整搜索 pipeline（fanout → search → extract → rerank → dedupe），只把压缩后的 top-N 结果带回上下文。

**核心原则**：搜索是可编程的，不是黑盒 API 调用。多源验证、追溯原始来源、标注时效性、处理冲突。

**跨 Agent 兼容**：本 skill 遵循 [Anthropic Agent Skills 标准](https://agentskills.io)（SKILL.md + references/）。方法论与工具调用分离——所有"做什么"是通用的，"用什么工具"按 Agent 环境映射（见 [Agent 兼容矩阵](references/agent-compatibility.md)）。

## Step 0 分流门（skill 加载后第一个动作）

**直通词「猎真」**——消息含此词 → 无条件深度通道（7 步），跳过出口判定与三问，零歧义。

**考古模式（直通词专属子路径）**——直通后若消息+当前任务语境提取不出明确问句（问题在语境里、用户没成形）→ 先造问题再答：
1. 数据源 = 当前对话上下文（本任务会话内），不跨会话检索历史——历史残影带偏问题树且拖慢响应
2. 用批判性思维扫语境找张力点（矛盾点、悬而未决的分歧、用户真正的关注点）→ 每个张力点转成一个可检索子问题，共 3±2 个
3. 剪枝：答案不改变用户任何行动的子问题砍掉
4. 出口分置信度：线索收敛（多信号指向同一疑问）→ 展示问题树同时直接开跑；线索发散 → 只出问题树等用户拍板，不跑
5. 收敛：分支答案互证 → 合成主答案；分支冲突 → **冲突即答案**——暴露用户真正纠结的矛盾，对矛盾做 trade-off 分析，进输出契约「矛盾记录」

考古是纯推理步骤，零额外工具调用；产出问题树后无缝接入 7 步的拆解→检索→收敛。

**出口 0：非调研加载**——任务是治理/运维/读文件（skill 合规检查、审计、抓取工具调用）→ 不启用任何流程，仅将 references/scripts 当工具箱，用完即走。多数历史加载属此类，禁止为仪式跑流程。

**快速通道**——三问全过 → 单引擎单查直答，不走 7 步：
1. 单一子问题（一问可闭环，无需拆解）
2. 答错后果限于当次对话（不牵动决策/产出）
3. 无冲突裁决预期（非争议/多方利益话题）

**深度通道**——任一命中 → 完整 7 步：
- ≥2 个相互依赖的子问题
- 答案影响决策（选品/定价/架构/对外发布）
- 预期来源冲突或需可信度分级
- 用户明说「深度调研/认真查/帮我验证」（或直通词「猎真」）

**升级规则**：快速通道跑完发现来源打架或答案牵动决策 → 立即转深度，已查结果作为子问题输入不浪费。

## When to Use

- 用户说"深度调研"、"研究一下"、"调查"、"investigate"、"research"
- 问题需要跨来源交叉验证（时效性、准确性会影响结论）
- 模型已有知识无法可靠回答，或需要最新数据
- 任务涉及竞争性分析、工具对比、市场研究

## When NOT to Use

- 简单事实查询（直接回答即可）
- 单一权威来源可回答的问题
- 仅总结用户提供的材料
- 使用普通 web search 更合适的情况

## 模式选择（ReAct vs CodeAct）

**先用 30 秒判断复杂度，不要无脑走完整编排：**

| 判断维度 | ReAct（轻量） | CodeAct（完整编排） |
|---------|-------------|-------------------|
| 子问题数 | 1 个 | 2-5 个 |
| 来源要求 | 单一权威可答 | 需跨源交叉验证 |
| 事实类型 | 静态（定义/理论） | 动态（版本/价格/CEO） |
| 冲突风险 | 低 | 高（可能矛盾） |

**ReAct 模式**：直接搜索 2-3 次 + 判断，不写编排代码、不做重排，报告精简版（结论 + 来源 URL）。

**CodeAct 模式**：走完整五段式管道（见 references/sac-search-orchestration.md）。

**判断标准**："一次搜索就够 + 模型已有知识能兜底" → ReAct；"多源验证 + 时效性 + 冲突处理" → CodeAct。

## 工作流（7 步）

### Step 1: 拆解 + Query Rephrasing

- 将问题拆解为 2-5 个子问题，按信息增益排序
- **差距型调研识别**：问题含"区别/优势/提升/现成方案/本质"信号 → 固定加入子问题"现有方案已经能做什么？X 的真正增量是什么？"（见 references/orion-research-user-patterns.md）
- 每个子问题生成 1-3 个查询变体（同义/英文/加限定词）
- **强制输出（写进报告"调研过程"）**：`原查询 → 变体1 / 变体2` 对照表
- 输出：子问题列表 + 查询组合（含变体）

### Step 2: SaC 编排搜索（核心）

- 用**代码执行环境**写一段 Python 编排搜索 pipeline
- **五段式管道**（模板见 `references/sac-search-orchestration.md`）：
  - **2a 引擎选择**：按子问题特性决策——核心→深度搜索引擎，一般→免费搜索，中文→中文搜索引擎（百度等），语义→语义搜索
  - **2b 统一 rerank**：所有引擎结果汇合 → 重排打分 → 过滤低分 → 排序
  - **2b.5 规则 filter**：snippet 过短/SEO 农场/关键词缺失 → 降权剔除（补语义重排盲区，不用 LLM）
  - **2c 内容形态工具**：验证原文→单页抓取 / 反爬→浏览器渲染 / 全站→整站抓取
  - **2d dedupe + top-N**：域名去重，只回压缩结果
  - **2e parse_field**：对 top-3 抓原文抽结构化字段（定义/时间/关键数字/观点），字段可追溯到来源 URL
- **限流铁律**：遵守各 API 速率限制 → 请求间必须 sleep；同一 API 绝不并发
- 控制：查询 ≤15 个，重排 API 一次调研只调 1-2 次

> 🔧 **Agent 工具映射**：代码执行（Hermes `execute_code` ≈ Claude Code bash ≈ OpenCode tool_code），搜索（各 agent 内置 web search / SerpAPI / Tavily）。完整映射见 [Agent 兼容矩阵](references/agent-compatibility.md)。

### Step 3: 动态搜索决策（Gap Analysis）

- 评估 top-N 结果质量：
  - 所有子问题有可靠证据 → 停止，进入 Step 4
  - 存在缺口 → 基于缺口定向补搜（最多 3 轮）
  - **对比类缺口**：差距型调研只有 X 的介绍、缺现有方案对比 → 补搜"X vs Y"/"X alternative"
- **不要盲目搜满固定轮数** — 质量够就停，省 token 省配额

🔴 **CHECKPOINT（强制）**：Step 3 结束后，必须显式评估「每个子问题是否有 A/B 级证据支撑」。有缺口 → 回到 Step 2 换引擎补搜；全部有证据 → 才允许进入 Step 4。不得跳过此判断直接写报告。

### Step 4: 验证（不变量 + Gotcha）

| 不变量 | 检查 |
|--------|------|
| 搜索 ≠ 证据 | 结果多不代表证据强 |
| 单源 ≠ 验证 | 一个来源再权威也要标注 |
| 转载 ≠ 独立 | 追溯引用链，按 source family 计数 |
| 冲突必须可见 | 不得静默选择其中一个 |
| 引用 ≠ 支撑 | claim 必须被 citation 原文支持 |
| 时效必须验证 | 动态事实标注时间戳 |

### Step 5: 报告

- 完整报告 = 核心结论（带可信度）+ 证据 + 来源列表（每行含可点击 URL）+ 冲突记录 + 分析过程
- **差距型调研**：核心结论第一条必须是 X 的真正价值（解决的痛点，非功能列表）+ 与现有方案的差距
- 必须包含 **Gotcha 检测** 章节（显式引用 G-001/G-012/G-013/G-014/G-017 状态）
- 输出形式按 Agent 环境：是否写盘/入库遵循各环境约定（Hermes 下仅用户明确说"存知识库"才写）

### Step 6: 独立验证 pass（确定性，非 LLM 自评）

报告写完 ≠ 完成。用**确定性规则**独立复核（不是作者自己勾 ✅）：

| 检查项 | 通过标准 | 失败处理 |
|--------|---------|---------|
| 来源可追溯 | 来源表每行有可点击 URL | 缺 URL → 报告不达标，回补 |
| 可信度不虚标 | 每个 A 级结论有 ≥2 个独立 source family | A 级无支撑 → 降为 B |
| 冲突可见 | 有矛盾未标注 = 失败 | 回补矛盾记录 |
| Gotcha 真实执行 | 每个 Gotcha 写"做了什么"而非"应该做" | 空洞勾选 → 回补 |

**禁用 LLM 自评可信度**——同质 LLM 验证无意义。

### Step 7: 搜索策略反馈

调研结束后，记录"引擎-子问题"经验（写文件或记忆系统，按 Agent 环境）：

```
[引擎-子问题匹配] 子问题类型 X → 引擎 Y 有效/无效 → 原因。
```

**只记录两类**：
1. 某引擎对某类子问题**失效**
2. 某引擎对某类子问题**意外有效**

**不记录**：一切正常、按预期工作的调用。

## 反模式（不要这样做）

| 反模式 | 后果 |
|--------|------|
| ❌ 盲搜满固定轮数 | 浪费配额和 token，且不解决真实缺口 |
| ❌ 同 API 并发调用 | 触发 429，全部请求失败 |
| ❌ 跳过重排直接写报告 | 低质量/无关来源混入结论 |
| ❌ 把外部 LLM 回答当搜索源 | 黑盒无法验证来源，G-001/G-014 全部失效 |
| ❌ 所有子问题都走付费深度引擎 | 配额很快烧光，一般子问题用免费层即可 |
| ❌ 结果全量塞进上下文 | 违反 SaC 核心（只回 top-N 压缩结果） |

## 核心 Gotchas

| Gotcha | 触发条件 | 失败表现 | 检查方法 |
|--------|---------|---------|---------|
| **G-001 伪独立来源** | 多结果声称同一数字 | 将转载视为独立证据 | 追溯引用链，按 source family 计数 |
| **G-012 SEO 内容农场** | 搜索结果含大量营销内容 | 将 SEO 内容视为有效来源 | 识别标题党/模板化内容，优先一手来源 |
| **G-013 时效性过时** | 涉及动态事实（定价、版本、CEO） | 使用过时信息 | 检查发布日期，标注时间戳 |
| **G-014 Citation 支撑失败** | 引用了但未验证 | 假设"Citation = 支持" | 抓取原文验证 entailment |
| **G-017 冲突静默** | 不同来源矛盾 | 未标注冲突 | 分类冲突并给出判断 |

## 输出契约

每次调研交付：
1. 核心结论 3-5 条（每条带可信度 A/B/C/D）
2. 来源列表（每行含可点击 URL + 类型 + 可信度）——缺 URL 判不达标
3. Gotcha 检测章节
4. 矛盾记录（不同来源说法差异 + 判断依据）
5. 聊天中：核心结论 +（若入库）文件路径

## 参考文档

- [SaC 搜索编排模板](references/sac-search-orchestration.md) ← 核心，Step 2 用
- [Agent 兼容矩阵](references/agent-compatibility.md) ← 跨 Agent 工具映射
- [来源可信度评估](references/source-evaluation.md)
- [报告模板](references/reporting.md)
- [用户调研模式（差距分析）](references/orion-research-user-patterns.md) ← Step 1/3/5 用
- [故障处理 F1-F7](references/failure-handling.md) ← Step 2 工具失败时用
- [中文调研专项 + 工具参考](references/tool-reference.md) ← 中文生态/辅助脚本
