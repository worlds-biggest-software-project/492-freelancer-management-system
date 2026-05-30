# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A PostgreSQL schema that keeps core entities and their relationships in traditional relational columns while using JSONB columns for flexible, nested, or jurisdiction-varying data. This is a pragmatic middle ground: you get referential integrity and indexing on the fields that matter for joins and lookups, plus schema-free extensibility for the parts of the domain that vary unpredictably across clients, jurisdictions, and integrations.

## Why This Suits a Freelancer Management System

Freelancer management spans multiple jurisdictions, each with distinct compliance requirements. A US contractor needs W-9/TIN verification and 1099-NEC filing. A UK contractor needs IR35 assessment fields. A French contractor needs URSSAF registration data. Modeling every jurisdiction's fields as relational columns produces a sparse, wide table riddled with nullable columns. JSONB solves this: the `compliance_data` column holds whatever jurisdiction-specific fields apply, indexed with GIN for fast lookups.

The same flexibility applies to configurable approval workflows (each org defines different rules), integration sync metadata (QuickBooks mapping differs from Xero), and AI classification results (factor lists vary as the model evolves). JSONB absorbs this variation without schema migrations.

The trade-off is weaker enforcement. PostgreSQL cannot enforce foreign keys or NOT NULL constraints inside JSONB. Application-level validation and JSON Schema checks must fill the gap. Queries against deeply nested JSONB paths are also less readable and harder to optimize than flat column access.

---

## Schema Definition

```sql
-- ============================================================
-- CORE IDENTITY & ORGANIZATION
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(50),
    country_code    CHAR(2) NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- settings holds: default_currency, payment_terms_days, approval_thresholds,
    --   branding (logo_url, colors for white-label), default_contract_jurisdiction,
    --   notification_preferences, scim_config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_country ON organizations(country_code);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(30) NOT NULL CHECK (role IN ('admin', 'manager', 'finance', 'hr', 'viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    preferences     JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- preferences holds: timezone, locale, notification_channels, dashboard_layout
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(organization_id);

-- ============================================================
-- CONTRACTOR PROFILE & ONBOARDING
-- ============================================================

CREATE TABLE contractors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    email               VARCHAR(255) NOT NULL,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    phone               VARCHAR(30),
    country_code        CHAR(2) NOT NULL,
    onboarding_status   VARCHAR(30) NOT NULL DEFAULT 'invited'
                        CHECK (onboarding_status IN ('invited', 'in_progress', 'documents_pending',
                                                      'background_check', 'approved', 'rejected')),

    -- Relational: fields queried/joined/filtered constantly
    classification      VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (classification IN ('pending', 'independent_contractor', 'employee', 'flagged')),
    classification_score DECIMAL(5,2),

    -- JSONB: tax & compliance data varies by jurisdiction
    tax_info            JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- US example: {"form_type": "W-9", "tin": "encrypted_ref", "tin_verified": true, "tin_match_date": "..."}
    -- UK example: {"form_type": "self_assessment", "utr": "encrypted_ref", "ir35_status": "outside", "ni_number": "..."}
    -- FR example: {"form_type": "urssaf", "siret": "...", "urssaf_registered": true}

    -- JSONB: identity verification results
    identity_verification JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"provider": "jumio", "status": "verified", "verified_at": "...", "document_type": "passport",
    --  "check_id": "...", "background_check": {"provider": "checkr", "status": "clear", "completed_at": "..."}}

    -- JSONB: skills and profile (searchable via GIN)
    profile             JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"skills": [{"name": "React", "proficiency": "expert"}, ...],
    --  "bio": "...", "portfolio_url": "...", "availability": "full_time",
    --  "preferred_rate": {"amount": 150, "currency": "USD", "unit": "hour"}}

    invite_token        VARCHAR(255),
    invite_expires_at   TIMESTAMPTZ,
    portal_password_hash VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

CREATE INDEX idx_contractors_org ON contractors(organization_id);
CREATE INDEX idx_contractors_status ON contractors(onboarding_status);
CREATE INDEX idx_contractors_classification ON contractors(classification);
CREATE INDEX idx_contractors_tax_info ON contractors USING GIN(tax_info jsonb_path_ops);
CREATE INDEX idx_contractors_profile ON contractors USING GIN(profile jsonb_path_ops);
CREATE INDEX idx_contractors_country ON contractors(country_code);

CREATE TABLE contractor_bank_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contractor_id   UUID NOT NULL REFERENCES contractors(id) ON DELETE CASCADE,
    account_type    VARCHAR(20) NOT NULL CHECK (account_type IN ('ach', 'wire', 'iban')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    verified        BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB: bank details vary by account type and country
    bank_details    JSONB NOT NULL,
    -- ACH: {"bank_name": "...", "routing_number": "...", "account_number_ref": "vault:xxx"}
    -- IBAN: {"bank_name": "...", "iban_ref": "vault:xxx", "swift_code": "..."}
    -- Wire: {"bank_name": "...", "account_ref": "vault:xxx", "swift_code": "...",
    --         "intermediary_bank": "...", "intermediary_swift": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bank_contractor ON contractor_bank_accounts(contractor_id);

-- ============================================================
-- DOCUMENTS & E-SIGNATURES
-- ============================================================

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contractor_id   UUID REFERENCES contractors(id),
    document_type   VARCHAR(30) NOT NULL,
    file_name       VARCHAR(255) NOT NULL,
    file_path       VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    uploaded_by     UUID REFERENCES users(id),
    -- JSONB: e-signature tracking embedded with document
    signatures      JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"signer_type": "contractor", "signer_id": "uuid", "status": "signed",
    --   "signed_at": "...", "ip_address": "...", "signature_hash": "sha256:..."},
    --  {"signer_type": "organization", "signer_id": "uuid", "status": "pending"}]
    metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"version": 2, "expiry_date": "...", "auto_renew": true, "tags": ["onboarding", "legal"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_contractor ON documents(contractor_id);
CREATE INDEX idx_documents_type ON documents(document_type);
CREATE INDEX idx_documents_signatures ON documents USING GIN(signatures jsonb_path_ops);

-- ============================================================
-- CONTRACTS
-- ============================================================

CREATE TABLE contract_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    jurisdiction    VARCHAR(50) NOT NULL,
    template_body   TEXT NOT NULL,
    -- JSONB: clause library and insertion rules
    clause_rules    JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"clause_id": "ip_assignment", "condition": {"jurisdiction": "US-CA"}, "text": "..."},
    --  {"clause_id": "non_compete", "condition": {"contract_type": "retainer"}, "text": "..."}]
    version         INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contractor_id   UUID NOT NULL REFERENCES contractors(id),
    template_id     UUID REFERENCES contract_templates(id),
    document_id     UUID REFERENCES documents(id),
    title           VARCHAR(255) NOT NULL,
    contract_type   VARCHAR(30) NOT NULL CHECK (contract_type IN ('fixed_price', 'hourly', 'retainer', 'milestone')),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'pending_signature', 'active', 'completed', 'terminated', 'expired')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE NOT NULL,
    end_date        DATE,
    jurisdiction    VARCHAR(50) NOT NULL,
    -- JSONB: rate and budget structure varies by contract type
    commercial_terms JSONB NOT NULL,
    -- Hourly: {"rate": 150.00, "unit": "hour", "max_hours_week": 40, "overtime_rate": 225.00}
    -- Fixed:  {"total_price": 50000, "payment_schedule": "on_milestones"}
    -- Retainer: {"monthly_fee": 10000, "included_hours": 60, "overage_rate": 175}
    -- Milestone: {"milestones": [{"title": "...", "amount": 10000, "due": "2026-03-01"}]}

    -- JSONB: jurisdiction-specific compliance terms
    compliance_terms JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- US: {"requires_1099": true, "state_withholding": false}
    -- UK: {"ir35_determination": "outside", "ir35_assessed_by": "uuid", "ir35_assessed_at": "..."}

    auto_renew      BOOLEAN NOT NULL DEFAULT FALSE,
    notice_period_days INT DEFAULT 30,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_org ON contracts(organization_id);
CREATE INDEX idx_contracts_contractor ON contracts(contractor_id);
CREATE INDEX idx_contracts_status ON contracts(status);
CREATE INDEX idx_contracts_dates ON contracts(start_date, end_date);
CREATE INDEX idx_contracts_commercial ON contracts USING GIN(commercial_terms jsonb_path_ops);

-- ============================================================
-- PROJECTS, MILESTONES & TIMESHEETS
-- ============================================================

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('draft', 'active', 'on_hold', 'completed', 'archived')),
    budget          DECIMAL(14,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    start_date      DATE,
    end_date        DATE,
    manager_id      UUID REFERENCES users(id),
    -- JSONB: flexible project metadata
    metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"department": "engineering", "cost_center": "CC-1001",
    --  "tags": ["q2-initiative", "critical"], "custom_fields": {...}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_org ON projects(organization_id);
CREATE INDEX idx_projects_metadata ON projects USING GIN(metadata jsonb_path_ops);

CREATE TABLE project_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    contractor_id   UUID NOT NULL REFERENCES contractors(id),
    contract_id     UUID REFERENCES contracts(id),
    role            VARCHAR(100),
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(project_id, contractor_id)
);

CREATE TABLE milestones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    due_date        DATE,
    amount          DECIMAL(12,2),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'in_progress', 'submitted', 'approved', 'rejected')),
    -- JSONB: deliverables and acceptance criteria
    deliverables    JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"title": "API endpoints", "description": "...", "completed": true},
    --  {"title": "Documentation", "description": "...", "completed": false}]
    completed_at    TIMESTAMPTZ,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_milestones_project ON milestones(project_id);

CREATE TABLE timesheets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contractor_id   UUID NOT NULL REFERENCES contractors(id),
    contract_id     UUID NOT NULL REFERENCES contracts(id),
    project_id      UUID REFERENCES projects(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    total_hours     DECIMAL(6,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'submitted', 'approved', 'rejected', 'invoiced')),
    -- JSONB: daily entries stored as structured array
    entries         JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"date": "2026-01-15", "hours": 8.0, "description": "Feature X", "task_ref": "PROJ-123"},
    --  {"date": "2026-01-16", "hours": 6.5, "description": "Code review", "task_ref": "PROJ-124"}]
    submitted_at    TIMESTAMPTZ,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timesheets_contractor ON timesheets(contractor_id);
CREATE INDEX idx_timesheets_contract ON timesheets(contract_id);
CREATE INDEX idx_timesheets_period ON timesheets(period_start, period_end);

-- ============================================================
-- INVOICING
-- ============================================================

CREATE TABLE invoices (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    contractor_id       UUID NOT NULL REFERENCES contractors(id),
    contract_id         UUID REFERENCES contracts(id),
    invoice_number      VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'submitted', 'under_review', 'approved',
                                          'rejected', 'scheduled', 'paid', 'void')),
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal            DECIMAL(14,2) NOT NULL,
    tax_amount          DECIMAL(12,2) NOT NULL DEFAULT 0,
    total_amount        DECIMAL(14,2) NOT NULL,
    due_date            DATE NOT NULL,

    -- JSONB: line items stored inline (avoids join for display)
    line_items          JSONB NOT NULL,
    -- [{"description": "Backend dev", "quantity": 40, "unit_price": 150, "amount": 6000,
    --   "timesheet_id": "uuid", "milestone_id": null}]

    -- JSONB: approval chain tracking
    approval_chain      JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"step": 1, "approver_id": "uuid", "role": "manager", "decision": "approved",
    --   "comment": "Looks good", "decided_at": "..."},
    --  {"step": 2, "approver_id": "uuid", "role": "finance", "decision": null}]

    -- JSONB: anomaly detection results
    anomaly_results     JSONB NOT NULL DEFAULT '[]'::jsonb,
    -- [{"type": "inflated_hours", "confidence": 0.87, "description": "...", "resolved": false}]

    submitted_at        TIMESTAMPTZ,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, invoice_number)
);

CREATE INDEX idx_invoices_org ON invoices(organization_id);
CREATE INDEX idx_invoices_contractor ON invoices(contractor_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due ON invoices(due_date);
CREATE INDEX idx_invoices_anomalies ON invoices USING GIN(anomaly_results jsonb_path_ops);

-- ============================================================
-- PAYMENTS & TAX FILING
-- ============================================================

CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    contractor_id       UUID NOT NULL REFERENCES contractors(id),
    invoice_id          UUID REFERENCES invoices(id),
    payment_method      VARCHAR(20) NOT NULL CHECK (payment_method IN ('ach', 'wire', 'check', 'other')),
    currency_code       CHAR(3) NOT NULL DEFAULT 'USD',
    amount              DECIMAL(14,2) NOT NULL,
    exchange_rate       DECIMAL(12,6) DEFAULT 1.0,
    amount_in_base      DECIMAL(14,2) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'reversed')),
    bank_account_id     UUID REFERENCES contractor_bank_accounts(id),
    -- JSONB: payment processor response and tracking
    processing_details  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"external_ref": "ACH-TXN-12345", "processor": "stripe",
    --  "batch_id": "BATCH-001", "nacha_trace": "...",
    --  "failure_reason": null, "retry_count": 0}
    scheduled_date      DATE,
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_org ON payments(organization_id);
CREATE INDEX idx_payments_contractor ON payments(contractor_id);
CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_status ON payments(status);

CREATE TABLE tax_filings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contractor_id   UUID NOT NULL REFERENCES contractors(id),
    tax_year        INT NOT NULL,
    form_type       VARCHAR(20) NOT NULL,
    total_paid      DECIMAL(14,2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'generated', 'reviewed', 'filed', 'corrected', 'void')),
    -- JSONB: form data varies by type (1099-NEC vs 1042-S vs UK self-assessment)
    form_data       JSONB NOT NULL,
    -- 1099-NEC: {"payer_tin": "...", "recipient_tin": "...", "nonemployee_comp": 72000,
    --            "state_tax_withheld": 0, "state_id": "CA", "box_amounts": {...}}
    -- 1042-S:   {"income_code": "17", "tax_rate": 0.30, "gross_income": 50000, ...}

    -- JSONB: filing submission tracking
    submission      JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"iris_submission_id": "...", "submitted_at": "...", "accepted": true,
    --  "correction_count": 0, "last_correction_at": null}

    -- JSONB: reconciliation details
    reconciliation  JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"payment_ids": ["uuid1", "uuid2"], "total_from_payments": 72000,
    --  "total_from_invoices": 72000, "discrepancy": 0, "auto_reconciled": true}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, contractor_id, tax_year, form_type)
);

CREATE INDEX idx_tax_filings_year ON tax_filings(tax_year);
CREATE INDEX idx_tax_filings_status ON tax_filings(status);

-- ============================================================
-- COMPLIANCE & JURISDICTION
-- ============================================================

CREATE TABLE jurisdictions (
    code            VARCHAR(20) PRIMARY KEY,
    name            VARCHAR(100) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    region          VARCHAR(100),
    -- JSONB: compliance requirements are jurisdiction-specific by nature
    requirements    JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"required_documents": ["w9"], "classification_rules": "abc_test",
    --  "tax_forms": ["1099-NEC"], "withholding_required": false,
    --  "ir35_applicable": false, "minimum_notice_days": 14}
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    jurisdiction_code VARCHAR(20) NOT NULL REFERENCES jurisdictions(code),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(10) NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    -- JSONB: affected entities and remediation steps
    details         JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"affected_contractors": ["uuid1", "uuid2"], "regulation_ref": "AB-5",
    --  "effective_date": "2026-07-01", "remediation_steps": ["Reclassify ...", "Update ..."]}
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_org ON compliance_alerts(organization_id);
CREATE INDEX idx_alerts_severity ON compliance_alerts(severity);

-- ============================================================
-- AI CLASSIFICATION
-- ============================================================

CREATE TABLE classification_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contractor_id   UUID NOT NULL REFERENCES contractors(id),
    contract_id     UUID REFERENCES contracts(id),
    assessment_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_score   DECIMAL(5,2) NOT NULL,
    risk_level      VARCHAR(10) NOT NULL CHECK (risk_level IN ('low', 'medium', 'high', 'critical')),
    recommendation  VARCHAR(30) NOT NULL,
    assessed_by     VARCHAR(20) NOT NULL DEFAULT 'ai',
    -- JSONB: factor details (variable count and structure as model evolves)
    factors         JSONB NOT NULL,
    -- [{"name": "behavioral_control", "category": "irs_20_factor", "score": 90,
    --   "weight": 0.25, "evidence": "Contractor sets own hours"},
    --  {"name": "financial_control", "category": "irs_20_factor", "score": 82,
    --   "weight": 0.25, "evidence": "Uses own equipment"}]
    -- JSONB: model metadata for reproducibility
    model_metadata  JSONB NOT NULL DEFAULT '{}'::jsonb
    -- {"model_version": "v2.3", "feature_count": 28, "training_date": "2026-01-01"}
);

CREATE INDEX idx_assessments_contractor ON classification_assessments(contractor_id);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id         UUID,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create yearly partitions
CREATE TABLE audit_log_2026 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
CREATE TABLE audit_log_2027 PARTITION OF audit_log
    FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
CREATE INDEX idx_audit_changes ON audit_log USING GIN(changes jsonb_path_ops);

-- ============================================================
-- INTEGRATIONS & WEBHOOKS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'disconnected',
    -- JSONB: provider-specific config and sync state
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- QuickBooks: {"realm_id": "...", "last_sync": "...", "account_mapping": {...}}
    -- Xero:       {"tenant_id": "...", "last_sync": "...", "tracking_categories": [...]}
    credentials_ref VARCHAR(255),  -- reference to vault, never store inline
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    event_types     TEXT[] NOT NULL,
    target_url      VARCHAR(500) NOT NULL,
    secret_hash     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- JSONB: delivery config
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    -- {"retry_count": 3, "retry_delay_seconds": 60, "headers": {"X-Custom": "value"},
    --  "filter": {"contractor_country": "US"}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id),
    event_type      VARCHAR(50) NOT NULL,
    payload         JSONB NOT NULL,
    response_status INT,
    response_body   TEXT,
    attempts        INT NOT NULL DEFAULT 1,
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE webhook_deliveries_2026 PARTITION OF webhook_deliveries
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

---

## Query Examples

### Find all UK contractors needing IR35 assessment

```sql
SELECT id, first_name, last_name, tax_info
FROM contractors
WHERE country_code = 'GB'
  AND organization_id = $1
  AND tax_info @> '{"ir35_status": null}'::jsonb;
```

### Search talent pool by skill

```sql
SELECT id, first_name, last_name, profile->'skills' AS skills
FROM contractors
WHERE organization_id = $1
  AND profile @> '{"skills": [{"name": "React"}]}'::jsonb
  AND onboarding_status = 'approved';
```

### Find invoices with unresolved anomalies

```sql
SELECT id, invoice_number, total_amount, anomaly_results
FROM invoices
WHERE organization_id = $1
  AND anomaly_results @> '[{"resolved": false}]'::jsonb
  AND status IN ('submitted', 'under_review');
```

---

## Trade-offs

**Strengths:**
- Jurisdiction-specific compliance data, variable contract terms, and evolving AI model outputs all fit naturally in JSONB without schema migrations.
- GIN indexes on JSONB columns provide fast containment queries (`@>`) for searching nested structures.
- Fewer tables overall: timesheet entries, invoice line items, approval chains, and signature lists are embedded, reducing join complexity.
- PostgreSQL's JSONB is battle-tested with full transactional support -- no separate document store needed.
- Schema evolution for JSONB fields requires only application-level changes, not ALTER TABLE migrations.

**Weaknesses:**
- No database-level enforcement of JSONB structure. A malformed `tax_info` object will be accepted without complaint. Application-layer JSON Schema validation is essential.
- JSONB columns cannot have foreign key constraints. An `approver_id` inside an `approval_chain` array cannot reference the `users` table at the DB level.
- Complex aggregations across JSONB fields (e.g., summing all line item amounts across invoices) are possible but syntactically awkward and slower than columnar aggregation.
- ORM support for JSONB varies. Some frameworks handle it well (SQLAlchemy, Prisma), others require raw queries.

**Scalability:**
- Same as pure relational PostgreSQL for the relational columns; JSONB adds modest storage overhead.
- GIN indexes on JSONB can become large; partial indexes (on specific paths) help control index size.
- Partitioning high-volume tables (audit_log, webhook_deliveries) by date keeps query performance predictable.
- Consider materializing frequently-queried JSONB paths into generated columns if query patterns stabilize.

**Migration Path:**
- Individual JSONB columns can be "promoted" to relational columns via `ALTER TABLE ... ADD COLUMN` + backfill if a field becomes a primary query/join target.
- Can layer event sourcing (suggestion 2) on top by adding an event store table and publishing events from the application layer.
- The JSONB columns serve as a natural intermediate step if eventually migrating to a document store for specific aggregates.
