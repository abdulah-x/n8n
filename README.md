# n8n

A collection of self-hosted [n8n](https://n8n.io) workflow automation templates, covering AI integrations, notifications, and data tracking.

## Workflows

| File | What it does |
|---|---|
| `ai-receptionist/` | Voice AI receptionist backend for Vapi: 9 workflows for booking, CRM and call logs (see below) |
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

## AI Receptionist (Vapi + n8n)

The backend for a voice AI receptionist, set up for a car-detailing business. A Vapi voice agent answers calls and uses n8n as its tools through an MCP server. It can book, look up, move and cancel appointments in Google Calendar, and keep client records and an appointment log in Google Sheets. When a call ends, a webhook writes a report to the call log.

Adapted from [Nate Herk](https://www.youtube.com/@nateherk)'s AI receptionist tutorial.

| File | Role |
|---|---|
| `vapi-mcp-server.json` | MCP server that exposes the 7 tool workflows below to Vapi |
| `check-availability.json` | Checks free slots in Google Calendar |
| `book-event.json` | Creates the calendar event and logs it to Sheets |
| `lookup-appointment.json` | Finds a caller's existing appointment |
| `update-appointment.json` | Moves an appointment and updates the log |
| `delete-appointment.json` | Cancels an appointment and updates the log |
| `client-lookup.json` | Finds a returning client in the CRM sheet |
| `new-client-crm.json` | Adds a new client to the CRM sheet |
| `hercules-receptionist-eoc-report.json` | End-of-call webhook that writes the call report to Sheets |

**Setup**

1. Import all 9 files. In `vapi-mcp-server.json`, point each tool node at your imported copy of that sub-workflow, because workflow IDs change on import.
2. Create a Google Sheet with tabs `Clients`, `Appointment Log` and `Call Log`, then replace `YOUR_GOOGLE_SHEET_ID`.
3. Replace `YOUR_CALENDAR_EMAIL@gmail.com` with the Google Calendar to book into.
4. Attach your Google Calendar and Google Sheets credentials.
5. In Vapi, add the MCP server URL as a tool, and set the end-of-call webhook to the EOC report URL.

## Usage

1. Import any `.json` file into your n8n instance.
2. Configure the required credentials and API keys.
3. Activate the workflow.

## Requirements

- n8n (self-hosted or cloud)
- API keys/credentials specific to each workflow
