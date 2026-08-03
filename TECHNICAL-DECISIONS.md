# Technical Decisions & Build Journey

The *why* behind the architecture — including the mistakes and tradeoffs, not just the final state.

---

## 1. Three independent workflows, not one

Monitoring and maintenance were briefly merged into the same canvas as the core send pipeline — the wrong call, caught in review and reverted.

Each piece has a different reason to change and a different blast radius if it breaks: a bug in pruning logic is a data-loss risk, a bug in monitoring is a silent-failure risk, neither should share a deploy unit with the revenue-facing pipeline. Splitting them means a change to one never requires re-verifying the others, and each can be paused independently (e.g. freezing sends during a rebrand while monitoring keeps running).

## 2. Diagnosing a silent platform bug, not just a code bug

The built-in Slack node was silently converting every approval-message payload into plain text, dropping the interactive buttons — no error, no warning. Isolated it methodically: confirmed the JSON itself was valid, then sent a fully static payload with zero dynamic content, and it still failed identically. That ruled out the content and pointed at the node's internal request construction.

**Lesson applied:** when a node's own preview is correct but the live result is wrong regardless of input, stop iterating on content and go straight to the underlying API. Replaced it with a direct HTTP call to `chat.postMessage`.

## 3. Making human approval time-independent

Early versions referenced the AI generation step's output directly from later nodes in the same execution. That silently breaks the moment approval happens in a separate execution — which it always does, since a Slack click arrives as its own webhook trigger, potentially days later.

**Fix:** the full generated content is serialized to JSON and persisted in Airtable at generation time. Every downstream step reads from that stored record instead of a live execution reference, so approval genuinely doesn't care how much time has passed.

## 4. Cost-gated, two-tier content verification

Rather than one LLM call to fact-check every draft, verification runs in two tiers: a deterministic, zero-cost pass checks numeric and version claims against source text on every draft; a second LLM-based semantic check only fires when the first tier flags something. Cheap screening on everything, expensive judgment only where it's earned — the same pattern production moderation pipelines use, and the right architecture independent of budget.

The verification layer isn't the real safety net, either way — the human approval step is. Verification exists to make that review faster and more targeted, not to replace it.

## 5. Reject → capped, same-day auto-regeneration

Reject used to be a dead end until the next scheduled run. Now it immediately re-triggers generation — but capped at 2 automatic regenerations per day, since an uncapped retry loop on repeated rejections is a real cost risk, not a hypothetical one. Past the cap, a human is notified instead of the system retrying blindly.

**Bug caught while testing this:** the rejection counter compared `{Status} = 'rejected'` against records actually written as `'Rejected'`. Airtable's string comparison is case-sensitive, so the count silently stayed at zero and the cap never engaged — every rejection would have triggered an unbounded regenerate. It only surfaced under repeated-rejection testing, not a single happy-path run, which is the more general reason multi-attempt failure paths need to be tested more than once.

## 6. Edit flow: Airtable as the editor, polling instead of a webhook

Editing happens directly in Airtable; a checkbox the reviewer ticks (not "did the content change," which would fire mid-keystroke) signals "done," and a 10-minute poll picks it up and reposts for approval.

Airtable's native Automations could push this instantly via webhook — genuinely faster. Deliberately avoided it anyway: this system has been kept as a small number of workflows fully contained and version-controlled in one place. Splitting logic into Airtable's separate automation layer reintroduces exactly the fragmentation an earlier decision (see #1) was meant to eliminate. A 10-minute delay is a small, known tradeoff for keeping one source of truth.

## 7. Closing a silent gap: article dedup had no memory

Deduplication was designed to check new articles against everything previously ranked — but nothing ever wrote ranked articles back into that table. The system ran without erroring for the entire build; the only symptom was ranking the same articles repeatedly, an efficiency loss with no visible failure. Added the write-back step once noticed.

Deliberately left one field (`Source`, the originating feed name) unmapped on principle rather than chasing it: by the point ranked data reaches storage, the feed name has passed through several node hops including one that fully replaces item data, where a cross-node reference risks silently resolving to the *wrong* feed rather than failing visibly. `URL` alone covers both dedup and traceability, so adding a dedicated lineage-preserving step for a label nothing depends on wasn't worth the added fragility.

---

## Smaller decisions worth a line each

- **Groq primary / Gemini fallback for ranking, reversed for generation** — ranking is many small calls (rate-limit bound), generation is one large call (context/token bound); matched each provider's actual strength to the call pattern rather than picking one model for everything.
- **Segments are re-angled, not regenerated** — one fact-checked, approved draft reframed per audience keeps facts consistent across segments and keeps cost linear instead of multiplying per segment.
- **Free-tier model IDs get deprecated without warning** (`gemini-2.5-flash` 404'd mid-build) — treated as a recurring maintenance cost of relying on any external provider's model catalog, not a one-time setup detail.

---

## Known deferred decisions

- **Subject line A/B testing** — three variants are generated per issue specifically to support this; implementation is a small, scoped change to the send request, deferred because it's gated to a paid Brevo tier. Design-ready, not a gap.
- **GDPR data-subject-request handling** — documented as a manual runbook for now, not yet automated.
- **List hygiene** — manual monthly review; worth automating once subscriber volume justifies it.
