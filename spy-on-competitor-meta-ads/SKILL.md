---
name: spy-on-competitor-meta-ads
description: Pull every ad your competitors are running on Meta into your own Airtable base. Reads a competitor list, scrapes the Meta Ad Library via Apify, downloads and transcribes video ads, classifies the angle and format, and writes it all to a table with previews. Use for "scrape competitor ads", "what ads is [brand] running", "spy on competitor ads", or to refresh an existing ad database.
---

# Spy On Your Competitors' Meta Ads

You are a competitive ad intelligence system. You pull every active Meta ad from a list of competitors, download the videos, transcribe them, classify the angle and format, and write everything into Airtable.

**What you produce:** a searchable database of competitor ads — copy, headlines, landing pages, media, transcripts, hooks, and how long each ad has been running.

**Why days-running matters:** nobody keeps a losing ad live. An ad that has been running for 60+ days is one they're making money on. That column is the whole point of the database.

---

## Prerequisites

1. **Apify token** — free account, $5/month in credit (no card). Get it at apify.com → Settings → API & Integrations.
2. **Airtable personal access token (PAT)** — airtable.com/create/tokens, with scopes `data.records:read`, `data.records:write`, and `schema.bases:read`, granted to the base you'll use.
3. **A base with two tables** — run **Step 0** below and this skill builds them for you.
4. *(Optional)* **whisper.cpp + ffmpeg** — for transcribing video ads. If either is missing, skip transcription and say so in the summary. Everything else still works.

Read tokens from the project's `.mcp.json` (an `apify-*` / `airtable-*` server block) or from `.env` as `APIFY_TOKEN` and `AIRTABLE_API_KEY`. Never print a token back to the user.

---

## Step 0 — Set up the base (first run only)

**Ask the user for their Airtable base ID** (it's in the URL: `airtable.com/appXXXXXXXXXXXXXX/...`).

Then read the base schema and either find or create the two tables below. **Always resolve fields by NAME, never by hard-coded field ID** — IDs are unique to each base, names are not. Look up the IDs at runtime and hold them in memory for the run.

### Table: `Competitors`

| Field | Type | Notes |
|---|---|---|
| `Name` | Single line text | Company name |
| `Facebook Page ID` | Single line text | The **Ad Library** page ID — not the profile ID. See "When it breaks". |
| `Status` | Single select | `Active` / `Paused` — only `Active` gets scraped |

### Table: `Ad Research`

| Field | Type | | Field | Type |
|---|---|---|---|---|
| `Ad Archive ID` | Single line text | | `Caption` | Single line text |
| `Page Name` | Single line text | | `Video URL` | URL |
| `Page ID` | Single line text | | `Image URL` | URL |
| `Competitor` | Link to `Competitors` | | `Video Duration (sec)` | Number |
| `Ad Library URL` | URL | | `Video Aspect Ratio` | Single select |
| `Start Date` | Date | | `Transcript` | Long text |
| `End Date` | Date | | `Hook (Video)` | Long text |
| `Days Running` | Formula | | `Word Count` | Number |
| `Is Active` | Checkbox | | `Angle Category` | Single select |
| `Platforms` | Multiple select | | `Ad Format Type` | Single select |
| `Display Format` | Single select | | `Scrape Date` | Date |
| `Body Text` | Long text | | `Scrape Batch ID` | Single line text |
| `Title` | Single line text | | `Preview` | Attachment |
| `CTA Text` | Single line text | | `Link URL` | URL |
| `CTA Type` | Single line text | | `Link Description` | Long text |

**`Days Running` formula** — this is the column that tells you which ads are working:

```
IF({Start Date}, DATETIME_DIFF(IF({Is Active}, TODAY(), {End Date}), {Start Date}, 'days'))
```

Select options: `Platforms` → Facebook, Instagram, Messenger, Audience Network · `Display Format` → Video, Image, Carousel, DCO · `Video Aspect Ratio` → 9:16, 4:5, 1:1, 16:9, Other. `Angle Category` and `Ad Format Type` options are listed in Step 6.

---

## Step 1 — Read the active competitors

Fetch every `Competitors` record where `Status = Active`, returning `Name` and `Facebook Page ID`. Skip any row with no page ID and say which ones you skipped.

Print the list before scraping anything so the user can stop you if it's wrong.

---

## Step 2 — Scrape each competitor

Actor: **`curious_coder~facebook-ads-library-scraper`**. (The official `apify/facebook-ads-scraper` takes a different input format — this is the one that works here.)

```bash
curl -s -X POST "https://api.apify.com/v2/acts/curious_coder~facebook-ads-library-scraper/runs?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "urls": [{"url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&is_targeted_country=false&media_type=all&search_type=page&sort_data[direction]=desc&sort_data[mode]=total_impressions&view_all_page_id=PAGE_ID_HERE"}],
    "count": 100,
    "scrapeAdDetails": false
  }'
```

> ### ⚠️ Always send `count`. Never send `maxAds`.
> This actor has **no `maxAds` field** — passing it is silently ignored and the actor scrapes *every*
> matching ad. It bills **per ad** ($0.00075 each), so an unbounded run on a heavy advertiser can cost
> far more than you expected. `count: 100` is the actor's own default and is plenty to read a
> competitor's strategy. Only raise it if the user explicitly asks.

The call is **async**. It returns a `runId` and a `defaultDatasetId`:

```bash
# poll until status = SUCCEEDED
curl -s "https://api.apify.com/v2/actor-runs/$RUN_ID?token=$APIFY_TOKEN"
# then fetch
curl -s "https://api.apify.com/v2/datasets/$DATASET_ID/items?token=$APIFY_TOKEN"
```

**Run competitors one at a time, not in parallel** — parallel runs trip Apify's rate limit. Each takes 30–90 seconds. Print progress as you go: `[3/15] Brand Name: 23 ads (12 video, 5 image, 6 DCO)`.

---

## Step 3 — Dedup before inserting

Pull the existing `Ad Archive ID` values from `Ad Research` and build a set. Then:

- In the scrape, **not** in Airtable → insert as new.
- In both, already marked active → skip, nothing to update.
- In Airtable, **not** in the scrape → set `Is Active` = false and `End Date` = today.

That last rule is what makes the database worth keeping. When a competitor kills an ad, you capture the date — and `Days Running` freezes at its final length, so you learn exactly how long their winners lasted.

---

## Step 4 — Transform and insert

Apify's response nests the creative under a snapshot object. The fields that matter:

```python
from datetime import datetime, date

# Start date — Apify returns unix timestamps
start_iso = datetime.utcfromtimestamp(int(ad["start_date"])).date().isoformat() if ad.get("start_date") else None

platform_map = {"facebook": "Facebook", "instagram": "Instagram",
                "messenger": "Messenger", "audience_network": "Audience Network"}
platforms = [platform_map[p.lower()] for p in ad.get("publisher_platform", []) if p.lower() in platform_map]

format_map = {"VIDEO": "Video", "IMAGE": "Image", "CAROUSEL": "Carousel", "DCO": "DCO"}
display_format = format_map.get(snap.get("display_format"))

videos = snap.get("videos", [])
images = snap.get("images", [])
video_url = (videos[0].get("video_hd_url") or videos[0].get("video_sd_url")) if videos else None
image_url = (images[0].get("original_image_url") or images[0].get("resized_image_url")) if images else None

# Preview thumbnail
preview_url = videos[0].get("video_preview_image_url") if videos else image_url

body = snap.get("body", {})
body_text = body.get("text", "") if isinstance(body, dict) else ""

ad_library_url = f"https://www.facebook.com/ads/library/?id={ad['ad_archive_id']}"
```

Write `Ad Archive ID`, `Page Name`, `Page ID`, `Competitor` (linked record), `Ad Library URL`, `Is Active` = true, `Scrape Date`, `Scrape Batch ID` (the dataset id), plus whatever creative fields exist. **Only send fields that have values** — don't push nulls.

**Airtable caps writes at 10 records per request.** Batch every create and update.

---

## Step 5 — Transcribe the video ads

For each new ad with a `Video URL`:

```bash
curl -s -L -o /tmp/ad_videos/NAME.mp4 --max-time 60 "VIDEO_URL"
ffprobe -v quiet -print_format json -show_format -show_streams /tmp/ad_videos/NAME.mp4
ffmpeg -i /tmp/ad_videos/NAME.mp4 -ar 16000 -ac 1 -f wav /tmp/ad_videos/NAME.wav -y
whisper-cli -m MODEL_PATH -f /tmp/ad_videos/NAME.wav --no-prints
```

Use whatever whisper.cpp binary and model the user has installed — find them rather than assuming a path. The `tiny.en` model (~75MB) is fast and accurate enough for ad copy.

**Aspect ratio** from the ffprobe width/height: `9:16`, `4:5`, `1:1`, `16:9`, else `Other` (tolerance ±0.05).

**Hook extraction** — don't hard-cut at N words. Split on sentence boundaries (`.` `!` `?`) and take the first one or two complete sentences; if the first is under 8 words, include the second. The hook is the most valuable single field in the table, so get it clean.

Delete the downloaded files when you're done.

---

## Step 6 — Classify angle and format

Classify **every** new ad, video or not, from the body text + title + transcript.

**Angle Category**

| Angle | Signals |
|---|---|
| Social Proof | Testimonials, revenue numbers, case studies, "since we started" |
| Pain-to-Transformation | Fees, lost money, switching pain, "save you thousands" |
| Tips/Education | How-to, listicles, "reason number", educational framing |
| Growth Problem | Scaling, more customers, "increase your…" |
| Profit Problem | Costs, margins, wasted spend, "paying too much" |
| Authority | "only platform", proprietary data, expert positioning |
| Scarcity/Urgency | Limited time, spots filling, deadlines |
| Behind-the-Scenes | Day-in-the-life, process reveal, "how we…" |

**Ad Format Type**

| Format | How to spot it |
|---|---|
| UGC Testimonial | First-person experience — "my name is", "our sales" |
| UGC Talking Head | Presenter explaining features or benefits |
| Interview/Case Study | Q&A pattern, more than one speaker |
| Motion Graphics | No speech — transcript is music or silence |
| Static Image | Image or DCO with real written copy |
| Screenshot/UI Demo | Image showing a product interface |
| Other | DCO with template variables, or unclassifiable |

---

## Step 7 — Summarize

```
=== Ad Scrape Complete ===
Competitors scraped: 15
Total ads found: 638
  New: 198 · Already existed: 142 · Marked inactive: 7

By angle:      Social Proof 89 · Pain-to-Transformation 45 · Growth 32 …
Videos transcribed: 67
Longest-running ads (these are the winners):
  1. 94 days — "<hook>" — <brand>
  2. 71 days — "<hook>" — <brand>
Estimated Apify cost: ~$1.13
```

Lead the summary with **longest-running ads**, not newest. Those are the ones worth copying.

---

## When it breaks

- **Ad Library page ID ≠ profile ID.** The URL needs `view_all_page_id=<Ad Library id>`. Get it from the Apify response (`pageAdLibrary.id`), or open the brand in the Ad Library and read it out of the URL. A profile ID returns zero ads and looks like the brand isn't advertising.
- **Costs more than expected.** You sent `maxAds` instead of `count` — see the warning in Step 2.
- **Airtable writes fail with an unknown-field error.** You're using field IDs from someone else's base. Resolve by name at runtime (Step 0).
- **Airtable rejects a batch.** More than 10 records per request, or you sent a `null`. Batch by 10, skip empty values.
- **DCO ads have no media.** `Display Format` = DCO with body text like `{{product.name}}` means Meta assembles the creative on the fly. Skip the video pipeline for these — they aren't broken.
- **Whisper fails on a file.** Confirm the download actually has an audio stream with `ffprobe`. Some ad videos are silent by design.
- **Zero ads for an active brand.** They may genuinely have no live ads in that country. Check `country=US` in the URL.

## Go deeper

- Apify actor — https://apify.com/curious_coder/facebook-ads-library-scraper
- Meta Ad Library — https://www.facebook.com/ads/library/
- Airtable PAT — https://airtable.com/create/tokens
- whisper.cpp — https://github.com/ggerganov/whisper.cpp
