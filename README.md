# NetGuardian — Technical approach & weekly deliverables (GSoC 2026)

> **Note for maintainers:** This is the **technical part only**. I'll add the SAT/personal sections (bio, coding skills, time commitment, etc.) once you're happy with this approach.

---

## 1. Introduction

This project extends work I have already contributed to OWASP BLT, including **PR #5057** (CVE search, filtering, caching, autocomplete, and CVE-aware indexing on the Issue model), merged in the main BLT repo. NetGuardian is not a from-scratch implementation: it builds on that CVE layer and on BLT's existing Django + DRF + templates/HTMX stack to deliver a zero-trust encrypted ingestion path for security findings, a small detection pack, CVE-aware triage, and verified events for downstream systems (Rewards, RepoTrust, University).

---

## 2. Architecture overview

```mermaid
flowchart TB
    subgraph External["External inputs"]
        Agent["NetGuardian Agent\n(Semgrep + HTTP checks)"]
        MgmtCmd["Management command\n(periodic scan)"]
    end

    subgraph BLT["BLT server (Django/DRF)"]
        Ingest["Ingestion API\n(signature + replay check)"]
        DB[( "PostgreSQL\nFinding, Envelope,\nEvidenceBlob, events_ng" )]
        CVE["CVE layer (PR #5057)\nnormalize_cve_id,\nget_cached_cve_score"]
        Triage["Triage UI\n(list, detail, filters)"]
        Convert["Convert to Issue"]
        Reports["CSV / PDF reports"]
        Webhook["Verified-events webhook\n(HMAC-signed)"]
    end

    subgraph Downstream["Downstream (out of GSoC scope)"]
        Rewards["BLT-Rewards (B)"]
        RepoTrust["RepoTrust (X)"]
        University["BLT University (C)"]
    end

    Agent -->|"ztr-finding-1 envelope"| Ingest
    MgmtCmd -->|"Finding rows"| DB
    Ingest --> DB
    DB --> CVE
    CVE --> Triage
    Triage --> Convert
    Convert --> DB
    DB --> Webhook
    Triage --> Reports
    Webhook --> Rewards
    Webhook --> RepoTrust
    Webhook -.->|"future"| University
```

**Data flow in short:** Scanner/agent or management command produces signed findings → ingestion API verifies and stores → CVE layer (existing) enriches → triage UI with server-side decrypt and “Convert to Issue” → verified events emitted via webhook for Rewards/RepoTrust (and later University). No new queue; periodic management commands and existing throttling only.

---

## 3. Stack & scope (CodeRabbit-aligned)

- **Stack:** Django 5.x + DRF, PostgreSQL, existing URL routing and auth. **No new queue:** periodic management commands and existing throttling middleware only; no Celery.
- **Evidence:** Server-side decrypt with audited access (permission checks + access logging). Client-side crypto is post–v1.
- **Reports:** CSV and PDF both in scope (PDF e.g. via WeasyPrint, timeboxed in Week 12).
- **Detection pack:** 3–5 Semgrep rules + 2 HTTP checks, with acceptance gates and curated fixtures; regression test in CI.
- **Security-critical code:** Hand-written from a written spec and covered by tests; LLMs only for scaffolding and docs.

---

## 4. Community bonding (pre–Week 1)

**Weekly deliverables:**
- A one-pager for `ztr-finding-1` that nails down the fields, signature algorithm, time window, nonce semantics, and how keys get distributed.
- An adoption/readiness checklist covering the target install modes and where data boundaries sit.
- Mentor sign-off on the v1 envelope, the minimal detection pack, and a clear "verified event" definition that downstream consumers can actually build against.

**Day 1–2:** Re-review BLT codebase and PR #5057 CVE paths; sync with other GSoC project owners (Preflight, Rewards, University, MCP) on shared interfaces and timeline touchpoints.  
**Day 3–4:** Draft and iterate `ztr-finding-1` one-pager with mentors; draft adoption checklist (install modes, data boundaries, retention).  
**Day 5:** Finalize both deliverables; confirm env, fixtures, and AI-tool rules; document decisions in a short design note.

---

## 5. Week-by-week plan (granular deliverables + day-level)

### Week 1 — Envelope & schema

**Weekly deliverables:**
- A written `ztr-finding-1` spec that covers fields, signatures, timestamps, and nonce — enough that someone else could implement against it.
- Django models for `Finding`, `Envelope`, and `EvidenceBlob` with migrations applied and everything registered in admin.
- A key registry model (per-org/per-sender) and a defined pattern for `SignedNonce` — either a DB table or a cache-key approach.
- Documented pagination defaults and a DB index strategy for findings, so we're not making it up later.
- Serializer stubs for `Finding`/`Envelope` and unit tests covering model constraints and validation.

**Day 1–2:** Finalize and document `ztr-finding-1` envelope format. Implement `Finding`, `Envelope`, `EvidenceBlob` models (encrypted bytes, digest, media_type, size); add key registry and SignedNonce (or cache pattern); write migrations.  
**Day 3–4:** Apply migrations; register models in admin. Define pagination defaults and DB index strategy for findings. Add serializer stubs for Finding and Envelope.  
**Day 5:** Unit tests for model constraints, validation, and uniqueness; review and fix; document schema in proposal/ADR.

---

### Week 2 — Ingestion & Zero-Trust plumbing

**Weekly deliverables:**
- A working ingestion API that accepts encrypted findings and verifies signatures and timestamps server-side.
- Solid replay protection: `Envelope` unique on `(sender_id, nonce)`, clock skew capped at ±5 min, `received_at`/`validated_at` stored, and anything expired or replayed gets rejected cleanly.
- `TokenAuthentication` with per-org scoping, body size caps, and rate limits wired into the existing throttling middleware.
- Property tests for the signature window, log redaction, and idempotency — plus one E2E test that proves a valid envelope actually ends up as a stored Finding.

**Day 1–2:** Implement ingestion view(s) and request parsing. Add server-side signature and timestamp verification (human-written from spec). Implement DB-backed nonce/replay: unique (sender_id, nonce), clock skew check, store received_at/validated_at.  
**Day 3–4:** Wire TokenAuthentication and per-org scoping; add body size caps; plug into existing throttling middleware. Add audit log stubs for accept/reject.  
**Day 5:** Property tests (valid/expired/future/replayed envelope, signature mismatch, canonicalization); log redaction and idempotency tests; E2E test: encrypted envelope → stored Finding.

---

### Week 3 — Detection MVP

**Weekly deliverables:**
- 3–5 Semgrep rules for Python/JS with fixture code that actually triggers them, plus 2 HTTP checks (security headers and a reflected-XSS canary in a fixture app).
- A rule metadata format that captures ID, category, CWE, default severity, and optional CVE association — consistent across all rules.
- A management command that runs the detectors with timeouts and per-target concurrency caps and feeds the output into a staging table or directly into Finding ingestion.
- At least one fixture per rule category with an expected finding, and tests covering 1–2 rules in each category.

**Day 1–2:** Define rule metadata schema and 3–5 Semgrep rules; add fixture code/snippets that trigger expected findings. Implement 2 HTTP checks (headers + XSS canary) against a small test fixture app.  
**Day 3–4:** Implement management command: invoke Semgrep + HTTP checks with timeouts and concurrency caps; parse output into Finding rows (staging or direct). Wire rule ID/category/severity from metadata.  
**Day 5:** Add fixtures and tests for at least 1–2 rules per category; document rule list and acceptance criteria for Week 9.

---

### Week 4 — Triage-lite UI

**Weekly deliverables:**
- A Finding list view with severity/rule/target filters, pagination, and sort — something you can actually use to triage.
- A detail view that decrypts evidence server-side, gates it behind explicit permissions, logs every access, and renders it in redacted form.
- A "Convert to Issue" flow sketched out and wired to the BLT Issue model (CVE columns come later).
- Org-scoped permissions enforced throughout, with basic template or browser tests for list and detail.

**Day 1–2:** Implement Finding list view (DRF or template): filters severity/rule/target, pagination, sort. Implement org-scoped permission checks; ensure only org members see org findings.  
**Day 3–4:** Implement detail view; server-side decrypt with permission check and access logging; render redacted evidence. Sketch "Convert to Issue" (button + view wiring to Issue model).  
**Day 5:** Basic template or browser tests for list/detail and filters; permission leakage test (Org A user cannot see Org B findings); document UX for midterm demo.

---

### Week 5 — CVE Intelligence plumbing

**Weekly deliverables:**
- Findings (or linked Issues) populated with `cve_id` and `cve_score`, using `normalize_cve_id` and `get_cached_cve_score` from `website/cache/cve_cache.py` — no reinventing the wheel.
- CVE columns surfaced in both the triage list and detail views.
- Mapping tests only; cache-miss fallback is already covered by the existing suite.

**Day 1–2:** Add nullable `cve_id`/`cve_score` on Finding (or link path to Issue). Implement mapping layer: rule → CVE lookup; call `normalize_cve_id` and `get_cached_cve_score`; populate on ingest or on demand.  
**Day 3–4:** Add CVE columns to list and detail templates/API; ensure existing CVE cache and indexes are used.  
**Day 5:** Unit tests for CVE mapping and cache-miss fallback; no new NVD/cache logic — reuse PR #5057 paths only.

---

### Week 6 — Validation & dedup

**Weekly deliverables:**
- A fingerprint defined as `(rule_id, target_url, optional selector, evidence_digest)` with a unique DB index backing it.
- Idempotent submission that upserts by fingerprint — new evidence attaches to an existing Finding rather than creating a duplicate.
- A confidence score field (Decimal [0, 1]) and optional FP checklist fields on Finding.
- Tests that cover duplicate collapse, evidence attachment rollup, and concurrent submission races.

**Day 1–2:** Implement fingerprint computation and unique index on (rule_id, target_url, selector, evidence_digest). Implement upsert logic: same fingerprint updates/attaches evidence; new fingerprint creates new Finding.  
**Day 3–4:** Add confidence score (Decimal, bounded) and FP checklist fields; migration; wire into ingestion and triage.  
**Day 5:** Tests: same fingerprint collapses; evidence variation creates new attachment; concurrent posts collapse correctly; document dedup semantics.

---

### Week 7 — CVE-aware triage UX

**Weekly deliverables:**
- Finding list filters for `cve_id`, `cve_score_min`, and `cve_score_max`, mirroring what already exists on the Issue side.
- A "Related CVEs" side panel rendered server-side from the existing CVE index — no new infrastructure.
- CVE autocomplete reused in the "Convert to Issue" flow, so people aren't typing CVE IDs by hand.
- UI tests confirming filters interact correctly and the related CVE list and autocomplete behave as expected.

**Day 1–2:** Add filter backend/query params for Finding list: cve_id, cve_score_min, cve_score_max (reuse patterns from Issue). Expose in API and wire to list template.  
**Day 3–4:** Implement "Related CVEs" side panel (server-rendered, existing CVE index). Integrate CVE autocomplete into "Convert to Issue" form.  
**Day 5:** UI/tests for filter combinations and related CVE panel; verify no permission leakage; update triage docs.

---

### Week 8 — Triage polish & RFI

**Weekly deliverables:**
- A noticeably better evidence viewer — improved layout, syntax highlighting or snippet context, something that makes reviewing findings less painful.
- Canned RFI templates as markdown/callout partials, ready to drop into the detail view without touching any email plumbing.
- Tests confirming templates don't leak secrets and render safely.
- **Midterm checkpoint:** A full E2E demo — signed ingestion → Finding → triage list → server-side decrypt → "Convert to Issue" with CVE autopopulated.

**Day 1–2:** Improve evidence viewer (layout, syntax highlighting or snippet context). Add markdown/callout partial for "Request more info" and 2–3 canned RFI templates (stored safely, no secrets).  
**Day 3–4:** Wire RFI templates into detail view; ensure all rendering is safe (escape, no raw HTML from findings).  
**Day 5:** Tests for template safety and non-leakage; run full E2E for midterm: signed envelope → Finding → triage list with filters → decrypt/view evidence → "Convert to Issue" with CVE autopopulated; record demo.

---

### Week 9 — Accuracy sampling & tuning

**Weekly deliverables:**
- A curated mini-suite of 5–8 fixtures (code or URLs) with known expected outcomes — the ground truth we'll measure against.
- A management command that runs the detectors against the suite and persists per-rule metrics (TP/FP/FN or precision/recall).
- Documented acceptance gates (e.g. precision ≥ X) and a regression test wired to CI so regressions don't sneak in.

**Day 1–2:** Finalize 5–8 curated fixtures and expected results per rule. Implement metrics persistence (per rule: counts or precision/recall).  
**Day 3–4:** Management command: run detectors on suite, compute and store metrics; add optional threshold check. Define acceptance gates (e.g. precision ≥ 0.8) in config or docs.  
**Day 5:** Wire regression test (run suite, assert gates); document rule list and gates for reviewers; tune 1–2 rules if needed.

---

### Week 10 — Consensus & resilience

**Weekly deliverables:**
- A reconfirmation gate for critical-severity findings — a second heuristic or rule has to agree before "Convert to Issue" goes through.
- Confidence scoring updated to factor in whether reconfirmation happened.
- Per-org/hour quotas via DB counters, hooked into the existing throttling and `IPRestrictMiddleware` — batch ingestion stays in DB transactions, no new queue.
- Tests for the reconfirmation gate and for 429/`Retry-After` behavior under load.

**Day 1–2:** Implement reconfirmation gate for critical findings (e.g. second rule or heuristic must agree). Update confidence scoring to reflect reconfirmation.  
**Day 3–4:** Add per-org/hour quota (DB counter); integrate with existing throttling and IPRestrictMiddleware. Ensure batch ingestion uses DB transactions only; document "no new queue" decision.  
**Day 5:** Tests: reconfirmation enforced; rate limit and 429/Retry-After under load; document resilience behavior.

---

### Week 11 — Remediation & insights

**Weekly deliverables:**
- Markdown remediation fragments for each rule type, with OWASP links — static content, nothing dynamic.
- CVE-based enrichment when `cve_id` is present: links to advisories and OWASP so reviewers have context without leaving the page.
- "Why this matters" callouts and remediation hints surfaced in the triage detail view and report output.
- Tests confirming fragments render safely and map correctly to the right rules and CVEs.

**Day 1–2:** Author markdown remediation fragments for each rule type; add OWASP links. Add optional CVE-based enrichment (advisory links) when cve_id present.  
**Day 3–4:** Surface "why this matters" and remediation in triage detail view and in report payload; ensure safe rendering (no raw user input in fragments).  
**Day 5:** Tests for fragment rendering and rule/CVE mapping; review with mentor for tone and completeness.

---

### Week 12 — Disclosure helpers & reports

**Weekly deliverables:**
- `security.txt` detection (fetch/parse or a stub) integrated into "Convert to Issue" and the report flow, so disclosure contacts are surfaced automatically.
- CSV export for findings with CVE metadata included; snapshot test confirming sensitive evidence doesn't leak in plain text.
- PDF export (e.g. WeasyPrint, timeboxed to 0.5–1 week); snapshot test confirming the same redaction guarantees hold.

**Day 1–2:** Implement security.txt detection (URL discovery, fetch, parse or minimal stub); add hints to "Convert to Issue" and report context. Implement CSV export (findings + CVE columns); ensure no sensitive evidence in plain text.  
**Day 3–4:** Snapshot tests for CSV (columns, redaction). Implement PDF export (WeasyPrint or chosen lib, simple template); snapshot test for PDF redaction and no plaintext secrets.  
**Day 5:** Document report formats and redaction policy; finalize both CSV and PDF as deliverables.

---

### Week 13 — Verified events for downstream

**Weekly deliverables:**
- An `events_ng` outbox table with a versioned payload schema: `cve_id`, `cve_score`, `rule_id`, `severity`, `org_id`/`repo`, `finding_id`/`issue_id`, `created_at`, `dedupe_key`, `version`.
- Webhook delivery signed with HMAC-SHA256 (reusing BLT's existing GitHub/Slack HMAC patterns), emitted on "Convert to Issue" and on resolution, with an idempotency key and exponential backoff.
- A read-only API for event retrieval with pagination and filtering, plus consumption docs written specifically for Rewards (B) and RepoTrust (X).
- Tests covering event emission, idempotency, payload shape, and webhook signature verification.

**Day 1–2:** Design and create `events_ng` outbox model; define payload schema and version. Implement emission on "Convert to Issue" and on issue resolution; dedupe_key and idempotency.  
**Day 3–4:** Implement webhook delivery with HMAC-SHA256 (reuse existing HMAC helpers); retry with exponential backoff. Add read-only API for events (pagination, filters).  
**Day 5:** Tests for emission, idempotency, and payload; document consumption for B and X; no downstream logic in NetGuardian.

---

### Week 14 — Hardening & security review

**Weekly deliverables:**
- A proper security review pass: key handling, nonce uniqueness, evidence redaction in logs and templates, permission checks everywhere, and cache-poisoning resistance.
- Dead code and over-generalized AI-generated code cleaned out; docs updated to match what's actually implemented.
- A checklist or short report summarizing what was reviewed and what was fixed.

**Day 1–2:** Review key storage and usage; nonce uniqueness and replay logic; evidence and PII redaction in logs and templates; permission checks on all Finding/evidence access.  
**Day 3–4:** Review cache usage for poisoning; fix any issues. Remove dead or overly generic code; align docs and comments with current behavior.  
**Day 5:** Document review findings and mitigations; run full test suite; prepare for pilot.

---

### Week 15 — Pilot prep & docs

**Weekly deliverables:**
- A pilot checklist covering configuration steps, runbooks, and a rollback plan — so the first run isn't improvised.
- A migration rollback note and data deletion playbook for evidence blobs.
- User, admin, setup, and contribution docs polished and reviewed end-to-end.

**Day 1–2:** Write pilot checklist (config, first run, validation). Document migration rollback steps and data deletion playbook for evidence blobs.  
**Day 3–4:** Polish user-facing and admin docs; setup and contribution guide; link to ztr-finding-1 and adoption checklist.  
**Day 5:** Review with mentor; fix gaps; confirm pilot orgs and timeline for Week 16.

---

### Week 16 — Pilot run & final polish

**Weekly deliverables:**
- A live pilot with 1–2 orgs, with real metrics collected: time-to-triage, FP/FN feedback, and how useful the CVE filters and reports actually are in practice.
- High-priority fixes applied based on feedback, v1.0 tagged, and a short "what was delivered" summary ready for the GSoC final report.

**Day 1–2:** Execute pilot with first org; collect feedback and metrics (time-to-triage, FP/FN, CVE filter usefulness); take notes.  
**Day 3–4:** Second pilot org if applicable; apply high-priority fixes from feedback; regression test and smoke checks.  
**Day 5:** Tag v1.0; write "what was delivered" summary (features, tests, docs, pilot results); submit final report.

---

## 6. Milestone checkpoints

- **Midterm (end of Week 8):** E2E demo — signed ingestion → Finding in DB → triage list with filters → server-side decrypt/view evidence → "Convert to Issue" with CVE autopopulated.
- **Final (end of Week 16):** Verified-events webhook + minimal detector pack + curated metrics + pilot feedback.

---

## 7. Security invariants & design constraints (ztr-finding-1)

- **Required envelope fields:** `version`, `sender_id`, `issued_at` (UTC), `nonce`, `signature`, `payload_digest`, `payload_ciphertext`|`plaintext_mode`, `alg`.
- **Signature:** HMAC-SHA256 or Ed25519 over canonical JSON; anything outside ±300s skew gets rejected; nonces must be strictly monotonic per `sender_id` within TTL; `unique(sender_id, nonce)` enforced in DB.
- **Evidence:** Store only the digest and size; enforce max size and allowed MIME types; never log ciphertext or plaintext under any circumstance.
- **Inputs:** Normalized and length-capped; logs redact secrets and long fields.
- **Permissions:** All Finding reads are org-scoped; evidence is gated by explicit permission with every access logged. "Convert to Issue" enforces org ownership and rate limits.

---

## 8. Tests to include (beyond PR #5057)

- **Envelope/ingestion:** Valid, expired, future-issued, and replayed envelopes; multi-process replay via DB uniqueness; signature mismatch and canonicalization drift; size/MIME caps; filename/path sanitization; access logging on evidence reads.
- **Dedup/idempotency:** Same fingerprint collapses; evidence variation creates a new attachment; concurrent submissions collapse correctly.
- **Triage/filters:** Pagination boundaries; permission leakage (Org A user cannot see Org B findings); CVE filters reuse `normalize_cve_id` and cache paths; cache-poisoning resistance.
- **Consensus/resilience:** Critical reconfirmation gate; throttle limits under load; back-pressure returns 429 with `Retry-After`.
- **Reports:** CSV/PDF snapshot tests for redaction and no plaintext secrets.

---

## 9. Cross-project integration (future-friendly, not over-coupled)

- Expose a signed webhook for Verified Events with a stable, versioned schema; write concrete consumption examples for BLT-Rewards and RepoTrust so they're not guessing.
- Don't implement downstream scoring, gamification, or education logic — NetGuardian emits clean events and stops there.
- **Versioning:** `version` field in both the envelope (`ztr-finding-1`) and event payloads; `dedupe_key` for idempotent consumption. HMAC signing on webhooks reuses existing BLT patterns — no new signing infrastructure.
