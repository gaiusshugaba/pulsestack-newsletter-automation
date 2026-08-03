# PulseStack Newsletter Automation

An AI-powered newsletter automation system built for a fictional B2B DevOps/infrastructure SaaS company (PulseStack). It researches industry news, ranks it for relevance, writes a brand-voiced newsletter, re-angles it per audience segment, and routes it through a human approval step before sending — end to end, on a weekly schedule, unattended.

Built with n8n, Google Gemini, Groq, Airtable, Brevo, and Slack.

---

## The problem this solves

Small SaaS marketing/growth teams typically spend 4–6 hours a week researching industry news, drafting a newsletter, and manually segmenting content for different audiences (founders, developers, marketing leads). The process is repetitive, inconsistent, and easy to let slip during a busy week.

## What this system does

Every Monday at 6am:
1. Pulls new articles from 6 curated RSS feeds, filters out anything already seen — including articles the pipeline itself ranked in prior runs, which get written back to the article pool after ranking
2. Ranks the new articles for relevance using an LLM, keeps the top 8
3. Writes a newsletter grounded in those articles and the company's brand voice, enforced into a strict JSON schema
4. Runs the draft through a two-tier content verification check (deterministic claim-checking, escalating to an LLM review only when something looks unverified)
5. Posts the draft to Slack with Approve / Edit / Reject buttons
6. On approval: re-angles the same newsletter for each audience segment (Founders, Developers, Marketing) without re-researching, sends a separate campaign to each segment's list via Brevo, and archives everything in Airtable

Two additional response paths, independent of the weekly schedule:
- **Reject** — marks the draft rejected and immediately triggers a fresh generation cycle the same day, capped at 2 automatic regenerations per day to prevent an unbounded retry loop; beyond the cap, a human is notified to intervene manually
- **Edit** — the reviewer edits the draft's content directly in Airtable and ticks a "ready for re-review" checkbox; a polling check picks it up within 10 minutes and re-posts a fresh approval message with the edited content

Two supporting workflows run independently:
- **Health Check** — daily, verifies a draft was actually generated this week and flags anything stuck awaiting approval for 24+ hours
- **Pruning** — weekly, removes article records older than 90 days to stay under Airtable's free-tier record limits

## Architecture

```
workflows/
├── newsletter-automation.json   ← main pipeline (ingestion → generation → approval → send)
├── health-check.json             ← independent monitoring workflow
└── pruning.json                  ← independent maintenance workflow
```

Three separate workflows, not one — each has a different change frequency and failure impact, and keeping them independent means a change to monitoring logic can never accidentally affect the core send pipeline. See `TECHNICAL-DECISIONS.md` for the full reasoning.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Orchestration | n8n (self-hosted) | Visual workflow engine, native LangChain node support |
| Ranking / Generation | Gemini 3.6 Flash (primary) + Groq Llama 3.3 70B (fallback) | Free-tier friendly, dual-provider redundancy |
| Data store | Airtable | Config-as-data (feeds, brand voice, segments), draft archive, consent records |
| Email delivery | Brevo | Genuinely usable free tier (300/day, 100K contacts) vs. Mailchimp's gutted free plan |
| Approval interface | Slack (via direct API calls, not the native n8n Slack node — see technical doc) | Human-in-the-loop gate before anything reaches subscribers |

Documents in this repo
TECHNICAL-DECISIONS.md — architecture reasoning, the debugging journey, and honest tradeoffs
CHANGE-LOG.md — running history of what changed and why
