# The Teardown

Download → transcribe → read the on-screen hook → extract structure → save.

Run this **identically on your own posts and your competitors' posts.** Same engine, same
fields, one store. That's what makes them comparable.

The caller hands you, per video: `platform_url`, `handle`, `platform`, `post_id`
(shortcode), and the Sandcastles metrics (`views`, `likes`, `comments`, `engagement_rate`,
`outlier_score`, `post_date`, `thumbnail_url`).

---

## 1. Download the video (with audio)

Prefer `yt-dlp` — it muxes audio+video into one file and handles IG / TikTok / Shorts:

```bash
yt-dlp -o "/tmp/analyze_%(id)s.%(ext)s" -f "mp4/best" "<URL>"
```

**If Instagram blocks yt-dlp** (login wall), fall back to a scraper and grab BOTH streams —
newer IG reels split the audio out, and a video-only stream will silently produce an empty
transcript.

**TikTok URLs** are rejected by most IG scrapers' URL regex — use a TikTok-specific one.

## 2. Transcribe the audio (verbatim)

```bash
curl -s https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F file="@/tmp/analyze_<id>.mp4" -F model="whisper-1" -F response_format="text"
```

If Whisper returns empty AND the audio isn't silent, you pulled a video-only stream —
re-download the audio track. If the video is genuinely music-only with no voice, the
transcript is `""` (correct) and the on-screen text hook carries the content.

## 3. Read the on-screen text hook

Grab the opening frame and actually look at it. Don't guess:

```bash
ffmpeg -y -i "/tmp/analyze_<id>.mp4" -ss 00:00:01.5 -frames:v 1 "/tmp/analyze_<id>_hook.jpg"
```

Then **read the frame**. Report ONLY text in the **top half** of the first ~3 seconds — the
title card. IGNORE the bottom auto-caption subtitles; those are the transcript, not the hook.
If there's no top text, or you can't read it → `text_hook = null`. Never invent text.

## 4. Extract the structure

Base this ONLY on the transcript + caption + what you actually saw in the frame.

**If the transcript is empty or unreadable, do NOT fabricate.** Set `spoken_hook`,
`hook_framework`, and `hook_category` to null, and infer topic/format from the caption alone
(or null). Never invent hooks, quotes, or numbers.

Extract:

1. **spoken_hook** — the exact first spoken line, verbatim (`""` if no speech)
2. **hook_framework** — that hook as a reusable `[bracketed]` mad-lib template (≤4 variables)
3. **hook_category** — EXACTLY one of:
   `list · secret_reveal_breakdown · question · problem · contrarian · authority · personal_experience · tutorial · ranking_rating · comparison · trap_mistake`
4. **format_category** — EXACTLY one of: `educational · storytelling`
5. **format_type** — EXACTLY one of:
   `listicle · tutorial · case_study · breakdown_explainer · ranking_rating_tier_list · levels · problem_solution · skit_humor · personal_update`
6. **topic** — a 2–4 word label
7. **core_idea** — the core idea phrased as an action

The `hook_framework` is the most valuable field in here. `"5 secret codes for [tool], number
one is [code]"` is something you can refill for your own niche tomorrow. `"10 secret codes
for ChatGPT"` is not.

## 5. Save it

Write one record per video. The fields:

```
source            'own' | 'competitor'   ← the only thing separating your posts from theirs
platform          instagram | tiktok | youtube
handle
post_id           the shortcode — this is your dedup key
url
caption
post_date
thumbnail_url
transcript
text_hook
spoken_hook
hook_framework
hook_category
format_category
format_type
topic
core_idea
views  likes  comments  engagement_rate  outlier_score
analyzed_at
```

**Simplest version that works:** append to a local `competitor-research.json`, keyed on
`post_id` so re-runs update instead of duplicating.

**Better:** put it in a real database or an Airtable base so you can filter and sort. The
schema above maps 1:1 to columns — add them once and point the write here.

Whatever you choose, **your own posts go in the same store with `source='own'`.** One table,
two sources. Then "which hook categories beat me last month" is one query.

## 6. Return a one-line result

`✓ @<handle> — <format_type> / <hook_category> — <outlier>x`

or `✗ @<handle> — no transcript, skipped` if it was unreadable. The caller aggregates these
into the brief.

---

## Rules

- **Never fabricate.** Empty or unreadable input → null fields, not plausible guesses.
- Enums are closed sets — pick exactly one, never invent a new value.
- `hook_framework` uses `[bracketed]` variables, not the literal words.
- Clean up `/tmp/analyze_*` when you're done.
