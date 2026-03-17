# Collection Strategies: Stage 1

This file is loaded after Stage 0 input parsing. Execute the collection pipeline in order.

---

## Stage 1: Multi-Platform Data Collection

Log: `[Stage 1] Multi-platform data collection — starting`

### Step 1: Detect available data sources

Check if Agent Reach is installed:

```bash
which agent-reach 2>/dev/null || which xreach 2>/dev/null || echo "NOT_FOUND"
```

- If **found** → set `agent_reach_available = true`, log: `[Stage 1] Agent Reach detected — using direct platform APIs where available`
- If **NOT_FOUND** → set `agent_reach_available = false`, log: `[Stage 1] Agent Reach not found — using Brave Search fallback for all platforms`

Also check if Brave Search is available by testing WebSearch. If WebSearch fails entirely:

> Rumor signal scanning requires web search. Please set up Brave Search (free, ~1 minute):
> 1. Go to https://brave.com/search/api/ and create a free account
> 2. Generate an API key (free plan = 1,000 searches/month)
> 3. Set `BRAVE_SEARCH_API_KEY` in your OpenClaw config, then retry.
>
> 中文：舆情信号扫描需要网络搜索。请访问 https://brave.com/search/api/ 注册免费账号，获取 API Key，配置到 OpenClaw 后重试。

Do NOT proceed without any search capability.

---

### Step 2: Execute collection per platform

For each platform, use the best available method. **Collect up to 20 posts per platform per keyword set.**

#### Platform: X (Twitter)

**Layer 1 (Agent Reach available):**
```bash
xreach search "{keywords}" --limit 20 --json
```
- Requires user to have configured Twitter Cookie via Agent Reach
- If xreach fails (cookie expired, not configured) → fall back to Layer 2

**Layer 2 (Brave Search fallback):**
```
WebSearch "site:x.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`

#### Platform: Reddit

**Layer 1 (Agent Reach available):**
```bash
curl -s -H "User-Agent: social-pulse/1.0" "https://www.reddit.com/search.json?q={keywords}&sort=new&limit=20"
```
- Parse JSON response: extract `data.children[].data` fields (title, selftext, author, score, num_comments, permalink, created_utc)
- Note: Reddit's public JSON API works without authentication but may rate-limit; Agent Reach handles proxy/retry

**Layer 2 (Brave Search fallback):**
```
WebSearch "site:reddit.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`

#### Platform: TikTok (International)

**Always Layer 2** — Agent Reach does not support TikTok international:
```
WebSearch "site:tiktok.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`
- Note: search engine indexing is limited to video titles/descriptions; comments are not indexed

#### Platform: Xiaohongshu (Little Red Book)

**Layer 1 (Agent Reach available):**
```bash
mcporter call 'xiaohongshu.search_feeds(keyword="{keywords}", limit=20)'
```
- Requires Docker + Cookie configured via Agent Reach
- If mcporter fails → fall back to Layer 2

**Layer 2 (Brave Search fallback):**
```
WebSearch "site:xiaohongshu.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`

#### Platform: Douyin

**Layer 1 (Agent Reach available):**
```bash
mcporter call 'douyin.search_videos(keyword="{keywords}", limit=20)'
```
- No login required

**Layer 2 (Brave Search fallback):**
```
WebSearch "site:douyin.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`

#### Platform: Weibo

**Always Layer 2** — Agent Reach does not support Weibo:
```
WebSearch "site:weibo.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`
- Note: Weibo is critical for Chinese-language rumor tracking; Brave Search coverage may lag

#### Platform: YouTube

**Layer 1 (Agent Reach available):**
```bash
xreach youtube-search "{keywords}" --limit 20 --json
```
- Uses yt-dlp under the hood; extracts video titles, descriptions, and comment snippets
- No API key required

**Layer 2 (Brave Search fallback):**
```
WebSearch "site:youtube.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`
- Note: only video titles/descriptions are indexed; comments are not available via search fallback

#### Platform: Bilibili

**Layer 1 (Agent Reach available):**
```bash
xreach bilibili-search "{keywords}" --limit 20 --json
```
- Uses yt-dlp; extracts video titles, descriptions, and danmaku/comment data
- No login required

**Layer 2 (Brave Search fallback):**
```
WebSearch "site:bilibili.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`
- Note: Bilibili is a major Chinese-language video platform; danmaku (弹幕) and comments are not indexed by search engines

#### Platform: WeChat Articles (微信公众号)

**Layer 1 (Agent Reach available):**
```bash
xreach wechat-search "{keywords}" --limit 20 --json
```
- Searches public WeChat articles (公众号文章)
- If xreach fails → fall back to Layer 2

**Layer 2 (Brave Search fallback):**
```
WebSearch "site:mp.weixin.qq.com {keywords}"
```
- Mark results with `data_source: "brave_search_fallback"`
- Note: WeChat articles are critical for Chinese-language opinion pieces and official statements; search engine indexing coverage is limited

---

### Step 3: Time-aware query construction

If Stage 0 extracted a time constraint (e.g., "today", "几个小时前", "this morning", "yesterday"), convert it to a concrete date or date range based on the current scan timestamp:

| User expression | Resolved range (example if scan time = 2026-03-14 04:50 UTC) |
|----------------|--------------------------------------------------------------|
| "today" / "今天" | 2026-03-14 only |
| "几个小时前" / "a few hours ago" | last 12 hours (2026-03-13T17:00Z → now) |
| "this morning" / "今天早上" | 2026-03-14 00:00Z → now |
| "yesterday" / "昨天" | 2026-03-13 only |
| "this week" / "这周" | last 7 days |
| "recently" / "最近" | last 3 days |
| No time constraint | no filtering (default) |

Store the resolved range as `time_filter_start` and `time_filter_end` (UTC timestamps). These are used in two places:
1. **Here (Step 4)**: Append date qualifiers to search queries to bias results toward the correct time window.
2. **Stage 2 Step 0**: Hard-filter results that fall outside this range.

Log: `[Stage 1] Step 3 — Time constraint: "{user_expression}" → {time_filter_start} to {time_filter_end}`
Log (if no constraint): `[Stage 1] Step 3 — No time constraint specified`

---

### Step 4: Bilingual search execution

Execute two rounds of search across all platforms:

1. **Round 1**: Use `keywords_original` for all platforms
2. **Round 2**: Use `keywords_english` for all platforms (mandatory cross-check, even if Round 1 found results)

**Time-qualified search queries**: When a time constraint exists, append a date qualifier to each search query to bias search engines toward recent results:

- For WebSearch queries, append the date or date range in natural language. Examples:
  - `"ICD Brookfield debris" March 14 2026` (specific date)
  - `"ICD Brookfield debris" today` (same day)
- For Agent Reach / API queries that support date parameters, use the `time_filter_start` / `time_filter_end` directly.

This does NOT replace the hard filter in Stage 2 — search engine date biasing is best-effort. Stage 2 Step 0 performs the authoritative time filter.

Log: `[Stage 1] Round 1 — searching with original keywords: "{keywords_original}" (time-qualified: {yes/no})`
Log: `[Stage 1] Round 2 — searching with English keywords: "{keywords_english}" (time-qualified: {yes/no})`

**Important**: For Chinese-language platforms (Xiaohongshu, Douyin, Weibo, Bilibili, WeChat), the English keyword round may yield few results — this is expected. Still execute it for X, Reddit, TikTok, and YouTube where English content is prevalent.

---

### Step 4: Normalize and deduplicate

Normalize all collected posts into this standard format, regardless of source platform:

```json
{
  "platform": "x|reddit|tiktok|weibo|xiaohongshu|douyin|youtube|bilibili|wechat",
  "author": "username or ID",
  "author_followers": 1200,
  "author_verified": false,
  "content": "post text content",
  "timestamp": "2026-03-13T10:30:00Z",
  "engagement": {
    "likes": 340,
    "reposts": 89,
    "comments": 56
  },
  "url": "original post URL",
  "is_media_attached": true,
  "language": "zh|en|...",
  "data_source": "agent_reach|brave_search_fallback"
}
```

**Field mapping by platform:**

| Field | X (Twitter) | Reddit | TikTok | Xiaohongshu | Douyin | Weibo | YouTube | Bilibili | WeChat |
|-------|-------------|--------|--------|-------------|--------|-------|---------|----------|--------|
| author | screen_name | author | author | user.nickname | author.nickname | user.screen_name | channel_name | uploader | author (公众号名) |
| author_followers | followers_count | null | null | user.fans | author.follower_count | followers_count | subscriber_count | follower_count | null |
| likes | favorite_count | score (upvotes) | digg_count | liked_count | statistics.digg_count | attitudes_count | like_count | like | read_count |
| reposts | retweet_count | null | share_count | shared_count | statistics.share_count | reposts_count | null | share | null |
| comments | reply_count | num_comments | comment_count | comments_count | statistics.comment_count | comments_count | comment_count | reply | null |

**Notes on Brave Search fallback data:**
- `author_followers`: set to `null`
- `author_verified`: set to `null`
- `engagement`: all fields set to `null`
- `data_source`: set to `"brave_search_fallback"`
- These null fields are expected and handled gracefully in Stage 2 analysis

**Deduplication**: Remove posts with identical URLs. When two posts have the same URL from different search rounds, keep the one with more complete data.

**Time pre-filter**: If a time constraint was resolved in Step 3, discard any post whose `timestamp` falls outside the `[time_filter_start, time_filter_end]` range. For posts where the timestamp cannot be determined (Brave Search fallback with no date in snippet), keep them but tag with `"time_verified": false` — Stage 2 Step 0 will handle these conservatively.

Log: `[Stage 1] Collection complete — {total_posts} posts from {N} platforms ({M} via Agent Reach, {K} via Brave Search fallback), {discarded} discarded by time filter`

---

### Step 5: Proceed to analysis

After collection is complete, load and follow `{baseDir}/references/analysis_pipeline.md` for Stage 2.
