# NetGuardian — Technical approach & weekly deliverables (GSoC 2026)

> **Note for maintainers:** This is the **technical part only**. I'll add the SAT/personal sections (bio, coding skills, time commitment, etc.) once you're happy with this approach.

---

## 1. Introduction

This project extends work I've already contributed to OWASP BLT, including PR `#5057` (CVE search, filtering, caching, autocomplete, and CVE-aware indexing on the Issue model), which is merged in the main repo. NetGuardian builds on that CVE layer and on the existing BLT-NetGuardian Worker (a Cloudflare Python Worker for autonomous discovery and scanning) to deliver a zero-trust ingestion path for security findings, CVE-aware triage, and verified events for downstream systems (Rewards, RepoTrust, University).

**Relationship to the existing BLT-NetGuardian Worker**

The Worker discovers targets (CT logs, GitHub API, blockchain) and runs security scanners (Web2, Web3, static, contract). Results stage in Cloudflare KV. This GSoC project doesn't rewrite the Worker. It connects that pipeline to BLT by:

1. Adding a BLT exporter in the Worker that converts scan results to signed ztr-finding-1 envelopes and POSTs to BLT.
2. Building BLT ingestion (`/api/ng/ingest`) with replay protection and Postgres storage.
3. Enriching findings via the PR `#5057` CVE cache.
4. Adding a triage UI and "Convert to Issue".
5. Emitting HMAC-signed verified events for downstream.

The split is clean: the Worker handles discovery, scanning, and KV; BLT handles ingestion, triage, CVE enrichment, Issue creation, and events.

**Prerequisites verified**

- Python 3.11+, Django 5.x dev environment.
- Access to the BLT main repo and BLT-NetGuardian Worker repo.
- Cloudflare Workers CLI (`wrangler`) for local Worker testing.
- PostgreSQL 14+ for local schema and management command testing.

---

## 2. Architecture, stack & key files

```mermaid
flowchart TB
    subgraph Worker["BLT-NetGuardian Worker - Cloudflare"]
        Discovery["Autonomous Discovery - CT logs, GitHub, blockchain"]
        Scanners["Security Scanners - Web2, Web3, static, contract"]
        KV["Cloudflare KV - temporary task/result storage"]
        Exporter["BLT Exporter NEW - canonical JSON + HMAC sign + POST"]
    end

    subgraph BLT["BLT Server - Django + Postgres"]
        Ingest["Ingestion API NEW - verify envelope, replay check, rate limit"]
        DB["PostgreSQL - Finding, Envelope, EvidenceBlob, SenderKey, EventOutbox"]
        CVE["CVE Cache EXISTS PR 5057 - normalize_cve_id, get_cached_cve_score"]
        Triage["Triage UI NEW - list, filters, detail, server-side decrypt"]
        Convert["Convert to Issue NEW - Finding to Issue with CVE"]
        Reports["CSV / PDF Reports NEW - export findings with CVE metadata"]
        Webhook["events_ng Outbox NEW - HMAC-signed webhook"]
    end

    subgraph Downstream["Downstream - out of GSoC scope"]
        Rewards["BLT-Rewards B"]
        RepoTrust["RepoTrust X"]
        University["BLT University C"]
    end

    Discovery --> Scanners
    Scanners --> KV
    KV --> Exporter
    Exporter -->|ztr-finding-1 envelope| Ingest
    Ingest --> DB
    DB --> CVE
    CVE --> Triage
    Triage --> Convert
    Convert --> DB
    DB --> Webhook
    Triage --> Reports
    Webhook --> Rewards
    Webhook --> RepoTrust
    Webhook -.->|future| University
```

**Downstream consumers:** B = BLT-Rewards (gamification); X = RepoTrust (repository trust scoring); C = BLT-University (education, future).

Worker already exists; GSoC adds the Exporter and all BLT-side pieces. Flow: Worker to KV to Exporter to signed envelopes to ingestion to CVE enrichment to triage UI to "Convert to Issue" to HMAC-signed webhook. No new queue; existing throttling only.

**Stack**

- **BLT server:** Django 5.x, Django REST Framework, PostgreSQL.
- **Worker:** Cloudflare Python Worker (existing, for autonomous scanning and KV storage).

**Architecture split**

- **Detection and scanning:** Already implemented in BLT-NetGuardian Worker. No new detector code in GSoC scope.
- **Ingestion API:** New DRF endpoints in BLT (`/api/ng/ingest`, `/api/ng/ingest/batch`) reusing existing auth patterns (TokenAuthentication, org-scoped permissions, throttling middleware).
- **Triage UI:** Server-rendered templates with HTMX, reusing existing BLT frontend patterns.
- **No new infrastructure:** No Celery, no separate worker daemons, no new queue systems. Periodic management commands (cron/Kubernetes CronJob) where needed; existing throttling middleware for rate limits.

**Key files**

- **Worker (existing):** `src/worker.py`, `src/scanners/*.py`, `src/models/*.py`, `ARCHITECTURE.md`, `API.md`.
- **BLT (existing):**
  - `website/cache/cve_cache.py` (PR `#5057`) for `normalize_cve_id()` and `get_cached_cve_score()`.
  - `website/models.py` for `Issue.cve_id` and `Issue.cve_score`.
  - `blt/middleware/throttling.py` and `blt/middleware/ip_restrict.py` for rate limiting and IP controls.
  - `website/views/user.py` for the HMAC pattern used by GitHub webhooks.
- **BLT (new in this project):**
  - `website/netguardian/` (models, views, templates for Finding, Envelope, EvidenceBlob, EventOutbox, triage UI, and ingestion endpoints).

---

## 3. Security invariants and design (ztr-finding-1)

Every envelope must carry `version`, `sender_id`, `issued_at` (UTC), `nonce`, `signature`, `payload_digest`, `alg`, and either `payload_ciphertext` or a `plaintext_mode` flag.

Signatures are HMAC-SHA256 over canonical JSON, chosen because it works natively in Cloudflare Workers without heavy crypto dependencies. Anything outside a +-300 second clock window gets rejected outright, and nonces must be strictly monotonic per `sender_id` within that TTL. The `unique(sender_id, nonce)` constraint is enforced at the DB level, not just in application code, so replay protection holds even under concurrent requests. Ed25519 is on the table for future BLT-side verification but isn't needed for the Worker in this project.

For evidence, NetGuardian stores only the digest and size, never the raw ciphertext or plaintext, and never logs either under any circumstances. Inbound inputs are normalized and length-capped before they touch anything, and logs redact secrets and long fields by default. All Finding reads are org-scoped at the query level, evidence access requires an explicit permission check and gets logged every time, and "Convert to Issue" enforces both org ownership and rate limits before it goes through.

These invariants are treated as contracts: tests are written against them, and any refactor or library change has to preserve them.

---

## 4. 12-week implementation plan (phases 1–16)

**GSoC 12-week calendar and AI usage**

Models (see 9): **Cursor** (IDE); **GPT-4.1** (reasoning/design); **o3-mini** (fast edits/boilerplate); **Claude 3.5 Sonnet** (second opinion on security).

| GSoC Week | Focus (phases) | Models most likely to be used | How AI helps (where, how, why — details under each phase below) |
|-----------|----------------|-------------|------------------------------------------------------------------|
| 1 | Phases 1–2: envelope/schema + ingestion and zero-trust | GPT-4.1, o3-mini | Spec draft and field names (GPT-4.1); serializer/test scaffolds (o3-mini). I refine and harden; Claude 3.5 for second pass on crypto/schema. |
| 2 | Phase 3: BLT Exporter integration | o3-mini, Claude 3.5 | Mapping ScanResult→envelope and retry/backoff (o3-mini); I hand-check signing/headers/errors; Claude 3.5 to double-check signing and error paths. |
| 3 | Phase 4: Triage-lite UI | o3-mini | Template markup, filter forms, HTMX snippets (o3-mini); I keep permission/decrypt/queryset logic explicit and reviewed. |
| 4 | Phases 5–6: CVE plumbing + validation/dedup | o3-mini | Repetitive CVE/fingerprint tests and migration boilerplate (o3-mini); I design fingerprint and validation rules. |
| 5 | Phase 7 + Phase 8 start: CVE-aware UX + polish | o3-mini | UX copy for filters and "Related CVEs", evidence viewer layout (o3-mini); I enforce a11y and security. |
| 6 | Phase 8: triage polish, RFIs, midterm E2E | o3-mini, GPT-4.1 | RFI template text and demo script (o3-mini); edge-case enumeration for E2E plan (GPT-4.1). Checkpoint week. |
| 7 | Phase 9: Fidelity and acceptance gates | o3-mini | Fixture generation and query/report boilerplate (o3-mini); I define thresholds and metrics. |
| 8 | Phase 10: consensus and resilience | o3-mini, GPT-4.1 | Quota/rate-limit patterns (o3-mini); schema and middleware integration choices (GPT-4.1). |
| 9 | Phase 11: remediation and insights | o3-mini | Remediation fragments and "why this matters" copy (o3-mini); I tune with mentors for OWASP alignment. |
| 10 | Phase 12: disclosure and reports | o3-mini | CSV/PDF templates and snapshot-test harnesses (o3-mini); I own redaction and verify no leaks. CSV required; PDF optional. |
| 11 | Phase 13: verified events for downstream | GPT-4.1, o3-mini | Event schema variants and pagination patterns (GPT-4.1); API/docs boilerplate (o3-mini); I lock schema and HMAC/idempotency. |
| 12 | Phases 14–16: hardening, pilot, v1.0 | o3-mini, Claude 3.5 | Checklists and runbooks (o3-mini); final summary draft (o3-mini); Claude 3.5 second opinion on security-critical diffs; I do final review. |

\* Phase 5 is fast (~2 days) because it reuses the existing CVE cache utilities from PR `#5057`. It is paired with Phase 6 in Week 4 to keep the 12-week timeline realistic. Phase numbers are **implementation milestones**, not weeks; the table above shows how 16 named phases map onto 12 GSoC weeks.

### Phase 1 — Envelope and schema

**Weekly deliverables**

- A written ztr-finding-1 spec covering fields, signatures, timestamps, and nonces, detailed enough that someone else could implement against it without coming back with questions.
- Database/ORM models for Finding, Envelope, EvidenceBlob, and SenderKey with migrations applied and everything wired into admin.
- A key registry model (per-org/per-sender) with `kid` for rotation and a clear approach to nonce uniqueness, either a DB table or a cache-key pattern, decided and documented.
- Pagination defaults and a DB index strategy for findings written down now, so we're not improvising under pressure later.
- Serializer stubs for Finding/Envelope and unit tests covering model constraints and validation.

**AI use (where, how, why)**

- **ztr-finding-1 spec draft:** GPT-4.1 in Cursor — to propose field names, structure, and wording for the spec; I then harden so another implementer could use it without ambiguity.
- **Serializer stubs and test scaffolds:** o3-mini in Cursor — to generate DRF serializer skeletons and pytest fixtures for model validation; I add edge cases and security assertions.
- **Second pass on schema/crypto wording:** Claude 3.5 — to review the spec for consistency and crypto clarity before locking the contract.

### Phase 2 — Ingestion and zero-trust

**Weekly deliverables**

- A working ingestion API (`/api/ng/ingest`) that accepts signed findings and verifies signatures and timestamps on the server side.
- Solid replay protection: Envelope unique on `(sender_id, nonce)`, clock skew capped at +-5 minutes, `received_at`/`validated_at` stored, and anything expired or replayed rejected cleanly.
- TokenAuthentication with per-org scoping, body size caps (<=1 MB), and rate limits wired into the existing throttling middleware.
- Property tests for the signature window, log redaction, and idempotency, plus one E2E test proving a valid envelope actually ends up as a stored Finding.

**AI use (where, how, why)**

- **View/serializer boilerplate for `/api/ng/ingest`:** o3-mini in Cursor — to draft DRF view and request/response serializers; I implement verification, replay check, and caps myself and hand-review.
- **Property-test cases (signature window, idempotency):** o3-mini — to suggest parametrized test cases and assertions; I add replay and concurrency cases and verify log redaction.
- **Security review of ingestion path:** Claude 3.5 — to double-check timestamp/nonce/signature handling and error paths before merge.

### Phase 3 — BLT Exporter integration

**Weekly deliverables**

- A `BLTExporter` class in BLT-NetGuardian Worker (`src/exporters/blt_exporter.py`) that maps Worker ScanResult to ztr-finding-1 envelopes, signs with HMAC-SHA256 using Cloudflare Workers stdlib (no heavy crypto libs), and POSTs to BLT `/api/ng/ingest` with retry/timeout.
- An integration point in `src/worker.py` via `handle_result_ingestion()` that calls the exporter after KV storage, best-effort, so the Worker keeps going if BLT is unreachable.
- Cloudflare secrets configured: `BLT_INGEST_URL`, `SENDER_ID`, `KID`, `SENDER_SECRET`.
- End-to-end test: Worker scan to KV to BLT exporter to BLT dev server to Finding row in Postgres.

**AI use (where, how, why)**

- **ScanResult → envelope mapping and retry/backoff helpers:** o3-mini in Cursor — to draft the mapping logic and small retry/backoff utilities; I hand-check all signing, headers, and error handling and own the canonical JSON construction.
- **Second opinion on signing and error paths:** Claude 3.5 — to review exporter code for HMAC usage, secret handling, and failure behavior so nothing leaks or breaks the Worker.

### Phase 4 — Triage-lite UI

**Weekly deliverables**

- A Finding list view with severity/rule/target filters, pagination, and sort, something you can actually sit down and use to triage.
- A detail view that decrypts evidence server-side, gates it behind explicit permissions, logs every access, and renders it in redacted form.
- A "Convert to Issue" flow sketched out and wired to the BLT Issue model (CVE columns come later).
- Org-scoped permissions enforced throughout, with basic template or browser tests for list and detail.

**AI use (where, how, why)**

- **Template markup and filter form wiring:** o3-mini in Cursor — to generate Django/HTMX template snippets and filter form fields; I keep permission checks, decrypt paths, and queryset logic explicit and review them.
- **List/detail view structure and tests:** o3-mini — to suggest view structure and basic template or browser test scaffolding; I enforce org-scoping and evidence access rules.

### Phase 5 — CVE intelligence

**Weekly deliverables**

- Findings (or linked Issues) populated with `cve_id` and `cve_score`, using `normalize_cve_id` and `get_cached_cve_score` from `website/cache/cve_cache.py`, no reinventing the wheel.
- CVE columns surfaced in both the triage list and detail views.
- Mapping tests only; cache-miss fallback is already covered by the existing suite.

**AI use (where, how, why)**

- **Repetitive CVE mapping tests and view/column wiring:** o3-mini in Cursor — to generate tests that assert CVE ID/score mapping and to suggest where to surface columns in list/detail; I design the integration points and rely on existing cache behavior.
- **Migration boilerplate for new CVE fields on Finding/Issue:** o3-mini — to draft migration and model changes; I verify indices and constraints.

### Phase 6 — Validation and dedup

**Weekly deliverables**

- A fingerprint defined as `(rule_id, target_url, optional selector, evidence_digest)` with a unique DB index backing it.
- Idempotent submission that upserts by fingerprint, so new evidence attaches to an existing Finding instead of creating a duplicate.
- A confidence score field (Decimal [0, 1]) and optional FP checklist fields on Finding.
- Tests covering duplicate collapse, evidence attachment rollup, and concurrent submission races.

**AI use (where, how, why)**

- **Fingerprint collision and dedup test cases:** o3-mini in Cursor — to generate repetitive tests for same/different fingerprints, evidence rollup, and concurrent upsert scenarios; I define the fingerprint and validation rules and ensure DB uniqueness is exercised.
- **Migration and model changes for fingerprint/confidence/FP fields:** o3-mini — to draft migration and field definitions; I set constraints and indexes.

### Phase 7 — CVE-aware triage UX

**Weekly deliverables**

- Finding list filters for `cve_id`, `cve_score_min`, and `cve_score_max`, mirroring what already exists on the Issue side.
- A "Related CVEs" side panel rendered server-side from the existing CVE index, no new infrastructure.
- CVE autocomplete reused in the "Convert to Issue" flow, so people aren't typing CVE IDs by hand.
- UI tests confirming filters interact correctly and the related CVE list and autocomplete behave as expected.

**AI use (where, how, why)**

- **UX copy for filters and "Related CVEs" panel:** o3-mini in Cursor — to suggest labels, placeholders, and short descriptions; I enforce accessibility and security (no leaking sensitive data in labels).
- **Evidence viewer layout and snippet context ideas:** o3-mini — to propose layout and syntax-highlighting approach; I implement and gate behind permissions.
- **Filter and autocomplete test scaffolding:** o3-mini — to draft UI/test cases for filter interaction and autocomplete; I verify correctness and org-scoping.

### Phase 8 — Triage polish and RFI

**Weekly deliverables**

- A noticeably better evidence viewer, with improved layout, syntax highlighting or snippet context, something that makes reviewing findings less of a slog.
- Canned RFI templates as markdown/callout partials, ready to drop into the detail view without touching email plumbing.
- Tests confirming templates don't leak secrets and render safely.
- **Midterm checkpoint:** A full E2E demo, Worker scan to BLT Exporter to signed ingestion to Finding to triage list to server-side decrypt to "Convert to Issue" with CVE autopopulated.

**AI use (where, how, why)**

- **RFI template text (markdown/callouts):** o3-mini in Cursor — to draft canned RFI partials; I review for safe rendering and no secret leakage.
- **Midterm demo script and edge-case list for E2E:** o3-mini for script wording; GPT-4.1 to enumerate edge cases and failure modes for the E2E test plan; I run the demo and own assertions.

### Phase 9 — Fidelity and acceptance gates

**Weekly deliverables**

- Five to eight curated fixtures with known expected outcomes (for example specific CVE IDs with known severities), the ground truth used to measure the Worker to BLT pipeline.
- A management command in BLT that queries the Worker `/api/vulnerabilities` endpoint and compares results against expected fixtures, persisting per-fixture metrics (ingestion success, CVE enrichment match).
- Documented acceptance gates: at least 95 percent ingestion success rate; at least 90 percent CVE match on known CVEs.
- Scope is pipeline integrity and CVE enrichment accuracy, not scanner rule tuning, which is already the Worker's job.

**AI use (where, how, why)**

- **Fixture generation and query/report boilerplate:** o3-mini in Cursor — to draft fixture JSON/YAML and management-command structure and reporting output; I define acceptance thresholds and assertions and ensure metrics are computed correctly.
- **Documentation of acceptance gates:** o3-mini — to draft the gate criteria and run instructions; I validate numbers and wording.

### Phase 10 — Consensus and resilience

**Weekly deliverables**

- A reconfirmation gate for critical-severity findings, a second heuristic or rule has to agree before "Convert to Issue" goes through.
- Confidence scoring updated to factor in whether reconfirmation happened.
- Per-org/hour quotas via DB counters, hooked into the existing throttling and IPRestrictMiddleware; batch ingestion stays in DB transactions only, no new queue.
- Tests for the reconfirmation gate and for 429/Retry-After behavior under load.

**AI use (where, how, why)**

- **Quota counter and rate-limit test patterns:** o3-mini in Cursor — to suggest test structure for 429/Retry-After and quota exhaustion; I implement DB schema and hook into existing middleware.
- **Reconfirmation gate and confidence logic design:** GPT-4.1 — to explore design options (when to require second signal, how to weight confidence); I choose the exact rules and implement.

### Phase 11 — Remediation and insights

**Weekly deliverables**

- Markdown remediation fragments for each rule type, with OWASP links, static content, nothing dynamic.
- CVE-based enrichment when `cve_id` is present: advisory links and OWASP context so reviewers do not have to leave the page.
- "Why this matters" callouts and remediation hints surfaced in the triage detail view and report output.
- Tests confirming fragments render safely and map correctly to the right rules and CVEs.

**AI use (where, how, why)**

- **Initial remediation fragments and "why this matters" copy:** o3-mini in Cursor — to draft markdown fragments and short callouts per rule type; I tune with mentors for tone, accuracy, and OWASP alignment and ensure no dynamic execution.
- **Test scaffolding for fragment mapping and safe render:** o3-mini — to generate tests that assert correct rule/CVE mapping and safe rendering; I add XSS/injection cases.

### Phase 12 — Disclosure helpers and reports

**Weekly deliverables**

- `security.txt` detection (fetch/parse or a stub) integrated into "Convert to Issue" and the report flow, so disclosure contacts surface automatically.
- CSV export for findings with CVE metadata included; snapshot tests confirming sensitive evidence does not leak in plain text.
- PDF export (for example WeasyPrint, timeboxed to 0.5–1 week); snapshot tests confirming the same redaction guarantees hold.

**AI use (where, how, why)**

- **CSV/PDF export templates and snapshot-test harnesses:** o3-mini in Cursor — to draft export templates and test structure for snapshot comparisons; I own redaction rules and verify no sensitive evidence in output.
- **security.txt stub or parser scaffolding:** o3-mini — to suggest integration points and minimal fetch/parse logic; I implement and gate behind existing permissions.

### Phase 13 — Verified events for downstream

**Weekly deliverables**

- An `EventOutbox` table with a versioned payload schema: `cve_id`, `cve_score`, `rule_id`, `severity`, `org_id`/`repo`, `finding_id`/`issue_id`, `created_at`, `dedupe_key`, `version`.
- Webhook delivery signed with HMAC-SHA256 (reusing BLT's existing GitHub/Slack HMAC patterns from `website/views/user.py`), emitted on "Convert to Issue" and on resolution, with an idempotency key and exponential backoff.
- A read-only API for event retrieval with pagination and filtering, plus consumption docs written specifically for Rewards (B) and RepoTrust (X).
- Tests covering event emission, idempotency, payload shape, and webhook signature verification.

**AI use (where, how, why)**

- **Event schema variants and pagination/filter patterns:** GPT-4.1 in Cursor — to explore payload shapes and filter options for consumers; I lock the final schema, HMAC details, and idempotency behavior.
- **Outbox model, API, and consumption doc boilerplate:** o3-mini — to draft model, serializers, and doc structure for Rewards/RepoTrust; I implement delivery and signing and verify against existing HMAC helpers.

### Phase 14 — Hardening and security review

**Weekly deliverables**

- A proper security review pass: key handling, nonce uniqueness, evidence redaction in logs and templates, permission checks everywhere, and cache-poisoning resistance.
- Dead code and over-generalized code cleaned out; docs updated to reflect what is actually implemented.
- A checklist or short report summarizing what was reviewed and what was fixed.

**AI use (where, how, why)**

- **Security review checklist and report draft:** o3-mini in Cursor — to generate a structured checklist (key handling, nonce, redaction, permissions, cache) and a report template; I perform the actual review and fill in findings and fixes.
- **Second opinion on security-critical diffs:** Claude 3.5 — to review changed code paths before final merge; I own the final sign-off.

### Phase 15 — Pilot prep and docs

**Weekly deliverables**

- A pilot checklist covering configuration steps, runbooks, and a rollback plan, so the first run is not improvised.
- A migration rollback note and data deletion playbook for evidence blobs.
- User, admin, setup, and contribution docs polished and reviewed end-to-end.

**AI use (where, how, why)**

- **Pilot checklist and runbook drafts:** o3-mini in Cursor — to draft configuration steps, runbooks, and rollback plan structure; I validate against actual deployment and add org-specific notes.
- **Migration rollback and data deletion playbook:** o3-mini — to draft the playbook text; I verify commands and ordering.
- **Doc polish (user, admin, setup, contribution):** o3-mini — to suggest improvements and fill gaps; I review end-to-end for accuracy.

### Phase 16 — Pilot run and final polish

**Weekly deliverables**

- A live pilot with one to two orgs, with real metrics collected: time-to-triage, FP/FN feedback, and how useful the CVE filters and reports actually are in practice.
- High-priority fixes applied based on feedback, v1.0 tagged, and a short "what was delivered" summary ready for the GSoC final report.

**AI use (where, how, why)**

- **"What we delivered" summary draft:** o3-mini in Cursor — to draft the GSoC final-report summary and feature list; I tailor to actual deliverables and pilot feedback.
- **Final pass on security-critical changes from pilot:** Claude 3.5 — to review any patches from pilot feedback before v1.0 tag; I do the final merge decision.

---

## 5. Flexibility and change management

NetGuardian is designed to stay flexible for maintainers and the community. If we collectively decide mid-project that a different library or pattern makes more sense, I will isolate third-party dependencies behind small wrappers so swapping them does not ripple through callers. Any non-trivial change gets a short Markdown note first (what, why, alternatives, schedule impact) before I touch anything. The acceptance criteria in this proposal (ingestion invariants, fidelity gates, verified-event schema, and so on) act as contracts; if we refactor, the tests stay the same and still have to pass.

---

## 6. Testing strategy (beyond PR `#5057`)

The ingestion tests cover the full envelope lifecycle: valid submission, expired timestamp, future-issued, and replay, with the replay case exercised via DB uniqueness so it holds under concurrent load rather than just in a single-threaded test. Signature mismatch and canonicalization drift are tested separately because they are easy to conflate. Size and MIME cap enforcement, filename sanitization, and access logging on evidence reads each get their own cases.

For the exporter, the main test is the full Worker to BLT E2E path. Beyond that: retry behavior when BLT is down, idempotency via nonce reuse, and how the exporter handles a malformed scan result without taking the Worker down with it.

Dedup tests focus on the fingerprint: the same fingerprint should collapse into one Finding, and concurrent submissions of the same fingerprint should also collapse cleanly rather than racing to create duplicates.

On the triage side: pagination boundaries, org permission leakage (Org A user cannot see Org B findings), CVE filter correctness reusing the `normalize_cve_id` and cache paths from PR `#5057`, and basic cache-poisoning resistance. For consensus and resilience: the critical reconfirmation gate and 429/Retry-After behavior under load. For reports: CSV and PDF snapshot tests that confirm sensitive evidence does not appear in plain text output.

---

## 7. Milestone checkpoints

- **Midterm (after Phase 8):** E2E demo — Worker scan to BLT Exporter to signed ingestion to Finding to triage list to server-side decrypt to "Convert to Issue" with CVE autopopulated.
- **Final (after Phase 16):** Full Worker to BLT pipeline live plus verified-events webhook plus fidelity metrics plus pilot feedback.

---

## 8. Cross-project integration

NetGuardian emits a signed webhook for Verified Events with a stable, versioned schema. I'll write concrete consumption examples specifically for BLT-Rewards and RepoTrust, not just the schema, but actual working examples so they're not guessing at edge cases. NetGuardian stops at emitting clean events; downstream scoring, gamification, and education logic are out of scope and should not live here.

Both the envelope and event payloads carry a `version` field and a `dedupe_key` for idempotent consumption. Webhook signing reuses the existing HMAC helpers from `website/views/user.py`, the same pattern BLT already uses for GitHub webhooks, so there is no new signing infrastructure to maintain.

---

## 9. AI tooling, usage and flexibility

**IDE (fixed) and models (most likely / tentative / subjective)**

- **IDE:** Cursor.
- **Strong reasoning (architecture, threat modeling, tricky refactors):** e.g. GPT-4.1 — used when design choices and edge cases matter; I validate all output.
- **Fast edits and boilerplate:** e.g. o3-mini — used for inline edits, serializers, migrations, docstrings, type hints, and repetitive scaffolding; I review before commit.
- **Second opinion (security-sensitive code and docs):** e.g. Claude 3.5 Sonnet — used to double-check ingestion, signing, nonce handling, permissions, and evidence redaction; I never ship without understanding and hand-review.

The constraint I'm holding to: every security-critical path (ingestion, signing, nonce handling, permissions, evidence redaction) gets hand-reviewed and test-covered before it merges. AI is useful for drafts and suggestions, but I will not ship something I don't fully understand just because a model generated it confidently.

In practice, this means using AI for scaffolding and spec text that I then tighten up, repetitive test generation and template markup while I keep permission and queryset logic explicit, and later for checklists, runbook drafts, and the final summary. Throughout, it handles mechanical work like renames, docstrings, and type hints, while I own the actual design decisions. Per-phase granularity (where AI is used, which model, how, and why) is spelled out under each phase's deliverables below.
