# n8n

A collection of self-hosted [n8n](https://n8n.io) workflow automation templates, covering AI integrations, notifications, and data tracking.

## Workflows

| File | What it does |
|---|---|
| `daily-ai-news-digest.json` | Emails a ranked, deduplicated AI news digest every morning (see below) |
| `discord-ai-auto-moderator.json` | Scheduled AI moderation of Discord messages |
| `gemini-chat-assistant.json` | Chat agent powered by Google Gemini |
| `portfolio-tracker.json` | Scheduled portfolio tracking in Google Sheets |
| `predictive-insights-oracle.json` | Form input → Gemini insights → logged to Sheets and emailed |
| `transactional-email-sender.json` | Sends emails via Gmail from a Google Sheet |

## Daily AI News Digest

Every day at 10 AM it emails the 5 to 8 most relevant AI news items from the last 24 hours. Each item gets a one or two sentence summary and its source. Stories already sent are never repeated, even when another outlet rewrites them. It runs on free tiers only.

```
Schedule (10 AM) → RSS feeds (TechCrunch, The Verge, MIT Tech Review,
VentureBeat, OpenAI, Google AI) → Merge → last-24h filter
→ Gemini embeddings → Pinecone similarity check (drops stories already sent)
→ score against your topic preferences (Google Sheet) → top 5–8
→ Gemini summaries → HTML email via Gmail
→ log sent stories to Pinecone + Google Sheet
```

**Design choices**

- **Semantic dedup with Pinecone.** Comparing embeddings catches the same story from different outlets, which URL matching misses.
- **Preferences in a Google Sheet.** You change which topics rank higher without editing the workflow.
- **The LLM only writes summaries.** Filtering, dedup and ranking use embeddings and code, which keeps the workflow fast and free.

**Setup**

1. Create a free Pinecone serverless index with 768 dimensions (the workflow requests 768-dimension `gemini-embedding-001` vectors).
2. In n8n, create credentials for Google Gemini (free API key from Google AI Studio), Pinecone, Gmail (OAuth2) and Google Sheets (OAuth2).
3. Create a Google Sheet with two tabs: `AI News Prefs` (topics to prioritise or deprioritise) and `Sent Articles` (log).
4. Import the JSON, then replace the placeholders:
   - `YOUR_GOOGLE_SHEET_ID`: your sheet's ID
   - `YOUR-PINECONE-INDEX-HOST.pinecone.io`: your index host
   - `YOUR_EMAIL@gmail.com`: where the digest is sent
5. Attach your credentials to each node, run it once manually, then activate it.

## Usage

1. Import any `.json` file into your n8n instance.
2. Configure the required credentials and API keys.
3. Activate the workflow.

## Requirements

- n8n (self-hosted or cloud)
- API keys/credentials specific to each workflow
