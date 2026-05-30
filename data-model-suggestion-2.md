# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourcing architecture where every state change is captured as an immutable domain event appended to an event store. The write side processes commands and emits events; the read side builds materialized projections optimized for specific query patterns. This follows the Command Query Responsibility Segregation (CQRS) pattern with a persistent event log as the system of record.

## Why This Suits a Freelancer Management System

A freelancer management system is dominated by lifecycle workflows: contractor onboarding moves through stages, invoices flow through approval chains, payments transition through processing states, and contracts evolve from draft to active to completed. Event sourcing captures every one of these transitions as a first-class record, giving the system a complete, immutable audit trail by default -- critical for tax compliance (1099 reconciliation) and regulatory audits.

The AI features described in the README (classification analysis, anomaly detection, spend forecasting) benefit enormously from a full event history. Rather than querying current state and trying to infer patterns, ML models can consume raw event streams to detect anomalies, reconstruct decision chains, and train on historical behavior.

The trade-off is operational complexity. Event-sourced systems require careful schema evolution for events, eventual consistency between write and read sides, and more infrastructure (event store, projection builders, potentially a message broker). Teams unfamiliar with the pattern face a steep learning curve.

---

## Event Store Schema

The event store is the single source of truth. All domain state is derived from replaying events.

```sql
-- ============================================================
-- EVENT STORE (PostgreSQL-backed)
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    event_version   INT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    correlation_id  UUID,
    causation_id    UUID,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(aggregate_type, aggregate_id, event_version)
);

CREATE INDEX idx_events_aggregate ON event_store(aggregate_type, aggregate_id, event_version);
CREATE INDEX idx_events_type ON event_store(event_type);
CREATE INDEX idx_events_created ON event_store(created_at);
CREATE INDEX idx_events_correlation ON event_store(correlation_id);

-- Snapshot table for performance (avoid replaying thousands of events)
CREATE TABLE event_snapshots (
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version INT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- Outbox for reliable event publishing to external consumers
CREATE TABLE event_outbox (
    id              BIGSERIAL PRIMARY KEY,
    event_id        UUID NOT NULL REFERENCES event_store(event_id),
    destination     VARCHAR(100) NOT NULL,
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished ON event_outbox(published) WHERE published = FALSE;
```

---

## Domain Events

Each aggregate (contractor, contract, invoice, payment, etc.) emits typed events. Below are the core event schemas defined as JSON structures.

### Contractor Aggregate Events

```json
// ContractorInvited
{
  "event_type": "ContractorInvited",
  "aggregate_type": "Contractor",
  "payload": {
    "organization_id": "uuid",
    "email": "string",
    "first_name": "string",
    "last_name": "string",
    "country_code": "string",
    "invite_token": "string",
    "invite_expires_at": "iso8601"
  }
}

// ContractorOnboardingStarted
{
  "event_type": "ContractorOnboardingStarted",
  "payload": {
    "portal_access_granted": true,
    "required_documents": ["w9", "nda", "ip_agreement"]
  }
}

// ContractorDocumentSubmitted
{
  "event_type": "ContractorDocumentSubmitted",
  "payload": {
    "document_id": "uuid",
    "document_type": "w9|w8ben|nda|ip_agreement|id_verification",
    "file_path": "string",
    "file_size_bytes": 0
  }
}

// ContractorTaxIdVerified
{
  "event_type": "ContractorTaxIdVerified",
  "payload": {
    "tax_id_masked": "***-**-1234",
    "tax_form_type": "W-9",
    "verification_source": "irs_tin_match",
    "verified": true
  }
}

// ContractorClassified
{
  "event_type": "ContractorClassified",
  "payload": {
    "classification": "independent_contractor|employee|flagged",
    "overall_score": 85.5,
    "risk_level": "low|medium|high|critical",
    "factors": [
      {"name": "behavioral_control", "score": 90, "weight": 0.25},
      {"name": "financial_control", "score": 82, "weight": 0.25}
    ],
    "assessed_by": "ai|manual"
  }
}

// ContractorApproved / ContractorRejected
{
  "event_type": "ContractorApproved",
  "payload": {
    "approved_by": "uuid",
    "background_check_passed": true
  }
}

// ContractorBankAccountAdded
{
  "event_type": "ContractorBankAccountAdded",
  "payload": {
    "bank_account_id": "uuid",
    "account_type": "ach|wire|iban",
    "currency_code": "USD",
    "bank_name": "string",
    "is_primary": true
  }
}

// ContractorSkillsUpdated
{
  "event_type": "ContractorSkillsUpdated",
  "payload": {
    "skills_added": [{"name": "React", "proficiency": "expert"}],
    "skills_removed": ["Angular"]
  }
}
```

### Contract Aggregate Events

```json
// ContractDrafted
{
  "event_type": "ContractDrafted",
  "payload": {
    "organization_id": "uuid",
    "contractor_id": "uuid",
    "template_id": "uuid",
    "contract_type": "fixed_price|hourly|retainer|milestone",
    "currency_code": "USD",
    "rate_amount": 150.00,
    "rate_unit": "hour",
    "total_budget": 50000.00,
    "start_date": "2026-01-15",
    "end_date": "2026-12-31",
    "jurisdiction": "US-CA",
    "auto_clauses_applied": ["ip_assignment_ca", "ndisclosure_standard"]
  }
}

// ContractSentForSignature / ContractSigned / ContractTerminated
{
  "event_type": "ContractSigned",
  "payload": {
    "signed_by_contractor_at": "iso8601",
    "signed_by_org_at": "iso8601",
    "document_id": "uuid",
    "signature_hash": "sha256:..."
  }
}
```

### Invoice Aggregate Events

```json
// InvoiceSubmitted
{
  "event_type": "InvoiceSubmitted",
  "payload": {
    "organization_id": "uuid",
    "contractor_id": "uuid",
    "contract_id": "uuid",
    "invoice_number": "INV-2026-001",
    "currency_code": "USD",
    "line_items": [
      {"description": "Backend development", "quantity": 40, "unit_price": 150, "amount": 6000}
    ],
    "subtotal": 6000.00,
    "tax_amount": 0,
    "total_amount": 6000.00,
    "due_date": "2026-02-28"
  }
}

// InvoiceAnomalyDetected
{
  "event_type": "InvoiceAnomalyDetected",
  "payload": {
    "anomaly_type": "duplicate|inflated_hours|off_contract|unusual_amount",
    "confidence": 0.92,
    "description": "Hours exceed contract maximum by 15%",
    "comparable_invoice_ids": ["uuid"]
  }
}

// InvoiceApproved / InvoiceRejected / InvoicePaid
{
  "event_type": "InvoiceApproved",
  "payload": {
    "approved_by": "uuid",
    "approval_step": 2,
    "workflow_id": "uuid"
  }
}
```

### Payment Aggregate Events

```json
// PaymentScheduled
{
  "event_type": "PaymentScheduled",
  "payload": {
    "invoice_id": "uuid",
    "contractor_id": "uuid",
    "payment_method": "ach|wire",
    "amount": 6000.00,
    "currency_code": "USD",
    "exchange_rate": 1.0,
    "bank_account_id": "uuid",
    "scheduled_date": "2026-03-01"
  }
}

// PaymentProcessing / PaymentCompleted / PaymentFailed / PaymentReversed
{
  "event_type": "PaymentCompleted",
  "payload": {
    "external_ref": "ACH-TXN-12345",
    "processed_at": "iso8601",
    "settlement_amount": 6000.00
  }
}
```

### TaxFiling Aggregate Events

```json
// TaxFilingGenerated
{
  "event_type": "TaxFilingGenerated",
  "payload": {
    "contractor_id": "uuid",
    "tax_year": 2026,
    "form_type": "1099-NEC",
    "total_paid": 72000.00,
    "nonemployee_comp": 72000.00,
    "payment_ids_reconciled": ["uuid1", "uuid2"]
  }
}

// TaxFilingSubmittedToIRS
{
  "event_type": "TaxFilingSubmittedToIRS",
  "payload": {
    "iris_submission_id": "IRIS-2026-XXXX",
    "submitted_at": "iso8601"
  }
}
```

---

## Command Handlers

Commands are the write-side entry points. Each validates business rules, then emits events.

```
SubmitInvoice
  -> validate contractor has active contract
  -> validate line items match contract terms
  -> run anomaly detection
  -> emit InvoiceSubmitted
  -> if anomalies: emit InvoiceAnomalyDetected

ApproveInvoice
  -> validate user has approval authority
  -> validate invoice is in correct workflow step
  -> emit InvoiceApproved
  -> if final approval step: emit InvoiceReadyForPayment

SchedulePayment
  -> validate invoice is approved
  -> validate bank account is verified
  -> calculate exchange rate if multi-currency
  -> emit PaymentScheduled

ClassifyContractor
  -> load engagement factors from contract and project data
  -> run classification model against 25+ factors
  -> emit ContractorClassified

Generate1099
  -> query all PaymentCompleted events for contractor+year
  -> reconcile totals across invoices
  -> emit TaxFilingGenerated
```

---

## Read Projections (Materialized Views)

Projections consume events and build query-optimized tables. Each projection can be rebuilt from scratch by replaying the event store.

```sql
-- ============================================================
-- PROJECTION: Contractor Directory (searchable talent pool)
-- ============================================================

CREATE TABLE proj_contractor_directory (
    contractor_id       UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    email               VARCHAR(255) NOT NULL,
    full_name           VARCHAR(200) NOT NULL,
    country_code        CHAR(2),
    skills              TEXT[],
    classification      VARCHAR(30),
    risk_level          VARCHAR(10),
    onboarding_status   VARCHAR(30),
    active_contracts    INT NOT NULL DEFAULT 0,
    total_paid_ytd      DECIMAL(14,2) NOT NULL DEFAULT 0,
    last_payment_date   TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_dir_org ON proj_contractor_directory(organization_id);
CREATE INDEX idx_proj_dir_skills ON proj_contractor_directory USING GIN(skills);
CREATE INDEX idx_proj_dir_classification ON proj_contractor_directory(classification);

-- ============================================================
-- PROJECTION: Invoice Dashboard
-- ============================================================

CREATE TABLE proj_invoice_dashboard (
    invoice_id          UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    contractor_id       UUID NOT NULL,
    contractor_name     VARCHAR(200),
    invoice_number      VARCHAR(50),
    status              VARCHAR(20),
    total_amount        DECIMAL(14,2),
    currency_code       CHAR(3),
    due_date            DATE,
    has_anomalies       BOOLEAN NOT NULL DEFAULT FALSE,
    anomaly_types       TEXT[],
    current_approver    UUID,
    approval_step       INT,
    submitted_at        TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_inv_org_status ON proj_invoice_dashboard(organization_id, status);

-- ============================================================
-- PROJECTION: Payment Ledger
-- ============================================================

CREATE TABLE proj_payment_ledger (
    payment_id          UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    contractor_id       UUID NOT NULL,
    invoice_id          UUID,
    amount              DECIMAL(14,2),
    currency_code       CHAR(3),
    payment_method      VARCHAR(20),
    status              VARCHAR(20),
    scheduled_date      DATE,
    processed_at        TIMESTAMPTZ,
    external_ref        VARCHAR(255),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_pay_org ON proj_payment_ledger(organization_id);
CREATE INDEX idx_proj_pay_contractor ON proj_payment_ledger(contractor_id);

-- ============================================================
-- PROJECTION: Spend Analytics (per project/contractor/period)
-- ============================================================

CREATE TABLE proj_spend_analytics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    period_year         INT NOT NULL,
    period_month        INT NOT NULL,
    contractor_id       UUID,
    project_id          UUID,
    total_invoiced      DECIMAL(14,2) NOT NULL DEFAULT 0,
    total_paid          DECIMAL(14,2) NOT NULL DEFAULT 0,
    total_hours         DECIMAL(10,2) NOT NULL DEFAULT 0,
    contractor_count    INT NOT NULL DEFAULT 0,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, period_year, period_month, contractor_id, project_id)
);

-- ============================================================
-- PROJECTION: Compliance Status
-- ============================================================

CREATE TABLE proj_compliance_status (
    contractor_id       UUID NOT NULL,
    organization_id     UUID NOT NULL,
    jurisdiction        VARCHAR(50),
    tax_form_status     VARCHAR(20),
    id_verified         BOOLEAN NOT NULL DEFAULT FALSE,
    background_check    VARCHAR(20),
    classification      VARCHAR(30),
    risk_level          VARCHAR(10),
    active_alerts       INT NOT NULL DEFAULT 0,
    last_assessment_at  TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (contractor_id, organization_id)
);

-- ============================================================
-- PROJECTION: Tax Filing Reconciliation
-- ============================================================

CREATE TABLE proj_tax_reconciliation (
    organization_id     UUID NOT NULL,
    contractor_id       UUID NOT NULL,
    tax_year            INT NOT NULL,
    form_type           VARCHAR(20),
    total_invoiced      DECIMAL(14,2) NOT NULL DEFAULT 0,
    total_paid          DECIMAL(14,2) NOT NULL DEFAULT 0,
    payment_count       INT NOT NULL DEFAULT 0,
    reconciliation_status VARCHAR(20) DEFAULT 'pending',
    discrepancy_amount  DECIMAL(14,2) DEFAULT 0,
    filing_status       VARCHAR(20) DEFAULT 'not_started',
    iris_submission_id  VARCHAR(100),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, contractor_id, tax_year)
);
```

---

## Trade-offs

**Strengths:**
- Complete audit trail is a natural by-product -- every state change is an immutable event. This is invaluable for compliance-heavy domains like tax filing and payment reconciliation.
- AI/ML features (anomaly detection, spend forecasting, classification) can consume the event stream directly, training on historical sequences rather than point-in-time snapshots.
- Temporal queries ("what was this contractor's classification status on March 15?") are trivial -- just replay events up to that timestamp.
- Read models can be independently optimized, scaled, and rebuilt without affecting the write path.
- Webhook delivery becomes trivial: events are the webhooks.

**Weaknesses:**
- Eventual consistency between the event store and projections requires careful handling in the UI (e.g., invoice just submitted but not yet visible in the dashboard).
- Event schema evolution is a real operational concern. Renaming or restructuring event payloads requires upcasters.
- Higher infrastructure complexity: event store + projection builders + potentially a message broker (Kafka, NATS, or PostgreSQL LISTEN/NOTIFY for simpler setups).
- The team must understand event sourcing patterns; the learning curve is steeper than traditional CRUD.

**Scalability:**
- The event store scales horizontally by partitioning on aggregate_id.
- Projections can be rebuilt from scratch on new hardware or in new formats without touching the event store.
- Event replay for large aggregates is mitigated by snapshots (stored every N events).

**Migration Path:**
- Can start with a simpler relational model (suggestion 1) and introduce event sourcing incrementally, beginning with the most audit-sensitive aggregates (payments, tax filings) while keeping less critical entities as standard CRUD.
- The outbox pattern enables reliable integration with external systems (accounting software, IRS IRIS API) via at-least-once event delivery.
