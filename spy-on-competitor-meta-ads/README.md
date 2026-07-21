# Spy On Your Competitors' Meta Ads

A Claude skill that pulls every ad your competitors are running on Meta into your own Airtable base — the copy, the headline, the landing page, and **how long each ad has been live**.

That last one is the point. Nobody keeps a losing ad running. An ad that's been live 60+ days is one they're making money on, and now you can see it.

## What's in here

| File | What it is |
|------|------------|
| `SKILL.md` | The skill. Install it into Claude Code and it does the whole job — builds your tables, scrapes, transcribes the video ads, classifies the angles. |

## Install it

In Claude Code:

```
npx skills add noe-varner/noeai-free-resources
```

No download needed — it drops straight into your skills folder.

Prefer to read it first? Open `SKILL.md` above. It's plain text, and you should always read a skill before you run it.

## What you'll need

- **An Apify account** — free, and it comes with $5/month of credit. A full run across 15 competitors costs about **$1.13**, so the free credit covers you.
- **An Airtable account** — free tier is fine.
- **Claude Code.**
- *Optional:* whisper.cpp + ffmpeg if you want the video ads transcribed. Everything else works without them.

## How to use it

Point Claude at it and say what you want:

```
Set up my competitor ad base, then scrape everyone in it.
```

First run, it builds the two tables for you and asks for your tokens. After that, just:

```
Refresh my competitor ads.
```

Ads that vanished since last time get marked inactive with an end date — so you learn exactly how long their winners lasted before they killed them.

---

Full written walkthrough: **[noevarner.com/resources/spy-on-competitor-meta-ads](https://noevarner.com/resources/spy-on-competitor-meta-ads)**
