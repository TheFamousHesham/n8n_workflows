# n8n Workflow Templates

A collection of production-grade n8n workflows I've built and use in my own automation stack. Sharing them here in case any of the patterns are useful to someone else.

These aren't toy examples. Most of them are pieces of actual pipelines I run for content production, research, or business automation — sanitized for public release.

---

## What's in here

The workflows here generally fall into a few categories:

- **AI agent pipelines** — multi-step LLM chains using OpenAI, Anthropic, Mistral, and others, often with Brave Search or Jina Reader as research tools and Qdrant for retrieval
- **Content production** — YouTube discovery, transcript extraction, virality scoring, blog generation
- **E-commerce / research** — Amazon product research, listing analysis, market intelligence
- **Data manipulation** — Google Sheets pipelines, scheduled aggregations, classification jobs
- **Infrastructure glue** — webhook chaining, API integrations, scheduled triggers

Each workflow lives in its own folder with its sanitized JSON and a short README explaining what it does, what it depends on, and what you need to wire up before it'll run.

---

## Using a template

1. **Download the JSON** for the workflow you want
2. **Import it** into your n8n instance: Workflows → Import from File
3. **Set up credentials** — every credential ID has been wiped, so n8n will flag every node that needs one. Click each flagged node and either select an existing credential or create a new one
4. **Replace the placeholders** (see Sanitization conventions below)
5. **Test in isolation** — these workflows often chain together via webhooks, so test each one on its own before activating the chain

---

## Sanitization conventions

Every workflow here has been scrubbed of personal data using a consistent set of placeholders. When you see one of these, it's a thing you need to fill in:

| Placeholder | What it is | Where to find the real value |
|---|---|---|
| `""` (empty credential `id`) | Credential reference | n8n will prompt you to select/create one |
| `Workflow Constants`, `Google Sheets Account`, etc. | Generic credential names | Rename to whatever you want |
| `YOUR_N8N_HOST` | The hostname of your n8n instance | Your own deployment (e.g. `localhost`, `n8n.yourdomain.com`) |
| `REPLACE_WITH_WEBHOOK_PATH_N` | Path of a webhook that another workflow calls | Copy the path from the corresponding `Webhook` node after import |
| `your-ghost-instance.example.com` | Ghost CMS / publisher endpoint | Your own publisher URL |
| `YourBlogCollection` | Qdrant collection name | Pick something meaningful for your project |
| Empty `meta.instanceId` | n8n instance ID | n8n auto-fills on import |

Random UUIDs in `webhookId` and node `id` fields are kept as-is. They're not sensitive — n8n regenerates `webhookId` on import anyway, and node IDs are just internal references.

---

## Repository layout

```
.
├── README.md                          # this file
├── workflow-name-1/
│   ├── README.md                      # what it does, dependencies, setup
│   └── workflow.json                  # importable n8n JSON
├── workflow-name-2/
│   ├── README.md
│   └── workflow.json
└── ...
```

Each workflow folder is self-contained. If a workflow depends on another (via webhook chaining), the dependency is called out in that workflow's README.

---

## Requirements

These workflows assume:

- **n8n** v1.50+ (most use newer node versions; older n8n may need adjustments)
- **Node.js 20+** if you're running n8n locally
- **Credentials** for whatever services the specific workflow touches — typically some combination of OpenAI, Anthropic, Mistral, Google Sheets, Brave Search, SerpAPI, Cohere, Jina AI, Qdrant, RapidAPI

Workflows that need self-hosted infrastructure (Qdrant, Redis, Postgres, etc.) note this in their individual READMEs.

---

## A note on cost

Some of these workflows hit paid APIs aggressively — multi-agent pipelines with web search tools can spend $1-5 per run on tokens and API calls. Read the individual READMEs and check the model selections before activating anything on a schedule. I generally use Claude Sonnet for the heavy lifting and Mistral Large for cheaper structured-output tasks; swap in whatever fits your budget.

---

## Contributing

These are my workflows, shared as-is. I'm not actively maintaining them as a community project, but:

- **Issues** — if a template is broken or has a bug, open an issue
- **Improvements** — PRs are welcome but I may not always merge them; feel free to fork
- **Questions** — open a discussion rather than an issue for "how do I adapt this for X"

---

## License

MIT. Use them, modify them, ship them in your own products. No warranty — if a workflow racks up an API bill or breaks your pipeline, that's on you.

See `LICENSE` for the full text.

---

## Disclaimer

These workflows interact with paid APIs, external services, and (in some cases) write to live systems. Always test in isolation with throttled inputs before letting anything run on a schedule against production data.
