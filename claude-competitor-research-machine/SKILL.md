---
name: analyze-competitors
description: Find the winning content in your niche and tear it down. Pulls your Sandcastles watchlist's top outlier videos (default 3x+), transcribes them, reads the on-screen hook, and extracts the hook framework, format, and topic — then saves everything so you can search it later. Use for "/analyze-competitors", "what's working in my niche", "pull the watchlist outliers".
---

# Analyze Competitors

Find the outlier winners in your niche and tear them down with a real teardown engine —
not vibes.

**Two layers, on purpose:**
- **Sandcastles = discovery (0 credits).** It crawls your watchlist and scores outliers for
  you. We only *read* — `search_my_videos` + `get_video_details`. We never call their
  `analyze_video`, because we run our own analysis and want every row to be identical.
- **Your pipeline = the analysis.** Every video goes through the same
  [teardown](references/teardown.md) — download, Whisper, opening-frame text hook, structured
  extraction.

**Run the same teardown on your OWN posts too.** That's the whole trick. When your content
and your competitors' content go through one engine into one table, "how do I compare" is a
single query instead of a guess.

---

## Setup

1. **Sandcastles account** — https://sandcastles.ai/?ref=noe (paid, no free tier)
2. **Connect it to Claude Code:**
   ```bash
   claude mcp add --transport http sandcastles https://mcp.sandcastles.ai/
   ```
3. **Build your watchlist** at https://app.sandcastles.ai — add the creators you want
   watched. This is edited in the app; there's no MCP endpoint that lists channels.
4. **Save your roster** to `watchlist-roster.json` next to this skill — `handle` + channel
   `uuid` for every creator. This file is your source of truth for the discovery loop.
   ```json
   [{ "handle": "somecreator", "uuid": "..." }]
   ```
5. **Tools:** `yt-dlp` and `ffmpeg` installed, plus an `OPENAI_API_KEY` in your `.env` for
   Whisper.

---

## Pipeline

### Step 1 — Discover: loop the roster creator-by-creator

**Do NOT use the global `search_my_videos` feed.** Its cursor pagination times out after
page 1, and one prolific creator floods a page and hides everyone else. Loop your saved
roster instead — a per-creator `channel_uuids` query is reliable and returns that creator's
full set.

1. Load `watchlist-roster.json`.
2. For EACH creator, query just that channel:
   ```
   search_my_videos(channel_uuids=[uuid], min_outlier_score=3, lookback_days=7,
                    min_views=25000, limit=5)
   ```
   - **One creator per call.** Go 1-by-1 (safest) or ≤5 in flight — **never 8-wide**, that
     trips a per-minute `rate_limited`.
   - **Retry on failure.** The server times out roughly 30% of the time and occasionally
     returns `rate_limited` (retry_after ~1s). On timeout or rate-limit, wait ~2s and retry
     the SAME creator up to 3× before giving up. Note any creator that never returned so the
     run is honest.
3. **Union the winners.** A creator with no 3x+ post in the window returns empty — that's
   correct, skip it.
4. **Self-heal the roster.** After the loop, run ONE low-threshold sweep
   (`search_my_videos(min_outlier_score=0, min_views=0, min_engagement=0, lookback_days=7,
   limit=25)`) and dedupe `channel.handle` against the roster. Any handle NOT in the file is
   a creator you added in the app but never wrote down → append `{handle, uuid}` to
   `watchlist-roster.json` and include its winners.

Defaults: **3x+ outlier, last 7 days, 25k+ views.** Override freely (14 days, 5x+, a single
handle).

Each result carries: `platform_url`, `channel.handle`, `platform`, `view_count`,
`engagement_rate`, `outlier_score`, `published_at`, `thumbnail`, and `uuid`.

### Step 2 — Dedup before you spend

Derive `post_id` = the platform shortcode and check your store for an existing row. **Skip
anything already analyzed** — don't re-spend the pipeline. Show the fresh set + count +
rough cost (~$0.15/video in Whisper + model calls) and **confirm before analyzing**.

`post_id` from `platform_url`:
- Instagram `…/reel/DZlUgsrTBNQ/` → `DZlUgsrTBNQ`
- TikTok `…/video/7412…` → `7412…`
- YouTube `…/shorts/AbC123` → `AbC123`

### Step 3 — Metrics + metadata (0 credits)

`get_video_details(video_uuid)` per video → `view_count`, `like_count`, `comment_count`,
`engagement_rate`, `outlier_score`, `published_at`, `thumbnail`, `channel.handle`,
`platform`, `platform_url`.

**Do NOT read the `analysis` field** even when `analyzed:true` — we run our own.

### Step 4 — Tear it down

Run [`references/teardown.md`](references/teardown.md) on each `platform_url`. For a batch,
fan out in parallel sub-agents to keep it fast. Pass each sub-agent the Sandcastles metrics
so it fills the metric fields without re-fetching.

Watchlist items are almost always short-form video. If one is a carousel or long-form,
handle it separately — this teardown assumes a video with audio.

### Step 5 — Deliver the brief

After the batch, summarize. Don't dump every teardown:

```
## Niche Winners — <N> videos, <lookback>, 3x+ outlier

**Top outliers analyzed:**
1. @<handle> — "<spoken or text hook>" — <outlier>x, <views> views — <format_type> / <hook_category>
2. …

**Recurring hook frameworks:** <the 2-3 [bracketed] madlibs showing up most>
**Hook categories that pop:** <e.g. trap_mistake, authority — by outlier>
**Formats winning:** <format_type breakdown>
**Topics clustering:** <top topics>
**What to steal:** 3-4 concrete, replicable moves for your content
```

---

## Rules

- **Watchlist only — non-negotiable.** Discovery is ALWAYS `search_my_videos` (watchlist
  scope). **NEVER** `search_all_videos` (the global index). If your watchlist has junk on it,
  fix that by pruning it at app.sandcastles.ai — not by widening the search here.
- **Never fabricate.** Empty or unreadable media → null the fields and skip the row. Don't
  invent a plausible hook. A guessed hook poisons every downstream decision you make.
- **Never call `analyze_video`.** Discovery and metadata only from Sandcastles; the analysis
  is yours.
- **Closed-set enums.** `hook_category` (11), `format_category` (2), `format_type` (9) — pick
  exactly one, never coin a new value. The whole point is that rows are comparable.
- **Dedup first, confirm cost, then analyze.**
- **First-party-only metrics don't exist for competitors.** Reach, saves, shares,
  impressions, watch-time, retention — leave them null. Never estimate them.
