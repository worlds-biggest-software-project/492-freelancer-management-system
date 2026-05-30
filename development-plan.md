# Freelancer Management System — Phased Development Plan

> Project: 492-freelancer-management-system · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four
`data-model-suggestion-*.md` files. The database design adopts **Suggestion 3 (Hybrid Relational +
JSONB on PostgreSQL)** as the foundation — it preserves ACID integrity for the financial/tax core
while absorbing per-jurisdiction compliance variation in JSONB without migrations. The **outbox +
immutable audit-event patterns from Suggestion 2** are layered onto payments, tax filings, and
webhooks where a tamper-evident trail is legally required.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The differentiating value (AI classification, 1099 reconciliation, anomaly detection, spend forecasting) is ML/LLM-heavy. Python has the strongest ecosystem for this (scikit-learn, pandas, the OpenAI/Anthropic SDKs) and first-class IRS/tax tooling. |
| API framework | FastAPI | Generates OpenAPI 3.x natively (standards.md requires OpenAPI as a deliverable), async-first for webhook/payment I/O, Pydantic validation maps directly onto JSON Schema Draft 2020-12 contractor/invoice models. |
| ASGI server | Uvicorn (dev) + Gunicorn/Uvicorn workers (prod) | Standard, production-proven FastAPI deployment. |
| Database | PostgreSQL 16 | Hybrid relational+JSONB model needs JSONB + GIN indexing, `TIMESTAMPTZ`, partial indexes, `pgcrypto`, and ACID for payments/tax. SQLite is insufficient for JSONB GIN and concurrent payment workloads. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM; Alembic gives reviewable, reversible migrations required for compliance auditability. |
| Validation models | Pydantic v2 | Single source of truth for request/response schemas and the portable contractor profile (HR Open + JSON Schema). |
| Task queue | Celery + Redis broker | Webhooks, payment dispatch, IRS filing, TIN matching, anomaly scoring, and forecasting are async/long-running and must retry. Redis doubles as cache + idempotency store. |
| Outbox dispatcher | Celery beat poller over `event_outbox` | Reliable at-least-once webhook + integration delivery (Suggestion 2 outbox pattern) without adding Kafka. |
| Frontend | Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui | Two distinct UIs (admin dashboard + contractor self-service portal, mobile-first per features.md). Server components reduce PII exposure; consumes the same OpenAPI spec via a generated client. |
| Auth | OAuth 2.1 (Authorization Code + PKCE) + OIDC, JWT access tokens | standards.md mandates OAuth 2.1 patterns; OIDC enables enterprise SSO (Okta/Entra/Google). Authlib server-side. |
| Identity provisioning | SCIM 2.0 (RFC 7644) | Enterprise contractor identity lifecycle (Deel/Remote parity). |
| Object storage | S3-compatible (AWS S3 / MinIO self-hosted) | Documents (W-9, contracts, IDs) need encrypted-at-rest blob storage, not the DB. |
| Field encryption | `pgcrypto` + application-layer AES-GCM via `cryptography` | SSN/TIN, bank account numbers, integration credentials must be encrypted (ISO 27701, GDPR/CCPA). |
| LLM provider | Pluggable provider abstraction (OpenAI + Anthropic), default configurable | Classification/compliance summarisation/talent matching use LLMs; self-hosters may swap providers. |
| e-signature | Pluggable adapter (DocuSign + native fallback signer) | Contracts/NDAs/IP agreements need e-sign; native fallback keeps the system self-hostable with no SaaS dependency. |
| Payments | Pluggable rails adapter: ACH (NACHA), wire/SWIFT, multi-currency | Avoids lock-in; NACHA + ISO 20022 pain.001 mapping per standards.md. |
| Tax filing | IRS IRIS API adapter + third-party fallback (Tax1099/TaxBandits) | FIRE retires 2026-12-31; IRIS is mandatory. Third-party adapter sidesteps the IRS TCC redistribution constraint noted in features.md. |
| Testing | pytest + pytest-asyncio + httpx test client; Playwright for frontend e2e | Standard Python stack; Playwright covers the two-portal e2e flows. |
| Code quality | Ruff (lint+format), mypy (strict), pre-commit | Enforced quality gate. |
| Packaging | uv + pyproject.toml (backend); pnpm (frontend) | Fast, reproducible. |
| Containerisation | Docker + docker-compose | Self-hosted is a primary deployment mode (README). compose wires Postgres, Redis, MinIO, API, worker, frontend. |
| Observability | structlog (JSON logs) + OpenTelemetry traces + Prometheus metrics | Required for payment/compliance incident response (ISO 27001). |

### Project Structure

```
freelancer-management-system/
├── pyproject.toml
├── uv.lock
├── Dockerfile                      # API + worker image
├── docker-compose.yml              # postgres, redis, minio, api, worker, beat, frontend
├── .env.example
├── alembic.ini
├── README.md
├── openapi/                        # exported OpenAPI 3.x spec (CI artifact)
│   └── fms-openapi.json
├── migrations/                     # Alembic revisions
│   └── versions/
├── src/fms/
│   ├── main.py                     # FastAPI app factory, router registration
│   ├── config.py                   # Pydantic Settings (env-driven)
│   ├── db/
│   │   ├── session.py              # async engine/session
│   │   ├── base.py                 # DeclarativeBase, mixins (UUID PK, timestamps, org scope)
│   │   └── models/                 # SQLAlchemy models grouped by domain
│   │       ├── organization.py
│   │       ├── contractor.py
│   │       ├── document.py
│   │       ├── contract.py
│   │       ├── project.py
│   │       ├── invoice.py
│   │       ├── payment.py
│   │       ├── tax.py
│   │       ├── compliance.py
│   │       ├── classification.py
│   │       ├── integration.py
│   │       ├── webhook.py
│   │       └── audit.py
│   ├── schemas/                    # Pydantic request/response + portable profile
│   │   └── ...
│   ├── api/
│   │   ├── deps.py                 # auth, org-scope, pagination, RBAC dependencies
│   │   ├── errors.py               # RFC 7807 problem+json handlers
│   │   └── routes/
│   │       ├── auth.py  contractors.py  documents.py  contracts.py
│   │       ├── projects.py  timesheets.py  invoices.py  payments.py
│   │       ├── tax.py  compliance.py  classification.py  talent.py
│   │       ├── integrations.py  webhooks.py  scim.py
│   ├── services/                   # business logic (framework-agnostic)
│   │   ├── onboarding.py  contract_gen.py  approvals.py
│   │   ├── invoicing.py  payments.py  reconciliation.py
│   │   ├── tax_filing.py  compliance_rules.py
│   ├── ai/
│   │   ├── provider.py             # LLM provider abstraction
│   │   ├── classification.py       # 25+ factor worker-classification engine
│   │   ├── anomaly.py              # invoice anomaly detection
│   │   ├── forecasting.py          # spend forecasting
│   │   └── talent_match.py         # talent-pool ranking
│   ├── integrations/
│   │   ├── esign/                  # docusign.py, native.py, base.py
│   │   ├── payments/               # ach_nacha.py, wire.py, base.py
│   │   ├── tax/                    # iris.py, tax1099.py, base.py
│   │   ├── tin_matching.py
│   │   ├── background_check.py
│   │   └── accounting/             # quickbooks.py, xero.py, freshbooks.py, base.py
│   ├── events/
│   │   ├── audit.py                # append-only audit + domain-event recorder
│   │   ├── outbox.py               # outbox writer + dispatcher
│   │   └── webhooks.py             # signed delivery, retry, HMAC
│   ├── security/
│   │   ├── crypto.py               # field-level AES-GCM helpers
│   │   ├── oauth.py                # OAuth2.1/OIDC, PKCE, JWT issue/verify
│   │   └── rbac.py
│   ├── workers/
│   │   ├── celery_app.py
│   │   └── tasks.py
│   └── storage/blob.py             # S3/MinIO adapter
├── tests/
│   ├── conftest.py                 # db fixture (testcontainers/pg), client, factories
│   ├── unit/  integration/  e2e/  fixtures/
└── frontend/
    ├── package.json
    ├── app/                        # Next.js app router
    │   ├── (admin)/                # dashboard
    │   └── (portal)/               # contractor self-service
    ├── lib/api-client/             # generated from openapi/fms-openapi.json
    └── tests/                      # Playwright
```

---

## Phase 1: Foundation — Project Skeleton, Config, Database Core, Auth

### Purpose
Establish a runnable, tested skeleton: app factory, configuration, the async DB layer with the
multi-tenant base model, the hybrid relational+JSONB conventions, RFC 7807 error handling, and the
OAuth 2.1 auth foundation. Everything later depends on this; it must be small but solid.

### Tasks

#### 1.1 — Project scaffold, config, and quality gate

**What**: Create the Python package, `pyproject.toml`, Docker/compose, and CI-enforced quality tooling.

**Design**:
- `pyproject.toml` declares deps (fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, authlib, celery[redis], redis, cryptography, structlog, boto3, httpx, pytest, pytest-asyncio, ruff, mypy).
- `config.py` — Pydantic `Settings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://localhost:6379/0"
      jwt_secret: str
      jwt_access_ttl_seconds: int = 900          # 15 min (standards.md JWT guidance)
      jwt_refresh_ttl_seconds: int = 1209600     # 14 days
      field_encryption_key: str                  # base64 32-byte AES-256 key
      s3_endpoint: str | None = None
      s3_bucket: str = "fms-documents"
      llm_provider: Literal["openai", "anthropic", "disabled"] = "disabled"
      llm_api_key: str | None = None
      environment: Literal["dev", "test", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="FMS_", env_file=".env")
  ```
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `minio`, `api`, `worker`, `beat`, `frontend`.
- `.pre-commit-config.yaml` runs Ruff + mypy --strict.

**Testing**:
- `Unit: Settings loads from env with FMS_ prefix → correct typed values, defaults applied.`
- `Unit: missing required FMS_DATABASE_URL → ValidationError naming the field.`
- `Integration: docker-compose up postgres+redis+minio → all healthchecks pass.`
- `CI: ruff check, ruff format --check, mypy --strict all exit 0.`

#### 1.2 — Async DB layer, base model, and tenancy mixins

**What**: SQLAlchemy 2.0 async engine/session plus reusable base mixins enforcing the hybrid model conventions.

**Design**:
- `db/base.py`:
  ```python
  class Base(DeclarativeBase): ...
  class UUIDPK:        id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
  class Timestamps:    created_at: Mapped[datetime]; updated_at: Mapped[datetime]   # server_default now()
  class OrgScoped:     organization_id: Mapped[UUID] = mapped_column(ForeignKey("organizations.id"), index=True)
  ```
- Convention: every flexible/jurisdiction-varying attribute is a `JSONB` column (`metadata_`, `compliance_data`, `config`) with a GIN index where queried.
- `db/session.py`: `async_sessionmaker`; `get_session()` FastAPI dependency yielding a transaction-scoped session.

**Testing**:
- `Integration (real pg): create a model using all mixins → row has UUID PK, server-set timestamps.`
- `Unit: JSONB column round-trips a nested dict unchanged.`
- `Integration: GIN index present on a JSONB column (query pg_indexes).`

#### 1.3 — RFC 7807 error handling + pagination (RFC 8288)

**What**: Standardised machine-readable errors and Link-header pagination shared by all routes.

**Design**:
- `api/errors.py`: exception handlers emitting `application/problem+json`:
  ```json
  {"type":"https://fms/errors/validation","title":"Validation failed","status":422,
   "detail":"line_items[0].amount must be > 0","instance":"/v1/invoices","errors":[...]}
  ```
- Distinct `type` URIs for `validation`, `compliance_failure`, `payment_rejected`, `authz`, `not_found`, `conflict`.
- `api/deps.py` `Paginator`: cursor pagination, emits `Link: <...>; rel="next"` header.

**Testing**:
- `Unit: ValidationError → 422 problem+json with errors[] and correct type URI.`
- `Unit: unknown id → 404 problem+json type=not_found.`
- `Integration: list endpoint with >page_size rows → Link rel=next present and resolvable.`

#### 1.4 — OAuth 2.1 / OIDC auth, JWT, RBAC

**What**: Token issuance/verification, Authorization Code + PKCE for users, Client Credentials for service-to-service, role-based access.

**Design**:
- Organizations + users tables (from Suggestion 1, lines 22–49) with roles `admin|manager|finance|hr|viewer`.
- `security/oauth.py`: `/oauth/authorize`, `/oauth/token` (PKCE), refresh-token rotation; JWT claims `{sub, org, role, scope, exp}`.
- `security/rbac.py`: `require_role(*roles)` and `require_scope(*scopes)` dependencies.
- Contractors authenticate to the portal via a separate audience (`aud=portal`) scoped only to their own records (OWASP API #1 BOLA defence — every query filtered by `organization_id` + `contractor_id`).

**Testing**:
- `Integration: Authorization Code + PKCE happy path → access+refresh JWT, valid signature/claims.`
- `Unit: expired token → 401 problem+json.`
- `Unit: finance role calling admin-only route → 403.`
- `Security: contractor token requesting another contractor's record → 404 (no existence leak), audited.`
- `Unit: reused refresh token → 401 and family revoked (rotation).`

---

## Phase 2: Contractor Records, Documents & e-Signature

### Purpose
Build the contractor profile (the portable, HR-Open-aligned core entity), encrypted PII handling,
the document store, and the e-signature abstraction. This is the data spine onboarding/contracts/
invoicing all attach to.

### Tasks

#### 2.1 — Contractor profile & encrypted PII

**What**: Contractor CRUD with field-level encryption for tax IDs and a JSON-Schema-validated portable profile.

**Design**:
- Model merges Suggestion 1 `contractors` (lines 55–80) with a Suggestion 3 `compliance_data JSONB`:
  ```python
  class Contractor(Base, UUIDPK, OrgScoped, Timestamps):
      email; first_name; last_name; phone; country_code
      tax_id_encrypted: Mapped[bytes | None]     # AES-GCM via security/crypto
      tax_id_last4: Mapped[str | None]           # for display/search
      tax_form_type: Mapped[str | None]          # W-9 | W-8BEN | W-8BEN-E
      tax_id_verified: Mapped[bool] = False
      classification: Mapped[str] = "pending"
      onboarding_status: Mapped[str] = "invited"
      compliance_data: Mapped[dict] = mapped_column(JSONB, default=dict)  # IR35/URSSAF/etc.
      __table_args__ = (UniqueConstraint("organization_id", "email"),)
  ```
- `schemas/contractor.py` Pydantic `ContractorProfile` doubles as the exported JSON Schema 2020-12 portable-profile (HR Open worker model alignment).
- `security/crypto.py`: `encrypt_field(plaintext) -> bytes`, `decrypt_field(ct) -> str` (AES-256-GCM, key from settings).

**Testing**:
- `Unit: encrypt_field then decrypt_field → original; ciphertext differs each call (random nonce).`
- `Unit: create contractor with SSN → tax_id_encrypted is bytes, tax_id_last4 == "1234", plaintext never persisted.`
- `Unit: duplicate (org,email) → 409 conflict.`
- `Integration: GET contractor as admin → tax_id masked ***-**-1234, never full.`

#### 2.2 — Document store (S3/MinIO) with encryption-at-rest

**What**: Upload/download of W-9, contracts, NDAs, IP agreements, ID docs to blob storage with DB metadata.

**Design**:
- `documents` table per Suggestion 1 (lines 117–130); files in S3 keyed `org/{org_id}/contractor/{id}/{doc_id}`.
- `storage/blob.py`: `put(key, bytes, content_type)`, `presigned_get(key, ttl)`; SSE-KMS or MinIO SSE enabled.
- Upload returns metadata only; downloads via short-lived presigned URLs (never proxy bytes through API).

**Testing**:
- `Integration (minio): upload pdf → object exists, row created with size/mime.`
- `Unit: presigned_get → URL expires after ttl.`
- `Security: contractor A requesting contractor B's doc → 404, audited.`

#### 2.3 — e-Signature abstraction (DocuSign + native fallback)

**What**: Pluggable signing of contracts/NDAs/IP agreements with status lifecycle.

**Design**:
- `integrations/esign/base.py`: `ESignProvider.create_envelope(doc, signers) -> envelope_id`; `status(envelope_id)`.
- `e_signatures` table per Suggestion 1 (lines 135–146); lifecycle `pending→sent→viewed→signed|declined|expired`.
- Native fallback signer records signer IP, timestamp, and SHA-256 hash of the signed bytes for non-repudiation.
- Provider webhooks update status; each transition appends an audit event (Phase 6).

**Testing**:
- `Integration (mocked DocuSign): create_envelope → envelope_id stored, status=sent.`
- `Unit: native signer → signature_hash = sha256(signed_bytes), ip recorded.`
- `Integration (mocked): signed webhook → status=signed, signed_at set.`
- `Unit: declined webhook → status=declined, downstream contract NOT activated.`

---

## Phase 3: Onboarding Orchestration, TIN Verification & Contract Generation

### Purpose
Deliver the first headline workflow: invite-link self-onboarding, jurisdiction-aware document
checklists, TIN verification, and jurisdiction-aware contract templating with e-sign. This is the
"smart onboarding orchestration" differentiator from research.md.

### Tasks

#### 3.1 — Invite + self-onboarding state machine

**What**: Admin invites a contractor; contractor self-onboards via tokenised link through a guided flow.

**Design**:
- `services/onboarding.py` state machine: `invited → in_progress → documents_pending → background_check → approved|rejected`.
- Invite: signed token (JWT, `aud=onboarding`, expiry), emailed link; `POST /v1/contractors/{id}/onboarding/start` validates token.
- Required-document checklist derived from `country_code` + engagement type via `services/compliance_rules.py` (Phase 7 supplies the rules; here use a seed ruleset: US→[w9, nda, ip_agreement], non-US→[w8ben, nda]).

**Testing**:
- `Unit: state transitions only valid edges (invited→approved directly → error).`
- `Integration: valid invite token → onboarding started, checklist returned for country.`
- `Unit: expired invite token → 401.`
- `Integration: all docs submitted + bg check pass → status=approved, ContractorApproved event emitted.`

#### 3.2 — TIN verification (IRS TIN Matching)

**What**: Verify W-9 TIN before payment/filing eligibility.

**Design**:
- `integrations/tin_matching.py`: `verify(tin, name) -> {matched: bool, code}` (real-time + bulk modes); async Celery task.
- On match → `contractor.tax_id_verified = True`, `ContractorTaxIdVerified` event. On mismatch → compliance alert, payment blocked.
- Idempotency: keyed on `(contractor_id, tin_hash)` in Redis to avoid duplicate IRS calls.

**Testing**:
- `Integration (mocked IRS): matching TIN → tax_id_verified True, event emitted.`
- `Integration (mocked IRS): mismatch → verified stays False, compliance alert created, payment guard trips.`
- `Unit: same TIN verified twice → second call short-circuits (idempotent).`

#### 3.3 — Jurisdiction-aware contract templating + signature

**What**: Generate contracts from templates with auto-inserted jurisdiction clauses, route to e-sign.

**Design**:
- `contract_templates` + `contracts` tables (Suggestion 1, lines 155–193); `auto_clauses TEXT[]` resolved by jurisdiction (e.g. `ip_assignment_ca`, `ir35_uk`).
- `services/contract_gen.py`: `render(template, contractor, terms) -> document_bytes`; persists a `documents` row, creates e-sign envelope, sets contract `pending_signature`.
- Contract lifecycle `draft→pending_signature→active→completed|terminated|expired`; activation requires both signatures (2.3).
- Contract type `fixed_price|hourly|retainer|milestone`; terms validated against Pydantic schema.

**Testing**:
- `Unit: render US-CA template → contains ip_assignment_ca clause, contractor name interpolated.`
- `Unit: render UK template → contains ir35 clause.`
- `Integration: generate → e-sign envelope created, contract=pending_signature.`
- `Integration: both parties sign → contract=active; only one signs → stays pending.`
- `Unit: activate contract for unverified-TIN contractor → 422 compliance_failure.`

---

## Phase 4: Projects, Timesheets, Invoicing & Approval Workflows

### Purpose
Build the project/deliverable layer (a stated gap in incumbents) and the invoicing engine with
configurable multi-step approvals and timesheet-to-invoice automation — TalentDesk-style
consolidated invoicing.

### Tasks

#### 4.1 — Projects, assignments, milestones

**What**: Project CRUD with budget, contractor assignment, and milestone tracking.

**Design**: `projects`, `project_assignments`, `milestones` tables (Suggestion 1, lines 199–241). Project status `draft|active|on_hold|completed|archived`; milestone `pending→in_progress→submitted→approved|rejected`.

**Testing**:
- `Unit: assign contractor twice to same project → 409 (unique constraint).`
- `Integration: approve milestone → status=approved, amount eligible for invoicing.`
- `Unit: sum(milestone.amount) > project.budget → warning flag in response.`

#### 4.2 — Timesheets + timesheet-to-invoice automation

**What**: Contractor timesheet submission with per-day entries and one-click consolidation into an invoice.

**Design**: `timesheets` + `timesheet_entries` (Suggestion 1, lines 243–273); entry hours `0<h<=24`. `services/invoicing.py: consolidate(contract_id, period) -> Invoice` rolls approved timesheets/milestones into one invoice (consolidated single-invoice model).

**Testing**:
- `Unit: entry hours=25 → validation error.`
- `Integration: approve timesheet then consolidate → invoice line items match hours×rate, timesheet status=invoiced.`
- `Unit: consolidate with no approved timesheets → 422.`

#### 4.3 — Invoices + configurable approval workflows

**What**: Invoice submission and multi-step, amount-thresholded approval chains.

**Design**:
- `invoices`, `invoice_line_items`, `approval_workflows`, `approval_steps`, `approval_records` (Suggestion 1, lines 279–355).
- Invoice lifecycle `draft→submitted→under_review→approved|rejected→scheduled→paid|void`.
- `services/approvals.py`: selects workflow by `entity_type=invoice` + amount band; advances `step_order`; only `approved` at the final step makes the invoice payment-eligible.

**Testing**:
- `Integration: $12k invoice with 2-step (manager→finance) workflow → first approval advances to step 2, not payable until step 2.`
- `Unit: approver without authority for step → 403.`
- `Unit: reject at any step → invoice=rejected, no payment scheduled.`
- `Integration: invoice_number duplicate within org → 409.`

---

## Phase 5: Payments & 1099 Tax Filing (with Outbox)

### Purpose
Ship payment orchestration (ACH/wire/multi-currency, NACHA-aligned) and the second headline
differentiator: intelligent 1099-NEC reconciliation + IRS IRIS e-filing. Introduces the
outbox/audit-event backbone for financial integrity.

### Tasks

#### 5.1 — Outbox + immutable audit-event backbone

**What**: Append-only event recording and a reliable outbox dispatcher (Suggestion 2 patterns) for payments, tax, and webhooks.

**Design**:
- `event_outbox` table (Suggestion 2, lines 57–66) + `audit_log` (Suggestion 1, lines 492–506).
- `events/audit.py: record(entity_type, entity_id, action, changes, actor)` writes audit row **in the same transaction** as the state change.
- `events/outbox.py`: writer enqueues `{event_id, destination, payload}`; Celery-beat dispatcher polls `published=FALSE`, delivers, marks published (at-least-once).

**Testing**:
- `Integration: state change + audit write in one tx → rollback drops both (atomicity).`
- `Integration: outbox row → dispatcher delivers once-or-more, sets published=TRUE.`
- `Unit: dispatcher delivery failure → row stays unpublished, retried next tick.`

#### 5.2 — Payment orchestration (ACH / wire / multi-currency)

**What**: Schedule and dispatch contractor payments through pluggable rails with FX handling.

**Design**:
- `payments` + `contractor_bank_accounts` (Suggestion 1, lines 96–111, 361–385); account numbers encrypted (2.1 crypto).
- `integrations/payments/base.py`: `PaymentRail.dispatch(payment) -> external_ref`; `ach_nacha.py` produces NACHA-compliant batch (and ISO 20022 pain.001 mapping); `wire.py` SWIFT.
- Lifecycle `pending→processing→completed|failed|reversed`; FX captures `exchange_rate` + `amount_in_base`.
- **Guards**: payment refused unless invoice `approved`, bank account `verified`, contractor `tax_id_verified`.
- Idempotency key per `payment_id` prevents double-dispatch on retry.

**Testing**:
- `Integration (mocked ACH): scheduled approved payment → NACHA batch generated, status=processing, PaymentScheduled event.`
- `Unit: dispatch for unverified bank account → 422 compliance_failure.`
- `Unit: dispatch for unapproved invoice → 409.`
- `Integration: rail returns completed webhook → status=completed, external_ref stored.`
- `Unit: retry of same payment_id → no second dispatch (idempotent).`
- `Unit: multi-currency EUR invoice → amount_in_base computed from exchange_rate.`

#### 5.3 — Intelligent 1099 reconciliation + IRIS e-filing

**What**: Auto-match completed payments to invoices per contractor/year, generate 1099-NEC, e-file via IRIS.

**Design**:
- `tax_filings` table (Suggestion 1, lines 387–404) + `proj_tax_reconciliation` projection (Suggestion 2, lines 459–473).
- `services/reconciliation.py: reconcile(org, contractor, year)` sums `PaymentCompleted` amounts, cross-checks invoice totals, flags `discrepancy_amount`; only contractors paid ≥ $600 generate 1099-NEC.
- `integrations/tax/base.py`: `TaxFiler.file(form) -> submission_id`; `iris.py` (IRS IRIS XML schema, requires TCC); `tax1099.py` fallback (sidesteps redistribution constraint).
- Filing lifecycle `draft→generated→reviewed→filed→corrected|void`.

**Testing**:
- `Integration: contractor paid $72k across 3 invoices → 1099-NEC nonemployee_comp=72000, payment_ids reconciled.`
- `Unit: contractor paid $400 → no 1099 generated (below threshold).`
- `Unit: payments total ≠ invoice total → discrepancy_amount set, reconciliation_status=needs_review.`
- `Integration (mocked IRIS): file generated 1099 → status=filed, iris_submission_id stored, TaxFilingSubmittedToIRS event.`
- `Unit: file without TCC configured → 422 with actionable error.`

---

## Phase 6: Public REST API, Webhooks, OpenAPI & SCIM

### Purpose
Expose the platform as a versioned public API with signed webhooks, an exported OpenAPI 3.x spec
(an enterprise procurement deliverable per standards.md), and SCIM 2.0 provisioning.

### Tasks

#### 6.1 — Versioned public API surface + OpenAPI export

**What**: Stabilise `/v1` routes, OAuth scopes, and export the OpenAPI spec as a build artifact.

**Design**: All routers under `/v1`; scopes (`contractors:read`, `invoices:write`, `payments:write`, etc.) enforced via `require_scope`. CI exports `app.openapi()` to `openapi/fms-openapi.json` and fails if it diverges from committed spec. OWASP API Top 10 checks: object-level authz on every `{id}` route, rate limiting via Redis.

**Testing**:
- `CI: exported OpenAPI validates against OpenAPI 3.x meta-schema.`
- `Security: each {id} route enforces org-scope (BOLA test matrix across roles).`
- `Unit: token missing required scope → 403.`
- `Integration: rate limit exceeded → 429 problem+json with Retry-After.`

#### 6.2 — Signed webhooks (events are webhooks)

**What**: Outbound webhooks for contract/payment/compliance lifecycle events with HMAC signing + retry.

**Design**: `webhook_subscriptions` + `webhook_deliveries` (Suggestion 1, lines 523–545). `events/webhooks.py` consumes outbox, POSTs payload with `X-FMS-Signature: sha256=hmac(secret, body)`, exponential-backoff retry (max attempts), records response. Event types: `contractor.approved`, `contract.signed`, `invoice.approved`, `payment.completed`, `tax.filed`, `compliance.alert`.

**Testing**:
- `Integration: subscribed event fires → delivery POSTed with valid HMAC signature.`
- `Unit: receiver 500 → retried with backoff up to max attempts, then dead-lettered.`
- `Security: signature computed over raw body; tampered body fails verification.`

#### 6.3 — SCIM 2.0 provisioning

**What**: `/scim/v2/Users` for enterprise IdP-driven contractor/user lifecycle.

**Design**: RFC 7644 endpoints (Create/Read/Update/Patch/Delete + `/Schemas`, `/ServiceProviderConfig`); maps SCIM User ↔ internal user/contractor; bearer-token auth from IdP.

**Testing**:
- `Integration: SCIM POST /Users → user created, returns SCIM resource with id+meta.`
- `Unit: PATCH active=false → user deactivated (de-provisioned).`
- `Unit: GET /Schemas → valid SCIM schema document.`

---

## Phase 7: AI-Native Differentiators

### Purpose
Implement the four AI capabilities that define the product's edge: pre-contract worker
classification (25+ factors), invoice anomaly detection, compliance-change monitoring, and spend
forecasting — plus talent matching.

### Tasks

#### 7.1 — Worker classification engine (25+ factors, pre-contract)

**What**: Score misclassification risk before a contract is issued.

**Design**:
- `ai/classification.py`: evaluates ≥25 factors across categories `behavioral_control`, `financial_control`, `relationship` (IRS three-factor + ABC test heuristics); weighted score → `risk_level low|medium|high|critical` and `recommendation independent_contractor|employee|review_needed`.
- Persists `classification_assessments` + `classification_factors` (Suggestion 1, lines 450–470); `assessed_by ai|manual|hybrid`.
- LLM (via `ai/provider.py`) generates a natural-language rationale per factor; deterministic scoring kept rule-based for auditability (LLM augments, does not decide).
- Invoked in 3.3 **before** contract activation; `high|critical` blocks activation pending review.

**Testing**:
- `Unit: full-control + fixed-salary inputs → risk=high, recommendation=employee.`
- `Unit: project-based + own-tools + multiple-clients → risk=low, independent_contractor.`
- `Integration: high-risk assessment → contract activation blocked, ContractorClassified event.`
- `Unit: score is deterministic for identical inputs (LLM rationale mocked).`

#### 7.2 — Invoice anomaly detection

**What**: Flag duplicates, inflated hours, and off-contract billing before approval.

**Design**: `ai/anomaly.py` runs at invoice submission (4.3); detectors: duplicate (hash of line items vs prior 90 days), inflated hours (> contract max / statistical outlier vs contractor history), off-contract (no active contract / period mismatch), unusual amount. Writes `invoice_anomalies` (Suggestion 1, lines 472–486) + sets `invoices.anomaly_flag`; high-confidence anomalies hold the invoice in `under_review`.

**Testing**:
- `Unit: identical line items to last week's invoice → duplicate, confidence>0.9.`
- `Unit: 60h on a 40h-cap contract → inflated_hours flagged.`
- `Unit: invoice with no active contract → off_contract flagged.`
- `Integration: high-confidence anomaly → invoice held under_review, not auto-approvable.`

#### 7.3 — Compliance-change monitoring + alerts

**What**: Ingest jurisdiction regulatory changes and surface natural-language alerts.

**Design**: `jurisdictions`, `compliance_rules`, `compliance_alerts` (Suggestion 1, lines 410–442). `services/compliance_rules.py` resolves effective rules by date; a scheduled task ingests a configurable regulatory feed, diffs against stored rules, and emits `compliance.alert` events with LLM-summarised impact (severity `info|warning|critical`).

**Testing**:
- `Unit: rule effective_from in future → not yet applied to checklist.`
- `Integration: feed introduces new IR35 rule → alert created for orgs with UK contractors, webhook fired.`
- `Unit: superseded rule (effective_to past) → excluded from active set.`

#### 7.4 — Spend forecasting + talent matching

**What**: Predict next-quarter contractor cost; rank talent-pool contractors against a brief.

**Design**: `ai/forecasting.py` over `proj_spend_analytics` (Suggestion 2, lines 421–434) — time-series on historical paid amounts/utilisation → next-quarter projection with confidence band. `ai/talent_match.py` ranks `contractor_skills` + history against brief requirements (embeddings + rule filters). Talent directory served from `proj_contractor_directory` (Suggestion 2, lines 351–369) with GIN-indexed skills search.

**Testing**:
- `Unit: 8 quarters of rising spend → forecast > last quarter, confidence band returned.`
- `Unit: insufficient history (<2 quarters) → low-confidence flag.`
- `Integration: brief requires [React, expert] → ranked contractors all match, ordered by relevance.`

---

## Phase 8: Frontend — Admin Dashboard & Contractor Portal

### Purpose
Deliver the two user-facing surfaces from features.md: an admin dashboard (HR/finance/ops) and a
mobile-first contractor self-service portal, both consuming the generated OpenAPI client.

### Tasks

#### 8.1 — Generated API client + auth/session

**What**: Type-safe client generated from `openapi/fms-openapi.json`; OIDC login + token refresh.

**Design**: `lib/api-client` generated in CI; Next.js middleware guards routes by audience (`admin` vs `portal`); tokens stored httpOnly; silent refresh on 401.

**Testing**:
- `E2E (Playwright): admin login via OIDC → dashboard loads; unauthenticated → redirect to login.`
- `Unit: 401 → silent refresh then retry; refresh fail → logout.`

#### 8.2 — Admin dashboard

**What**: Contractor directory, onboarding pipeline, invoice approvals, payments, compliance alerts, classification review, spend analytics.

**Design**: Progressive-disclosure compliance detail (Deel pattern); consolidated payment-schedule view; invoice approval queue showing anomaly flags; classification review screen surfacing 25-factor breakdown before contract activation.

**Testing**:
- `E2E: approve a flagged invoice → moves out of queue, status updates.`
- `E2E: review high-risk classification → activation blocked until override with reason.`

#### 8.3 — Contractor self-service portal (mobile-first)

**What**: Invite-link onboarding wizard, document upload, invoice submission, payment status, tax docs.

**Design**: Guided onboarding wizard driven by the jurisdiction checklist (3.1); invoice submission from approved timesheets; payment status timeline; 1099 download. Mobile-first responsive layout.

**Testing**:
- `E2E: contractor completes onboarding via invite link → status=approved in admin.`
- `E2E: submit invoice from portal → appears in admin approval queue.`
- `E2E (mobile viewport): portal usable at 375px width.`
- `Security E2E: contractor cannot navigate to another contractor's records.`

---

## Phase 9: Accounting Integrations

### Purpose
Add QuickBooks, Xero, and FreshBooks sync so invoices/payments reconcile in the customer's books —
a table-stakes integration set.

### Tasks

#### 9.1 — Integration framework + OAuth connect

**What**: Pluggable accounting connector with encrypted credential storage and connection lifecycle.

**Design**: `integrations` table (Suggestion 1, lines 512–521), credentials encrypted (2.1). `integrations/accounting/base.py`: `AccountingConnector.sync_invoice(invoice)`, `sync_payment(payment)`, `pull_status()`. OAuth connect flow per provider; status `connected|disconnected|error`, `last_sync_at`.

**Testing**:
- `Integration (mocked QBO): connect OAuth → status=connected, creds encrypted at rest.`
- `Unit: provider token expired → status=error, alert raised.`

#### 9.2 — Invoice/payment sync (QuickBooks, Xero, FreshBooks)

**What**: Push approved invoices and completed payments to the connected ledger.

**Design**: Outbox-driven (5.1) — `invoice.approved` / `payment.completed` events trigger sync tasks; field-mapping per provider in JSONB `config`; idempotent upsert by external id.

**Testing**:
- `Integration (mocked Xero): invoice.approved → invoice created in Xero, external id stored.`
- `Unit: re-sync same invoice → upsert, no duplicate.`
- `Integration: sync failure → retried via outbox, surfaced as integration error.`

---

## Phase 10: Hardening, Compliance & Release

### Purpose
Make the system production- and procurement-ready: security posture (OWASP/ISO 27001), privacy
(GDPR/CCPA, ISO 27701), observability, and the self-hosted release.

### Tasks

#### 10.1 — Security & privacy hardening

**What**: OWASP API Top 10 pass, GDPR/CCPA data-subject operations, DPA-supporting controls.

**Design**: BOLA/object-authz audit across all routes; rate limiting; secrets via env/secret-manager; right-to-erasure (`DELETE` with crypto-shredding of encrypted PII while retaining legally-required tax records anonymised); data-export endpoint; configurable retention.

**Testing**:
- `Security: automated BOLA matrix (each role × each {id} route) → no cross-tenant access.`
- `Integration: erasure request → PII crypto-shredded, tax_filings retained but de-identified.`
- `Integration: data-export → machine-readable bundle of contractor's data.`

#### 10.2 — Observability

**What**: Structured logs, traces, metrics, health/readiness endpoints.

**Design**: structlog JSON logs with correlation_id propagated from API → worker → integration; OpenTelemetry spans on payment/tax/webhook paths; Prometheus metrics (`/metrics`); `/healthz`, `/readyz`.

**Testing**:
- `Integration: a payment flow emits a single correlation_id across API+worker logs.`
- `Unit: /readyz returns 503 when DB unreachable.`

#### 10.3 — Self-hosted release packaging

**What**: One-command self-hosted deploy with migrations, seed data, and docs.

**Design**: `docker-compose.yml` full stack; entrypoint runs Alembic migrations + seeds jurisdictions/compliance rules; signed images; `.env.example` documents every setting; release notes + OpenAPI spec attached to the GitHub release.

**Testing**:
- `E2E: fresh `docker-compose up` → migrations applied, health green, admin can log in and onboard a contractor end-to-end.`
- `Integration: re-run compose → migrations idempotent, no data loss.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, db, errors)        ─── required by everything
    │
Phase 2: Contractors, Documents, e-Sign       ─── requires 1
    │
Phase 3: Onboarding, TIN, Contracts           ─── requires 2
    │
Phase 4: Projects, Timesheets, Invoices        ─── requires 3
    │
Phase 5: Payments + 1099 Filing (+outbox)      ─── requires 4
    │
    ├── Phase 6: Public API, Webhooks, SCIM     ─── requires 5 (outbox); can parallel with 7
    ├── Phase 7: AI Differentiators             ─── requires 4 (5 for forecasting); can parallel with 6
    │
Phase 8: Frontend (admin + portal)             ─── requires 6 (OpenAPI client); can parallel with 9
Phase 9: Accounting Integrations               ─── requires 5 (outbox) + 6; can parallel with 8
    │
Phase 10: Hardening, Compliance, Release       ─── requires all prior phases
```

**Parallelism opportunities**
- Phases 6 and 7 can be built concurrently once Phase 5 lands (both depend on the core engine + outbox; 7's forecasting needs Phase 5 data).
- Phases 8 and 9 can be built concurrently after Phase 6 (frontend needs the OpenAPI client; integrations need the outbox + public API).
- Within Phase 2, the e-sign adapter (2.3) and document store (2.2) are independent.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass; new code has meaningful coverage of happy-path and edge cases.
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes.
5. Docker image builds and `docker-compose up` brings the affected services to a healthy state.
6. The phase's feature works end-to-end (verified by an integration or e2e test).
7. New config options added to `Settings` and documented in `.env.example`.
8. New/changed endpoints appear in the exported `openapi/fms-openapi.json`, which validates against the OpenAPI 3.x meta-schema.
9. Alembic migration(s) created, reversible, and idempotent on re-run.
10. State changes touching money, tax, or compliance write an audit-log row in the same transaction.
11. Any PII field added is encrypted at rest and never returned unmasked.
12. OWASP object-level authorization (org/contractor scoping) enforced and tested on every new `{id}` route.
```
