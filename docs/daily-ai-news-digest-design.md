# Daily AI News Digest (Gmail, free stack)

An n8n automation that curates AI news every morning and emails a personalized, deduplicated digest at 10 AM — using embeddings + a vector DB for ranking/dedup and a free-tier LLM only for the final summary-writing step. $0 to run.

> Design document for [`daily-ai-news-digest.json`](../daily-ai-news-digest.json). The workflow is built; this records the plan and the reasoning behind it. Where the two differ in small details (for example, Pinecone and Gemini are called through HTTP Request nodes), the JSON is the source of truth.

## Goal

Every day at 10:00 AM, receive one email with the 5-8 most important AI news items from the last 24 hours — each a brief 1-2 sentence summary naming who/what and which outlet it came from, ranked by relevance to your interests, never repeating a story already sent (even if a different outlet reworded it).

## Architecture

```
Schedule Trigger (10 AM daily, explicit timezone)
        │
        ▼
RSS Feed Read × N  ──▶  Merge (combine all sources into one item list)
(curated sources: TechCrunch AI, The Verge AI, Ars Technica AI,
 MIT Tech Review AI, Anthropic blog, OpenAI blog, arXiv cs.AI, ...)
        │
        ▼
Filter (keep items published in the last 24h)
        │
        ▼
Gemini Embeddings (embed title + snippet per article, free tier)
        │
        ▼
Pinecone query: compare against "sent-articles" namespace
──▶ drop anything above similarity threshold (semantic dedup —
    catches the same story reworded by a different outlet,
    not just exact-URL repeats)
        │
        ▼
Code node: embed "AI News Prefs" (Google Sheet) fresh each run,
score each surviving article by cosine similarity to your
prioritize/deprioritize topics → sort → take top 5–8
        │
        ▼
Gemini Chat (free tier): write a 1-2 sentence summary per
selected article → {title, summary, link, source}
        │
        ▼
If (list empty?) ──▶ yes: send "nothing notable today" email
        │ no
        ▼
Format (Code/Set node → HTML email, each item shows its source)
        │
        ▼
Gmail node → send to your address
        │
        ▼
Pinecone upsert (log sent embeddings → powers tomorrow's dedup)
+ Google Sheets: log URL/date/source to "Sent Articles" (audit trail)
```

### Key design decisions

| Decision | Choice | Why |
|---|---|---|
| Delivery channel | **Gmail (OAuth2 node)** | Far simpler setup than WhatsApp — one OAuth consent, no Meta Developer app/test-number/template approval. WhatsApp remains a documented future add-on as a parallel send branch. |
| News source | **Curated RSS feeds** | Free, deterministic ingestion — no API key/rate limits — with ranking applied afterward, not a raw dump. |
| Freshness | **24h Filter node** | Only same-day news, no stale reposts. |
| LLM / embeddings | **Google Gemini (free tier)** | Replaces Claude entirely — $0 cost. One credential covers both embeddings and chat/summarization. |
| Dedup | **Pinecone (free Starter tier)** semantic similarity against everything ever sent | Catches the same story covered by multiple outlets under different URLs — exact-URL dedup alone misses this. Hosted, so no local container to keep running. |
| Personalization | Fresh-embedded **Google Sheet** prefs, cosine-scored per run | Human-editable without touching the workflow; cheap since the prefs list is small, no need to persist those vectors. |
| Summarization | **Gemini Chat**, one call per selected article | The one deliberately generative step — everything upstream (dedup, ranking) is retrieval/scoring, not LLM reasoning, which keeps it fast and free. |
| Output format | Multiple brief items per email, source shown per item | You asked for several short "who did what" items (e.g. "Anthropic launched X"), not one long story — and to always know where each item came from. |

### Deferred / future enhancements (not building yet)

- **WhatsApp as a second delivery branch** — parallel to Gmail, no upstream changes needed.
- **Interactive replies** — reply-to-email parsing ("tell me more about #2", "less about X") to fetch full articles or adjust the Prefs sheet.
- **Weekly meta-digest** — Sunday-only branch summarizing the week's sent articles into themes.
- **Persisted preference vectors** — if the prefs list grows large, move from "re-embed every run" to a persisted `preferences` namespace in Pinecone, synced on Sheet changes.

## Setup (outside n8n)

1. **Create a Pinecone free-tier serverless index** at pinecone.io → grab its host + API key → create the Pinecone credential in n8n.
2. **Google Gemini API key** (free, via Google AI Studio) → create the Gemini credential in n8n.
3. **Gmail OAuth2 credential** in n8n — connect your Google account.
4. **Google Sheets credential** in n8n, plus two sheets:
   - `AI News Prefs` — columns for topics to prioritize / deprioritize.
   - `Sent Articles` — log of sent article URL/date/source (audit trail; Pinecone is the actual dedup source of truth).

## Verification checklist

- [ ] Manual execution sends a correctly formatted email with multiple brief items, each showing its source.
- [ ] Only articles from the last 24h appear.
- [ ] Immediate re-run does **not** resend the same or reworded-duplicate stories (semantic dedup working).
- [ ] Editing the Prefs sheet to exclude a topic changes ranking on the next run.
- [ ] Schedule trigger's cron + timezone actually fires at 10:00 AM local time.

