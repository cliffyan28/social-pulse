# social-pulse

**Open-source social media signal detection for OpenClaw — scan 6 platforms for rumor signals and assess their strength. Zero config.**

## Install

```bash
claw install social-pulse
```

That's it. No Python, no pip install, no API keys required beyond what you already have in OpenClaw.

## What it does

- Scan 6 social media platforms (X, Reddit, TikTok, Xiaohongshu, Douyin, Weibo) for rumor signals
- Assess signal strength with a 0-100 score based on volume, source independence, credibility, and recency
- Detect single-source propagation vs multi-source independent reporting
- Cross-platform verification — check if a topic is discussed independently on multiple platforms
- Bilingual search — search in both the user's language and English for maximum coverage
- Output structured signal reports with representative posts and engagement data
- Works in any language — reports follow the user's language automatically
- Graceful degradation — works with or without Agent Reach installed

## Usage

Just ask your agent:

```
Is there buzz about TSMC leaving Taiwan?
```

```
听说台积电要从台湾撤走，查一下舆情
```

```
What are people saying about the Dubai floods?
```

```
查一下最近关于 OpenAI 收购 Anthropic 的传言
```

## How It Works

The skill runs a 4-stage pipeline:

1. **Input Parsing** — Extract keywords, detect language, generate bilingual search terms
2. **Multi-Platform Collection** — Search 6 platforms via Agent Reach (direct) or Brave Search (fallback)
3. **Signal Analysis** — Assess volume, source independence, credibility, and recency
4. **Report Generation** — Structured signal report with score, platform breakdown, and representative posts

### Signal Strength Levels

| Score | Level | Meaning |
|-------|-------|---------|
| 80-100 | Strong | Multiple platforms, many independent sources, high engagement |
| 60-79 | Moderate-Strong | Multiple sources but may cluster on 1-2 platforms |
| 40-59 | Moderate | Some discussion but limited sources |
| 20-39 | Weak | Scattered discussion, few sources |
| 0-19 | None | Virtually no social media discussion found |

### Key Differentiator: Source Independence

The most important metric is **source independence** — social-pulse distinguishes between:
- **Multi-source independent reporting**: many authors describe the same event with different wording (signal upgrade)
- **Single-source propagation**: most posts trace back to one original source (signal downgrade)

This prevents viral retweet chains from inflating signal scores.

## Data Sources

### Agent Reach (optional, recommended)

If you have [Agent Reach](https://github.com/anthropics/agent-reach) installed, social-pulse uses it for direct platform access:

| Platform | Agent Reach | Requirements |
|----------|:-----------:|-------------|
| X (Twitter) | Supported | Twitter Cookie |
| Reddit | Supported | May need proxy |
| TikTok | Not supported | Always uses Brave Search |
| Xiaohongshu | Supported | Docker + Cookie |
| Douyin | Supported | No login needed |
| Weibo | Not supported | Always uses Brave Search |

### Brave Search (fallback)

When Agent Reach is unavailable or a specific platform isn't configured, social-pulse falls back to Brave Search `site:` queries. This still works but with limitations:
- Search engine indexing may lag real-time posts
- Engagement metrics (likes, reposts, comments) are unavailable
- Author details (followers, verification status) are unavailable
- Reports note these limitations transparently

## Configuration

### Web Search (most users already have this)

The skill uses your OpenClaw agent's built-in web search. If you already have Brave Search configured, **you're good to go**.

If not, the skill guides you through setup (free, ~1 minute):
1. Go to https://brave.com/search/api/ and create a free account
2. Generate an API key (free plan = 1,000 searches/month)
3. Set `BRAVE_SEARCH_API_KEY` in your OpenClaw config

### Agent Reach (optional, enhances coverage)

Install [Agent Reach](https://github.com/anthropics/agent-reach) for direct platform access with richer data. Without it, the skill still works via Brave Search fallback.

## Relationship with fact-checker

social-pulse and [fact-checker](https://github.com/cliffyan28/fact-checker) form a product pair:

| | social-pulse | fact-checker |
|---|---|---|
| **Purpose** | Signal detection — "how much are people talking about this?" | Verification — "is this true or false?" |
| **Output** | Signal strength score (0-100) | Verdict (TRUE / FALSE / UNVERIFIED / ...) |
| **Use when** | Checking buzz, trends, public attention | Verifying claims, detecting misinformation |

Every social-pulse report includes a suggestion to use fact-checker for deep verification.

## Output Formats

- **Markdown** (default) — Human-readable signal report in conversation
- **JSON** — Structured data, request with "output as JSON". Schema: [`references/output_schema.json`](references/output_schema.json)

## FAQ

### What is social-pulse?
An open-source social media signal detection skill for OpenClaw that scans 6 platforms and assesses how widely a rumor or topic is being discussed.

### Does it tell me if a rumor is true or false?
No. social-pulse only measures signal strength — how much discussion exists. For truth verification, use fact-checker.

### Does it require API keys?
No additional keys beyond your existing OpenClaw setup. Brave Search API key is optional but recommended for fallback coverage.

### What languages does it support?
Any language. The skill searches in both the user's language and English, and reports are generated in the user's language.

### Can it monitor topics automatically?
Automatic monitoring is planned for Phase 2. Currently, social-pulse supports manual queries.

## GitHub Topics

`openclaw` `social-pulse` `social-media` `signal-detection` `real-time-alert` `osint` `clawhub` `multilingual` `sentiment-analysis` `misinformation` `public-opinion` `social-listening` `trend-detection` `early-warning`

## License

MIT
