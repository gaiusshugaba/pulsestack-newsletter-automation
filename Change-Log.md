# PulseStack Newsletter Automation — Change Log

Format: newest entries at top. One entry per meaningful change.

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
