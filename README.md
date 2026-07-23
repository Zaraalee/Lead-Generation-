# AI Lead Generation Pipeline

An end-to-end n8n workflow that turns a CSV of prospect companies into personalized outreach at scale — scraping each prospect's site, regenerating a tailored prototype page with OpenAI, deploying it live on GitHub Pages, and closing the loop with a Supabase-backed RAG chatbot embedded right on the page.

Built as a real production workflow at Matrix AE, not a proof of concept.

![n8n](https://img.shields.io/badge/-n8n-EA4B71?style=flat-square&logo=n8n&logoColor=white)
![LangChain](https://img.shields.io/badge/-LangChain-1C3C3C?style=flat-square)
![Supabase](https://img.shields.io/badge/-Supabase-3ECF8E?style=flat-square&logo=supabase&logoColor=white)
![OpenAI](https://img.shields.io/badge/-OpenAI-412991?style=flat-square&logo=openai&logoColor=white)
![GitHub API](https://img.shields.io/badge/-GitHub%20API-181717?style=flat-square&logo=github&logoColor=white)

---

## What it does

1. **Upload a CSV** of prospect companies (name, website, contact email)
2. **Scrape** each prospect's site (Firecrawl) and pull their existing content, branding, and offering
3. **Analyze + regenerate** — OpenAI reads the scraped content and generates a personalized prototype site tailored to that specific business
4. **Deploy** — a new GitHub repo is created per prospect and the generated site is pushed live via GitHub Pages
5. **Outreach** — a personalized email goes out (Gmail/SMTP) pointing the prospect to their own live demo
6. **RAG chatbot** — scraped content is embedded into a Supabase vector store in parallel, powering a chatbot widget embedded in the generated site so visitors can ask questions about the business in real time

Batch progress is tracked across all 5 stages in a Supabase `batch_progress` table, so you can monitor exactly where each prospect is in the pipeline.

## Architecture

```
CSV upload
   │
   ▼
Scrape (Firecrawl) ──────────────► Supabase vector store
   │                                   │
   ▼                                   ▼
OpenAI: analyze + generate        Embeddings (per companyId)
   │                                   │
   ▼                                   ▼
GitHub: create repo + push        RAG chatbot webhook
   │                                   │
   ▼                                   ▼
Deploy to GitHub Pages ◄────── Chatbot widget embedded in generated site
   │
   ▼
Send outreach email
```

The workflow is split into two halves — **Lead Gen Intake** (upload → scrape → analysis) and **Lead Gen Processing** (generation → deploy → outreach) — connected via a fire-and-forget sub-workflow handoff, with binary CSV data passed as base64 to avoid data loss across the split.

## Stack

| Layer | Tool |
|---|---|
| Orchestration | n8n |
| Scraping | Firecrawl |
| Site generation | OpenAI |
| Vector store | Supabase (pgvector) |
| RAG chatbot | LangChain + Supabase Vector Store node |
| Deployment | GitHub API + GitHub Pages |
| Outreach | Gmail / SMTP |
| Progress tracking | Supabase (`batch_progress` table) |

## Setup

1. Import `Lead_Generation_with_chatbot_wihtout_UI.json` into your n8n instance
2. Set the following credentials in n8n:
   - Firecrawl API key
   - OpenAI API key
   - GitHub personal access token (repo + pages scope)
   - Supabase project URL + service key
   - Gmail/SMTP credentials for outreach
3. Create the Supabase tables:
   - A vector store table (matching the `vectorStoreSupabase` node config), tagged by `companyId`
   - A `batch_progress` table with columns for `company_id`, `stage`, `status`, `updated_at`
4. Upload a CSV with columns: `company_name`, `website_url`, `contact_email`
5. Trigger the workflow via the form trigger node, or run it manually against a test CSV first

## Notes on production issues solved

- **CORS**: headers required on both the Webhook trigger node and the Respond to Webhook node for the chatbot widget to work when embedded on external generated sites
- **Binary data loss**: fire-and-forget sub-workflow handoffs drop binary payloads — solved by base64-encoding the CSV before handoff
- **Local testing**: an ngrok tunnel exposes the local n8n instance during widget development, though free-tier ngrok URLs rotate and will break a previously deployed chatbot endpoint — swap for a stable tunnel or hosted n8n instance before going to production

## Status

Module 1 (scrape → generate → deploy → outreach) is fully working. Module 2 (RAG chatbot) is in active development, with the current focus on extending it to a voice-based chatbot for the hospital/clinic vertical.
