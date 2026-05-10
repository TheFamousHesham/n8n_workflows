# Financial Blog Automation

An end-to-end pipeline that turns trending investing keywords into published, SEO-optimized blog articles for a Ghost CMS instance — including ETF relevance scoring, YouTube video discovery, transcript-based research, AI-written drafts, image sourcing, and final publishing.

This is a single workflow JSON containing **six chained sub-flows**, wired together via internal webhooks. Each flow can be triggered independently, but they're designed to run in sequence.

---

## Flow overview

| # | Flow | What it does |
|---|------|---|
| 1 | **Keyword-to-ETF Relevance Scoring** | For each tracked keyword, an AI agent (Claude Sonnet, with Mistral and GPT-5 fallbacks) scores how relevant the topic is to each of 8 ETF/fund profiles. Uses Brave Search for live market context. Writes per-fund verdicts back to Google Sheets. *Only run this when keywords or ETF profiles are updated — it's expensive.* |
| 2 | **YouTube Video Discovery** | Picks a random "fair game" keyword from the master sheet, searches YouTube via RapidAPI, fetches related videos, captures channel metadata. Stores candidate videos with status `MASTER` or `READY`. |
| 4 | **Virality Score & Transcriber** | Scores each candidate video using a custom virality formula (view velocity × engagement ratio × subscriber outperformance). Transcribes the videos that clear the threshold using a RapidAPI transcript service. |
| 5 | **Best From The Batch** | An AI editor (Mistral Large) picks the single most article-worthy video from the batch, with structured reasoning for each candidate. The winner is marked `SELECTED`; losers get `REJECTED` with a reason. |
| 6 | **Source Discovery + Blog Generation** | A research agent (Mistral with Claude fallback) finds 5-8 high-quality supplementary sources, fetches them with Jina Reader, chunks and embeds them with Cohere into a Qdrant collection. A blog writer agent (Claude Sonnet) then composes the article using the transcript and the Qdrant knowledge base as a tool, outputting structured JSON blocks. |
| 7 | **Image Sourcing + Publishing** | Fans out each `PLACEHOLDER` image in the article, runs Brave image search per alt-text, has GPT-4o pick the best candidate per slot, patches the blocks, then POSTs the final post to a [Ghost Publisher API](https://github.com/TheFamousHesham/ghost-blocks) endpoint. |

There's no Flow 3 — that number was reserved for a step that ended up not being needed.

---

## Why it's structured this way

Each sub-flow is gated by a status column in Google Sheets (`FAIR GAME`, `MASTER`, `READY`, `VIRAL`, `WAITING`, `SELECTED`, `COMPLETED`, etc.). This makes the pipeline:

- **Resumable** — if step 5 crashes, you can re-run from step 5 without redoing the previous work
- **Observable** — at any point you can look at the sheet and see exactly which keywords/videos are in which state
- **Retriable** — failed items stay in their status until reprocessed

The flows hand off via internal webhooks (`Call Webhook 1`, `Call Webhook 2`, etc.) rather than being one giant pipeline, which makes each piece independently testable.

---

## Dependencies

### Services / APIs

| Service | Used for | Cost |
|---|---|---|
| **OpenAI** | GPT-5 Mini (output parser fallback), GPT-4o (image picker) | ~$0.10–0.50 / run |
| **Anthropic** | Claude Sonnet 4.6 (relevance scoring, blog writing — primary LLM) | ~$1.00–3.00 / run |
| **Mistral Cloud** | Mistral Large (cheap structured-output, source picker, blog writer fallback) | ~$0.05–0.20 / run |
| **Google Sheets** | State tracking across all flows | Free |
| **Brave Search** | Web search (relevance scoring, source discovery) and image search (Flow 7) | Free tier or paid |
| **RapidAPI: youtube-v2** | YouTube search and channel metadata | Free tier |
| **RapidAPI: youtube-multi-api** | Video transcription (DeepSeek-backed) | Paid |
| **SerpAPI** | YouTube video details + related videos | Paid |
| **Cohere** | `embed-english-v3.0` for source embeddings | Free tier sufficient |
| **Jina AI Reader** | Fetching and parsing source URLs into clean text | Free tier sufficient |
| **Redis** | Chat memory for the relevance scoring agent | Self-hosted |
| **Qdrant** | Vector store for the per-article research knowledge base | Self-hosted |
| **Ghost Publisher API** | Final publishing endpoint — see [TheFamousHesham/ghost-blocks](https://github.com/TheFamousHesham/ghost-blocks) | Self-hosted |

Every credential is stripped on import — n8n will flag each one when you open the workflow.

### Google Sheets schema

The workflow expects a Google Sheet with **four sheets** (configured via the `Global Constants` credential):

- **GoogleSheet1** — Keywords master list. Columns: `ID`, `STATUS`, `KEYWORD`, `DATE ADDED`, `DATE TODAY`, `LAST ACCESSED`, `DAYS SINCE LAST ACCESS`, `Keyword Summary`, plus 8 `{TICKER}_relevance` columns (one per fund).
- **GoogleSheet2** — ETF/fund bios. Columns: `ETF`, `BIO`, `STATUS`.
- **GoogleSheet3** — Video candidates. Columns: `STATUS`, `TYPE`, `NOTE`, `PUBDATE`, `KW`, `VIDEOID`, `VIDEO TITLE`, `VIDEO VIEWS`, `VIDEO LIKES`, `CHANNEL NAME`, `SUBSCRIBER COUNT`, `CHANNEL LINK`, `TRANSCRIPT`, `VERDICT REASONING`, `WINNER REASONING`.
- **Blog Outputs** (sheet 4) — Generated articles. Columns: `STATUS`, `DATE`, `VIDEO_ID`, `KEYWORD`, `FEATURE_IMAGE`, `TITLE`, `SUBTITLE`, `META_DESCRIPTION`, `BLOCKS_JSON`, `BLOCK_COUNT`.

### Environment expectations

- A self-hosted Qdrant instance reachable at `http://qdrant:6333` from your n8n container (adjust the URL in the *Create Qdrant Collection* and *Qdrant Vector Store* nodes if yours is elsewhere)
- A self-hosted Redis instance (used by the relevance scoring agent's chat memory)
- A Ghost Publisher API endpoint — replace `your-ghost-instance.example.com` with your own. See [ghost-blocks](https://github.com/TheFamousHesham/ghost-blocks) for the publisher.

---

## Setup

1. **Import** `workflow.json` into your n8n instance (Workflows → Import from File)
2. **Create credentials** for every flagged service. Keep credential names consistent if you want — n8n will let you map any name to the placeholder.
3. **Configure Global Constants** — the workflow uses 8 `Global Constants` nodes to read your Google Sheet ID and tab names. Set up a `Global Constants` credential with at least:
   - `GoogleSheetID` — the spreadsheet's ID
   - `GoogleSheet1` — keywords sheet name
   - `GoogleSheet2` — ETFs sheet name
   - `GoogleSheet3` — videos sheet name
   - `GoogleSheet4` — blog outputs sheet name
4. **Replace placeholders** in the JSON:
   - `your-ghost-instance.example.com` — your Ghost Publisher URL
   - `YourBlogCollection` — the Qdrant collection name (e.g., `TheSkepticInvestor`)
   - `REPLACE_WITH_WEBHOOK_PATH_N` — copy the webhook paths from the corresponding `Webhook` nodes after import
5. **Replace internal webhook URLs** — the four `Call Webhook *` HTTP Request nodes call back into the same n8n instance via `localhost:5678`. If your instance is elsewhere, update the URLs.
6. **Replace the ETF tickers** — the relevance-scoring agent's prompt and output schema name 8 specific funds (MPLY, GOLY, HNDL, ATRFX, ARKK, QQQ, SCHD, BND). Swap these for your own portfolio if you're not building the same publication.
7. **Test each sub-flow in isolation** before activating the schedule trigger or chaining them.

---

## Cost warning

A full end-to-end run from keyword → published post can spend **$2–5** on AI tokens and APIs (the blog writer agent with thinking enabled is the biggest cost). The `Schedule Trigger` is set to fire on its default cadence — review and adjust before activating.

---

## Customizing for your own publication

This is a template. The investing-specific bits — ETF tickers, fund relevance criteria, the "Skeptic Investor" voice — all live in:

- **Flow 1 system prompt** (`AI Agent2` node) — replace the ETF list and the YES/NO calibration examples
- **Flow 1 schema** (`Structured Output Parser2`) — replace the 8 `{TICKER}_relevance` properties with your own taxonomy
- **Flow 6 blog writer prompt** (`Blog Writer Agent` node) — replace the voice/persona description with your own publication's style guide
- **Sheet column names** if you change the schema

The pipeline architecture (status-gated, sheet-tracked, webhook-chained) works for any niche — health, food, sports, etc. Just swap the prompts and the relevance taxonomy.

---

## Known gotchas

- **Limit node before Loop Over Items2** — set to `maxItems: 3` on purpose to throttle YouTube searches. Increase if you have higher API quotas.
- **Wait nodes** — there are 30s waits after some HTTP requests for rate limiting. Adjust based on your API tier.
- **Qdrant collection deleted at start of Flow 6** — the workflow wipes and recreates `TheSkepticInvestor` collection on every run. If you want persistent embeddings across articles, change the collection name to be article-specific or comment out the delete step.
- **The transcript length check** in Flow 4 (the IF node) only proceeds if the transcript is longer than 50,000 characters. Short videos are skipped.
