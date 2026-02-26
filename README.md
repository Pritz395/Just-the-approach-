# NetGuardian — Technical approach & weekly deliverables (GSoC 2026)

> **Note for maintainers:** This is the **technical part only**. I'll add the SAT/personal sections (bio, coding skills, time commitment, etc.) once you're happy with this approach.

---

## 1. Introduction

This project extends work I've already contributed to OWASP BLT, including PR #5057 (CVE search, filtering, caching, autocomplete, and CVE-aware indexing on the Issue model), which is merged in the main repo. NetGuardian builds on that CVE layer and on the existing BLT-NetGuardian Worker (a Cloudflare Python Worker for autonomous discovery and scanning) to deliver a zero-trust ingestion path for security findings, CVE-aware triage, and verified events for downstream systems (Rewards, RepoTrust, University).

**Relationship to the existing BLT-NetGuardian Worker**

The Worker discovers targets (CT logs, GitHub API, blockchain) and runs security scanners (Web2, Web3, static, contract). Results stage in Cloudflare KV. This GSoC project doesn't rewrite the Worker. It connects that pipeline to BLT by:

1. Adding a BLT exporter in the Worker that converts scan results to signed ztr-finding-1 envelopes and POSTs to BLT.
2. Building BLT ingestion (`/api/ng/ingest`) with replay protection and Postgres storage.
3. Enriching findings via the PR #5057 CVE cache.
4. Adding a triage UI and "Convert to Issue".
5. Emitting HMAC-signed verified events for downstream.

The split is clean: the Worker handles discovery, scanning, and KV; BLT handles ingestion, triage, CVE enrichment, Issue creation, and events.

**Prerequisites verified**

- Python 3.11+, Django 5.x dev environment.
- Access to the BLT main repo and BLT-NetGuardian Worker repo.
- Cloudflare Workers CLI (wrangler) for local Worker testing.
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

Worker already exists; GSoC adds the Exporter and all BLT-side pieces. Flow: Worker to KV to Exporter to signed envelopes to ingestion to CVE enrichment to triage UI to "Convert to Issue" to HMAC-signed webhook. No new queue; existing throttling only.

**Stack**

- BLT server: Django 5.x, Django REST Framework, PostgreSQL.
- Worker: Cloudflare Python Worker (existing, for autonomous scanning and KV storage).

**Architecture split**

- Detection and scanning: already implemented in BLT-NetGuardian Worker, no new detector code in GSoC scope.
- Ingestion API: new DRF endpoints in BLT (`/api/ng/ingest`, `/api/ng/ingest/batch`) reusing existing auth patterns (TokenAuthentication, org-scoped permissions, throttling middleware).
- Triage UI: server-rendered templates with HTMX, reusing existing BLT frontend patterns.
- No new infrastructure: no Celery, no separate worker daemons, no new queue systems. Periodic management commands (cron/Kubernetes CronJob) where needed; existing throttling middleware for rate limits.

**Key files**

- Worker (existing): `src/worker.py`, `src/scanners/*.py`, `src/models/*.py`, `ARCHITECTURE.md`, `API.md`.
- BLT (existing): `website/cache/cve_cache.py` (PR #5057) for `normalize_cve_id()` and `get_cached_cve_score()`; `website/models.py` for `Issue.cve_id` and `Issue.cve_score`; `blt/middleware/throttling.py` and `blt/middleware/ip_restrict.py` for rate limiting and IP controls; `website/views/user.py` for the HMAC pattern used by GitHub webhooks.
- BLT (new): `website/netguardian/` (models, views, templates for Finding, Envelope, EvidenceBlob, EventOutbox, triage UI, and ingestion endpoints).

---

## 3. Security invariants and design (ztr-finding-1)

Every envelope must carry `version`, `sender_id`, `issued_at` (UTC), `nonce`, `signature`, `payload_digest`, `alg`, and either `payload_ciphertext` or a `plaintext_mode` flag.

Signatures are HMAC-SHA256 over canonical JSON, chosen because it works natively in Cloudflare Workers without heavy crypto dependencies. Anything outside a +-300 second clock window gets rejected outright, and nonces must be strictly monotonic per `sender_id` within that TTL. The `unique(sender_id, nonce)` constraint is enforced at the DB level, not just in application code, so replay protection holds even under concurrent requests. Ed25519 is on the table for future BLT-side verification but isn't needed for the Worker in this project.

For evidence, NetGuardian stores only the digest and size, never the raw ciphertext or plaintext, and never logs either under any circumstances. Inbound inputs are normalized and length-capped before they touch anything, and logs redact secrets and long fields by default. All Finding reads are org-scoped at the query level, evidence access requires an explicit permission check and gets logged every time, and "Convert to Issue" enforces both org ownership and rate limits before it goes through.

These invariants are treated as contracts: tests are written against them, and any refactor or library change has to preserve them.

---

## 4. 12-week implementation plan (phases 1-16)

**GSoC 12-week calendar and AI usage**

Models (see section 9): Cursor (IDE); Claude Opus 4.5 (reasoning/design); Claude Sonnet 4.5 (fast edits/boilerplate); GPT-5.2 (second opinion on security).

| GSoC Week | Focus (phases) | Models most likely to be used | How AI helps |
|-----------|---------------|-------------------------------|--------------|
| 1 | Phases 1-2: envelope/schema + ingestion and zero-trust | Claude Opus 4.5, Claude Sonnet 4.5 | Spec draft and field names (Claude Opus 4.5); serializer/test scaffolds (Claude Sonnet 4.5). I refine and harden; GPT-5.2 for second pass on crypto/schema. |
| 2 | Phase 3: BLT Exporter integration | Claude Sonnet 4.5, GPT-5.2 | Mapping ScanResult to envelope and retry/backoff (Claude Sonnet 4.5); I hand-check signing/headers/errors; GPT-5.2 to double-check signing and error paths. |
| 3 | Phase 4: Triage-lite UI | Claude Sonnet 4.5 | Template markup, filter forms, HTMX snippets (Claude Sonnet 4.5); I keep permission/decrypt/queryset logic explicit and reviewed. |
| 4 | Phases 5-6: CVE plumbing + validation/dedup | Claude Sonnet 4.5 | Repetitive CVE/fingerprint tests and migration boilerplate (Claude Sonnet 4.5); I design fingerprint and validation rules. |
| 5 | Phase 7 + Phase 8 start: CVE-aware UX + polish | Claude Sonnet 4.5 | UX copy for filters and "Related CVEs", evidence viewer layout (Claude Sonnet 4.5); I enforce accessibility and security. |
| 6 | Phase 8: triage polish, RFIs, midterm E2E | Claude Sonnet 4.5, Claude Opus 4.5 | RFI template text and demo script (Claude Sonnet 4.5); edge-case enumeration for E2E plan (Claude Opus 4.5). Checkpoint week. |
| 7 | Phase 9: Fidelity and acceptance gates | Claude Sonnet 4.5 | Fixture generation and query/report boilerplate (Claude Sonnet 4.5); I define thresholds and metrics. |
| 8 | Phase 10: consensus and resilience | Claude Sonnet 4.5, Claude Opus 4.5 | Quota/rate-limit patterns (Claude Sonnet 4.5); schema and middleware integration choices (Claude Opus 4.5). |
| 9 | Phase 11: remediation and insights | Claude Sonnet 4.5 | Remediation fragments and "why this matters" copy (Claude Sonnet 4.5); I tune with mentors for OWASP alignment. |
| 10 | Phase 12: disclosure and reports | Claude Sonnet 4.5 | CSV/PDF templates and snapshot-test harnesses (Claude Sonnet 4.5); I own redaction and verify no leaks. CSV required; PDF optional. |
| 11 | Phase 13: verified events for downstream | Claude Opus 4.5, Claude Sonnet 4.5 | Event schema variants and pagination patterns (Claude Opus 4.5); API/docs boilerplate (Claude Sonnet 4.5); I lock schema and HMAC/idempotency. |
| 12 | Phases 14-16: hardening, pilot, v1.0 | Claude Sonnet 4.5, GPT-5.2 | Checklists and runbooks (Claude Sonnet 4.5); final summary draft (Claude Sonnet 4.5); GPT-5.2 second opinion on security-critical diffs; I do final review. |

*Phase 5 is fast (~2 days) because it reuses the existing CVE cache utilities from PR #5057. It is paired with Phase 6 in Week 4 to keep the 12-week timeline realistic. Phase numbers are implementation milestones, not weeks; the table above shows how 16 named phases map onto 12 GSoC weeks.*

---

### Phase 1 — Envelope and schema

**Weekly deliverables**

- A written ztr-finding-1 spec covering fields, signatures, timestamps, and nonces, detailed enough that someone else could implement against it without coming back with questions.
- Database/ORM models for Finding, Envelope, EvidenceBlob, and SenderKey with migrations applied and everything wired into admin.
- A key registry model (per-org/per-sender) with `kid` for rotation and a clear approach to nonce uniqueness, either a DB table or a cache-key pattern, decided and documented.
- Pagination defaults and a DB index strategy for findings written down now, so we're not improvising under pressure later.
- Serializer stubs for Finding/Envelope and unit tests covering model constraints and validation.

**AI use**

The main thing I need AI for here is getting a first draft of the ztr-finding-1 spec on paper quickly. Claude Opus 4.5 handles proposing field names and structure, which I then tighten until another implementer could use it without ambiguity. For the serializer skeletons and pytest fixtures, Claude Sonnet 4.5 handles the repetitive scaffolding while I add the edge cases and security assertions that matter. Before locking the contract, I'll run the spec past GPT-5.2 to check crypto wording and consistency; it's easier to catch those issues before anything is built against the schema.

---

### Phase 2 — Ingestion and zero-trust

**Weekly deliverables**

- A working ingestion API (`/api/ng/ingest`) that accepts signed findings and verifies signatures and timestamps on the server side.
- Solid replay protection: Envelope unique on `(sender_id, nonce)`, clock skew capped at +-5 minutes, `received_at`/`validated_at` stored, and anything expired or replayed rejected cleanly.
- TokenAuthentication with per-org scoping, body size caps (<=1 MB), and rate limits wired into the existing throttling middleware.
- Property tests for the signature window, log redaction, and idempotency, plus one E2E test proving a valid envelope actually ends up as a stored Finding.

**AI use**

The DRF view and request/response serializer boilerplate is where Claude Sonnet 4.5 saves the most time. The parts I write myself are the verification logic, the replay check, and the size caps, because those are where bugs matter. For the property tests, I'll use Claude Sonnet 4.5 to suggest parametrized cases and then add the replay and concurrency scenarios myself. The ingestion path gets a GPT-5.2 review before merge, focused on the timestamp/nonce/signature handling and what happens on each error path.

---

### Phase 3 — BLT Exporter integration

**Weekly deliverables**

- A `BLTExporter` class in BLT-NetGuardian Worker (`src/exporters/blt_exporter.py`) that maps Worker ScanResult to ztr-finding-1 envelopes, signs with HMAC-SHA256 using Cloudflare Workers stdlib (no heavy crypto libs), and POSTs to BLT `/api/ng/ingest` with retry/timeout.
- An integration point in `src/worker.py` via `handle_result_ingestion()` that calls the exporter after KV storage, best-effort, so the Worker keeps going if BLT is unreachable.
- Cloudflare secrets configured: `BLT_INGEST_URL`, `SENDER_ID`, `KID`, `SENDER_SECRET`.
- End-to-end test: Worker scan to KV to BLT exporter to BLT dev server to Finding row in Postgres.

**AI use**

The ScanResult to envelope field mapping is mechanical but easy to get wrong in subtle ways, so I'll use Claude Sonnet 4.5 to propose a first-pass mapping and small retry/backoff utilities, then go through every field manually. The canonical JSON construction and HMAC signing I write myself; those aren't places to trust generated code without fully understanding it. GPT-5.2 reviews the finished exporter before it goes in, specifically for secret handling and failure behavior when BLT is unreachable.

---

### Phase 4 — Triage-lite UI

**Weekly deliverables**

- A Finding list view with severity/rule/target filters, pagination, and sort, something you can actually sit down and use to triage.
- A detail view that decrypts evidence server-side, gates it behind explicit permissions, logs every access, and renders it in redacted form.
- A "Convert to Issue" flow sketched out and wired to the BLT Issue model (CVE columns come later).
- Org-scoped permissions enforced throughout, with basic template or browser tests for list and detail.

**AI use**

This phase has a lot of template markup and filter form wiring that Claude Sonnet 4.5 handles fine. What I keep explicit and review carefully are the permission checks, the decrypt paths, and the queryset scoping, because those are where org data leaks happen. For test scaffolding, Claude Sonnet 4.5 gets me the structure; I fill in the permission leakage cases and evidence access assertions.

---

### Phase 5 — CVE intelligence

**Weekly deliverables**

- Findings (or linked Issues) populated with `cve_id` and `cve_score`, using `normalize_cve_id` and `get_cached_cve_score` from `website/cache/cve_cache.py`.
- CVE columns surfaced in both the triage list and detail views.
- Mapping tests only; cache-miss fallback is already covered by the existing suite.

**AI use**

Most of this phase is wiring existing utilities into new places, so AI usage is narrow: Claude Sonnet 4.5 for the repetitive mapping tests and migration boilerplate, and suggestions on where to surface the CVE columns in list/detail. The integration points and index constraints I decide myself.

---

### Phase 6 — Validation and dedup

**Weekly deliverables**

- A fingerprint defined as `(rule_id, target_url, optional selector, evidence_digest)` with a unique DB index backing it.
- Idempotent submission that upserts by fingerprint, so new evidence attaches to an existing Finding instead of creating a duplicate.
- A confidence score field (Decimal [0, 1]) and optional FP checklist fields on Finding.
- Tests covering duplicate collapse, evidence attachment rollup, and concurrent submission races.

**AI use**

The fingerprint definition and upsert logic I design from scratch; those decisions have downstream consequences for dedup correctness. Where I use Claude Sonnet 4.5 is generating the repetitive test cases: same fingerprint, different fingerprint, concurrent upsert races. There are enough permutations that generating the scaffolding is worthwhile, and I then go through each case to make sure the DB uniqueness constraint is actually being exercised and not just mocked away.

---

### Phase 7 — CVE-aware triage UX

**Weekly deliverables**

- Finding list filters for `cve_id`, `cve_score_min`, and `cve_score_max`, mirroring what already exists on the Issue side.
- A "Related CVEs" side panel rendered server-side from the existing CVE index, no new infrastructure.
- CVE autocomplete reused in the "Convert to Issue" flow, so people aren't typing CVE IDs by hand.
- UI tests confirming filters interact correctly and the related CVE list and autocomplete behave as expected.

**AI use**

Claude Sonnet 4.5 is useful here for UX copy (filter labels, placeholders, panel headings) and layout ideas for the evidence viewer. I don't need a model to decide what filters to build, but having something suggest wording quickly is genuinely faster than writing it from scratch. I enforce accessibility and make sure nothing in the UI surface leaks sensitive data through labels or autocomplete suggestions.

---

### Phase 8 — Triage polish and RFI

**Weekly deliverables**

- A noticeably better evidence viewer, with improved layout, syntax highlighting or snippet context, something that makes reviewing findings less of a slog.
- Canned RFI templates as markdown/callout partials, ready to drop into the detail view without touching email plumbing.
- Tests confirming templates don't leak secrets and render safely.
- Midterm checkpoint: A full E2E demo, Worker scan to BLT Exporter to signed ingestion to Finding to triage list to server-side decrypt to "Convert to Issue" with CVE autopopulated.

**AI use**

RFI template text is a good fit for Claude Sonnet 4.5 since the output is markdown prose, not logic. I review each template to make sure it renders safely and doesn't reference anything that could leak. For the midterm E2E, Claude Sonnet 4.5 helps draft the demo script while Claude Opus 4.5 is more useful for thinking through failure modes and edge cases in the test plan. I run the actual demo and write the assertions.

---

### Phase 9 — Fidelity and acceptance gates

**Weekly deliverables**

- Five to eight curated fixtures with known expected outcomes (for example specific CVE IDs with known severities), the ground truth used to measure the Worker to BLT pipeline.
- A management command in BLT that queries the Worker `/api/vulnerabilities` endpoint and compares results against expected fixtures, persisting per-fixture metrics (ingestion success, CVE enrichment match).
- Documented acceptance gates: at least 95% ingestion success rate; at least 90% CVE match on known CVEs.
- Scope is pipeline integrity and CVE enrichment accuracy, not scanner rule tuning, which is already the Worker's job.

**AI use**

Fixture generation is tedious, and Claude Sonnet 4.5 speeds it up. The thresholds (95% ingestion, 90% CVE match) I choose based on what the pipeline can actually achieve rather than what sounds good, so those don't come from a model. Same for the management command structure; Claude Sonnet 4.5 can scaffold the reporting output, but I write the assertions and verify the metrics are computed correctly against real fixture data.

---

### Phase 10 — Consensus and resilience

**Weekly deliverables**

- A reconfirmation gate for critical-severity findings, a second heuristic or rule has to agree before "Convert to Issue" goes through.
- Confidence scoring updated to factor in whether reconfirmation happened.
- Per-org/hour quotas via DB counters, hooked into the existing throttling and IPRestrictMiddleware; batch ingestion stays in DB transactions only, no new queue.
- Tests for the reconfirmation gate and for 429/Retry-After behavior under load.

**AI use**

The reconfirmation gate design is something I want to think through with Claude Opus 4.5 before committing to an implementation, specifically when to require a second signal and how to weight confidence. There are real tradeoffs between false negative rate and friction that are worth exploring before anything is built. For the rate-limit and 429 test structure, Claude Sonnet 4.5 handles the boilerplate; I implement the DB schema and make sure it hooks into the existing middleware correctly rather than adding a parallel path.

---

### Phase 11 — Remediation and insights

**Weekly deliverables**

- Markdown remediation fragments for each rule type, with OWASP links, static content, nothing dynamic.
- CVE-based enrichment when `cve_id` is present: advisory links and OWASP context so reviewers don't have to leave the page.
- "Why this matters" callouts and remediation hints surfaced in the triage detail view and report output.
- Tests confirming fragments render safely and map correctly to the right rules and CVEs.

**AI use**

This is one of the phases where AI earns its keep most clearly. Drafting remediation prose for each rule type is time-consuming, and Claude Sonnet 4.5 gets a reasonable first pass on paper quickly. What I bring is tuning the tone and accuracy with mentors, verifying OWASP link targets are correct, and making sure nothing in the fragments is dynamic or executable. The rendering safety tests I write myself since those are the ones that actually catch XSS.

---

### Phase 12 — Disclosure helpers and reports

**Weekly deliverables**

- `security.txt` detection (fetch/parse or a stub) integrated into "Convert to Issue" and the report flow, so disclosure contacts surface automatically.
- CSV export for findings with CVE metadata included; snapshot tests confirming sensitive evidence does not leak in plain text.
- PDF export (for example WeasyPrint, timeboxed to 0.5-1 week); snapshot tests confirming the same redaction guarantees hold.

**AI use**

Export templates and snapshot-test harnesses are mechanical work that Claude Sonnet 4.5 handles well. The redaction rules I define myself and verify manually against the snapshot output; the test passing isn't enough if I haven't checked what's actually in the file. For `security.txt`, Claude Sonnet 4.5 can suggest the fetch/parse structure, but the integration point and permission gating I implement directly.

---

### Phase 13 — Verified events for downstream

**Weekly deliverables**

- An `EventOutbox` table with a versioned payload schema: `cve_id`, `cve_score`, `rule_id`, `severity`, `org_id`/`repo`, `finding_id`/`issue_id`, `created_at`, `dedupe_key`, `version`.
- Webhook delivery signed with HMAC-SHA256 (reusing BLT's existing GitHub/Slack HMAC patterns from `website/views/user.py`), emitted on "Convert to Issue" and on resolution, with an idempotency key and exponential backoff.
- A read-only API for event retrieval with pagination and filtering, plus consumption docs written specifically for Rewards (B) and RepoTrust (X).
- Tests covering event emission, idempotency, payload shape, and webhook signature verification.

**AI use**

I'll use Claude Opus 4.5 to explore payload shape options before locking the schema. The downstream consumers (Rewards, RepoTrust) have different needs and it's worth thinking through the tradeoffs before anything is built against it. Once the schema is decided, Claude Sonnet 4.5 handles the model, serializer, and doc boilerplate. The HMAC signing and idempotency behavior I implement myself, reusing the existing helpers from `website/views/user.py` rather than generating new signing code.

---

### Phase 14 — Hardening and security review

**Weekly deliverables**

- A proper security review pass: key handling, nonce uniqueness, evidence redaction in logs and templates, permission checks everywhere, and cache-poisoning resistance.
- Dead code and over-generalized code cleaned out; docs updated to reflect what is actually implemented.
- A checklist or short report summarizing what was reviewed and what was fixed.

**AI use**

Claude Sonnet 4.5 is useful for generating a structured review checklist so nothing gets missed. The actual review I do myself: reading the code, checking log statements, tracing permission paths. GPT-5.2 gets a look at any changed security-critical code before final merge, as a second set of eyes rather than a substitute for my own review.

---

### Phase 15 — Pilot prep and docs

**Weekly deliverables**

- A pilot checklist covering configuration steps, runbooks, and a rollback plan, so the first run is not improvised.
- A migration rollback note and data deletion playbook for evidence blobs.
- User, admin, setup, and contribution docs polished and reviewed end-to-end.

**AI use**

Runbook and checklist drafting is where Claude Sonnet 4.5 saves real time; getting structure on paper fast matters here. I validate every step against the actual deployment and add org-specific notes. The migration rollback playbook I verify command by command since the order matters and a wrong step is hard to recover from.

---

### Phase 16 — Pilot run and final polish

**Weekly deliverables**

- A live pilot with one to two orgs, with real metrics collected: time-to-triage, FP/FN feedback, and how useful the CVE filters and reports actually are in practice.
- High-priority fixes applied based on feedback, v1.0 tagged, and a short "what was delivered" summary ready for the GSoC final report.

**AI use**

The final report summary is a good fit for Claude Sonnet 4.5 to draft since it's largely a factual account of what was built. I tailor it to actual deliverables and pilot feedback rather than what was planned. Any patches from pilot feedback with security implications get a GPT-5.2 review before the v1.0 tag.

---

## 5. Flexibility and change management

NetGuardian is designed to stay flexible for maintainers and the community. If we collectively decide mid-project that a different library or pattern makes more sense, I'll isolate third-party dependencies behind small wrappers so swapping them doesn't ripple through callers. Any non-trivial change gets a short Markdown note first (what, why, alternatives, schedule impact) before I touch anything. The acceptance criteria in this proposal (ingestion invariants, fidelity gates, verified-event schema) act as contracts; if we refactor, the tests stay the same and still have to pass.

---

## 6. Testing strategy (beyond PR #5057)

The ingestion tests cover the full envelope lifecycle: valid submission, expired timestamp, future-issued, and replay, with the replay case exercised via DB uniqueness so it holds under concurrent load rather than just in a single-threaded test. Signature mismatch and canonicalization drift are tested separately because they're easy to conflate. Size and MIME cap enforcement, filename sanitization, and access logging on evidence reads each get their own cases.

For the exporter, the main test is the full Worker to BLT E2E path. Beyond that: retry behavior when BLT is down, idempotency via nonce reuse, and how the exporter handles a malformed scan result without taking the Worker down with it.

Dedup tests focus on the fingerprint: the same fingerprint should collapse into one Finding, and concurrent submissions of the same fingerprint should also collapse cleanly rather than racing to create duplicates.

On the triage side: pagination boundaries, org permission leakage (Org A user cannot see Org B findings), CVE filter correctness reusing the `normalize_cve_id` and cache paths from PR #5057, and basic cache-poisoning resistance. For consensus and resilience: the critical reconfirmation gate and 429/Retry-After behavior under load. For reports: CSV and PDF snapshot tests that confirm sensitive evidence does not appear in plain text output.

---

## 7. Milestone checkpoints

- **Midterm (after Phase 8):** E2E demo — Worker scan to BLT Exporter to signed ingestion to Finding to triage list to server-side decrypt to "Convert to Issue" with CVE autopopulated.
- **Final (after Phase 16):** Full Worker to BLT pipeline live plus verified-events webhook plus fidelity metrics plus pilot feedback.

---

## 8. Cross-project integration

NetGuardian emits a signed webhook for Verified Events with a stable, versioned schema. I'll write concrete consumption examples specifically for BLT-Rewards and RepoTrust, not just the schema, but working examples so they're not guessing at edge cases. NetGuardian stops at emitting clean events; downstream scoring, gamification, and education logic are out of scope and should not live here.

Both the envelope and event payloads carry a `version` field and a `dedupe_key` for idempotent consumption. Webhook signing reuses the existing HMAC helpers from `website/views/user.py`, the same pattern BLT already uses for GitHub webhooks, so there is no new signing infrastructure to maintain.

---

## 9. AI tooling

**IDE (fixed) and models (tentative)**

- IDE: Cursor.
- Strong reasoning (architecture, threat modeling, tricky refactors): Claude Opus 4.5, used when design choices and edge cases matter.
- Fast edits and boilerplate: Claude Sonnet 4.5, used for inline edits, serializers, migrations, docstrings, type hints, and repetitive scaffolding.
- Second opinion on security-sensitive code and docs: GPT-5.2, used to double-check ingestion, signing, nonce handling, permissions, and evidence redaction.

**Why these models**

- Claude Opus 4.5 leads SWE-bench at 77-82% and is widely cited as the best available model for architecture planning and agentic/multi-step reasoning in coding contexts, making it the right fit for design-heavy phases.
- Claude Sonnet 4.5 is the fast workhorse; developers using Cursor and Claude Code consistently describe it as the default for quick iteration, and it handles boilerplate-heavy phases well without needing a heavier model.
- GPT-5.2 covers security review; its strong multi-language handling and code correctness make it a solid second opinion on signing, nonce handling, and redaction paths specifically.

Every security-critical path (ingestion, signing, nonce handling, permissions, evidence redaction) gets hand-reviewed and test-covered before it merges. AI is useful for drafts and suggestions, but I won't ship something I don't fully understand just because a model generated it confidently. The per-phase AI notes above describe specifically where each model is used and why.
