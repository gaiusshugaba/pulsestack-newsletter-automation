# PulseStack Newsletter Automation — Change Log

Format: newest entries at top. One entry per meaningful change.

---

## 2026-07-29 — Production Readiness pass: all five roadmap areas
**What:**
- Compliance: CAN-SPAM footer (physical address + privacy link) added to both HTML templates; unsubscribe processing verified against real Brevo account behavior; Subscribers table added to both bases for consent record-keeping.
- Reliability: new `Newsletter Health Check` workflow — daily dead-man's-switch (alerts if no draft found for current week by 9am) and stale-draft check (alerts if a draft sits in Draft status >24h). Alerts post to dedicated `#newsletter-automation-alerts` channel.
- Cost & Scaling: `Run_Costs` table logs articles-ranked and emails-sent per run; usage-anomaly alert flags runs that rank unusually many articles (dedup/Limit failure signal); weekly pruning job removes Article_Pool records older than 90 days to stay under Airtable free-tier record limits.
- Data Quality: two-tier Content Verification Layer — Tier 1 (deterministic, zero-cost) extracts numeric/version claims from generated copy and checks them against source article text; Tier 2 (LLM semantic check) only triggers when Tier 1 flags something, keeping cost gated to the cases that need judgment. Findings surface directly in the Slack approval message. Subject-line A/B testing designed (3 variants generated per issue specifically to support it) but implementation deferred — gated to Brevo's paid tier; documented as a scoped proposition rather than built.
- Testing & Change Management: staging Airtable base + staging Slack channel + duplicated staging n8n workflow; git repo initialized and pushed to a private GitHub remote; sticky-note documentation added to the canvas covering every phase's key decisions, credentials needed, and known failure modes.

**Why:** Move from "working prototype" to a system whose failure modes are caught automatically, whose changes can be tested safely and rolled back, and whose legal/compliance exposure is addressed rather than assumed.

**Known deferred items (intentional, documented, not oversights):**
- Subject line A/B testing — paid-tier Brevo feature, architecture ready.
- List hygiene — manual monthly review, not yet automated.
- Visual content (header images) — cosmetic, deferred.
- GDPR data-subject-request handling — documented as a manual runbook need, not yet built.

**Tested in staging:** Yes — full run through ingestion → generation → verification → Slack approval → segmented Brevo send → Mark as Sent, plus Health Check workflow's three alert conditions, tested individually with forced trigger conditions.
**Rolled out to production:** [update after confirming production nodes match staging]

---

## 2026-07-27 — Testing & Change Management setup
**What:** Created staging Airtable base, staging Slack channel, duplicated n8n workflow for staging use. Initialized git version control for production workflow JSON.
**Why:** No safe way to test changes before this — every fix during initial build was tested directly against production, real Slack channel, and real Airtable data.
**Risk if skipped:** A bad change could corrupt production data or spam the real approval channel with test noise.

---

## [Template for future entries]

## YYYY-MM-DD — Short title
**What:** What changed, concretely.
**Why:** The problem this solves or the goal it serves.
**Risk if skipped:** What could go wrong without this change (helps future-you understand priority when reviewing old entries).
**Tested in staging:** Yes/No
**Rolled out to production:** Yes/No, date
