# n8n Job Finder Bot

[![n8n](https://img.shields.io/badge/n8n-workflow%20automation-EA4B71?logo=n8n&logoColor=white)](https://n8n.io/)
[![Telegram](https://img.shields.io/badge/Telegram-bot%20control-26A5E4?logo=telegram&logoColor=white)](https://telegram.org/)
[![Local LLM](https://img.shields.io/badge/local%20LLM-Qwen%202.5%207B-4B8BBE)](https://qwenlm.github.io/)
[![Google Sheets](https://img.shields.io/badge/logging-Google%20Sheets-34A853?logo=googlesheets&logoColor=white)](https://www.google.com/sheets/about/)

Telegram-controlled job hunter built with n8n. Send a search query, get a ranked list of remote jobs scored by a local AI model. Paste any job URL, get a fit verdict. Every result is logged to Google Sheets, so the same job never comes back twice. Runs daily at 10:00 automatically.

## What it does

- **Telegram remote control** — `find junior developer` → the bot scans
- **Daily automatic scan** — 10:00 every morning, no tunnel, no interaction
- **Job analysis** — paste any job URL → the bot scrapes the page (r.jina.ai), extracts title/salary/location with AI, answers with a fit verdict 🟢🟡🔴
- **Remotive API** — free remote-jobs source
- **Dedupe** — checks every job URL against the Google Sheets log, only new jobs pass
- **AI scoring** — local Qwen 2.5 7B (LM Studio) scores jobs 0–100 against my profile: entry-level dev, automation/n8n/QA, remote Europe
- **Ranked reply** — 🟢 (≥70) / 🟡 (≥40) / 🔴 to my Telegram
- **Logging** — Date | Query | Score | Title | Company | Location | URL | Reason

## Architecture: caller → engine

The bot is **three workflows**, split on purpose:

| **Workflow** | **Role** |
|---|---|
| **`Job finder`** | Telegram remote control — chat / find / job URL commands |
| **`Job Scan Engine`** | stateless worker — receives **`{query, chatId}`**, scans, scores, delivers |
| **`Daily 10am Job Scan`** | scheduler — at 10:00 (Europe/Skopje) calls the engine |

### Daily 10am Job Scan → Job Scan Engine

```text
Daily 10am Job Scan (caller)
            │
            ▼
Schedule 10:00
            │
            ▼
Daily Config
            │
            ▼
Call Engine ═══════════════════════════════╗
                                           ║
                                           ▼
                                  Job Scan Engine (worker)
                                           │
                                           ▼
                                       Scan Input
                                           │
                                           ▼
                                    Remotive Search
                                           │
                                           ▼
                                     Get Seen URLs
                                           │
                                           ▼
                                    Prepare Scoring
                                     (dedupe, cap 10)
                                           │
                                           ▼
                                    IF has new jobs?
                                      /          \
                                   TRUE          FALSE
                                    │               │
                                    ▼               ▼
                              Qwen Score Jobs   Telegram
                                    │           "no new jobs"
                                    ▼
                                Merge & Rank
                                  /      \
                                 ▼        ▼
                         Telegram list  Split Rows
                                           │
                                           ▼
                                   Google Sheets log
```

## Job URL analysis branch

```text
Telegram URL → Route Command (regex) → Switch

  └─ job → Fetch Job Page (r.jina.ai reader) → Prepare Extract

           (clean junk, cap 3500 chars) → Qwen Extract Job (7 labeled lines)

           → Format Job Reply (tolerant parser) → Telegram verdict

```


## Design decisions

- **Google Sheet = the dedupe database** — no extra infrastructure, human-readable log.
- **1 item into HTTP nodes** — n8n runs HTTP nodes once PER input item; uncontrolled upstream rows = duplicate API calls.
- **Local LLM = $0 cost** — LM Studio serves Qwen 2.5 7B on a second PC via an OpenAI-compatible API. Provider swap = change 3 fields.
- **Worker without Telegram trigger** — the scheduled pair runs without the cloudflared tunnel (tunnel is only needed for inbound webhooks).

## Tech stack

n8n (npm, Windows) · Telegram Bot API · Remotive API · r.jina.ai reader · Google Sheets (OAuth) · LM Studio + Qwen 2.5 7B · cloudflared tunnel (Telegram inbound only)

## Setup

1. Import the JSON files from **`workflow/`** into n8n.
2. Create credentials: Telegram bot token, Google Sheets OAuth.
3. Point AI nodes at your OpenAI-compatible endpoint (LM Studio: **`http://<ai-pc-ip>:1234/v1/chat/completions`**).
4. Create a **"Job Tracker"** sheet with columns: `Date | Query | Score | Title | Company | Location | URL | Reason`.
5. In Telegram send **`find automation`** - bot replies with a ranked list.
6. Or paste any job URL - bot replies with a fit verdict.
