# Analysis Pipeline: Stage 2

This file is loaded after Stage 1 data collection. Execute the analysis steps in order.

---

## Stage 2: Signal Analysis

Log: `[Stage 2] Signal analysis — starting with {total_posts} posts`

### Step 0: Time Constraint Hard Filter

**This step runs first, before any scoring, when a time constraint was specified by the user.**

If `time_filter_start` and `time_filter_end` exist from Stage 1 Step 3:

1. **Drop posts outside the time range**: Remove any post whose `timestamp` falls before `time_filter_start` or after `time_filter_end`.
2. **Handle ambiguous timestamps**: For posts tagged `"time_verified": false` (timestamp could not be determined):
   - Inspect the post's content snippet and URL for date clues (e.g., URL contains `/2026/03/14/`, snippet mentions "Friday morning").
   - If clues suggest the post is within range → keep it, set `"time_verified": true`.
   - If clues suggest the post is outside range → drop it.
   - If no clues at all → keep it but flag as `"time_unverified": true` in the report.
3. **Warn on excessive filtering**: If >80% of collected posts were dropped by this filter, add a note to the report: "Most search results fell outside the specified time range. Coverage for this time window may be limited."

Log: `[Stage 2] Step 0 — Time filter: {kept} kept, {dropped} dropped, {unverified} unverified (range: {time_filter_start} to {time_filter_end})`

**If no time constraint was specified, skip this step entirely.**

---

### Step 1: Volume Statistics

Calculate basic quantity metrics from the collected data:

- **Total hits**: total number of posts collected
- **Platform distribution**: count of posts per platform
- **Time distribution**:
  - Earliest appearance timestamp
  - Posts in the last 1 hour / 6 hours / 24 hours
  - Time trend: accelerating, steady, or decelerating
- **Total engagement**: sum of (likes + reposts + comments) across all posts where data is available
  - For posts with `null` engagement (Brave Search fallback), exclude from engagement totals but include in hit counts

Log: `[Stage 2] Step 1 — Volume: {total_hits} hits, {platforms_with_hits} platforms, total engagement: {engagement_sum}`

**Volume Score (0-100):**

| Total Hits | Engagement | Score Range |
|-----------|------------|-------------|
| 50+ hits | 10,000+ total engagement | 80-100 |
| 20-49 hits | 1,000-9,999 engagement | 60-79 |
| 10-19 hits | 100-999 engagement | 40-59 |
| 3-9 hits | 10-99 engagement | 20-39 |
| 0-2 hits | < 10 engagement | 0-19 |

When engagement data is unavailable (all Brave Search fallback), score based on hit count only and note this in the report.

---

### Step 2: Source Independence Analysis

**This is the most critical step — it distinguishes genuine multi-source signals from single-source amplification.**

#### 2a. Count independent sources

- Group posts by author (username/ID)
- Count unique authors = **independent source count**
- Calculate **independent source ratio** = unique authors / total posts

#### 2b. Cross-platform verification

- Check if the topic appears on 2+ different platforms
- If yes → `cross_platform_verified = true` (signal boost)
- If only on 1 platform → `cross_platform_verified = false`

#### 2c. Single-source propagation detection

Analyze whether most posts trace back to a single origin:

1. Look for patterns indicating reposting/quoting the same source:
   - Multiple posts containing the same quoted text or @-mentioning the same original author
   - Reddit posts linking to the same external URL
   - Posts that are clearly retweets/reposts of one original

2. Calculate the origin concentration:
   - If >80% of posts reference the same original source → **single-source propagation detected** (signal downgrade)
   - If multiple authors describe the same event with different wording and no shared source → **multi-source independent reporting** (signal upgrade)

#### 2d. Phrasing diversity check

- Analyze the linguistic diversity of posts:
  - If most posts use identical or near-identical phrasing → likely copy-paste or repost chain
  - If posts describe the same event with varied phrasing → more likely independent observation

Log: `[Stage 2] Step 2 — Independence: {independent_sources}/{total_posts} unique sources, cross-platform: {yes/no}, single-source: {detected/not detected}`

**Independence Score (0-100):**

| Independent Source Ratio | Cross-Platform | Single-Source | Score Range |
|------------------------|----------------|---------------|-------------|
| >60% + cross-platform | Yes | No | 80-100 |
| 40-60% + cross-platform | Yes | No | 60-79 |
| 30-50% | Partial | Partial | 40-59 |
| 15-30% | No | Yes | 20-39 |
| <15% | No | Yes | 0-19 |

---

### Step 3: Source Credibility Weighting

Assign a credibility weight to each post based on the author's profile:

| Source Type | Weight | How to Identify |
|------------|--------|-----------------|
| Verified media account | x3.0 | Blue/yellow checkmark, official media organization |
| Verified journalist / industry KOL | x2.0 | Verified badge, or widely cited by other sources |
| High-follower regular user | x1.5 | Followers > 10,000 |
| Regular user | x1.0 | Baseline |
| New/low-activity account | x0.5 | Very few posts, recently created |

**Brave Search fallback rule**: When author data is unavailable (follower count = null, verification = null), apply default weight x1.0 for all sources. Note in the report: "部分来源权重因数据不足采用默认值" / "Some source weights use default values due to insufficient data".

**Credibility Score (0-100):**

Calculate the weighted credibility of the source pool:
- Compute: `weighted_sum = sum(post_weight * post_engagement)` for all posts
- Compare against baseline where all posts have weight 1.0
- Higher concentration of credible sources → higher score

| Weighted Profile | Score Range |
|-----------------|-------------|
| Multiple verified media/journalist sources | 80-100 |
| Mix of verified and regular, some high-follower | 60-79 |
| Mostly regular users with some engagement | 40-59 |
| Mostly anonymous/low-follower accounts | 20-39 |
| All low-activity or new accounts | 0-19 |

Log: `[Stage 2] Step 3 — Credibility: highest weight source: {description}, credibility score: {score}`

---

### Step 4: Recency Score

Evaluate the timeliness of the signal:

| Time Since Earliest Post | Score Range |
|------------------------|-------------|
| < 1 hour ago | 90-100 |
| 1-6 hours ago | 70-89 |
| 6-24 hours ago | 50-69 |
| 1-3 days ago | 30-49 |
| 3-7 days ago | 15-29 |
| > 7 days ago | 0-14 |

**Acceleration bonus**: If the post rate is increasing over time (more posts in recent hours than earlier), add +10 to recency score (cap at 100).

Log: `[Stage 2] Step 4 — Recency: earliest {time_ago}, trend: {accelerating/steady/decelerating}`

---

### Step 5: Composite Signal Score

Combine all four dimensions:

```
Signal Score = (
  volume_score       * 0.25 +
  independence_score * 0.35 +
  credibility_score  * 0.25 +
  recency_score      * 0.15
)
```

Round to the nearest integer (0-100).

**Map to signal level:**

| Score | Level | Emoji |
|-------|-------|-------|
| 80-100 | Strong | Red circle |
| 60-79 | Moderate-Strong | Orange circle |
| 40-59 | Moderate | Yellow circle |
| 20-39 | Weak | Blue circle |
| 0-19 | None | White circle |

Log: `[Stage 2] Step 5 — Signal Score: {score}/100 ({level}) [V:{volume} I:{independence} C:{credibility} R:{recency}]`

---

### Step 6: Select representative posts

Select up to 5 representative posts for the report. Prioritize by:

1. **Source credibility weight** (verified media > verified journalist > KOL > regular > new)
2. **Engagement** (higher total engagement preferred)
3. **Recency** (newer preferred)
4. **Platform diversity** (try to include posts from different platforms)

For each selected post, prepare:
- Platform and author info
- Content snippet (max 200 characters, truncate with "...")
- Engagement metrics (or "N/A" for Brave Search fallback)
- Relative time (e.g., "5h ago")
- URL

---

### Step 7: Proceed to report generation

Return to the main SKILL.md Stage 3 (Report Generation) with the following data ready:

- `signal_score` (0-100)
- `signal_level` (strong/moderate_strong/moderate/weak/none)
- `total_hits`
- `independent_sources`
- `cross_platform_verified` (boolean)
- `single_source_propagation` (boolean)
- `earliest_appearance` (timestamp)
- `platform_breakdown` (per-platform stats)
- `representative_posts` (up to 5)
- `data_limitations` (list of platforms using Brave Search fallback)
- `volume_score`, `independence_score`, `credibility_score`, `recency_score` (sub-scores)
