# NetGuardian — Technical approach & weekly deliverables (GSoC 2026)

> **Note for maintainers:** This is the technical part only. I'll add the SAT/personal sections (bio, time commitment, availability, etc.) after you're happy with the approach.

> **Scope note:** This GSoC scope does not add new Django endpoints, templates, or PostgreSQL models to the BLT main repo. All NetGuardian backend logic runs in Cloudflare Workers with D1, and the triage UI is a static app on GitHub Pages consuming Worker JSON APIs. CVE data and Issue creation are delegated to BLT-API.

---

## 1. Introduction

This proposal connects the existing BLT-NetGuardian autonomous Worker to BLT's triage and issue-tracking via a zero-trust ingestion path, CVE-aware triage, and verified downstream events. It builds on my recent BLT contribution (PR #5057: CVE search, filtering, caching, autocomplete, and Issue model CVE columns) by enriching new security Findings with CVE metadata for faster triage.

**Important pivot (per maintainer guidance)**

- Serverless-first stack: Cloudflare Worker (Python) + Cloudflare D1 (SQLite) + GitHub Pages (static triage SPA).
- No new Django/DRF endpoints or PostgreSQL models in BLT.
- BLT-API provides CVE normalization/score and Issue creation on Convert-to-Issue.

**Relationship to the existing BLT-NetGuardian Worker**

- **Worker (already exists):** autonomous discovery (CT logs, GitHub, blockchain) and scanners (Web2, Web3, static, contract) staging intermediate results in KV.
- This GSoC project does not rewrite scanners. It:
  - Adds a BLT exporter (in Worker) that builds/sends signed ztr-finding-1 envelopes to a dedicated ingestion route.
  - Implements ingestion, triage APIs, CVE enrichment, dedup/idempotency, and verified events in Worker with D1 storage.
  - Serves a static triage SPA on GitHub Pages that calls Worker JSON APIs (no secrets on the client).

**Prerequisites verified**

- Cloudflare Workers + D1 (SQLite-based).
- GitHub Pages for static SPA hosting.
- BLT-API endpoints for CVE normalization/score and Issue creation.

---

## 2. Architecture & Stack (Cloudflare Worker + GitHub Pages + D1 + BLT-API)

```mermaid
flowchart TB
  subgraph Clients["Clients"]
    FC["Flutter / local agents"]
    WX["BLT-NetGuardian Worker<br/>scanner + exporter"]
  end

  subgraph CF["Cloudflare Worker"]
    Ingest["POST /api/ng/ingest<br/>+/batch"]
    Triage["GET /api/ng/findings<br/>detail, convert"]
    CVE["CVE via BLT-API"]
    Events["Verified events webhook"]
  end

  subgraph Storage["D1"]
    D1[(sender_keys, envelopes,<br/>findings, evidence_meta,<br/>events_outbox, access_logs)]
  end

  subgraph Frontend["GitHub Pages"]
    SPA["Triage SPA"]
  end

  subgraph External["External"]
    BLTAPI["BLT-API<br/>CVE + Issue creation"]
    Down["Downstream (Rewards, RepoTrust)"]
  end

  FC -->|ztr-finding-1| Ingest
  WX -->|ztr-finding-1| Ingest
  Ingest --> D1
  Triage --> D1
  CVE --> BLTAPI
  Triage --> CVE
  SPA -->|HTTPS| Triage
  Triage --> BLTAPI
  Events --> Down
```

**2.1 Stack overview**

- **Backend:** Cloudflare Worker (Python) + Cloudflare D1 (SQLite-based) for storage (no VPS; Cloudflare Cron Triggers for housekeeping).
- **Frontend:** GitHub Pages (static SPA) that calls Worker JSON APIs over CORS-controlled HTTPS (no secrets on the client).
- **External services:** BLT-API for CVE normalization/score and Issue creation.
- No new Django endpoints, no PostgreSQL models, no Celery/queue infra.

**2.2 Responsibilities split**

- **Cloudflare Worker**
  - Ingestion API: POST /api/ng/ingest (+/batch) — verify signature, skew, size; dedupe by fingerprint; persist to D1; throttle under load.
  - Triage APIs: list/detail finders; server-side decrypt; evidence access audit log; "Convert to Issue" → BLT-API.
  - CVE plumbing: call BLT-API normalize_cve_id/get_cve_score; store cve_id/cve_score on Finding.
  - Outbox/webhooks: HMAC-signed verified events with retries and dedupe_key idempotency.
  - Auth: GitHub OAuth + PKCE (or Cloudflare Access) → session cookie; org-scoped authorization. X-BLT-Timestamp is advisory for logs/rate-limit; issued_at in the signed envelope governs expiry.
- **GitHub Pages SPA**
  - Triage UI: list/filter/detail, CSV export, "Convert to Issue" button; all calls go to Worker APIs.
  - No local secrets, no direct DB/BLT-API calls, no decryption on client.
- **BLT-API**
  - CVE normalization and CVSS score; Issue creation on "Convert to Issue".
  - Optional: org/user membership check for triage authorization.

**2.3 Non-goals (explicit)**

- No new Django/DRF endpoints or templates in BLT main repo.
- No PostgreSQL migrations/models for NetGuardian.
- No heavy server-side PDF pipeline by default (CSV required; PDF optional/timeboxed).
- Never log ciphertext/plaintext; never expose raw evidence to the SPA.

**2.4 Key files/endpoints (conceptual)**

- Worker endpoints: /api/ng/ingest, /api/ng/ingest/batch, /api/ng/findings, /api/ng/findings/{id}, /api/ng/findings/{id}/convert
- D1 tables: sender_keys, envelopes, findings, evidence_meta, events_outbox, access_logs

**2.5 Ingestion endpoints (Worker)**

- **Headers**
  - Content-Type: application/json
  - X-BLT-Signature: sha256=\<hex\>
  - X-BLT-Timestamp: \<unix_ts\>
- **Endpoints and responses**
  - **POST /api/ng/ingest**
  - 201 Created: `{ finding_id, status: "created"|"merged", evidence_id, replay: false }`
  - 200 OK (duplicate): `{ status: "duplicate", replay: true }`
  - 400 Bad Request: `{ error: "clock_skew"|"digest_mismatch"|"bad_sig"|"invalid_envelope" }`
  - 401 Unauthorized
  - 413 Payload Too Large
  - 429 Too Many Requests (includes Retry-After)
- **POST /api/ng/ingest/batch**
  - 200 OK with per-item array:
  - `[{ index, status: "created"|"merged"|"duplicate"|"error", finding_id?, evidence_id?, error_code? }, …]`
  - Clients retry only items with status="error".

**2.6 Webhook headers (Worker to downstream)**

- X-BLT-Webhook-Signature: sha256=\<hex\>
- X-BLT-Webhook-Timestamp: \<unix_ts\>

**2.7 D1 schema essentials**

- **sender_keys**(org_id TEXT, sender_id TEXT, kid TEXT, alg TEXT, secret_or_pubkey BLOB, active INT, rotated_at TEXT, created_at TEXT)
  - UNIQUE(org_id, sender_id, kid)

- **envelopes**(org_id TEXT, sender_id TEXT, kid TEXT, alg TEXT, nonce TEXT, issued_at TEXT, received_at TEXT, payload_digest TEXT, signature BLOB, validated_at TEXT, error_code TEXT)
  - UNIQUE(sender_id, nonce)
  - INDEX(issued_at, sender_id), INDEX(org_id, received_at)

- **evidence_meta**(digest_sha256 TEXT PRIMARY KEY, media_type TEXT, size_bytes INTEGER, pointer TEXT, created_at TEXT)

- **findings**(org_id TEXT, rule_id TEXT, severity TEXT, target_url TEXT, selector TEXT, fingerprint TEXT, cve_id TEXT, cve_score REAL, evidence_digest_sha256 TEXT, latest_evidence_pointer TEXT, confidence REAL, false_positive INT, reconfirmed INT, converted_issue_ref TEXT, created_at TEXT, updated_at TEXT)
  - UNIQUE(rule_id, target_url, selector, evidence_digest_sha256)
  - INDEX(org_id, created_at), INDEX(org_id, severity), INDEX(org_id, cve_id), INDEX(fingerprint)

- **events_outbox**(event_type TEXT, payload_json TEXT, dedupe_key TEXT UNIQUE, status TEXT, attempt_count INTEGER, next_attempt_at TEXT, signature_hmac TEXT, created_at TEXT, sent_at TEXT)
  - INDEX(status, next_attempt_at)

- **access_logs**(finding_id TEXT, user_id TEXT, action TEXT, at TEXT, ip_hash TEXT, user_agent_hash TEXT)
  - INDEX(finding_id, at)

**2.8 Reports**

- CSV export: required; snapshot tests ensure no plaintext secrets leak.
- PDF export: optional/timeboxed (0.5–1 week). If included, render in Worker and preserve the same redaction guarantees as CSV.

**2.9 Companion (optional; not core to GSoC): Flutter Desktop Client**

- Focused local runner (Windows/macOS/Linux) that builds/signs ztr-finding-1 envelopes and POSTs to Worker ingest; offline queue + retry; small local history; integrates via the same contract. Not required for v1 GSoC milestones.

---

## 3. Security invariants (ztr-finding-1)

**3.1 Required fields (in the signed JSON)**

- version: "ztr-finding-1"
- sender_id: string
- kid: Key ID string (for rotation)
- alg: MUST be "hmac-sha256" for v1 (Ed25519 may be added post-v1)
- issued_at: RFC 3339 UTC timestamp
- nonce: unique per sender_id
- payload_digest: hex(SHA-256(payload_bytes))
- signature: base64
- Exactly one payload:
  - payload_ciphertext: base64, OR
  - payload_plaintext: JSON object with plaintext_mode=true
- payload_digest must be computed over the exact bytes sent (ciphertext bytes, or the UTF-8 canonical JSON of payload_plaintext).

**3.2 Canonicalization and signing**

- Compute canonical JSON with signature field omitted (RFC 8785-style key ordering; UTF-8; no extra whitespace).
- Example (Python): `json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False).encode("utf-8")`
- HMAC-SHA256 over canonical JSON; compare with timing-safe method.

**3.3 Replay and freshness**

- Enforce ±5 minutes clock skew on issued_at.
- DB uniqueness on (sender_id, nonce); duplicates return 200 with status="duplicate".

**3.4 Caps, logging, and privacy**

- Max envelope body size: 1 MiB (1,048,576 bytes) by default (org-configurable); 413 on overflow.
- Store encrypted evidence bytes at rest (or pointer), plus digest and size.
- Evidence encryption: AES-GCM with a Worker-managed app key (rotatable); only digests/sizes or pointers are exposed to the SPA; every decrypt is audited.
- Never log ciphertext or plaintext; redact long/sensitive fields in logs/templates.
- Evidence access is server-side and always audited (access_logs).

**3.5 Nonce guidance**

- MUST be unique per sender_id. Recommended format: "<unix_ts>-" (ordering not required; uniqueness is).

**3.6 Permissions and scoping**

- All Finding queries are org-scoped; SPA calls must include an authenticated session; Convert-to-Issue checks org ownership; rate limits/quotas enforced per org.

---

## 4. 12-week implementation plan (Worker + D1 + GH Pages + BLT-API)

| Week | Focus |
|------|--------|
| 1 | Spec + D1 schema + scaffolding |
| 2 | Ingestion verification, replay, caps |
| 3 | Auth + triage SPA + server-side decrypt + audit |
| 4 | CVE plumbing (BLT-API) + CVE filters |
| 5 | Dedup/idempotency + CSV export (baseline) |
| 6 | Triage polish + RFIs + midterm E2E |
| 7 | Fidelity fixtures & acceptance gates |
| 8 | Consensus for criticals + quotas/back-pressure |
| 9 | Remediation & insights (static) |
| 10 | Disclosure helpers & reports |
| 11 | Verified events & minimal events API |
| 12 | Hardening, docs, pilot, v1.0 |

**Week 1 — Spec + D1 schema + scaffolding**

- ztr-finding-1 spec finalized (fields, canonicalization, caps, errors); D1 schema (sender_keys, envelopes, evidence_meta, findings, events_outbox, access_logs) with unique indexes; Worker project + CI skeleton; canonicalization/digest utilities and tests.

**Week 2 — Ingestion verification, replay, caps**

- Implement POST /api/ng/ingest (+/batch): signature verify, ±5 min skew, 1 MiB cap, DB uniqueness on (sender_id, nonce) with idempotent duplicate path; consistent error codes and JSON responses; property tests.

**Week 3 — Auth + triage SPA + server-side decrypt + audit**

- GitHub OAuth (Auth Code + PKCE) in Worker; SPA login; /api/ng/findings (list with filters/paging) and /api/ng/findings/{id} (redacted detail + server-side decrypted snippet); access_logs on evidence view; org-scoped permission checks.

**Week 4 — CVE plumbing (BLT-API) + CVE filters**

- Integrate BLT-API normalize_cve_id/get_cve_score; store cve_id/cve_score on Finding; extend filters (cve_id, cve_score_min/max); SPA CVE filters, Related CVEs panel; mapping tests.

**Week 5 — Dedup/idempotency + CSV export (baseline)**

- Enforce fingerprint uniqueness (rule_id, target_url, selector?, evidence_digest) and upsert semantics; concurrency/race tests; Worker CSV export endpoint with redaction; SPA "Export CSV" button; snapshot tests.

**Week 6 — Triage polish + RFIs + midterm E2E**

- Evidence viewer improvements; canned RFI markdown fragments safely rendered; midterm end-to-end demo: login → signed ingestion → Finding in D1 → triage list/detail → server-side decrypt → Convert-to-Issue (via BLT-API) → verified event queued → CSV export.

**Week 7 — Fidelity fixtures & acceptance gates**

- 5–8 curated fixtures; ingestion and CVE enrichment metrics persisted; acceptance thresholds enforced in CI (≥95% ingestion success; ≥90% CVE match on curated set); docs on fixtures and regression.

**Week 8 — Consensus for criticals + quotas/back-pressure**

- Consensus gate for criticals (2+ corroborating signals) before auto-convert; confidence scoring updated; per-org/hour quotas; 429 with Retry-After; SPA handles back-pressure gracefully; tests.

**Week 9 — Remediation & insights (static)**

- Rule-mapped remediation fragments with OWASP links; "why this matters" callouts; SPA renders safe static content; mapping and XSS-safety tests.

**Week 10 — Disclosure helpers & reports**

- security.txt detection integrated into Convert-to-Issue and reports; finalize CSV; PDF optional/timeboxed (0.5–1 week) with same redaction rules—defer if unstable.

**Week 11 — Verified events & minimal events API**

- events_outbox insert on convert/resolution; webhook delivery (HMAC headers, retries, dedupe_key idempotency); read-only /api/ng/events for consumers; signature/retry/idempotency tests; example consumers/docs.

**Week 12 — Hardening, docs, pilot, v1.0**

- Security review: key handling, AES-GCM usage, nonce uniqueness, cache-poisoning resistance, permission paths, log redaction; Cron Triggers for retries/cleanup; WAF/Rate-Limit tuning; pilot with 1–2 orgs; fix high-priority feedback; runbook + rollback/data deletion docs; tag v1.0.

---

## 5. Flexibility and change management

- Serverless-first isolation: Worker modules (ingestion, triage, CVE, events) are thin and test-covered; swapping implementations (e.g. batch code paths or events retry backoff) does not ripple across SPA or schema.
- All externally visible contracts (envelope ztr-finding-1, ingest/webhook headers, events payload) are versioned; tests enforce invariants.
- Any non-trivial change ships with a short ADR-style note (what, why, alternatives, impact).

---

## 6. Testing strategy

**Envelope/ingestion**

- Valid/expired/future/replayed; duplicate idempotency; signature mismatch; canonicalization stability; 1 MiB cap; consistent error codes; no plaintext in logs (redaction).

**Dedup/idempotency**

- Same fingerprint collapses; variant evidence_digest creates new attachment; concurrent races collapse correctly.

**Triage and permissions**

- Org scoping and leakage negatives; pagination/sort; evidence access always audited; CVE filter correctness; cache-poisoning resistance.

**Consensus and resilience**

- Critical reconfirmation gate; quotas; 429 Retry-After honored end-to-end.

**Reports**

- CSV snapshot tests: no plaintext secrets; (optional) PDF snapshot tests with same redaction guarantees.

**Metrics/fidelity**

- Worker→BLT ingestion success and CVE match thresholds enforced in CI.

---

## 7. Milestone checkpoints

**Midterm (end of Week 6)**

- E2E demo: signed ingestion → D1 Finding → CVE-aware triage list/detail → server-side decrypt (audited) → Convert-to-Issue (BLT-API) → verified event queued → CSV export.

**Final (end of Week 12)**

- Productionized Worker + D1 + SPA; verified events/webhooks; curated metrics; pilot feedback applied; v1.0 tagged; docs/runbooks shipped.

---

## 8. Cross-project integration

**BLT-API**

- normalize_cve_id/get_cve_score and Issue creation; optional org/user membership check.

**Downstream consumers**

- HMAC-signed verified events (versioned, with dedupe_key) + minimal read-only events API for polling; example consumers for BLT-Rewards/RepoTrust included in docs; do not embed downstream logic in NetGuardian.

**Webhook contract**

- Both the envelope and event payloads carry a version field and a dedupe_key for idempotent consumption. Webhook HMAC header: X-BLT-Webhook-Signature: sha256=\<hex\>; X-BLT-Webhook-Timestamp: \<unix_ts\>.

---

## 9. AI tooling and safety

**9.1 IDE (fixed) and models (tentative)**

- IDE: Cursor.
- Strong reasoning (architecture, threat modeling, tricky refactors): Claude Opus 4.5.
- Fast edits and boilerplate: Claude Sonnet 4.5.
- Second opinion on security-sensitive code and docs: GPT-5.2.

**9.2 How AI is used across phases**

- Claude Opus 4.5: design-heavy phases (ztr-finding-1 spec, reconfirmation gate tradeoffs, events payload shape, midterm failure modes).
- Claude Sonnet 4.5: repetitive scaffolding (Worker handlers, SPA filters/UI, D1 queries, UX copy, RFI prose, fixtures, rate-limit tests, remediation fragments, CSV/PDF snapshot tests, events docs, checklists, runbooks, final summary).
- GPT-5.2: security review of critical paths (secrets/failure behavior in exporter, timestamp/nonce/signature and error paths, security-impacting patches).

**9.3 Guardrails (AI safety and usage policy)**

- Security-critical code (envelope verification, canonicalization, signing, nonce handling, server-side decrypt, permission checks) is hand-written from specs and covered by tests. No unreviewed generation in these paths.
- AI assistance is limited to boilerplate, documentation, and fixtures; all changes pass code review and test gates.
- Invariants are codified in tests; refactors must preserve them.
