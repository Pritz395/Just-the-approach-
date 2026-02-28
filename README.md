# NetGuardian — Technical approach & weekly deliverables (GSoC 2026)

> **Note for maintainers:** This is the technical part only. I'll add the SAT/personal sections (bio, coding skills, time commitment, etc.) once you're happy with this approach.

> **Scope note:** This GSoC scope does not add new Django endpoints, templates, or PostgreSQL models to the BLT main repo. All NetGuardian backend logic runs in Cloudflare Workers with D1, and the triage UI is a static app on GitHub Pages consuming Worker JSON APIs. CVE data and Issue creation are delegated to BLT-API.

---

## 1. Introduction

This project extends work I've already contributed to OWASP BLT and builds on the existing BLT-NetGuardian Worker (a Cloudflare Python Worker for autonomous discovery and scanning) to deliver a zero-trust ingestion path for security findings, CVE-aware triage, and verified events for downstream systems (Rewards, RepoTrust, University). The implementation runs on **Cloudflare Workers + D1 + GitHub Pages**, with CVE normalization and Issue creation delegated to **BLT-API**; no new Django endpoints or PostgreSQL models are in scope.

**Relationship to the existing BLT-NetGuardian Worker**

The Worker discovers targets (CT logs, GitHub API, blockchain) and runs security scanners (Web2, Web3, static, contract). Results stage in Cloudflare KV. This GSoC project connects that pipeline to a dedicated **ingestion and triage backend** (also on Cloudflare) by:

- Adding a BLT exporter in the Worker that converts scan results to signed ztr-finding-1 envelopes and POSTs to the NetGuardian ingestion Worker.
- Implementing ingestion and triage in **Cloudflare Workers + D1**: verify envelope, replay protection, dedupe by fingerprint, CVE enrichment via BLT-API, and HMAC-signed verified events.
- Delivering a **triage UI as a static SPA on GitHub Pages** that calls Worker JSON APIs (no secrets on the client).

NetGuardian may also be used by a Flutter desktop client or other agents that send signed ztr-finding-1 envelopes to the same ingestion endpoint; the contract is designed so those can integrate in parallel.

**Prerequisites verified**

- Cloudflare Workers CLI (wrangler), D1 bindings, and Python Worker dev environment.
- Access to the BLT-NetGuardian Worker repo and BLT-API (for CVE and Issue creation).
- Static site hosting (e.g. GitHub Pages) for the triage SPA.

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

- **Backend:** Cloudflare Worker (Python) + Cloudflare D1 (SQLite-based) for storage.
- **Frontend:** GitHub Pages (static SPA) that calls Worker JSON APIs (no secrets on the client).
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

- MUST be unique per sender_id. Recommended format: "<unix_ts>-<random>" (ordering not required; uniqueness is).

---

## 4. 12-week implementation plan (Worker + D1 + GH Pages + BLT-API)

| Week | Focus (phases) |
|------|----------------|
| 1 | Phases 1–2: ztr-finding-1 spec; D1 schema + unique indexes; ingest skeleton |
| 2 | Phase 2: ingest verification (sig/skew/size), replay/idempotency, property tests |
| 3 | Phase 4: Triage SPA (GH Pages) + Worker read APIs; server-side decrypt; audit logs; Convert to Issue |
| 4 | Phases 5–6: CVE via BLT-API (normalize/score) + dedup/fingerprint; filters in SPA |
| 5 | Phase 7 + Phase 8 start: CVE-aware triage UX + polish; CSV export scaffold |
| 6 | Phase 8: triage polish, RFIs, midterm E2E (Worker ingest → D1 → SPA → BLT-API Issue → verified event) |
| 7 | Phase 9: Worker→BLT fidelity fixtures & acceptance gates (≥95% ingest success; ≥90% CVE match) |
| 8 | Phase 10: consensus for criticals; quotas/back-pressure (429 + Retry-After) |
| 9 | Phase 11: remediation fragments (static + OWASP links) in SPA |
| 10 | Phase 12: CSV export (required); PDF optional/timeboxed (if added, via Worker) |
| 11 | Phase 13: verified events/webhooks + minimal read-only events API (Worker) |
| 12 | Phases 14–16: hardening, docs, pilot, v1.0 tag |

---

## 5. Flexibility and change management

NetGuardian is designed to stay flexible for maintainers and the community. If we collectively decide mid-project that a different library or pattern makes more sense, I'll isolate third-party dependencies behind small wrappers so swapping them doesn't ripple through callers. Any non-trivial change gets a short Markdown note first (what, why, alternatives, schedule impact) before I touch anything. The acceptance criteria in this proposal (ingestion invariants, fidelity gates, verified-event schema) act as contracts; if we refactor, the tests stay the same and still have to pass.

---

## 6. Testing strategy

The ingestion tests cover the full envelope lifecycle: valid submission, expired timestamp, future-issued, and replay, with the replay case exercised via D1 uniqueness so it holds under concurrent load. Signature mismatch and canonicalization drift are tested separately. Size and MIME cap enforcement, filename sanitization, and access logging on evidence reads each get their own cases.

For the exporter (in the BLT-NetGuardian Worker), the main test is the full scan → KV → exporter → NetGuardian ingest Worker → D1 path. Beyond that: retry behavior when the ingest Worker is down, idempotency via nonce reuse, and malformed scan result handling.

Dedup tests focus on the fingerprint: the same fingerprint should collapse into one Finding, and concurrent submissions of the same fingerprint should collapse cleanly rather than racing to create duplicates.

On the triage side: pagination boundaries, org permission leakage (Org A user cannot see Org B findings), CVE filter correctness via BLT-API responses, and basic cache-poisoning resistance. For consensus and resilience: the critical reconfirmation gate and 429/Retry-After behavior under load. For reports: CSV (and PDF if implemented) snapshot tests that confirm sensitive evidence does not appear in plain text output.

---

## 7. Milestone checkpoints

- **Midterm (after Week 6):** E2E demo — Worker ingest → D1 → SPA triage → server-side decrypt → "Convert to Issue" via BLT-API → verified event.
- **Final (after Week 12):** Full Worker + D1 + GH Pages pipeline live, verified-events webhook, fidelity metrics, and pilot feedback.

---

## 8. Cross-project integration

NetGuardian emits a signed webhook for Verified Events with a stable, versioned schema. I'll write concrete consumption examples specifically for BLT-Rewards and RepoTrust, not just the schema, but working examples so they're not guessing at edge cases. NetGuardian stops at emitting clean events; downstream scoring, gamification, and education logic are out of scope and should not live here.

Both the envelope and event payloads carry a version field and a dedupe_key for idempotent consumption. Webhook HMAC header: X-BLT-Webhook-Signature: sha256=\<hex\>; X-BLT-Webhook-Timestamp: \<unix_ts\>.

---

## 9. AI tooling

**9.1 IDE (fixed) and models (tentative)**

- IDE: Cursor.
- Strong reasoning (architecture, threat modeling, tricky refactors): Claude Opus 4.5, used when design choices and edge cases matter.
- Fast edits and boilerplate: Claude Sonnet 4.5, used for inline edits, Worker handlers, serialization, docstrings, type hints, and repetitive scaffolding.
- Second opinion on security-sensitive code and docs: GPT-5.2, used to double-check ingestion, signing, nonce handling, permissions, and evidence redaction.

**9.2 How AI is used across phases**

Claude Opus 4.5 is used for the design-heavy phases: locking the ztr-finding-1 spec (Phase 1), reconfirmation gate tradeoffs (Phase 10), payload shape for the downstream event schema (Phase 13), and failure modes in the midterm test plan (Phase 8).

Claude Sonnet 4.5 handles high-volume repetitive work: Worker request/response handling (Phase 2), ScanResult-to-envelope mapping and retry utilities (Phase 3), SPA UI and filter wiring (Phase 4), D1 queries and tests (Phases 5–6), UX copy and filter labels (Phase 7), RFI template prose (Phase 8), fixture generation (Phase 9), rate-limit test structure (Phase 10), remediation fragments (Phase 11), export and snapshot tests (Phase 12), events schema and docs (Phase 13), security review checklists (Phase 14), runbook and pilot drafts (Phase 15), and the final GSoC report summary (Phase 16).

GPT-5.2 is reserved for security review of critical paths: exporter secret handling and failure behavior (Phase 3), timestamp/nonce/signature and error paths in the ingestion Worker (Phase 2), security-critical changes before final merge (Phase 14), and patches with security implications before the v1.0 tag (Phase 16).

**9.3 Guardrails**

Every security-critical path (ingestion, signing, nonce handling, permissions, evidence redaction) gets hand-reviewed and test-covered before it merges. AI is useful for drafts and suggestions, but nothing ships that isn't fully understood. Canonical JSON and HMAC signing are implemented explicitly; verification logic, replay check, and size caps are written directly rather than generated. AI-assisted security-sensitive code gets a GPT-5.2 review before merge as a second set of eyes, not a substitute for the primary review.
