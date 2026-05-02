# 📊 Guest Sentiment Dashboard

> An automated AI-powered sentiment analysis system that aggregates guest feedback from Facebook, Instagram, and Google Reviews — then delivers weekly business intelligence directly to the resort owner via Telegram.

***

## 🗂️ Table of Contents

- [Overview](#overview)
- [Core Objectives](#core-objectives)
- [System Architecture](#system-architecture)
- [Workflow Diagram](#workflow-diagram)
- [Node-by-Node Breakdown](#node-by-node-breakdown)
- [AI Analysis Engine](#ai-analysis-engine)
- [Knowledge Base Leak Detection](#knowledge-base-leak-detection)
- [Output: Telegram Reports](#output-telegram-reports)
- [Google Sheets Data Structure](#google-sheets-data-structure)
- [Environment Variables & Credentials](#environment-variables--credentials)
- [Setup & Installation](#setup--installation)
- [Tech Stack](#tech-stack)
- [Sample Output](#sample-output)
- [Project Status](#project-status)

***

## Overview

The **Guest Sentiment Dashboard** is a fully automated n8n workflow that transforms raw, unstructured guest feedback into actionable business intelligence for resort owners. Instead of manually reading hundreds of comments and reviews each week, the system collects, normalizes, and analyzes all guest interactions using AI — then delivers a concise executive summary directly to the owner's Telegram.

**The system answers three core business questions:**
- Are guests happy with their experience?
- What is the #1 complaint this week?
- What are guests asking about most frequently?

***

## Core Objectives

| Objective | Description |
|-----------|-------------|
| **Sentiment Scoring** | Classify every piece of feedback as Positive, Neutral, or Negative |
| **Category Analysis** | Break down feedback by business area: Price, Facility, Staff, Nature |
| **Trend Detection** | Identify recurring themes and topics across all channels |
| **KB Leak Alerting** | Flag questions asked 3+ times as gaps in the resort's FAQ/Knowledge Base |
| **Weekly Reporting** | Deliver a formatted executive summary via Telegram every Sunday midnight |
| **Long-term Tracking** | Log all data to Google Sheets for trend analysis and Looker Studio dashboards |

***

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        WEEKLY TRIGGER (Sunday Midnight)              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
                     ┌─────────────────┐
                     │  Set Date Range  │  (last 7 days, Unix timestamps)
                     └────────┬────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                  ▼
   ┌──────────────┐  ┌──────────────────┐  ┌───────────────────┐
   │   Facebook   │  │    Instagram     │  │   Google Reviews  │
   │   Comments   │  │    Comments      │  │                   │
   └──────┬───────┘  └────────┬─────────┘  └────────┬──────────┘
          │                   │                      │
          └───────────────────┴──────────────────────┘
                               │
                               ▼
                  ┌────────────────────────┐
                  │  Normalize & Merge All  │  (unified schema)
                  │       Feedback          │
                  └────────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────┐
                    │  Prepare AI Batch │  (aggregate into single payload)
                    └────────┬─────────┘
                             │
                             ▼
                  ┌─────────────────────────┐
                  │  AI Sentiment Engine     │
                  │  (GPT-4o-mini via OpenAI)│
                  └────────────┬────────────┘
                               │
                               ▼
                    ┌──────────────────┐
                    │  Parse AI Analysis│  (build Telegram message + data)
                    └────────┬─────────┘
                             │
          ┌──────────────────┼──────────────────────┐
          ▼                  ▼                       ▼
  ┌──────────────┐  ┌─────────────────┐   ┌──────────────────────┐
  │ Send Telegram│  │ Prepare Sheets  │   │ Prepare KB Leaks Data│
  │   Report     │  │     Data        │   └──────────┬───────────┘
  └──────────────┘  └───────┬─────────┘              │
                            │                         ▼
                            ▼                  ┌──────────────┐
                  ┌─────────────────────┐      │ Has KB Leaks?│
                  │ Log to Google Sheets│      └──────┬───────┘
                  │   (WeeklySummary)   │    ┌────────┴────────┐
                  └─────────────────────┘    ▼                 ▼
                                     ┌──────────────┐  ┌──────────────┐
                                     │ Telegram KB  │  │Log KB Leaks  │
                                     │    Alert     │  │(PendingFAQ)  │
                                     └──────────────┘  └──────────────┘

          ┌─────────────────────────┐
          │ Prepare Feedback Log    │
          │ Data                    │
          └──────────┬──────────────┘
                     ▼
          ┌─────────────────────────┐
          │ Log to Google Sheets    │
          │    (FeedbackLog)        │
          └─────────────────────────┘
```

***

## Workflow Diagram

The visual schema below shows the actual n8n canvas layout:



> The workflow runs in a linear-then-fan-out pattern: triggers → data collection (parallel) → normalization → AI analysis → fan out to 4 simultaneous output branches.

***

## Node-by-Node Breakdown

### 1. 🕛 Weekly Trigger (Sunday Midnight)
- **Type:** `n8n-nodes-base.scheduleTrigger`
- **Schedule:** Every week (Sunday at midnight)
- **Purpose:** Kicks off the entire pipeline on a fixed cadence. No manual intervention needed.

***

### 2. 📅 Set Date Range
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:** Calculates the Unix timestamps for "now" and "7 days ago". Passes `since`, `until`, `sinceISO`, `untilISO`, and a formatted `reportDate` string downstream.

```javascript
const since = Math.floor(sevenDaysAgo.getTime() / 1000);
const until = Math.floor(now.getTime() / 1000);
```

***

### 3. 📘 Fetch Facebook Comments
- **Type:** `n8n-nodes-base.httpRequest`
- **Endpoint:** `https://graph.facebook.com/v19.0/{FB_PAGE_ID}/feed`
- **Fields fetched:** `message`, `comments{message, from, created_time}`, `created_time`
- **Limit:** 100 posts
- **Auth:** Facebook Page Access Token (via environment variable)

***

### 4. 📷 Fetch Instagram Comments
- **Type:** `n8n-nodes-base.httpRequest`
- **Endpoint:** `https://graph.facebook.com/v19.0/{IG_USER_ID}/media`
- **Fields fetched:** `caption`, `comments{text, username, timestamp}`
- **Limit:** 50 media items
- **Auth:** HTTP Header Auth (`Authorization: Bearer {token}`)

***

### 5. ⭐ Fetch Google Reviews
- **Type:** `n8n-nodes-base.httpRequest`
- **Endpoint:** `https://mybusiness.googleapis.com/v4/accounts/{GOOGLE_ACCOUNT_ID}/locations/{GOOGLE_LOCATION_ID}/reviews`
- **Page size:** 50
- **Auth:** Google OAuth2 API (predefined credential)
- **Note:** Filters reviews server-side to only include entries from the last 7 days.

***

### 6. 🔀 Normalize & Merge All Feedback
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:** Takes raw, disparate data from all three sources and maps them into a **unified schema**:

```javascript
{
  source: "Facebook" | "Instagram" | "Google",
  type: "post" | "comment" | "review",
  text: "<feedback content>",
  author: "<name or username>",
  timestamp: "<ISO string>",
  rating: "<FIVE|FOUR|THREE|TWO|ONE>"  // Google only
}
```

- **Fallback:** If all three APIs return empty (e.g., during testing), the node injects a set of **12 realistic mock feedback items** so the rest of the pipeline can be tested end-to-end without live API connections.

***

### 7. 📦 Prepare AI Batch
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:** Aggregates all normalized feedback items into a **single JSON payload** with sequential IDs. This avoids sending individual AI requests per message — instead, the entire week's feedback is analyzed in one efficient batch call.

```javascript
{
  totalFeedback: <number>,
  feedbackBatch: "<JSON string of all items>",
  reportDate: "<formatted date>"
}
```

***

### 8. 🤖 AI Sentiment Engine (GPT-4o)
- **Type:** `@n8n/n8n-nodes-langchain.openAi`
- **Model:** `gpt-4o-mini`
- **Temperature:** `0.2` (deterministic, consistent outputs)
- **Max Tokens:** `2000`
- **Role:** Hospitality business intelligence analyst

See [AI Analysis Engine](#ai-analysis-engine) section for full prompt and output schema.

***

### 9. 🔍 Parse AI Analysis
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:** Parses the raw AI JSON response, strips any accidental markdown fencing, and builds the formatted **Telegram message string**. Also passes the full parsed `analysis` object downstream to all output branches.
- **Fallback:** If JSON parsing fails, the node injects a hardcoded mock analysis object to prevent pipeline failure.

***

### 10. 📤 Send Telegram Report
- **Type:** `n8n-nodes-base.telegram`
- **Parse Mode:** Markdown
- **Chat ID:** `$env.TELEGRAM_CHAT_ID`
- Sends the complete weekly health report. See [Sample Output](#sample-output).

***

### 11. 📊 Prepare Sheets Data → Log to Google Sheets (WeeklySummary)
- Prepares a **single summary row** per week including all sentiment counts, category breakdowns, top keyword, top alert, and the executive summary text.
- Appends to the `WeeklySummary` sheet tab.

***

### 12. 🚩 Prepare KB Leaks Data → Has KB Leaks? → Conditional Branch
- **Has KB Leaks?** is an `if` node that checks whether any knowledge base leaks were identified.
- **True branch:** Sends a **Telegram KB Alert** AND logs to the `PendingFAQ` sheet.
- **False branch:** Does nothing (no leaks this week).

***

### 13. 📋 Prepare Feedback Log Data → Log to Google Sheets (FeedbackLog)
- Maps every individual feedback item (with its AI-assigned sentiment, category, and key theme) into a row in the `FeedbackLog` sheet tab.
- Enables long-term, item-level trend analysis.

***

## AI Analysis Engine

The AI node uses a carefully structured system prompt to act as a **hospitality business intelligence analyst**. It receives the full batch of feedback and returns a single, strictly-typed JSON object.

### System Prompt Role
> *"You are a hospitality business intelligence analyst specializing in guest sentiment analysis for resorts. You analyze guest feedback with precision and return ONLY valid JSON."*

### AI Output Schema

```json
{
  "summary": {
    "totalAnalyzed": 12,
    "positiveCount": 8,
    "neutralCount": 2,
    "negativeCount": 2,
    "positivePercent": 67,
    "overallSentiment": "Positive",
    "topKeyword": "Sunset",
    "topAlert": "Weak WiFi Signal"
  },
  "categories": {
    "Price":    { "mentions": 3, "sentiment": "Neutral",  "sample": "Prices seem high but worth it" },
    "Facility": { "mentions": 4, "sentiment": "Negative", "sample": "Weak signal throughout our stay" },
    "Staff":    { "mentions": 3, "sentiment": "Positive", "sample": "Staff was incredibly friendly" },
    "Nature":   { "mentions": 6, "sentiment": "Positive", "sample": "Sunset was breathtaking" }
  },
  "individualResults": [
    { "id": 1, "sentiment": "Positive", "category": "Nature", "keyTheme": "Sunset View" }
  ],
  "trends": [
    { "theme": "Weak WiFi / Signal", "count": 3, "description": "...", "isKnowledgeBaseLeak": true }
  ],
  "knowledgeBaseLeaks": [
    { "question": "Is there a swimming pool?", "frequency": 3, "suggestedAnswer": "..." }
  ],
  "actionableInsights": [
    { "priority": "High", "area": "Facility", "finding": "...", "recommendation": "..." }
  ],
  "execSummary": "2-3 sentence executive summary..."
}
```

### Sentiment Categories

| Category | What It Covers |
|----------|---------------|
| **Price** | Comments about cost, value for money, pricing perception |
| **Facility** | WiFi, rooms, amenities, infrastructure, utilities |
| **Staff** | Service quality, friendliness, responsiveness |
| **Nature** | Views, environment, outdoor experience, wildlife |

***

## Knowledge Base Leak Detection

One of the most powerful features of this system is its ability to automatically identify **gaps in the resort's public FAQ or information materials**.

**How it works:**
1. The AI analyzes all feedback for repeated questions.
2. Any question or concern mentioned **3 or more times** in a single week is flagged as a **"Knowledge Base Leak"** — meaning guests are asking because the information is not easily accessible.
3. The AI generates a **draft FAQ answer** for each leak.
4. A dedicated **Telegram KB Alert** is sent to the owner.
5. The leak is logged to the `PendingFAQ` Google Sheet tab with a `Pending Review` status for the owner to approve and publish.

**Example KB Leak:**

```
🚩 KNOWLEDGE BASE ALERT

❓ Repeated Question: "Is there a swimming pool?"
📊 Asked 3 times this week

💬 Suggested Answer for FAQ:
We currently do not have a swimming pool, but we offer natural
swimming spots nearby. We recommend bringing swimwear for the river.

➡️ Action: Add this to your booking page FAQ & Messenger auto-reply.
✅ Logged to PendingFAQ sheet.
```

***

## Output: Telegram Reports

### Weekly Health Report

The system sends a formatted Markdown message to the owner's Telegram chat every Sunday:

```
📊 WEEKLY RESORT HEALTH REPORT
📅 Sunday, May 3, 2026
━━━━━━━━━━━━━━━━━━━━

🟢 SENTIMENT SCORE: 67% Positive
   Total Feedback Analyzed: 12
   ✅ Positive: 8 | ⚠️ Neutral: 2 | ❌ Negative: 2

🏷️ TOP KEYWORD: "Sunset"
🚨 TOP ALERT: "Weak WiFi Signal"

📂 CATEGORY BREAKDOWN:
   😊 Nature: 6 mention(s) — Positive
   😊 Staff: 3 mention(s) — Positive
   😐 Price: 3 mention(s) — Neutral
   😟 Facility: 4 mention(s) — Negative

📈 THIS WEEK'S TRENDS:
   • "Weak WiFi / Signal" — 3x 🚩 KB Gap
   • "Swimming Pool Request" — 3x 🚩 KB Gap
   • "Sunset Experience" — 4x

💡 KNOWLEDGE BASE GAPS (3+ asks):
   • "Is there a swimming pool?" (3x asked)
   • "How is the WiFi/signal here?" (3x asked)

🎯 HIGH-PRIORITY ACTIONS:
   🔺 [Facility] Add a clear "Off-Grid Digital Detox" disclaimer to all booking pages.
   🔺 [FAQ] Add pool/swimming FAQ to booking page and Messenger auto-reply.

📝 EXECUTIVE SUMMARY:
This week showed strong overall positive sentiment driven by
nature experiences and staff warmth. The primary alert is recurring
WiFi complaints — an immediate FAQ/disclaimer update is recommended.

━━━━━━━━━━━━━━━━━━━━
✅ Full data logged to Google Sheets
```

**Sentiment indicator logic:**
- 🟢 Green = 70%+ positive
- 🟡 Yellow = 50–69% positive
- 🔴 Red = below 50% positive

***

## Google Sheets Data Structure

The system writes to **three tabs** in a single Google Sheets document:

### Tab 1: `WeeklySummary`

| Column | Description |
|--------|-------------|
| Report Date | Week of analysis |
| Total Analyzed | Total feedback count |
| Positive Count | Number of positive items |
| Neutral Count | Number of neutral items |
| Negative Count | Number of negative items |
| Positive % | Percentage positive |
| Overall Sentiment | Positive / Neutral / Negative |
| Top Keyword | Most mentioned positive word |
| Top Alert | Most mentioned complaint |
| Price Mentions | Count of price-related feedback |
| Price Sentiment | Sentiment for Price category |
| Facility Mentions | Count of facility-related feedback |
| Facility Sentiment | Sentiment for Facility category |
| Staff Mentions | Count of staff-related feedback |
| Staff Sentiment | Sentiment for Staff category |
| Nature Mentions | Count of nature-related feedback |
| Nature Sentiment | Sentiment for Nature category |
| Executive Summary | AI-generated 2-3 sentence summary |

### Tab 2: `PendingFAQ`

| Column | Description |
|--------|-------------|
| Date Flagged | Week the leak was detected |
| Repeated Question | The verbatim question guests are asking |
| Times Asked | Frequency count (threshold: 3+) |
| Suggested Answer | AI-drafted FAQ answer for owner review |
| Status | `Pending Review` (owner updates to `Published`) |

### Tab 3: `FeedbackLog`

| Column | Description |
|--------|-------------|
| Report Date | Week of analysis |
| Source | Facebook / Instagram / Google |
| Type | post / comment / review |
| Author | Guest name or username |
| Feedback Text | Raw feedback content |
| Timestamp | Original post/review time |
| Sentiment | Positive / Neutral / Negative |
| Category | Price / Facility / Staff / Nature |
| Key Theme | 2-3 word AI-assigned theme label |

> 💡 **Pro Tip:** Connect the `WeeklySummary` tab to **Looker Studio** for a live visual dashboard showing sentiment trends over time.

***

## Environment Variables & Credentials

Configure these in your n8n instance before activating the workflow:

### Environment Variables (n8n `.env` or Settings → Variables)

| Variable | Description | Example |
|----------|-------------|---------|
| `FB_PAGE_ID` | Facebook Page numeric ID | `123456789012345` |
| `IG_USER_ID` | Instagram Business User ID | `987654321098765` |
| `GOOGLE_ACCOUNT_ID` | Google My Business account ID | `accounts/1234567890` |
| `GOOGLE_LOCATION_ID` | Google My Business location ID | `locations/9876543210` |
| `GOOGLE_SHEET_ID` | Google Sheets document ID (from URL) | `1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms` |
| `TELEGRAM_CHAT_ID` | Owner's Telegram Chat ID | `-1001234567890` |

### n8n Credentials Required

| Credential Type | Used By | Notes |
|----------------|---------|-------|
| **HTTP Header Auth** | Fetch Instagram Comments | Set header `Authorization: Bearer {IG_ACCESS_TOKEN}` |
| **Google OAuth2 API** | Fetch Google Reviews | Scope: `https://www.googleapis.com/auth/business.manage` |
| **Google Sheets OAuth2** | All Google Sheets nodes | Scope: `https://www.googleapis.com/auth/spreadsheets` |
| **OpenAI API** | AI Sentiment Engine | Standard API key, model: `gpt-4o-mini` |
| **Telegram API** | Send Telegram Report, Telegram KB Alert | Bot token from @BotFather |

***

## Setup & Installation

### Prerequisites
- n8n instance (self-hosted or cloud) — v1.0+
- OpenAI API account with GPT-4o-mini access
- Facebook Developer App with Pages API access
- Instagram Business Account connected to Facebook Page
- Google My Business account with verified location
- Google Cloud project with My Business API enabled
- Telegram bot created via @BotFather

### Step-by-Step Setup

**1. Import the Workflow**
```
n8n → Workflows → Import from File
Select: RRE-AUTO-04-_-Guest-Sentiment-Dashboard.json
```

**2. Configure Credentials**
Go to `Settings → Credentials` in n8n and create:
- OpenAI API key credential
- HTTP Header Auth for Instagram (Bearer token)
- Google OAuth2 for Google Reviews
- Google Sheets OAuth2
- Telegram Bot API

**3. Set Environment Variables**
In n8n, go to `Settings → Environment Variables` and add all variables listed in the table above.

**4. Set Up Google Sheets**
Create a new Google Spreadsheet with three tabs named exactly:
- `WeeklySummary`
- `PendingFAQ`
- `FeedbackLog`

Add the column headers matching the tables in the [Google Sheets Data Structure](#google-sheets-data-structure) section.

**5. Get Your Telegram Chat ID**
- Message your bot on Telegram
- Visit: `https://api.telegram.org/bot{YOUR_BOT_TOKEN}/getUpdates`
- Copy the `chat.id` value and set it as `TELEGRAM_CHAT_ID`

**6. Test the Workflow**
- Click **Execute Workflow** manually to test
- If APIs are not yet connected, the `Normalize & Merge All Feedback` node will automatically use **mock data** — the full pipeline will still run and you can verify all outputs

**7. Activate**
Toggle the workflow to **Active**. It will now run every Sunday at midnight automatically.

***

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Orchestration** | n8n (self-hosted) | Workflow automation engine |
| **Data Sources** | Facebook Graph API v19.0 | Page comments and posts |
| **Data Sources** | Instagram Graph API | Media comments |
| **Data Sources** | Google My Business API v4 | Location reviews |
| **AI Engine** | OpenAI GPT-4o-mini | Sentiment analysis and categorization |
| **Notifications** | Telegram Bot API | Owner alerts and weekly reports |
| **Data Storage** | Google Sheets | Long-term trend tracking |
| **Visualization** | Looker Studio (optional) | Dashboard from Sheets data |
| **Language** | JavaScript (Node.js) | All Code nodes |

***

## Sample Output

### Telegram Weekly Report (Live Example)

Based on the built-in mock data used for testing, the system produces output like this:

```
📊 WEEKLY RESORT HEALTH REPORT
📅 Sunday, May 3, 2026
━━━━━━━━━━━━━━━━━━━━

🟡 SENTIMENT SCORE: 67% Positive
   Total Feedback Analyzed: 12
   ✅ Positive: 8 | ⚠️ Neutral: 2 | ❌ Negative: 2

🏷️ TOP KEYWORD: "Sunset"
🚨 TOP ALERT: "Weak WiFi Signal"

📂 CATEGORY BREAKDOWN:
   😊 Nature: 6 mention(s) — Positive
   😊 Staff: 3 mention(s) — Positive
   😐 Price: 3 mention(s) — Neutral
   😟 Facility: 4 mention(s) — Negative

📈 THIS WEEK'S TRENDS:
   • "Weak WiFi / Signal" — 3x 🚩 KB Gap
   • "Swimming Pool Request" — 3x 🚩 KB Gap
   • "Sunset Experience" — 4x

💡 KNOWLEDGE BASE GAPS (3+ asks):
   • "Is there a swimming pool?" (3x asked)
   • "How is the WiFi/signal here?" (3x asked)

🎯 HIGH-PRIORITY ACTIONS:
   🔺 [Facility] Add "Off-Grid Digital Detox" disclaimer to booking pages
   🔺 [FAQ] Add pool FAQ to booking page and Messenger auto-reply
   🔺 [Marketing] Feature sunset content in social media and reels

📝 EXECUTIVE SUMMARY:
This week showed strong positive sentiment driven by nature experiences
and staff warmth. Primary alert: recurring WiFi complaints. Three
guests asked about a pool — flagged as a Knowledge Base gap.

━━━━━━━━━━━━━━━━━━━━
✅ Full data logged to Google Sheets
```

***

## Project Status

| Item | Status |
|------|--------|
| n8n Blueprint | ✅ Complete |
| AI Prompt Engineering | ✅ Complete |
| Mock Data Fallback | ✅ Complete |
| Telegram Integration | ✅ Complete |
| Google Sheets Logging | ✅ Complete |
| KB Leak Detection | ✅ Complete |
| Facebook API Integration | ⚙️ Requires API credentials |
| Instagram API Integration | ⚙️ Requires API credentials |
| Google Reviews Integration | ⚙️ Requires OAuth2 setup |
| Looker Studio Dashboard | 📋 Planned |

***

<img width="1770" height="578" alt="RRE-AUTO-04 _ Guest Sentiment Dashboard Schema" src="https://github.com/user-attachments/assets/18d145d8-09ff-46cd-b328-5b6c9c032219" />


## Project Metadata

```
Project Code:  RRE-AUTO-04
Project Name:  Guest Sentiment Dashboard
Workflow File: RRE-AUTO-04-_-Guest-Sentiment-Dashboard.json
n8n Version:   1.x (executionOrder: v1)
AI Model:      gpt-4o-mini
Trigger:       Weekly — Every Sunday at Midnight
Status:        Ready for Deployment
```

***

*Built with ❤️ using n8n, OpenAI, and a little bit of automation magic.*
