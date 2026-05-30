# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Anti-Money Laundering Platform (Candidate #432)
> Date: 2026-05-25

## Overview

This model uses a fully normalized relational schema in PostgreSQL, designed around the six core domains of an AML platform: party management (KYC/CDD), account and instrument tracking, transaction monitoring, alert and case management, sanctions screening, and regulatory reporting. Every table enforces referential integrity, and temporal columns support the bitemporal auditing that regulators require.

PostgreSQL is chosen for its ACID compliance, mature partitioning (critical for multi-billion-row transaction tables), row-level security (multi-tenant data isolation), and the broadest ecosystem of extensions relevant to AML workloads (pg_trgm for fuzzy name matching, pgcrypto for field-level encryption, pg_partman for automated partition management).

---

## Schema Design Principles

1. **Bitemporal tracking**: Core entities carry both `valid_from`/`valid_to` (business time) and `recorded_at` (system time) to satisfy regulatory requirements for point-in-time reconstruction.
2. **Immutable audit trail**: All state changes are recorded in dedicated `_audit` tables using INSERT-only patterns. No UPDATE or DELETE on audit tables.
3. **Soft deletes**: Operational tables use `is_active` flags rather than physical deletes, preserving data for regulatory retention periods (typically 5-7 years post-relationship termination).
4. **UUID primary keys**: All primary keys use `uuid` to support distributed ingestion and avoid sequence contention at high throughput.
5. **ISO 20022 alignment**: Transaction and party field naming aligns with ISO 20022 MX message elements where applicable, reducing impedance mismatch during ingestion from SWIFT/payment rails.

---

## Complete Schema

### Domain 1: Party Management (KYC/CDD)

```sql
-- Parties: individuals and organisations subject to CDD
CREATE TABLE party (
    party_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_type          VARCHAR(20) NOT NULL CHECK (party_type IN ('INDIVIDUAL', 'ORGANISATION', 'TRUST', 'GOVERNMENT')),
    legal_name          TEXT NOT NULL,
    trading_name        TEXT,
    date_of_birth       DATE,                         -- individuals only
    date_of_incorporation DATE,                       -- organisations only
    country_of_residence VARCHAR(3),                  -- ISO 3166-1 alpha-3
    country_of_incorporation VARCHAR(3),
    nationality         VARCHAR(3),
    tax_id              TEXT,                          -- encrypted at rest
    lei                 VARCHAR(20),                   -- Legal Entity Identifier (ISO 17442)
    risk_rating         VARCHAR(10) NOT NULL DEFAULT 'MEDIUM' CHECK (risk_rating IN ('LOW', 'MEDIUM', 'HIGH', 'VERY_HIGH', 'PROHIBITED')),
    risk_rating_reason  TEXT,
    cdd_level           VARCHAR(20) NOT NULL DEFAULT 'STANDARD' CHECK (cdd_level IN ('SIMPLIFIED', 'STANDARD', 'ENHANCED')),
    pep_status          BOOLEAN NOT NULL DEFAULT FALSE,
    pep_category        VARCHAR(50),
    source_of_wealth    TEXT,
    source_of_funds     TEXT,
    occupation          TEXT,
    industry_code       VARCHAR(10),                  -- NAICS or SIC
    onboarding_date     DATE NOT NULL,
    last_review_date    DATE,
    next_review_date    DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL,
    updated_by          UUID
);

CREATE INDEX idx_party_legal_name_trgm ON party USING gin (legal_name gin_trgm_ops);
CREATE INDEX idx_party_risk_rating ON party (risk_rating);
CREATE INDEX idx_party_pep ON party (pep_status) WHERE pep_status = TRUE;
CREATE INDEX idx_party_country ON party (country_of_residence);
CREATE INDEX idx_party_next_review ON party (next_review_date) WHERE is_active = TRUE;

-- Party aliases for fuzzy matching and sanctions screening
CREATE TABLE party_alias (
    alias_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id            UUID NOT NULL REFERENCES party(party_id),
    alias_type          VARCHAR(20) NOT NULL CHECK (alias_type IN ('AKA', 'FKA', 'DBA', 'MAIDEN', 'TRANSLITERATION', 'ABBREVIATION')),
    alias_name          TEXT NOT NULL,
    script              VARCHAR(10),                  -- ISO 15924 script code
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alias_name_trgm ON party_alias USING gin (alias_name gin_trgm_ops);
CREATE INDEX idx_alias_party ON party_alias (party_id);

-- Party addresses (multiple per party, temporal)
CREATE TABLE party_address (
    address_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id            UUID NOT NULL REFERENCES party(party_id),
    address_type        VARCHAR(20) NOT NULL CHECK (address_type IN ('REGISTERED', 'TRADING', 'RESIDENTIAL', 'MAILING', 'OTHER')),
    address_line_1      TEXT NOT NULL,
    address_line_2      TEXT,
    city                TEXT NOT NULL,
    state_province      TEXT,
    postal_code         TEXT,
    country             VARCHAR(3) NOT NULL,          -- ISO 3166-1 alpha-3
    is_current          BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_address_party ON party_address (party_id);
CREATE INDEX idx_address_country ON party_address (country);

-- Party identification documents
CREATE TABLE party_identification (
    identification_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    party_id            UUID NOT NULL REFERENCES party(party_id),
    id_type             VARCHAR(30) NOT NULL CHECK (id_type IN ('PASSPORT', 'NATIONAL_ID', 'DRIVERS_LICENCE', 'TAX_ID', 'COMPANY_REGISTRATION', 'LEI', 'BIC', 'OTHER')),
    id_number           TEXT NOT NULL,                -- encrypted at rest
    issuing_country     VARCHAR(3) NOT NULL,
    issue_date          DATE,
    expiry_date         DATE,
    verification_status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (verification_status IN ('PENDING', 'VERIFIED', 'FAILED', 'EXPIRED')),
    verified_at         TIMESTAMPTZ,
    verified_by         UUID,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_identification_party ON party_identification (party_id);

-- Relationships between parties (beneficial ownership, directorships, etc.)
CREATE TABLE party_relationship (
    relationship_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_party_id       UUID NOT NULL REFERENCES party(party_id),
    to_party_id         UUID NOT NULL REFERENCES party(party_id),
    relationship_type   VARCHAR(30) NOT NULL CHECK (relationship_type IN (
        'BENEFICIAL_OWNER', 'DIRECTOR', 'SHAREHOLDER', 'AUTHORIZED_SIGNATORY',
        'SPOUSE', 'PARENT', 'CHILD', 'BUSINESS_PARTNER', 'EMPLOYER',
        'AGENT', 'TRUSTEE', 'BENEFICIARY', 'NOMINEE', 'OTHER'
    )),
    ownership_percentage DECIMAL(5,2),                -- for BENEFICIAL_OWNER / SHAREHOLDER
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_relationship_from ON party_relationship (from_party_id);
CREATE INDEX idx_relationship_to ON party_relationship (to_party_id);
CREATE INDEX idx_relationship_type ON party_relationship (relationship_type);
```

### Domain 2: Accounts and Instruments

```sql
-- Financial accounts held by parties
CREATE TABLE account (
    account_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number      TEXT NOT NULL,                -- encrypted at rest
    account_type        VARCHAR(30) NOT NULL CHECK (account_type IN (
        'CURRENT', 'SAVINGS', 'LOAN', 'CREDIT_CARD', 'BROKERAGE',
        'CUSTODIAL', 'TRUST', 'CORRESPONDENT', 'NOSTRO', 'VOSTRO',
        'WALLET', 'PREPAID', 'OTHER'
    )),
    currency            VARCHAR(3) NOT NULL,          -- ISO 4217
    institution_id      UUID NOT NULL REFERENCES party(party_id),
    branch_code         VARCHAR(20),
    iban                VARCHAR(34),
    bic                 VARCHAR(11),
    opened_date         DATE NOT NULL,
    closed_date         DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'DORMANT', 'SUSPENDED', 'CLOSED', 'FROZEN')),
    risk_rating         VARCHAR(10) NOT NULL DEFAULT 'MEDIUM' CHECK (risk_rating IN ('LOW', 'MEDIUM', 'HIGH', 'VERY_HIGH', 'PROHIBITED')),
    is_correspondent    BOOLEAN NOT NULL DEFAULT FALSE,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL,
    updated_by          UUID
);

CREATE INDEX idx_account_status ON account (status);
CREATE INDEX idx_account_institution ON account (institution_id);
CREATE INDEX idx_account_risk ON account (risk_rating);

-- Many-to-many: parties linked to accounts with roles
CREATE TABLE account_party (
    account_party_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id          UUID NOT NULL REFERENCES account(account_id),
    party_id            UUID NOT NULL REFERENCES party(party_id),
    role                VARCHAR(30) NOT NULL CHECK (role IN ('HOLDER', 'JOINT_HOLDER', 'AUTHORIZED_SIGNATORY', 'BENEFICIAL_OWNER', 'POWER_OF_ATTORNEY', 'CUSTODIAN')),
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (account_id, party_id, role)
);

CREATE INDEX idx_account_party_account ON account_party (account_id);
CREATE INDEX idx_account_party_party ON account_party (party_id);
```

### Domain 3: Transactions

```sql
-- Financial transactions (partitioned by transaction_date for performance)
CREATE TABLE transaction (
    transaction_id      UUID NOT NULL DEFAULT gen_random_uuid(),
    external_ref        TEXT,                          -- source system reference
    transaction_date    TIMESTAMPTZ NOT NULL,
    value_date          DATE,
    transaction_type    VARCHAR(30) NOT NULL CHECK (transaction_type IN (
        'WIRE_TRANSFER', 'ACH', 'SEPA', 'FASTER_PAYMENT', 'CARD_PAYMENT',
        'CARD_WITHDRAWAL', 'CASH_DEPOSIT', 'CASH_WITHDRAWAL', 'CHECK',
        'INTERNAL_TRANSFER', 'CRYPTO_TRANSFER', 'TRADE', 'FEE', 'INTEREST',
        'LOAN_DISBURSEMENT', 'LOAN_REPAYMENT', 'OTHER'
    )),
    direction           VARCHAR(10) NOT NULL CHECK (direction IN ('INBOUND', 'OUTBOUND', 'INTERNAL')),
    amount              DECIMAL(18,4) NOT NULL,
    currency            VARCHAR(3) NOT NULL,           -- ISO 4217
    amount_usd          DECIMAL(18,4),                 -- normalised for threshold checks
    exchange_rate       DECIMAL(18,8),

    -- Originator
    originator_account_id   UUID REFERENCES account(account_id),
    originator_party_id     UUID REFERENCES party(party_id),
    originator_name         TEXT,
    originator_country      VARCHAR(3),
    originator_bic          VARCHAR(11),

    -- Beneficiary
    beneficiary_account_id  UUID REFERENCES account(account_id),
    beneficiary_party_id    UUID REFERENCES party(party_id),
    beneficiary_name        TEXT,
    beneficiary_country     VARCHAR(3),
    beneficiary_bic         VARCHAR(11),

    -- Intermediary (correspondent banking)
    intermediary_bic        VARCHAR(11),
    intermediary_name       TEXT,
    intermediary_country    VARCHAR(3),

    -- Payment rail metadata
    payment_rail        VARCHAR(30),
    payment_channel     VARCHAR(20) CHECK (payment_channel IN ('BRANCH', 'ONLINE', 'MOBILE', 'ATM', 'API', 'BATCH', 'SWIFT', 'OTHER')),
    remittance_info     TEXT,
    end_to_end_id       TEXT,                          -- ISO 20022 EndToEndId
    instruction_id      TEXT,                          -- ISO 20022 InstrId

    -- Risk scoring (populated by monitoring engine)
    risk_score          DECIMAL(5,2),
    risk_factors        TEXT[],                         -- array of triggered rule/model codes

    -- Metadata
    ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    source_system       VARCHAR(50) NOT NULL,
    batch_id            UUID,
    is_reversal         BOOLEAN NOT NULL DEFAULT FALSE,
    reversed_transaction_id UUID,

    PRIMARY KEY (transaction_id, transaction_date)
) PARTITION BY RANGE (transaction_date);

-- Create monthly partitions (example for 2026)
CREATE TABLE transaction_2026_01 PARTITION OF transaction FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE transaction_2026_02 PARTITION OF transaction FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE transaction_2026_03 PARTITION OF transaction FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
-- ... additional partitions created by pg_partman

CREATE INDEX idx_txn_originator_account ON transaction (originator_account_id, transaction_date);
CREATE INDEX idx_txn_beneficiary_account ON transaction (beneficiary_account_id, transaction_date);
CREATE INDEX idx_txn_originator_party ON transaction (originator_party_id, transaction_date);
CREATE INDEX idx_txn_beneficiary_party ON transaction (beneficiary_party_id, transaction_date);
CREATE INDEX idx_txn_risk_score ON transaction (risk_score DESC, transaction_date) WHERE risk_score IS NOT NULL;
CREATE INDEX idx_txn_amount_usd ON transaction (amount_usd, transaction_date);
CREATE INDEX idx_txn_type ON transaction (transaction_type, transaction_date);
CREATE INDEX idx_txn_source ON transaction (source_system, transaction_date);
CREATE INDEX idx_txn_external_ref ON transaction (external_ref, transaction_date);

-- Materialized view: daily entity transaction aggregates for behavioural profiling
CREATE MATERIALIZED VIEW mv_entity_daily_stats AS
SELECT
    COALESCE(originator_party_id, beneficiary_party_id) AS party_id,
    transaction_date::date AS txn_date,
    transaction_type,
    direction,
    COUNT(*) AS txn_count,
    SUM(amount_usd) AS total_amount_usd,
    AVG(amount_usd) AS avg_amount_usd,
    MAX(amount_usd) AS max_amount_usd,
    COUNT(DISTINCT CASE WHEN direction = 'OUTBOUND' THEN beneficiary_country END) AS distinct_dest_countries,
    COUNT(DISTINCT CASE WHEN direction = 'OUTBOUND' THEN beneficiary_party_id END) AS distinct_counterparties
FROM transaction
WHERE originator_party_id IS NOT NULL OR beneficiary_party_id IS NOT NULL
GROUP BY 1, 2, 3, 4;

CREATE UNIQUE INDEX idx_mv_entity_daily ON mv_entity_daily_stats (party_id, txn_date, transaction_type, direction);
```

### Domain 4: Detection Rules and Monitoring

```sql
-- Detection rule definitions
CREATE TABLE detection_rule (
    rule_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_code           VARCHAR(50) NOT NULL UNIQUE,
    rule_name           TEXT NOT NULL,
    rule_version        INTEGER NOT NULL DEFAULT 1,
    typology            VARCHAR(50) NOT NULL CHECK (typology IN (
        'STRUCTURING', 'LAYERING', 'SMURFING', 'MULE_ACCOUNT',
        'VELOCITY', 'DORMANT_ACTIVATION', 'HIGH_RISK_JURISDICTION',
        'ROUND_TRIPPING', 'TRADE_BASED', 'CASH_INTENSIVE',
        'THRESHOLD', 'BEHAVIORAL_ANOMALY', 'SANCTIONS_EVASION',
        'PEP_TRANSACTION', 'CUSTOM'
    )),
    description         TEXT NOT NULL,
    severity            VARCHAR(10) NOT NULL CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    rule_logic          TEXT NOT NULL,                  -- rule definition (DSL or SQL predicate)
    parameters          JSONB NOT NULL DEFAULT '{}',    -- configurable thresholds
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    effective_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    effective_to        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL,
    approved_by         UUID,
    approved_at         TIMESTAMPTZ
);

CREATE INDEX idx_rule_typology ON detection_rule (typology);
CREATE INDEX idx_rule_active ON detection_rule (is_active) WHERE is_active = TRUE;

-- ML model registry for model governance (SR 11-7)
CREATE TABLE ml_model (
    model_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name          TEXT NOT NULL,
    model_version       VARCHAR(20) NOT NULL,
    model_type          VARCHAR(30) NOT NULL CHECK (model_type IN (
        'ANOMALY_DETECTION', 'CLASSIFICATION', 'CLUSTERING',
        'ENTITY_RESOLUTION', 'NETWORK_ANALYSIS', 'NLP_NARRATIVE',
        'TYPOLOGY_DISCOVERY'
    )),
    description         TEXT NOT NULL,
    framework           VARCHAR(30),                   -- e.g. 'XGBOOST', 'PYTORCH', 'SKLEARN'
    artifact_path       TEXT NOT NULL,                 -- path to serialized model
    training_date       TIMESTAMPTZ NOT NULL,
    training_data_desc  TEXT NOT NULL,                  -- description of training dataset
    performance_metrics JSONB NOT NULL,                 -- precision, recall, F1, AUC, etc.
    validation_status   VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (validation_status IN ('PENDING', 'VALIDATED', 'REJECTED', 'RETIRED')),
    validated_by        UUID,
    validated_at        TIMESTAMPTZ,
    next_validation_date DATE,
    is_active           BOOLEAN NOT NULL DEFAULT FALSE,
    deployed_at         TIMESTAMPTZ,
    retired_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL
);

CREATE INDEX idx_model_active ON ml_model (is_active) WHERE is_active = TRUE;
CREATE INDEX idx_model_validation ON ml_model (next_validation_date) WHERE validation_status = 'VALIDATED';
```

### Domain 5: Alerts and Case Management

```sql
-- Alerts generated by detection engine
CREATE TABLE alert (
    alert_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_number        BIGINT GENERATED ALWAYS AS IDENTITY,
    alert_date          TIMESTAMPTZ NOT NULL DEFAULT now(),
    alert_source        VARCHAR(20) NOT NULL CHECK (alert_source IN ('RULE', 'MODEL', 'SCREENING', 'MANUAL', 'EXTERNAL')),
    rule_id             UUID REFERENCES detection_rule(rule_id),
    model_id            UUID REFERENCES ml_model(model_id),
    party_id            UUID NOT NULL REFERENCES party(party_id),
    account_id          UUID REFERENCES account(account_id),
    risk_score          DECIMAL(5,2) NOT NULL,
    severity            VARCHAR(10) NOT NULL CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    status              VARCHAR(20) NOT NULL DEFAULT 'NEW' CHECK (status IN (
        'NEW', 'ASSIGNED', 'UNDER_REVIEW', 'ESCALATED',
        'CLOSED_FALSE_POSITIVE', 'CLOSED_TRUE_POSITIVE', 'CLOSED_INCONCLUSIVE',
        'SAR_FILED'
    )),
    typology            VARCHAR(50),
    summary             TEXT NOT NULL,
    explanation         TEXT,                           -- AI explainability output
    assigned_to         UUID REFERENCES analyst(analyst_id),
    assigned_at         TIMESTAMPTZ,
    due_date            TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    disposition_reason  TEXT,
    case_id             UUID REFERENCES investigation_case(case_id),
    is_bulk_dispositioned BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_status ON alert (status);
CREATE INDEX idx_alert_party ON alert (party_id);
CREATE INDEX idx_alert_assigned ON alert (assigned_to, status) WHERE status IN ('ASSIGNED', 'UNDER_REVIEW');
CREATE INDEX idx_alert_risk ON alert (risk_score DESC) WHERE status = 'NEW';
CREATE INDEX idx_alert_date ON alert (alert_date);
CREATE INDEX idx_alert_case ON alert (case_id) WHERE case_id IS NOT NULL;

-- Transactions linked to an alert
CREATE TABLE alert_transaction (
    alert_transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id            UUID NOT NULL REFERENCES alert(alert_id),
    transaction_id      UUID NOT NULL,
    transaction_date    TIMESTAMPTZ NOT NULL,          -- needed for partition pruning
    relevance_score     DECIMAL(5,2),
    UNIQUE (alert_id, transaction_id)
);

CREATE INDEX idx_alert_txn_alert ON alert_transaction (alert_id);

-- Analysts / compliance officers
CREATE TABLE analyst (
    analyst_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL,                 -- FK to auth system
    employee_id         VARCHAR(50),
    full_name           TEXT NOT NULL,
    email               TEXT NOT NULL,
    role                VARCHAR(30) NOT NULL CHECK (role IN ('L1_ANALYST', 'L2_ANALYST', 'SENIOR_ANALYST', 'TEAM_LEAD', 'COMPLIANCE_OFFICER', 'MLRO', 'ADMIN')),
    team                VARCHAR(50),
    max_concurrent_cases INTEGER NOT NULL DEFAULT 20,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analyst_role ON analyst (role);
CREATE INDEX idx_analyst_active ON analyst (is_active) WHERE is_active = TRUE;

-- Investigation cases
CREATE TABLE investigation_case (
    case_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_number         BIGINT GENERATED ALWAYS AS IDENTITY,
    case_type           VARCHAR(30) NOT NULL CHECK (case_type IN (
        'SUSPICIOUS_ACTIVITY', 'SANCTIONS_HIT', 'PEP_REVIEW',
        'PERIODIC_REVIEW', 'REGULATORY_REQUEST', 'TIP_OFF', 'OTHER'
    )),
    status              VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN (
        'OPEN', 'ASSIGNED', 'UNDER_INVESTIGATION', 'ESCALATED',
        'PENDING_SAR', 'SAR_FILED', 'CLOSED_NO_ACTION', 'CLOSED_SAR_FILED',
        'CLOSED_REFERRED', 'REOPENED'
    )),
    priority            VARCHAR(10) NOT NULL DEFAULT 'MEDIUM' CHECK (priority IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    subject_party_id    UUID NOT NULL REFERENCES party(party_id),
    summary             TEXT NOT NULL,
    findings            TEXT,
    recommendation      TEXT,
    assigned_to         UUID REFERENCES analyst(analyst_id),
    assigned_at         TIMESTAMPTZ,
    escalated_to        UUID REFERENCES analyst(analyst_id),
    escalated_at        TIMESTAMPTZ,
    due_date            TIMESTAMPTZ,
    opened_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at           TIMESTAMPTZ,
    closure_reason      TEXT,
    sar_id              UUID,                          -- set when SAR is filed
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL
);

CREATE INDEX idx_case_status ON investigation_case (status);
CREATE INDEX idx_case_assigned ON investigation_case (assigned_to, status);
CREATE INDEX idx_case_party ON investigation_case (subject_party_id);
CREATE INDEX idx_case_priority ON investigation_case (priority, status) WHERE status NOT LIKE 'CLOSED%';

-- Case activity log (immutable audit trail)
CREATE TABLE case_activity (
    activity_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id             UUID NOT NULL REFERENCES investigation_case(case_id),
    activity_type       VARCHAR(30) NOT NULL CHECK (activity_type IN (
        'CREATED', 'ASSIGNED', 'REASSIGNED', 'NOTE_ADDED', 'EVIDENCE_ATTACHED',
        'STATUS_CHANGED', 'ESCALATED', 'DE_ESCALATED', 'SAR_DRAFTED',
        'SAR_REVIEWED', 'SAR_SUBMITTED', 'CLOSED', 'REOPENED',
        'ALERT_LINKED', 'ALERT_UNLINKED'
    )),
    description         TEXT NOT NULL,
    old_value           TEXT,
    new_value           TEXT,
    performed_by        UUID NOT NULL,
    performed_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_case_activity_case ON case_activity (case_id, performed_at);
CREATE INDEX idx_case_activity_analyst ON case_activity (performed_by, performed_at);

-- Evidence / document attachments
CREATE TABLE case_evidence (
    evidence_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id             UUID NOT NULL REFERENCES investigation_case(case_id),
    evidence_type       VARCHAR(30) NOT NULL CHECK (evidence_type IN (
        'DOCUMENT', 'SCREENSHOT', 'TRANSACTION_EXTRACT', 'SCREENING_RESULT',
        'NETWORK_DIAGRAM', 'EXTERNAL_REPORT', 'CORRESPONDENCE', 'OTHER'
    )),
    file_name           TEXT NOT NULL,
    file_path           TEXT NOT NULL,                  -- object storage path
    file_size_bytes     BIGINT NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    description         TEXT,
    uploaded_by         UUID NOT NULL,
    uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    sha256_hash         VARCHAR(64) NOT NULL            -- integrity verification
);

CREATE INDEX idx_evidence_case ON case_evidence (case_id);
```

### Domain 6: Sanctions and Watchlist Screening

```sql
-- Watchlist sources (OFAC, UN, EU, etc.)
CREATE TABLE watchlist_source (
    source_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_code         VARCHAR(30) NOT NULL UNIQUE,
    source_name         TEXT NOT NULL,
    source_url          TEXT,
    list_type           VARCHAR(20) NOT NULL CHECK (list_type IN ('SANCTIONS', 'PEP', 'ADVERSE_MEDIA', 'LAW_ENFORCEMENT', 'CUSTOM')),
    update_frequency    VARCHAR(20) NOT NULL,          -- e.g. 'DAILY', 'REAL_TIME'
    last_updated_at     TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Watchlist entries (denormalized for screening performance)
CREATE TABLE watchlist_entry (
    entry_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id           UUID NOT NULL REFERENCES watchlist_source(source_id),
    source_entry_id     TEXT NOT NULL,                 -- e.g. OFAC UID
    entry_type          VARCHAR(20) NOT NULL CHECK (entry_type IN ('INDIVIDUAL', 'ENTITY', 'VESSEL', 'AIRCRAFT')),
    primary_name        TEXT NOT NULL,
    program             TEXT,                          -- e.g. 'SDGT', 'IRAN', 'UKRAINE-EO13662'
    remarks             TEXT,
    listed_date         DATE,
    delisted_date       DATE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, source_entry_id)
);

CREATE INDEX idx_watchlist_name_trgm ON watchlist_entry USING gin (primary_name gin_trgm_ops);
CREATE INDEX idx_watchlist_source ON watchlist_entry (source_id);
CREATE INDEX idx_watchlist_active ON watchlist_entry (is_active) WHERE is_active = TRUE;

-- Watchlist entry aliases
CREATE TABLE watchlist_alias (
    alias_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id            UUID NOT NULL REFERENCES watchlist_entry(entry_id),
    alias_name          TEXT NOT NULL,
    alias_type          VARCHAR(20),
    script              VARCHAR(10),                   -- ISO 15924
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wl_alias_name_trgm ON watchlist_alias USING gin (alias_name gin_trgm_ops);
CREATE INDEX idx_wl_alias_entry ON watchlist_alias (entry_id);

-- Watchlist entry identifiers (passports, national IDs, etc.)
CREATE TABLE watchlist_identifier (
    identifier_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_id            UUID NOT NULL REFERENCES watchlist_entry(entry_id),
    id_type             VARCHAR(30) NOT NULL,
    id_number           TEXT NOT NULL,
    issuing_country     VARCHAR(3),
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wl_identifier_entry ON watchlist_identifier (entry_id);
CREATE INDEX idx_wl_identifier_number ON watchlist_identifier (id_number);

-- Screening results
CREATE TABLE screening_result (
    result_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_type      VARCHAR(20) NOT NULL CHECK (screening_type IN ('ONBOARDING', 'ONGOING', 'TRANSACTION', 'BATCH', 'MANUAL')),
    screened_party_id   UUID REFERENCES party(party_id),
    screened_name       TEXT NOT NULL,
    matched_entry_id    UUID REFERENCES watchlist_entry(entry_id),
    match_score         DECIMAL(5,4) NOT NULL,         -- 0.0000 to 1.0000
    match_algorithm     VARCHAR(30) NOT NULL,          -- 'JARO_WINKLER', 'LEVENSHTEIN', 'SOUNDEX', 'EXACT'
    status              VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'TRUE_MATCH', 'FALSE_POSITIVE', 'ESCALATED')),
    reviewed_by         UUID REFERENCES analyst(analyst_id),
    reviewed_at         TIMESTAMPTZ,
    review_notes        TEXT,
    alert_id            UUID REFERENCES alert(alert_id),
    screened_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_party ON screening_result (screened_party_id);
CREATE INDEX idx_screening_status ON screening_result (status) WHERE status = 'PENDING';
CREATE INDEX idx_screening_date ON screening_result (screened_at);
CREATE INDEX idx_screening_match ON screening_result (matched_entry_id);
```

### Domain 7: Regulatory Reporting

```sql
-- Suspicious Activity Reports
CREATE TABLE sar_report (
    sar_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sar_number          BIGINT GENERATED ALWAYS AS IDENTITY,
    case_id             UUID NOT NULL REFERENCES investigation_case(case_id),
    report_type         VARCHAR(20) NOT NULL CHECK (report_type IN ('SAR', 'STR', 'CTR', 'CMIR', 'DOEP', 'OTHER')),
    jurisdiction        VARCHAR(3) NOT NULL,           -- ISO 3166-1 alpha-3
    regulator_code      VARCHAR(20) NOT NULL,          -- 'FINCEN', 'FCA', 'AUSTRAC', 'BAFIN', etc.
    filing_type         VARCHAR(20) NOT NULL CHECK (filing_type IN ('INITIAL', 'CONTINUING', 'CORRECTED', 'JOINT')),
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT' CHECK (status IN (
        'DRAFT', 'NARRATIVE_GENERATED', 'UNDER_REVIEW', 'APPROVED',
        'SUBMITTED', 'ACCEPTED', 'REJECTED', 'AMENDMENT_REQUIRED'
    )),

    -- Subject information
    subject_party_id    UUID NOT NULL REFERENCES party(party_id),

    -- Suspicious activity details
    activity_start_date DATE NOT NULL,
    activity_end_date   DATE,
    total_amount        DECIMAL(18,4),
    total_amount_currency VARCHAR(3),
    suspicious_activity_types TEXT[] NOT NULL,          -- array of activity category codes

    -- Narrative
    narrative           TEXT,
    ai_generated_narrative TEXT,
    narrative_reviewed  BOOLEAN NOT NULL DEFAULT FALSE,
    narrative_reviewed_by UUID REFERENCES analyst(analyst_id),

    -- Filing details
    filing_institution  TEXT NOT NULL,
    filing_institution_id TEXT,                         -- RSSD, EIN, etc.
    prepared_by         UUID NOT NULL REFERENCES analyst(analyst_id),
    reviewed_by         UUID REFERENCES analyst(analyst_id),
    approved_by         UUID REFERENCES analyst(analyst_id),
    approved_at         TIMESTAMPTZ,
    submitted_at        TIMESTAMPTZ,
    submission_ref      TEXT,                           -- BSA E-Filing confirmation number
    regulator_ack_ref   TEXT,                           -- regulator acknowledgement

    -- Retention
    retention_until     DATE NOT NULL,                  -- typically 5 years from filing
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ
);

CREATE INDEX idx_sar_case ON sar_report (case_id);
CREATE INDEX idx_sar_status ON sar_report (status);
CREATE INDEX idx_sar_subject ON sar_report (subject_party_id);
CREATE INDEX idx_sar_jurisdiction ON sar_report (jurisdiction, regulator_code);
CREATE INDEX idx_sar_submitted ON sar_report (submitted_at) WHERE submitted_at IS NOT NULL;

-- CTR (Currency Transaction Report) for cash transactions over threshold
CREATE TABLE ctr_report (
    ctr_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ctr_number          BIGINT GENERATED ALWAYS AS IDENTITY,
    transaction_date    DATE NOT NULL,
    party_id            UUID NOT NULL REFERENCES party(party_id),
    account_id          UUID REFERENCES account(account_id),
    total_cash_in       DECIMAL(18,4) NOT NULL DEFAULT 0,
    total_cash_out      DECIMAL(18,4) NOT NULL DEFAULT 0,
    currency            VARCHAR(3) NOT NULL,
    jurisdiction        VARCHAR(3) NOT NULL,
    regulator_code      VARCHAR(20) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'GENERATED', 'SUBMITTED', 'ACCEPTED', 'REJECTED')),
    submission_ref      TEXT,
    submitted_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ctr_party ON ctr_report (party_id);
CREATE INDEX idx_ctr_date ON ctr_report (transaction_date);
CREATE INDEX idx_ctr_status ON ctr_report (status);
```

### Domain 8: System Administration and Audit

```sql
-- Comprehensive audit log (immutable, append-only)
CREATE TABLE audit_log (
    log_id              UUID NOT NULL DEFAULT gen_random_uuid(),
    event_time          TIMESTAMPTZ NOT NULL DEFAULT now(),
    event_type          VARCHAR(50) NOT NULL,           -- e.g. 'PARTY.UPDATED', 'ALERT.DISPOSITIONED'
    entity_type         VARCHAR(30) NOT NULL,           -- e.g. 'party', 'alert', 'case'
    entity_id           UUID NOT NULL,
    action              VARCHAR(20) NOT NULL CHECK (action IN ('CREATE', 'UPDATE', 'DELETE', 'VIEW', 'EXPORT', 'LOGIN', 'LOGOUT')),
    old_values          JSONB,
    new_values          JSONB,
    performed_by        UUID NOT NULL,
    ip_address          INET,
    user_agent          TEXT,
    session_id          UUID,
    PRIMARY KEY (log_id, event_time)
) PARTITION BY RANGE (event_time);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id, event_time);
CREATE INDEX idx_audit_user ON audit_log (performed_by, event_time);
CREATE INDEX idx_audit_type ON audit_log (event_type, event_time);

-- Tenant configuration (multi-tenant support)
CREATE TABLE tenant (
    tenant_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_name         TEXT NOT NULL,
    tenant_code         VARCHAR(20) NOT NULL UNIQUE,
    jurisdiction        VARCHAR(3) NOT NULL,
    timezone            TEXT NOT NULL DEFAULT 'UTC',
    ctr_threshold       DECIMAL(18,4) NOT NULL DEFAULT 10000.00,
    default_currency    VARCHAR(3) NOT NULL DEFAULT 'USD',
    data_retention_years INTEGER NOT NULL DEFAULT 7,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Pros and Cons

### Pros

1. **Regulatory examination readiness**: Normalized tables with strict foreign keys and constraints produce the exact audit trail structure that FFIEC examiners and FCA supervisors expect. Every relationship is traceable and every state change is documented.

2. **Referential integrity guarantees**: Cascading foreign keys ensure that no alert can reference a nonexistent party, no SAR can be filed for a nonexistent case, and no screening result can point to a deleted watchlist entry. This is critical in a domain where data integrity errors can result in regulatory penalties.

3. **Mature tooling ecosystem**: PostgreSQL has the richest ecosystem for AML-relevant operations: `pg_trgm` for fuzzy name matching in sanctions screening, `pgcrypto` for field-level encryption of PII, `pg_partman` for automated transaction table partitioning, and Row Level Security for multi-tenant data isolation.

4. **Partitioning for transaction volume**: Range partitioning on `transaction_date` allows the system to handle hundreds of millions of transactions with predictable query performance. Old partitions can be moved to cheaper storage or archived without affecting operational queries.

5. **Standards alignment**: Column naming follows ISO 20022 terminology (originator, beneficiary, remittance_info, end_to_end_id), reducing mapping complexity when ingesting from SWIFT/payment rails.

6. **Bitemporal queries**: The `valid_from`/`valid_to` and `recorded_at` pattern supports regulatory requirements for point-in-time reconstruction ("What was this party's risk rating on the date of the suspicious transaction?").

### Cons

1. **Network analysis limitations**: Graph traversal queries (finding layering chains, mule networks, ring structures) are expensive in SQL. Recursive CTEs work for simple 2-3 hop paths but degrade rapidly for deeper traversals across millions of transactions.

2. **Schema rigidity**: Adding new transaction types, risk factors, or regulatory report fields requires schema migrations. In a domain where regulations change frequently (e.g., new EU AMLA requirements in 2027, evolving FATF VASP guidance), this creates operational overhead.

3. **Entity resolution complexity**: Matching party records across aliases, transliterations, and fuzzy name variants requires specialized indexing (trigrams, phonetic algorithms) that adds storage overhead and query complexity compared to purpose-built entity resolution engines.

4. **Materialized view maintenance**: The `mv_entity_daily_stats` view and similar aggregations must be refreshed periodically, creating a lag between transaction ingestion and behavioural profile updates. This matters for real-time detection.

5. **Horizontal scaling constraints**: PostgreSQL scales vertically well but horizontal sharding (for multi-region deployments with data residency requirements) requires external tooling like Citus, adding operational complexity.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary database | PostgreSQL 16+ | ACID compliance, partitioning, RLS, pg_trgm |
| Partitioning management | pg_partman | Automated monthly partition creation and retention |
| Fuzzy matching | pg_trgm + fuzzystrmatch | Trigram similarity and Soundex/Metaphone for name screening |
| Field encryption | pgcrypto | AES-256 encryption for PII columns (tax_id, account_number) |
| Connection pooling | PgBouncer | Transaction pooling for high-concurrency alert processing |
| Replication | Streaming replication + pg_basebackup | HA with read replicas for reporting queries |
| Migration tool | Flyway or Liquibase | Version-controlled schema migrations with rollback |
| Monitoring | pg_stat_statements + Prometheus + Grafana | Query performance tracking |

---

## Migration and Scaling Considerations

### Initial Deployment (< 1M transactions/month)
- Single PostgreSQL instance with streaming replication standby
- Monthly partitions on transaction table, quarterly on audit_log
- Materialized views refreshed every 15 minutes
- Estimated storage: ~50 GB/year at 1M txn/month

### Growth Phase (1M - 50M transactions/month)
- Add read replicas for reporting and analytics queries
- Move to PgBouncer for connection pooling
- Increase partition frequency to weekly for transaction table
- Implement pg_partman retention policies (drop partitions older than retention period)
- Consider partial indexes for high-selectivity queries
- Estimated storage: ~500 GB - 2 TB/year

### Scale Phase (50M+ transactions/month)
- Deploy Citus for horizontal sharding by tenant_id
- Separate OLTP (transaction ingestion, alert processing) from OLAP (reporting, analytics) databases
- Archive cold partitions to columnar storage (Citus columnar or separate analytical DB)
- Implement logical replication for cross-region data residency
- Consider moving network analysis queries to a dedicated graph database (see suggestion 4)
- Estimated storage: 5-20 TB/year

### Data Retention Strategy
- Active data: current year + 2 years in hot storage
- Warm data: years 3-5 in compressed partitions
- Cold data: years 5-7 in archived columnar format
- Purge: after regulatory retention period (jurisdiction-dependent, typically 5-7 years post-relationship)

### Multi-Jurisdiction Deployment
- Use PostgreSQL Row Level Security policies keyed on `tenant_id` for logical data isolation
- For physical data residency (e.g., EU data must stay in EU), deploy separate PostgreSQL clusters per region with logical replication of shared reference data (watchlists, rules)
- Configure per-tenant `data_retention_years` to match jurisdiction requirements
