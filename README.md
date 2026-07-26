# Governance-Aware-Contract-Analyzer

A contract / policy document analyzer where every LLM call is treated as a governed data event. Uploads a contract, returns clause extraction, risk flags, and a summary — and produces a complete, tamper-evident audit trail of what went into the model, what came out, and what policy checks fired along the way.

The differentiator is not the analysis. It is that the analysis is **auditable by design**.

---

## 1. Purpose & scope

**Purpose.** Help a legal / procurement / compliance reviewer triage a contract faster by surfacing the clauses and risks that matter, while giving their own compliance function full visibility into how AI was used to produce that output.

**In scope (v1).**
- Single-user, single-tenant local app.
- English-language contracts and policy documents, PDF or plain text, ≤ ~50 pages.
- Clause extraction (parties, term, termination, liability, IP, data-processing, governing law), risk flagging against a small fixed ruleset, and an executive summary.
- A governance wrapper around every LLM call: input redaction, policy checks, output classification, immutable audit log, hash-chain integrity.
- A read-only audit-trail viewer.

**Out of scope (v1).** Multi-tenant auth, contract editing / redlining, negotiation suggestions, non-English documents, OCR of scanned PDFs, comparison across contracts, e-signature integration, multi-user review workflows.

**Success criterion.** A reviewer + a compliance officer can each open the app, run one contract through it, and both walk away satisfied — the reviewer with a useful analysis, the compliance officer with a defensible audit record.

## 2. Data categories processed

| Category | Examples in contracts | Sensitivity |
|---|---|---|
| Identifiers | Names, email addresses, signatory titles | Personal data |
| Business identifiers | Company names, registration numbers, addresses | Commercial-sensitive |
| Financial terms | Prices, payment schedules, penalties | Commercial-sensitive |
| Contractual dates | Effective date, term, notice periods | Non-sensitive |
| Free-text clauses | Any clause body | Mixed — may contain any of the above |
| Special category (GDPR Art. 9) | Health, biometric, political — occasionally referenced in DPAs | High; must be flagged, not silently processed |

**Data minimization principle.** The redaction layer's job is to prevent identifiers and special-category signals from leaving the local process unless the analysis genuinely requires them. When redaction happens, both the redacted and the original stay in the audit log; only the redacted version goes to the model.

## 3. Lawful basis (for the demo)

- **Legitimate interest** of the reviewer to analyze a document they lawfully hold.
- No consent flow in v1 because there is no data subject interaction — this is document-at-rest analysis by the document controller.
- The design doc records this assumption explicitly so that any future multi-user version has to revisit lawful basis rather than inherit it silently.

## 4. Sub-processors & data flow

```
[User] → [Angular UI] → [FastAPI backend]
                            │
                            ├─→ [Local PDF parser]  (no network)
                            ├─→ [Presidio PII detector]  (no network)
                            ├─→ [Policy engine — rule-based]  (no network)
                            ├─→ [Anthropic API]  ← only network egress
                            └─→ [Postgres audit log]  (local)
```

**Sub-processors.**
- **Anthropic** — the only third party that sees document content (in redacted form). Model: `claude-sonnet-5` for analysis, `claude-haiku-4-5-20251001` for the output-risk judge in a later phase. No training on API data per Anthropic's data-usage terms.
- **Postgres host** — local for dev; a specific managed host is a Phase-4 decision, not v1.

**Egress rule.** Exactly one outbound network call per LLM invocation, and only via the `GovernedLLMClient` wrapper. Any code path that reaches `anthropic.Anthropic()` directly is a bug.

## 5. Risks & mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | PII leaks to the model in prompt | High | High | Presidio pre-scan; redact-then-send; original stays local |
| R2 | Model output leaks back PII originally present in the document | Medium | High | Output classifier / regex sweep before returning to UI; flagged outputs quarantined for review |
| R3 | Model hallucinates a clause or risk that is not in the document | Medium | Medium | Prompt requires citations (page + span); UI shows the source span; unciteable claims dropped |
| R4 | Prompt injection via document contents (contract says "ignore instructions and…") | Medium | Medium | Documents are passed as data, not as instructions; system prompt states the document is untrusted input |
| R5 | Audit log tampered with post-hoc | Low | High (breaks the whole value prop) | Append-only table + per-row hash chain; integrity check on read |
| R6 | Cost / token blow-up on a large document | Medium | Low | Pre-flight token count with a hard cap; user sees cost estimate before the call is made |
| R7 | Special-category data (Art. 9) processed without acknowledgement | Low | High | Presidio flags health/biometric/etc.; app refuses to proceed until the user explicitly acknowledges |
| R8 | User treats the analysis as legal advice | Medium | Medium (out-of-domain) | Persistent disclaimer in UI and in every exported report |

The audit log **must** be able to answer, for any past analysis: what went in, what was redacted, what policies fired, what the model was asked, what it returned, what was flagged, what the user saw. If any of those cannot be reconstructed, the audit log has failed.

## 6. Retention

- **Uploaded documents.** Kept for 30 days locally, then hard-deleted. User can delete on demand.
- **Audit records.** Kept for 12 months (aligns with typical internal audit cycles). Hash-chain preserved across deletions of the underlying document — the audit row survives even when the document is gone, but content fields are nulled and a `document_deleted_at` timestamp is set.
- **Model outputs.** Same retention as the audit row that contains them.
- **No retention on Anthropic's side** beyond what their API terms specify; we do not enable any opt-in data-sharing.

## 7. What "auditable by design" means concretely

Every LLM call produces exactly one audit record with, at minimum:

- `id`, `created_at`, `prev_hash`, `row_hash` (SHA-256 of the canonical serialization + `prev_hash`)
- `document_id`, `stage` (`clause_extraction` | `risk_flag` | `summary` | `output_judge`)
- `model_id`, `model_params`
- `input_raw_len`, `input_redacted`, `redactions[]` (type, span, replacement token)
- `policy_checks[]` (rule id, passed / failed, detail)
- `output_raw`, `output_flags[]`
- `tokens_in`, `tokens_out`, `cost_usd`, `latency_ms`
- `status` (`ok` | `blocked_by_policy` | `error`), `error_detail`

If any field cannot be populated, the call does not happen.

## 8. Out of scope for v1 — explicit list

Recorded here so scope creep is a conscious choice, not a drift:

- Authentication / multi-user
- Role-based access to audit records
- Contract redlining or edit suggestions
- Multi-language support
- OCR for scanned PDFs
- Comparison across two or more contracts
- Fine-tuned or self-hosted models
- Export to a formal audit-package PDF (planned for Phase 4)
- Framework mapping UI (EU AI Act / GDPR article tagging) (planned for Phase 4)

---

This document is the closest thing this project has to a DPIA. Every later architectural decision should be checkable against it. If a change makes a section here false, update the section — do not let the code and the doc drift.
