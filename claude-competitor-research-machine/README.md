# Competitor Research Machine

A Claude Code skill that watches every competitor in your niche, pulls anything that beat its
own channel's average, and tears it down — hook, framework, format, topic — so you stop
guessing what's working.

**Full setup guide:** https://noevarner.com/resources/claude-competitor-research-machine

## What's in here

| File | What it is |
|---|---|
| `SKILL.md` | The skill. Discovery loop, dedup, the run, the weekly brief. |
| `references/teardown.md` | The analysis engine — download, transcribe, read the hook, extract structure. |

## Install

Drop the folder into your skills directory:

```
~/.claude/skills/analyze-competitors/
├── SKILL.md
└── references/
    └── teardown.md
```

Then type `/analyze-competitors` in Claude Code.

## You'll also need

- A **Sandcastles** account — https://sandcastles.ai/?ref=noe (paid, no free tier). This is
  the discovery layer that crawls your watchlist and scores outliers.
- `yt-dlp` and `ffmpeg` installed
- An `OPENAI_API_KEY` in your `.env` for Whisper

Full step-by-step in the guide linked above.

## The part most people skip

Run the same teardown on **your own posts**, saved with `source='own'` alongside the
competitor rows. One store, one schema, both sides. Then "where are they beating me" is a
query instead of a feeling.
