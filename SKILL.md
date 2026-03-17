---
name: social-pulse
description: >
  Scan social media for rumor signals and assess their strength.
  Use when someone wants to check if a rumor or breaking event
  is being discussed online, gauge public attention level,
  or monitor specific topics for emerging signals. Any language.
version: 1.0.0
metadata:
  openclaw:
    emoji: "\U0001F4E1"
    requires: {}
    optionalBins:
      - agent-reach
    optionalEnv:
      - BRAVE_SEARCH_API_KEY
    homepage: https://github.com/cliffyan28/social-pulse
---

# Social Pulse: Social Media Signal Detection Skill

## When to Use

- User asks "有没有听说…" / "Is it true that..." / "最近关于…的舆情"
- User asks to check if something is trending or being discussed online
- User wants to know "人们怎么说" / "what are people saying about..."
- User mentions 舆情、传言、传闻、rumor、buzz、chatter
- User asks about public reaction or social media attention on any topic

## When NOT to Use (boundary with fact-checker)

- User explicitly requests "fact-check" or "verify if true/false" → guide to fact-checker
- User provides an image/video for authenticity verification → guide to fact-checker
- social-pulse answers "how much are people discussing this online", NOT "is this true or false"

If the user's intent is ambiguous, default to social-pulse (signal detection) and include the fact-checker suggestion at the end.

---

## Stage 0: Input Parsing

### 1. Detect input type

Determine the user's intent and mode:

- **Manual query mode**: User provides a rumor, topic, or question about social media buzz.
  - Examples: "听说台积电要从台湾撤走", "Is OpenAI acquiring Anthropic?", "查一下关于迪拜暴雨的舆情"

Log: `[Stage 0] Input parsing — mode: manual, language: {detected}`

### 2. Extract keywords

From the user's input, extract:

- **Core entities**: people, organizations, locations
- **Event keywords**: action verbs, event descriptions
- **Time constraints**: if the user mentions a timeframe ("today", "几个小时前", "recently", "this week", "yesterday", etc.), record the **original expression** as `time_constraint_raw`. This is resolved to concrete UTC timestamps in Stage 1 Step 3 and used for both search query biasing and hard post-filtering. If no time expression is present, set `time_constraint_raw = null`.

Generate two keyword sets (bilingual search strategy):
- `keywords_original`: keywords in the user's original language (do NOT include time expressions in keywords — time filtering is handled separately)
- `keywords_english`: keywords translated to English (skip if already English; use a different phrasing for the second set)

### 3. Detect language

Identify the user's language. This determines the report output language.

### 4. Store internally

`query_text`, `language`, `mode` (manual), `keywords_original`, `keywords_english`, `time_constraint_raw`, `timestamp`

### 5. Route to collection

Load and follow `{baseDir}/references/collection_strategies.md` for Stage 1.

---

## Execution Logging

At the **start** of each stage/step, output a status line. Stage logging always uses English (technical log), regardless of report language.

```
[Stage N] Stage Name — status/result summary
```

---

## Report Generation (Stage 3)

After Stage 2 analysis is complete (from `{baseDir}/references/analysis_pipeline.md`), generate the signal report.

### Report Template (English)

Use this template when the user's language is English:

```
# Rumor Signal Report

**Query**: [user's rumor/topic]
**Scan Time**: [current UTC time]
**Platforms Scanned**: [list of platforms actually scanned]

---

## Signal Strength: [emoji] [level label] ([score]/100)

### Signal Summary

Found [N] related posts across [N] platforms from [N] independent sources.
Earliest appearance: [time] ([relative time]).
Cross-platform verification: [yes/no with details].

### Platform Distribution

| Platform | Hits | Independent Sources | Total Engagement |
|----------|------|-------------------|-----------------|
| ... | ... | ... | ... |

### Source Independence Analysis

- Independent source ratio: [X]% ([N]/[total])
- Single-source propagation: [detected / not detected]
- Most-cited origin: [description]

### Representative Posts (up to 5)

1. **[Platform] @author** (author type) — "content snippet..."
   - Engagement: [metrics]
   - Time: [relative time]
   - URL: [link]

[repeat for up to 5 posts, prioritize by: credibility weight > engagement > recency]

### Data Limitations

[Only include this section if any platform used Brave Search fallback]
- [Platform X]: Data retrieved via search engine (indirect). Coverage may be incomplete; engagement metrics and author details unavailable.

### Disclaimer

- This report assesses social media signal strength only. **It does not determine whether the event is true or false.**
- High signal strength does not mean the event is real. High signals can result from coordinated amplification or viral misinformation.

---

**Deep Verification**: To verify the truthfulness of this rumor, use fact-checker:
> Please fact-check: [auto-fill user's original query]
```

### Report Template (Chinese)

**When the user writes in Chinese, use this template. All titles, labels, and text must be in Chinese. No English mixing.**

```
# 舆情信号报告

**查询内容**：[用户输入的传言/话题]
**扫描时间**：[当前 UTC 时间]
**覆盖平台**：[实际扫描的平台列表]

---

## 信号强度：[emoji] [等级标签]（[分数]/100）

### 信号摘要

在 [N] 个平台上发现 [N] 条相关内容，来自 [N] 个独立来源。
最早出现时间：[时间]（[相对时间]）。
跨平台验证：[有/无，附详情]。

### 平台分布

| 平台 | 命中数 | 独立来源数 | 总互动量 |
|------|-------|-----------|---------|
| ... | ... | ... | ... |

### 来源独立性分析

- 独立来源比例：[X]%（[N]/[总数]）
- 单源传播检测：[检测到 / 未检测到]
- 最被引用的源头：[描述]

### 代表性内容（最多 5 条）

1. **[平台] @作者** (来源类型) — "内容摘要..."
   - 互动：[指标]
   - 时间：[相对时间]
   - 链接：[URL]

[最多 5 条，按来源可信度权重 > 互动量 > 时效性排序]

### 数据局限性

[仅在部分平台使用了 Brave Search 降级时显示此节]
- [平台X]：该平台数据通过搜索引擎间接获取，覆盖可能不完整，互动数据和作者信息不可用。

### 注意事项

- 本报告仅评估社交媒体上的舆情信号强度，**不对事件真实性做出判断**。
- 信号强度高不等于事件一定为真。高信号也可能由协调传播或误传导致。

---

**深度验证建议**：如需验证该传言的真实性，可使用 fact-checker：
> 请用 fact-checker 验证：[自动填入用户的原始传言]
```

### Language rule

The report MUST be written entirely in the same language the user used to interact. This includes:
- **Report title**: e.g., "舆情信号报告" not "Rumor Signal Report"
- **Signal level labels**: e.g., "强信号" not "Strong"
- **Section headings**: e.g., "信号摘要" not "Signal Summary"
- **All explanations and analysis text**

The ONLY exception is Stage logging (technical log), which always uses English.

For any language other than English or Chinese, translate all labels and text naturally.

### Signal level label translations

| English | Chinese |
|---------|---------|
| Strong | 强信号 |
| Moderate-Strong | 中强信号 |
| Moderate | 中等信号 |
| Weak | 弱信号 |
| None | 无信号 |

### JSON output

If the user requests JSON output, use the schema defined in `{baseDir}/references/output_schema.json`.

---

## Signal Strength Framework

Apply consistently when generating reports:

| Score | Level | Emoji | Meaning |
|-------|-------|-------|---------|
| **80-100** | Strong | Red circle | Multiple platforms, many independent sources, high engagement — event very likely already occurred |
| **60-79** | Moderate-Strong | Orange circle | Multiple sources but may cluster on 1-2 platforms, moderate independence |
| **40-59** | Moderate | Yellow circle | Some discussion but limited sources, possibly early-stage single-source propagation |
| **20-39** | Weak | Blue circle | Scattered discussion, few sources, low independence |
| **0-19** | None | White circle | Virtually no related discussion found on social media |

**Scoring formula (guide for LLM, not hard-coded):**

```
Signal Score = f(
  volume_score       * 0.25,   // quantity and engagement
  independence_score * 0.35,   // source independence (most important)
  credibility_score  * 0.25,   // source credibility weighting
  recency_score      * 0.15    // timeliness (newer = more important)
)
```

Source independence has the highest weight (0.35) because it is the key indicator for distinguishing genuine multi-source signals from single-source amplification.

---

## Important Rules

1. **No truth/false verdicts.** social-pulse only assesses signal strength. Never output "true", "false", "real", or "fake". If the user asks about truthfulness, guide them to fact-checker.
2. **Cite sources.** Every representative post in the report must include a URL. If a URL is unavailable, note it explicitly.
3. **Acknowledge limitations.** If a platform's data collection fails or is incomplete, state this clearly in the report. Brave Search fallback must be noted with "data retrieved via search engine (indirect)".
4. **Bilingual search.** For non-English input, always perform an English cross-check search. This is mandatory, not optional.
5. **No fabricated data.** Only reference content that actually appeared in search results. If searches return nothing, Signal Score is 0, report "None".
6. **Graceful degradation.** Agent Reach is optional. When unavailable, automatically fall back to Brave Search. Never error out or interrupt — degraded mode still produces a report, just with reduced coverage.
7. **Privacy.** Do not collect or store user data. Each query executes independently with no history retention.
8. **No invented dependencies.** Only use Agent Reach upstream tools (xreach, mcporter, etc.), WebSearch, WebFetch, curl, bash. Do NOT require the user to pip install any Python packages or install tools not listed in this skill.

---

## Setup Suggestion

After the **first successful signal report**, suggest to the user (once only):

> "Would you like me to automatically use social-pulse whenever you ask about rumors or trending topics? I can add this to my AGENTS.md."
>
> "你希望我以后遇到舆情查询的请求都自动使用 social-pulse 吗？我可以写入 AGENTS.md 让它永久生效。"

If the user agrees, append to AGENTS.md:
```
When asked about rumors, trending topics, social media buzz, or public attention on events, always use the social-pulse skill.
When asked to fact-check or verify truthfulness, always use the fact-checker skill.
```
