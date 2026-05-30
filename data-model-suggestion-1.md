# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity gets its own table with proper foreign keys, check constraints, and indexes. This is the bread-and-butter approach for transactional systems where data integrity, ACID compliance, and well-understood tooling matter most.

## Why This Suits a Freelancer Management System

Freelancer management is fundamentally a transactional domain: contracts are signed, invoices are submitted, payments are approved, tax forms are filed. Each of these operations involves multiple related entities that must stay consistent. A normalized relational model enforces referential integrity at the database level, preventing orphaned invoices, payments without contracts, or tax filings that reference deleted contractors. The domain also demands strong auditability -- 1099 filing, compliance tracking, and payment reconciliation all benefit from the rigid structure and constraint enforcement that a normalized schema provides.

The trade-off is verbosity. Queries that span onboarding, invoicing, and compliance often require multi-table joins. Flexible or jurisdiction-specific metadata (varying compliance fields per country) can be awkward to model without resorting to EAV patterns or many nullable columns.

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
    industry        VARCHAR(100),
    billing_email   VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    role            VARCHAR(30) NOT NULL CHECK (role IN ('admin', 'manager', 'finance', 'hr', 'viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_email ON users(email);

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
    tax_id              VARCHAR(50),
    tax_form_type       VARCHAR(20) CHECK (tax_form_type IN ('W-9', 'W-8BEN', 'W-8BEN-E', NULL)),
    tax_id_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    classification      VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (classification IN ('pending', 'independent_contractor', 'employee', 'flagged')),
    classification_score DECIMAL(5,2),
    onboarding_status   VARCHAR(30) NOT NULL DEFAULT 'invited'
                        CHECK (onboarding_status IN ('invited', 'in_progress', 'documents_pending',
                                                      'background_check', 'approved', 'rejected')),
    background_check_id VARCHAR(100),
    id_verified         BOOLEAN NOT NULL DEFAULT FALSE,
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

CREATE TABLE contractor_skills (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contractor_id   UUID NOT NULL REFERENCES contractors(id) ON DELETE CASCADE,
    skill_name      VARCHAR(100) NOT NULL,
    proficiency     VARCHAR(20) CHECK (proficiency IN ('beginner', 'intermediate', 'advanced', 'expert')),
    UNIQUE(contractor_id, skill_name)
);

CREATE INDEX idx_contractor_skills_name ON contractor_skills(skill_name);

CREATE TABLE contractor_bank_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contractor_id   UUID NOT NULL REFERENCES contractors(id) ON DELETE CASCADE,
    account_type    VARCHAR(20) NOT NULL CHECK (account_type IN ('ach', 'wire', 'iban')),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    bank_name       VARCHAR(255),
    routing_number  VARCHAR(30),
    account_number_encrypted BYTEA,
    iban_encrypted  BYTEA,
    swift_code      VARCHAR(11),
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    verified        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_bank_accounts_contractor ON contractor_bank_accounts(contractor_id);

-- ============================================================
-- DOCUMENTS & E-SIGNATURES
-- ============================================================

CREATE TABLE documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contractor_id   UUID REFERENCES contractors(id),
    document_type   VARCHAR(30) NOT NULL
                    CHECK (document_type IN ('w9', 'w8ben', 'contract', 'nda', 'ip_agreement',
                                              'id_verification', 'certificate', 'other')),
    file_name       VARCHAR(255) NOT NULL,
    file_path       VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_contractor ON documents(contractor_id);
CREATE INDEX idx_documents_type ON documents(document_type);

CREATE TABLE e_signatures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id     UUID NOT NULL REFERENCES documents(id),
    signer_type     VARCHAR(20) NOT NULL CHECK (signer_type IN ('contractor', 'organization')),
    signer_id       UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'sent', 'viewed', 'signed', 'declined', 'expired')),
    signed_at       TIMESTAMPTZ,
    ip_address      INET,
    signature_hash  VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esignatures_document ON e_signatures(document_id);
CREATE INDEX idx_esignatures_status ON e_signatures(status);

-- ============================================================
-- CONTRACTS
-- ============================================================

CREATE TABLE contract_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    jurisdiction    VARCHAR(50) NOT NULL,
    template_body   TEXT NOT NULL,
    auto_clauses    TEXT[],
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
    rate_amount     DECIMAL(12,2),
    rate_unit       VARCHAR(20) CHECK (rate_unit IN ('hour', 'day', 'week', 'month', 'project', 'milestone')),
    total_budget    DECIMAL(14,2),
    start_date      DATE NOT NULL,
    end_date        DATE,
    jurisdiction    VARCHAR(50) NOT NULL,
    auto_renew      BOOLEAN NOT NULL DEFAULT FALSE,
    notice_period_days INT DEFAULT 30,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_org ON contracts(organization_id);
CREATE INDEX idx_contracts_contractor ON contracts(contractor_id);
CREATE INDEX idx_contracts_status ON contracts(status);
CREATE INDEX idx_contracts_dates ON contracts(start_date, end_date);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projects_org ON projects(organization_id);

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
    submitted_at    TIMESTAMPTZ,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timesheets_contractor ON timesheets(contractor_id);
CREATE INDEX idx_timesheets_contract ON timesheets(contract_id);
CREATE INDEX idx_timesheets_period ON timesheets(period_start, period_end);

CREATE TABLE timesheet_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timesheet_id    UUID NOT NULL REFERENCES timesheets(id) ON DELETE CASCADE,
    work_date       DATE NOT NULL,
    hours           DECIMAL(5,2) NOT NULL CHECK (hours > 0 AND hours <= 24),
    description     VARCHAR(500),
    task_reference  VARCHAR(100)
);

CREATE INDEX idx_timesheet_entries_sheet ON timesheet_entries(timesheet_id);

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
    submitted_at        TIMESTAMPTZ,
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    anomaly_flag        BOOLEAN NOT NULL DEFAULT FALSE,
    anomaly_reason      VARCHAR(500),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, invoice_number)
);

CREATE INDEX idx_invoices_org ON invoices(organization_id);
CREATE INDEX idx_invoices_contractor ON invoices(contractor_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due ON invoices(due_date);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,
    quantity        DECIMAL(10,2) NOT NULL,
    unit_price      DECIMAL(12,2) NOT NULL,
    amount          DECIMAL(14,2) NOT NULL,
    timesheet_id    UUID REFERENCES timesheets(id),
    milestone_id    UUID REFERENCES milestones(id),
    sort_order      INT NOT NULL DEFAULT 0
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

CREATE TABLE approval_workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(30) NOT NULL CHECK (entity_type IN ('invoice', 'timesheet', 'contract', 'expense')),
    min_amount      DECIMAL(14,2),
    max_amount      DECIMAL(14,2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id) ON DELETE CASCADE,
    step_order      INT NOT NULL,
    approver_role   VARCHAR(30) NOT NULL,
    approver_id     UUID REFERENCES users(id),
    UNIQUE(workflow_id, step_order)
);

CREATE TABLE approval_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES approval_workflows(id),
    entity_type     VARCHAR(30) NOT NULL,
    entity_id       UUID NOT NULL,
    step_order      INT NOT NULL,
    approver_id     UUID NOT NULL REFERENCES users(id),
    decision        VARCHAR(20) NOT NULL CHECK (decision IN ('approved', 'rejected', 'returned')),
    comment         TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_approval_records_entity ON approval_records(entity_type, entity_id);

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
    external_ref        VARCHAR(255),
    scheduled_date      DATE,
    processed_at        TIMESTAMPTZ,
    failure_reason      VARCHAR(500),
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
    form_type       VARCHAR(20) NOT NULL CHECK (form_type IN ('1099-NEC', '1099-MISC', '1042-S')),
    total_paid      DECIMAL(14,2) NOT NULL,
    nonemployee_comp DECIMAL(14,2),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'generated', 'reviewed', 'filed', 'corrected', 'void')),
    iris_submission_id VARCHAR(100),
    filed_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, contractor_id, tax_year, form_type)
);

CREATE INDEX idx_tax_filings_year ON tax_filings(tax_year);

-- ============================================================
-- COMPLIANCE & JURISDICTION RULES
-- ============================================================

CREATE TABLE jurisdictions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(20) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    country_code    CHAR(2) NOT NULL,
    region          VARCHAR(100),
    compliance_notes TEXT
);

CREATE TABLE compliance_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    rule_type       VARCHAR(50) NOT NULL,
    rule_key        VARCHAR(100) NOT NULL,
    rule_value      TEXT NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(jurisdiction_id, rule_type, rule_key, effective_from)
);

CREATE TABLE compliance_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    jurisdiction_id UUID NOT NULL REFERENCES jurisdictions(id),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(10) NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_alerts_org ON compliance_alerts(organization_id);

-- ============================================================
-- AI CLASSIFICATION & ANOMALY DETECTION
-- ============================================================

CREATE TABLE classification_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contractor_id   UUID NOT NULL REFERENCES contractors(id),
    contract_id     UUID REFERENCES contracts(id),
    assessment_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_score   DECIMAL(5,2) NOT NULL,
    risk_level      VARCHAR(10) NOT NULL CHECK (risk_level IN ('low', 'medium', 'high', 'critical')),
    recommendation  VARCHAR(30) NOT NULL
                    CHECK (recommendation IN ('independent_contractor', 'employee', 'review_needed')),
    assessed_by     VARCHAR(20) NOT NULL DEFAULT 'ai' CHECK (assessed_by IN ('ai', 'manual', 'hybrid'))
);

CREATE TABLE classification_factors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id   UUID NOT NULL REFERENCES classification_assessments(id) ON DELETE CASCADE,
    factor_name     VARCHAR(100) NOT NULL,
    factor_category VARCHAR(50) NOT NULL,
    score           DECIMAL(5,2) NOT NULL,
    weight          DECIMAL(5,2) NOT NULL,
    evidence        TEXT
);

CREATE TABLE invoice_anomalies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    anomaly_type    VARCHAR(50) NOT NULL
                    CHECK (anomaly_type IN ('duplicate', 'inflated_hours', 'off_contract',
                                            'unusual_amount', 'frequency_spike', 'other')),
    confidence      DECIMAL(5,2) NOT NULL,
    description     TEXT NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT FALSE,
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomalies_invoice ON invoice_anomalies(invoice_id);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(30) NOT NULL,
    changes         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log(organization_id);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);

-- ============================================================
-- INTEGRATIONS & WEBHOOKS
-- ============================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        VARCHAR(50) NOT NULL CHECK (provider IN ('quickbooks', 'xero', 'freshbooks', 'custom')),
    status          VARCHAR(20) NOT NULL DEFAULT 'disconnected'
                    CHECK (status IN ('connected', 'disconnected', 'error')),
    credentials_encrypted BYTEA,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    event_type      VARCHAR(50) NOT NULL,
    target_url      VARCHAR(500) NOT NULL,
    secret_hash     VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
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
);

CREATE INDEX idx_webhook_deliveries_sub ON webhook_deliveries(subscription_id);
```

---

## Trade-offs

**Strengths:**
- Full referential integrity enforced at database level -- no orphaned records possible.
- Well-understood by every backend developer; enormous ecosystem of ORMs, migration tools, and admin panels.
- Straightforward compliance auditing: every relationship is explicit and queryable with standard SQL.
- PostgreSQL's mature ACID transactions perfectly suit payment and tax-filing workflows where partial writes are unacceptable.

**Weaknesses:**
- Jurisdiction-specific compliance fields require either nullable columns (sparse) or an EAV sub-table (complexity).
- Multi-table joins for reporting queries (e.g., "show me all contractors, their contracts, invoices, payments, and tax filings for Q3") can become expensive at scale.
- Schema changes (adding a new compliance field, new document type) require migrations, which can be operationally heavy on large tables.

**Scalability:**
- PostgreSQL handles millions of rows per table comfortably with proper indexing.
- Partitioning `audit_log`, `payments`, and `webhook_deliveries` by date is recommended once volumes exceed ~50M rows.
- Read replicas can offload reporting workloads.

**Migration Path:**
- This schema can evolve toward the hybrid JSONB approach (suggestion 3) by adding JSONB columns to existing tables for flexible metadata -- no rewrite needed.
- Event sourcing (suggestion 2) can be layered on top by adding an event store table and publishing domain events alongside relational writes.
