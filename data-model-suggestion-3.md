# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL JSONB)

> Project: Anti-Money Laundering Platform (Candidate #432)
> Date: 2026-05-25

## Overview

This model uses PostgreSQL as a single database engine but strategically combines normalized relational columns for stable, frequently-queried data with JSONB columns for flexible, variable, and rapidly-evolving data. The approach recognizes a fundamental tension in AML platforms: some data structures are stable and well-understood (party demographics, account details, alert workflow states), while others vary unpredictably across jurisdictions, payment rails, transaction types, and regulatory regimes.

The hybrid approach avoids the schema migration burden of a fully normalized model (Suggestion 1) when regulations change, while retaining the query performance and referential integrity guarantees that a pure document database would sacrifice. PostgreSQL's JSONB support -- with GIN indexing, containment operators, JSON path queries, and JSON schema validation via CHECK constraints -- makes this practical without introducing a second database technology.

---

## Design Philosophy

The guiding principle is: **normalize what is stable, document what varies**.

| Data characteristic | Storage approach | Examples |
|--------------------|-----------------|----------|
| Queried in WHERE clauses or JOINs | Relational columns with indexes | party_id, status, risk_rating, transaction_date |
| Used for aggregations (SUM, COUNT, GROUP BY) | Relational columns | amount, currency, transaction_type |
| Foreign key relationships | Relational columns with constraints | party_id -> party, case_id -> case |
| Varies by jurisdiction or regulation | JSONB column | SAR form fields, CDD requirements, regulatory metadata |
| Varies by payment rail or source system | JSONB column | Payment rail-specific metadata, ISO 20022 extended elements |
| Evolves frequently with new requirements | JSONB column | ML model features, risk factor details, typology parameters |
| Free-form or semi-structured | JSONB column | Investigation notes metadata, evidence metadata, screening match details |

---

## Complete Schema

### Domain 1: Party Management (KYC/CDD)

```sql
CREATE TABLE party (
    party_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    party_type          VARCHAR(20) NOT NULL CHECK (party_type IN ('INDIVIDUAL', 'ORGANISATION', 'TRUST', 'GOVERNMENT')),

    -- Stable, frequently queried fields (relational)
    legal_name          TEXT NOT NULL,
    country_of_residence VARCHAR(3),
    nationality         VARCHAR(3),
    risk_rating         VARCHAR(10) NOT NULL DEFAULT 'MEDIUM' CHECK (risk_rating IN ('LOW', 'MEDIUM', 'HIGH', 'VERY_HIGH', 'PROHIBITED')),
    cdd_level           VARCHAR(20) NOT NULL DEFAULT 'STANDARD' CHECK (cdd_level IN ('SIMPLIFIED', 'STANDARD', 'ENHANCED')),
    pep_status          BOOLEAN NOT NULL DEFAULT FALSE,
    onboarding_date     DATE NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- Variable/jurisdiction-specific fields (JSONB)
    -- Structure varies by party_type and jurisdiction
    demographics        JSONB NOT NULL DEFAULT '{}',
    -- INDIVIDUAL: {"date_of_birth": "1985-03-15", "occupation": "Engineer", "source_of_wealth": "Employment", "source_of_funds": "Salary"}
    -- ORGANISATION: {"date_of_incorporation": "2010-06-01", "industry_code": "522110", "lei": "549300ABC123", "registration_number": "12345678"}

    identification      JSONB NOT NULL DEFAULT '[]',
    -- Array of ID documents, structure varies by jurisdiction:
    -- [
    --   {"type": "PASSPORT", "number": "encrypted:...", "issuing_country": "USA", "expiry_date": "2030-06-20", "verified": true, "verified_at": "2026-01-15"},
    --   {"type": "TAX_ID", "number": "encrypted:...", "issuing_country": "USA", "verified": true}
    -- ]

    addresses           JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "RESIDENTIAL", "line_1": "123 Main St", "city": "New York", "state": "NY", "postal_code": "10001", "country": "USA", "is_current": true},
    --   {"type": "MAILING", "line_1": "PO Box 456", "city": "New York", "state": "NY", "postal_code": "10002", "country": "USA", "is_current": true}
    -- ]

    aliases             JSONB NOT NULL DEFAULT '[]',
    -- [{"name": "J. Doe", "type": "AKA"}, {"name": "Jon Doe", "type": "TRANSLITERATION", "script": "Latn"}]

    cdd_details         JSONB NOT NULL DEFAULT '{}',
    -- Jurisdiction-specific CDD data. Structure varies significantly:
    -- US: {"beneficial_owners": [...], "cip_verification": {...}, "ofac_screening_date": "2026-05-25"}
    -- EU: {"ultimate_beneficial_owners": [...], "pep_screening_date": "2026-05-25", "high_risk_country_assessment": {...}}
    -- UK: {"source_of_wealth_evidence": "...", "enhanced_due_diligence_reason": "PEP"}

    risk_assessment     JSONB NOT NULL DEFAULT '{}',
    -- {"risk_factors": ["HIGH_RISK_JURISDICTION", "PEP"], "risk_score": 0.72,
    --  "last_assessment_date": "2026-05-25", "next_review_date": "2026-11-25",
    --  "assessment_history": [{"date": "2026-01-15", "rating": "MEDIUM", "reason": "Initial onboarding"}]}

    -- Metadata
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL,
    updated_by          UUID
);

-- Relational indexes for common query patterns
CREATE INDEX idx_party_tenant ON party (tenant_id);
CREATE INDEX idx_party_risk ON party (risk_rating) WHERE is_active = TRUE;
CREATE INDEX idx_party_pep ON party (pep_status) WHERE pep_status = TRUE AND is_active = TRUE;
CREATE INDEX idx_party_country ON party (country_of_residence) WHERE is_active = TRUE;

-- GIN indexes on JSONB for flexible querying
CREATE INDEX idx_party_name_trgm ON party USING gin (legal_name gin_trgm_ops);
CREATE INDEX idx_party_demographics ON party USING gin (demographics jsonb_path_ops);
CREATE INDEX idx_party_identification ON party USING gin (identification jsonb_path_ops);
CREATE INDEX idx_party_aliases ON party USING gin (aliases jsonb_path_ops);
CREATE INDEX idx_party_risk_assessment ON party USING gin (risk_assessment jsonb_path_ops);

-- JSON schema validation for critical JSONB fields
ALTER TABLE party ADD CONSTRAINT chk_demographics_structure CHECK (
    demographics IS NOT NULL AND jsonb_typeof(demographics) = 'object'
);
ALTER TABLE party ADD CONSTRAINT chk_identification_structure CHECK (
    identification IS NOT NULL AND jsonb_typeof(identification) = 'array'
);
ALTER TABLE party ADD CONSTRAINT chk_aliases_structure CHECK (
    aliases IS NOT NULL AND jsonb_typeof(aliases) = 'array'
);

-- Relationships between parties (relational for JOIN performance)
CREATE TABLE party_relationship (
    relationship_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_party_id       UUID NOT NULL REFERENCES party(party_id),
    to_party_id         UUID NOT NULL REFERENCES party(party_id),
    relationship_type   VARCHAR(30) NOT NULL CHECK (relationship_type IN (
        'BENEFICIAL_OWNER', 'DIRECTOR', 'SHAREHOLDER', 'AUTHORIZED_SIGNATORY',
        'SPOUSE', 'PARENT', 'CHILD', 'BUSINESS_PARTNER', 'EMPLOYER',
        'AGENT', 'TRUSTEE', 'BENEFICIARY', 'NOMINEE', 'OTHER'
    )),
    ownership_percentage DECIMAL(5,2),
    details             JSONB NOT NULL DEFAULT '{}',   -- relationship-type-specific metadata
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rel_from ON party_relationship (from_party_id);
CREATE INDEX idx_rel_to ON party_relationship (to_party_id);
CREATE INDEX idx_rel_type ON party_relationship (relationship_type);
```

### Domain 2: Accounts

```sql
CREATE TABLE account (
    account_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    account_type        VARCHAR(30) NOT NULL CHECK (account_type IN (
        'CURRENT', 'SAVINGS', 'LOAN', 'CREDIT_CARD', 'BROKERAGE',
        'CUSTODIAL', 'TRUST', 'CORRESPONDENT', 'NOSTRO', 'VOSTRO',
        'WALLET', 'PREPAID', 'OTHER'
    )),
    currency            VARCHAR(3) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE' CHECK (status IN ('ACTIVE', 'DORMANT', 'SUSPENDED', 'CLOSED', 'FROZEN')),
    risk_rating         VARCHAR(10) NOT NULL DEFAULT 'MEDIUM',
    institution_id      UUID NOT NULL REFERENCES party(party_id),
    opened_date         DATE NOT NULL,
    closed_date         DATE,

    -- JSONB: account details vary by type and institution
    account_details     JSONB NOT NULL DEFAULT '{}',
    -- CURRENT: {"account_number": "encrypted:...", "iban": "GB29NWBK60161331926819", "bic": "NWBKGB2L", "branch_code": "601613"}
    -- WALLET: {"wallet_address": "0x742d35...", "blockchain": "ethereum", "wallet_type": "custodial"}
    -- CORRESPONDENT: {"nostro_vostro": "NOSTRO", "correspondent_bic": "DEUTDEFF", "relationship_manager": "uuid"}

    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL
);

CREATE INDEX idx_account_tenant ON account (tenant_id);
CREATE INDEX idx_account_status ON account (status);
CREATE INDEX idx_account_institution ON account (institution_id);
CREATE INDEX idx_account_details ON account USING gin (account_details jsonb_path_ops);

-- Many-to-many: parties to accounts
CREATE TABLE account_party (
    account_party_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id          UUID NOT NULL REFERENCES account(account_id),
    party_id            UUID NOT NULL REFERENCES party(party_id),
    role                VARCHAR(30) NOT NULL CHECK (role IN ('HOLDER', 'JOINT_HOLDER', 'AUTHORIZED_SIGNATORY', 'BENEFICIAL_OWNER', 'POWER_OF_ATTORNEY', 'CUSTODIAN')),
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    valid_from          TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to            TIMESTAMPTZ,
    UNIQUE (account_id, party_id, role)
);

CREATE INDEX idx_ap_account ON account_party (account_id);
CREATE INDEX idx_ap_party ON account_party (party_id);
```

### Domain 3: Transactions

```sql
CREATE TABLE transaction (
    transaction_id      UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    transaction_date    TIMESTAMPTZ NOT NULL,

    -- Stable relational columns (always present, always queried)
    transaction_type    VARCHAR(30) NOT NULL CHECK (transaction_type IN (
        'WIRE_TRANSFER', 'ACH', 'SEPA', 'FASTER_PAYMENT', 'CARD_PAYMENT',
        'CARD_WITHDRAWAL', 'CASH_DEPOSIT', 'CASH_WITHDRAWAL', 'CHECK',
        'INTERNAL_TRANSFER', 'CRYPTO_TRANSFER', 'TRADE', 'FEE', 'INTEREST',
        'LOAN_DISBURSEMENT', 'LOAN_REPAYMENT', 'OTHER'
    )),
    direction           VARCHAR(10) NOT NULL CHECK (direction IN ('INBOUND', 'OUTBOUND', 'INTERNAL')),
    amount              DECIMAL(18,4) NOT NULL,
    currency            VARCHAR(3) NOT NULL,
    amount_usd          DECIMAL(18,4),
    originator_party_id UUID REFERENCES party(party_id),
    originator_account_id UUID REFERENCES account(account_id),
    beneficiary_party_id UUID REFERENCES party(party_id),
    beneficiary_account_id UUID REFERENCES account(account_id),
    payment_channel     VARCHAR(20),
    source_system       VARCHAR(50) NOT NULL,
    risk_score          DECIMAL(5,2),

    -- JSONB: payment-rail-specific and source-system-specific metadata
    -- This is where the hybrid model shines: every payment rail has different fields
    originator_details  JSONB NOT NULL DEFAULT '{}',
    -- SWIFT: {"name": "John Doe", "bic": "CHASUS33", "country": "USA", "address": "123 Main St, NY"}
    -- ACH: {"name": "John Doe", "routing_number": "021000021", "account_number": "encrypted:..."}
    -- CRYPTO: {"wallet_address": "0x742d35...", "blockchain": "ethereum", "tx_hash": "0xabc..."}

    beneficiary_details JSONB NOT NULL DEFAULT '{}',
    -- Same structure variation as originator_details

    intermediary_details JSONB,
    -- Correspondent banking: {"bic": "DEUTDEFF", "name": "Deutsche Bank", "country": "DEU"}

    payment_rail_metadata JSONB NOT NULL DEFAULT '{}',
    -- ISO 20022 fields: {"end_to_end_id": "E2E-001", "instruction_id": "INSTR-001",
    --   "remittance_info": "Invoice 2026-0042", "charge_bearer": "SHA",
    --   "settlement_method": "INDA", "interbank_settlement_date": "2026-05-26"}
    -- SWIFT MT: {"mt_type": "MT103", "field_20": "REF123", "field_70": "Invoice payment"}
    -- Card: {"merchant_id": "MCC5411", "merchant_name": "Grocery Store", "terminal_id": "T001",
    --   "card_last_four": "4321", "authorization_code": "AUTH001"}

    risk_details        JSONB NOT NULL DEFAULT '{}',
    -- {"risk_factors": ["HIGH_RISK_JURISDICTION", "VELOCITY_BREACH"],
    --  "model_id": "uuid", "model_version": "2.3.1",
    --  "scoring_latency_ms": 12, "explanation": "..."}

    -- Metadata
    external_ref        TEXT,
    batch_id            UUID,
    is_reversal         BOOLEAN NOT NULL DEFAULT FALSE,
    ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (transaction_id, transaction_date)
) PARTITION BY RANGE (transaction_date);

-- Create partitions (managed by pg_partman)
CREATE TABLE transaction_2026_01 PARTITION OF transaction FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE transaction_2026_02 PARTITION OF transaction FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ...

-- Relational indexes for common query patterns
CREATE INDEX idx_txn_tenant_date ON transaction (tenant_id, transaction_date);
CREATE INDEX idx_txn_originator ON transaction (originator_party_id, transaction_date);
CREATE INDEX idx_txn_beneficiary ON transaction (beneficiary_party_id, transaction_date);
CREATE INDEX idx_txn_amount ON transaction (amount_usd, transaction_date);
CREATE INDEX idx_txn_risk ON transaction (risk_score DESC, transaction_date) WHERE risk_score IS NOT NULL;
CREATE INDEX idx_txn_type ON transaction (transaction_type, transaction_date);
CREATE INDEX idx_txn_source ON transaction (source_system, transaction_date);

-- GIN indexes on JSONB for flexible querying
CREATE INDEX idx_txn_rail_metadata ON transaction USING gin (payment_rail_metadata jsonb_path_ops);
CREATE INDEX idx_txn_risk_details ON transaction USING gin (risk_details jsonb_path_ops);

-- Example queries enabled by the hybrid model:
-- Find all SWIFT transfers with a specific BIC in intermediary chain:
-- SELECT * FROM transaction WHERE intermediary_details @> '{"bic": "DEUTDEFF"}';

-- Find transactions with specific ISO 20022 end-to-end ID:
-- SELECT * FROM transaction WHERE payment_rail_metadata @> '{"end_to_end_id": "E2E-001"}';

-- Find all card transactions at a specific merchant category:
-- SELECT * FROM transaction WHERE payment_rail_metadata @> '{"merchant_id": "MCC5411"}';

-- Find transactions where risk model flagged high-risk jurisdiction:
-- SELECT * FROM transaction WHERE risk_details @> '{"risk_factors": ["HIGH_RISK_JURISDICTION"]}';
```

### Domain 4: Detection Rules and ML Models

```sql
CREATE TABLE detection_rule (
    rule_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    rule_code           VARCHAR(50) NOT NULL,
    rule_name           TEXT NOT NULL,
    rule_version        INTEGER NOT NULL DEFAULT 1,
    typology            VARCHAR(50) NOT NULL,
    severity            VARCHAR(10) NOT NULL CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: rule configuration varies dramatically by typology
    rule_config         JSONB NOT NULL,
    -- STRUCTURING: {
    --   "type": "structuring",
    --   "threshold_amount": 10000.00, "threshold_currency": "USD",
    --   "window_hours": 24, "min_transactions": 2,
    --   "max_individual_amount_pct": 0.95,
    --   "applicable_transaction_types": ["CASH_DEPOSIT", "CASH_WITHDRAWAL"],
    --   "exclude_known_patterns": ["PAYROLL"]
    -- }
    -- VELOCITY: {
    --   "type": "velocity",
    --   "metric": "outbound_wire_count", "window_hours": 48,
    --   "threshold_absolute": 10, "threshold_sigma": 3.0,
    --   "baseline_window_days": 90,
    --   "min_total_amount_usd": 50000.00
    -- }
    -- DORMANT_ACTIVATION: {
    --   "type": "dormant_activation",
    --   "dormancy_period_days": 180, "min_reactivation_amount_usd": 5000.00,
    --   "lookback_window_days": 7, "min_transactions_post_activation": 3
    -- }
    -- HIGH_RISK_JURISDICTION: {
    --   "type": "high_risk_jurisdiction",
    --   "high_risk_countries": ["IRN", "PRK", "MMR", "SYR"],
    --   "elevated_risk_countries": ["VGB", "CYM", "PAN", "BLZ"],
    --   "min_amount_usd": 1000.00,
    --   "aggregate_threshold_usd": 25000.00, "aggregate_window_days": 30
    -- }

    -- Rule validation and approval metadata
    approval_details    JSONB NOT NULL DEFAULT '{}',
    -- {"approved_by": "uuid", "approved_at": "2026-05-25", "approval_notes": "...",
    --  "backtesting_results": {"true_positive_rate": 0.12, "false_positive_rate": 0.88, "sample_size": 50000}}

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL,
    UNIQUE (tenant_id, rule_code, rule_version)
);

CREATE INDEX idx_rule_tenant ON detection_rule (tenant_id);
CREATE INDEX idx_rule_typology ON detection_rule (typology) WHERE is_active = TRUE;
CREATE INDEX idx_rule_config ON detection_rule USING gin (rule_config jsonb_path_ops);

-- ML model registry
CREATE TABLE ml_model (
    model_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name          TEXT NOT NULL,
    model_version       VARCHAR(20) NOT NULL,
    model_type          VARCHAR(30) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT FALSE,
    artifact_path       TEXT NOT NULL,

    -- JSONB: model metadata varies by model type and framework
    model_metadata      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "framework": "xgboost",
    --   "training_date": "2026-05-01",
    --   "training_data": {"start_date": "2025-01-01", "end_date": "2026-04-30", "record_count": 15000000},
    --   "features": ["txn_count_24h", "txn_amount_zscore", "counterparty_risk", "jurisdiction_risk", ...],
    --   "performance": {"precision": 0.85, "recall": 0.72, "f1": 0.78, "auc": 0.94, "false_positive_rate": 0.03},
    --   "validation_status": "VALIDATED",
    --   "validated_by": "uuid", "validated_at": "2026-05-10",
    --   "next_validation_date": "2026-11-10",
    --   "drift_monitoring": {"baseline_distribution": {...}, "alert_threshold": 0.15},
    --   "sr11_7_documentation": {"conceptual_soundness": "APPROVED", "outcome_analysis": "PASSED"}
    -- }

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL
);

CREATE INDEX idx_model_active ON ml_model (is_active) WHERE is_active = TRUE;
CREATE INDEX idx_model_metadata ON ml_model USING gin (model_metadata jsonb_path_ops);
```

### Domain 5: Alerts and Cases

```sql
CREATE TABLE alert (
    alert_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    alert_number        BIGINT GENERATED ALWAYS AS IDENTITY,
    alert_date          TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Stable relational columns (queried constantly in alert queue)
    alert_source        VARCHAR(20) NOT NULL CHECK (alert_source IN ('RULE', 'MODEL', 'SCREENING', 'MANUAL', 'EXTERNAL')),
    party_id            UUID NOT NULL REFERENCES party(party_id),
    risk_score          DECIMAL(5,2) NOT NULL,
    severity            VARCHAR(10) NOT NULL CHECK (severity IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')),
    status              VARCHAR(20) NOT NULL DEFAULT 'NEW' CHECK (status IN (
        'NEW', 'ASSIGNED', 'UNDER_REVIEW', 'ESCALATED',
        'CLOSED_FALSE_POSITIVE', 'CLOSED_TRUE_POSITIVE', 'CLOSED_INCONCLUSIVE',
        'SAR_FILED'
    )),
    typology            VARCHAR(50),
    assigned_to         UUID,
    case_id             UUID,

    -- JSONB: alert details vary by source, typology, and detection method
    alert_details       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "rule_id": "uuid", "rule_code": "STRUCT-001",
    --   "account_id": "uuid",
    --   "summary": "Three cash deposits totaling $9,800 within 24 hours",
    --   "explanation": "Each deposit below $10,000 CTR threshold. Pattern consistent with structuring.",
    --   "triggering_transactions": [
    --     {"transaction_id": "uuid", "amount": 3200.00, "date": "2026-05-24T09:15:00Z"},
    --     {"transaction_id": "uuid", "amount": 3300.00, "date": "2026-05-24T14:30:00Z"},
    --     {"transaction_id": "uuid", "amount": 3300.00, "date": "2026-05-25T10:00:00Z"}
    --   ],
    --   "risk_factor_breakdown": [
    --     {"factor": "STRUCTURING", "weight": 0.45, "description": "Amount pattern below CTR threshold"},
    --     {"factor": "CASH_INTENSIVE", "weight": 0.25, "description": "All transactions in cash"},
    --     {"factor": "VELOCITY", "weight": 0.15, "description": "3 deposits in 24 hours vs avg 1/week"}
    --   ],
    --   "entity_context": {
    --     "party_risk_rating": "MEDIUM", "account_age_days": 45,
    --     "avg_monthly_deposits": 2, "avg_deposit_amount": 1500.00
    --   }
    -- }

    -- Disposition data (populated when alert is closed)
    disposition_details JSONB,
    -- {"disposition_reason": "Confirmed structuring", "dispositioned_by": "uuid",
    --  "disposition_date": "2026-05-26T11:00:00Z", "time_to_disposition_minutes": 42,
    --  "was_bulk_dispositioned": false}

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Relational indexes for alert queue performance
CREATE INDEX idx_alert_open ON alert (severity DESC, risk_score DESC)
    WHERE status IN ('NEW', 'ASSIGNED', 'UNDER_REVIEW');
CREATE INDEX idx_alert_assigned ON alert (assigned_to, status);
CREATE INDEX idx_alert_party ON alert (party_id);
CREATE INDEX idx_alert_tenant_status ON alert (tenant_id, status);
CREATE INDEX idx_alert_case ON alert (case_id) WHERE case_id IS NOT NULL;
CREATE INDEX idx_alert_date ON alert (alert_date);

-- GIN index for querying alert details
CREATE INDEX idx_alert_details ON alert USING gin (alert_details jsonb_path_ops);

-- Investigation cases
CREATE TABLE investigation_case (
    case_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    case_number         BIGINT GENERATED ALWAYS AS IDENTITY,

    -- Stable relational columns
    case_type           VARCHAR(30) NOT NULL CHECK (case_type IN (
        'SUSPICIOUS_ACTIVITY', 'SANCTIONS_HIT', 'PEP_REVIEW',
        'PERIODIC_REVIEW', 'REGULATORY_REQUEST', 'TIP_OFF', 'OTHER'
    )),
    status              VARCHAR(20) NOT NULL DEFAULT 'OPEN' CHECK (status IN (
        'OPEN', 'ASSIGNED', 'UNDER_INVESTIGATION', 'ESCALATED',
        'PENDING_SAR', 'SAR_FILED', 'CLOSED_NO_ACTION', 'CLOSED_SAR_FILED',
        'CLOSED_REFERRED', 'REOPENED'
    )),
    priority            VARCHAR(10) NOT NULL DEFAULT 'MEDIUM',
    subject_party_id    UUID NOT NULL REFERENCES party(party_id),
    assigned_to         UUID,
    opened_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at           TIMESTAMPTZ,

    -- JSONB: investigation content (evolves throughout investigation)
    case_content        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "summary": "Suspected structuring across multiple accounts",
    --   "findings": "Analysis reveals systematic deposits below CTR threshold...",
    --   "recommendation": "File SAR with FinCEN",
    --   "linked_alert_ids": ["uuid-1", "uuid-2"],
    --   "investigation_timeline": [
    --     {"date": "2026-05-25T09:00:00Z", "action": "Case opened", "by": "uuid"},
    --     {"date": "2026-05-25T10:30:00Z", "action": "Reviewed transaction history", "by": "uuid"},
    --     {"date": "2026-05-25T14:00:00Z", "action": "Network analysis completed", "by": "uuid"}
    --   ]
    -- }

    -- JSONB: escalation and assignment history
    workflow_history    JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"action": "ASSIGNED", "to": "uuid", "by": "uuid", "at": "2026-05-25T09:00:00Z"},
    --   {"action": "ESCALATED", "to": "uuid", "by": "uuid", "at": "2026-05-25T14:00:00Z", "reason": "Complex layering"}
    -- ]

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL
);

CREATE INDEX idx_case_open ON investigation_case (priority DESC, opened_at)
    WHERE status NOT LIKE 'CLOSED%';
CREATE INDEX idx_case_assigned ON investigation_case (assigned_to, status);
CREATE INDEX idx_case_party ON investigation_case (subject_party_id);
CREATE INDEX idx_case_tenant ON investigation_case (tenant_id, status);
CREATE INDEX idx_case_content ON investigation_case USING gin (case_content jsonb_path_ops);

-- Case activity log (append-only, immutable)
CREATE TABLE case_activity (
    activity_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id             UUID NOT NULL REFERENCES investigation_case(case_id),
    activity_type       VARCHAR(30) NOT NULL,
    performed_by        UUID NOT NULL,
    performed_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- JSONB: activity details vary by type
    activity_data       JSONB NOT NULL DEFAULT '{}',
    -- NOTE_ADDED: {"note_text": "...", "note_type": "INVESTIGATION_FINDING"}
    -- EVIDENCE_ATTACHED: {"file_name": "...", "file_path": "...", "sha256": "..."}
    -- STATUS_CHANGED: {"old_status": "OPEN", "new_status": "UNDER_INVESTIGATION"}
    -- SAR_DRAFTED: {"sar_id": "uuid", "narrative_source": "AI_GENERATED"}
    -- SCREENING_PERFORMED: {"lists_checked": [...], "matches_found": 2}

    description         TEXT NOT NULL
);

CREATE INDEX idx_activity_case ON case_activity (case_id, performed_at);
CREATE INDEX idx_activity_type ON case_activity (activity_type, performed_at);
```

### Domain 6: Sanctions Screening

```sql
-- Watchlist sources
CREATE TABLE watchlist_source (
    source_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_code         VARCHAR(30) NOT NULL UNIQUE,
    source_name         TEXT NOT NULL,
    list_type           VARCHAR(20) NOT NULL CHECK (list_type IN ('SANCTIONS', 'PEP', 'ADVERSE_MEDIA', 'LAW_ENFORCEMENT', 'CUSTOM')),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: source-specific configuration
    source_config       JSONB NOT NULL DEFAULT '{}',
    -- {"url": "https://...", "format": "XML", "schema_version": "2.0",
    --  "update_frequency": "DAILY", "last_updated_at": "2026-05-25T14:30:00Z",
    --  "parser_config": {"root_element": "sdnList", "entry_path": "sdnEntry"}}

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Watchlist entries
CREATE TABLE watchlist_entry (
    entry_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id           UUID NOT NULL REFERENCES watchlist_source(source_id),
    source_entry_id     TEXT NOT NULL,
    entry_type          VARCHAR(20) NOT NULL CHECK (entry_type IN ('INDIVIDUAL', 'ENTITY', 'VESSEL', 'AIRCRAFT')),
    primary_name        TEXT NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: entry details vary significantly by source and entry type
    entry_details       JSONB NOT NULL DEFAULT '{}',
    -- OFAC SDN Individual: {
    --   "program": "SDGT", "title": "Leader", "remarks": "...",
    --   "listed_date": "2020-03-15",
    --   "aliases": [{"name": "John DOE", "type": "AKA"}, {"name": "Jean DOE", "type": "AKA"}],
    --   "addresses": [{"city": "Tehran", "country": "IRN"}],
    --   "identifiers": [{"type": "PASSPORT", "number": "A1234567", "country": "IRN"}],
    --   "dates_of_birth": ["1970-01-01"],
    --   "nationalities": ["IRN"],
    --   "citizenships": ["IRN"]
    -- }
    -- UN Consolidated Entity: {
    --   "reference_number": "QDe.001",
    --   "listed_on": "2001-10-08",
    --   "un_list_type": "AL-QAIDA",
    --   "aliases": [...],
    --   "addresses": [...],
    --   "comments": "Listed pursuant to resolution 1267"
    -- }

    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, source_entry_id)
);

CREATE INDEX idx_wl_name_trgm ON watchlist_entry USING gin (primary_name gin_trgm_ops);
CREATE INDEX idx_wl_source ON watchlist_entry (source_id) WHERE is_active = TRUE;
CREATE INDEX idx_wl_details ON watchlist_entry USING gin (entry_details jsonb_path_ops);

-- Screening results
CREATE TABLE screening_result (
    result_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    screening_type      VARCHAR(20) NOT NULL CHECK (screening_type IN ('ONBOARDING', 'ONGOING', 'TRANSACTION', 'BATCH', 'MANUAL')),
    screened_party_id   UUID REFERENCES party(party_id),
    screened_name       TEXT NOT NULL,
    matched_entry_id    UUID REFERENCES watchlist_entry(entry_id),
    match_score         DECIMAL(5,4) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'TRUE_MATCH', 'FALSE_POSITIVE', 'ESCALATED')),
    screened_at         TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- JSONB: match details and disposition
    match_details       JSONB NOT NULL DEFAULT '{}',
    -- {"match_algorithm": "JARO_WINKLER", "matched_name": "JOHN DOE",
    --  "matched_list": "OFAC_SDN", "matched_program": "SDGT",
    --  "field_matches": {"name": 0.97, "dob": 0.0, "country": 0.5},
    --  "disposition": {"reviewed_by": "uuid", "reviewed_at": "2026-05-25",
    --    "reason": "Different DOB", "alert_generated": false}}

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_screening_party ON screening_result (screened_party_id);
CREATE INDEX idx_screening_status ON screening_result (status) WHERE status = 'PENDING';
CREATE INDEX idx_screening_tenant ON screening_result (tenant_id, screened_at);
CREATE INDEX idx_screening_details ON screening_result USING gin (match_details jsonb_path_ops);
```

### Domain 7: Regulatory Reporting

```sql
CREATE TABLE regulatory_report (
    report_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    report_number       BIGINT GENERATED ALWAYS AS IDENTITY,

    -- Stable relational columns
    report_type         VARCHAR(20) NOT NULL CHECK (report_type IN ('SAR', 'STR', 'CTR', 'CMIR', 'EFT', 'OTHER')),
    jurisdiction        VARCHAR(3) NOT NULL,
    regulator_code      VARCHAR(20) NOT NULL,
    filing_type         VARCHAR(20) NOT NULL CHECK (filing_type IN ('INITIAL', 'CONTINUING', 'CORRECTED', 'JOINT')),
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT' CHECK (status IN (
        'DRAFT', 'NARRATIVE_GENERATED', 'UNDER_REVIEW', 'APPROVED',
        'SUBMITTED', 'ACCEPTED', 'REJECTED', 'AMENDMENT_REQUIRED'
    )),
    case_id             UUID REFERENCES investigation_case(case_id),
    subject_party_id    UUID NOT NULL REFERENCES party(party_id),

    -- JSONB: report content varies dramatically by jurisdiction and report type
    report_content      JSONB NOT NULL DEFAULT '{}',
    -- FinCEN SAR: {
    --   "filing_institution": {"name": "Acme Bank", "ein": "12-3456789", "rssd": "123456"},
    --   "subject": {
    --     "name": "John Doe", "dob": "1985-03-15", "ssn": "encrypted:...",
    --     "address": {"line_1": "123 Main St", "city": "New York", "state": "NY"}
    --   },
    --   "suspicious_activity": {
    --     "date_range": {"start": "2026-03-01", "end": "2026-05-20"},
    --     "total_amount": 287500.00, "currency": "USD",
    --     "activity_types": ["a", "g"],
    --     "instrument_types": ["a", "e"]
    --   },
    --   "narrative": "Between March 1, 2026 and May 20, 2026...",
    --   "ai_generated_narrative": "...",
    --   "narrative_reviewed": true, "narrative_reviewed_by": "uuid"
    -- }
    -- FCA STR (UK): {
    --   "reporting_entity": {"frn": "123456", "name": "Acme Bank UK"},
    --   "subject": {"name": "...", "national_insurance": "encrypted:..."},
    --   "suspicious_activity": {"sars_code": "SAR-001", ...},
    --   "consent_requested": true, "consent_type": "DEFENCE"
    -- }
    -- goAML STR: {
    --   "report_code": "STR",
    --   "reporting_person": {...},
    --   "transaction_info": {...},
    --   "involved_parties": [...],
    --   "goaml_version": "5.0.1"
    -- }

    -- Submission tracking
    submission_details  JSONB,
    -- {"method": "BSA_EFILING_BATCH", "submitted_at": "2026-05-25T16:00:00Z",
    --  "submission_ref": "BSA-2026-0525-001", "regulator_ack_ref": "FINCEN-ACK-12345",
    --  "accepted_at": "2026-05-25T18:30:00Z"}

    -- Workflow
    workflow_details    JSONB NOT NULL DEFAULT '{}',
    -- {"prepared_by": "uuid", "prepared_at": "2026-05-25T09:00:00Z",
    --  "reviewed_by": "uuid", "reviewed_at": "2026-05-25T11:00:00Z",
    --  "approved_by": "uuid", "approved_at": "2026-05-25T14:00:00Z",
    --  "review_notes": "Narrative reviewed. Minor edits to amount description."}

    retention_until     DATE NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_report_status ON regulatory_report (status, regulator_code);
CREATE INDEX idx_report_jurisdiction ON regulatory_report (jurisdiction, report_type);
CREATE INDEX idx_report_case ON regulatory_report (case_id);
CREATE INDEX idx_report_subject ON regulatory_report (subject_party_id);
CREATE INDEX idx_report_tenant ON regulatory_report (tenant_id, status);
CREATE INDEX idx_report_content ON regulatory_report USING gin (report_content jsonb_path_ops);
```

### Domain 8: Audit and Administration

```sql
-- Immutable audit log
CREATE TABLE audit_log (
    log_id              UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    event_time          TIMESTAMPTZ NOT NULL DEFAULT now(),
    event_type          VARCHAR(50) NOT NULL,
    entity_type         VARCHAR(30) NOT NULL,
    entity_id           UUID NOT NULL,
    action              VARCHAR(20) NOT NULL,
    performed_by        UUID NOT NULL,

    -- JSONB: change details and context
    change_details      JSONB NOT NULL DEFAULT '{}',
    -- {"old_values": {"status": "NEW"}, "new_values": {"status": "ASSIGNED"},
    --  "ip_address": "192.168.1.1", "user_agent": "Mozilla/5.0...",
    --  "session_id": "uuid", "reason": "Auto-assigned by round-robin"}

    PRIMARY KEY (log_id, event_time)
) PARTITION BY RANGE (event_time);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id, event_time);
CREATE INDEX idx_audit_user ON audit_log (performed_by, event_time);
CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, event_time);
CREATE INDEX idx_audit_details ON audit_log USING gin (change_details jsonb_path_ops);

-- Analyst table (relational -- stable structure)
CREATE TABLE analyst (
    analyst_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    user_id             UUID NOT NULL,
    full_name           TEXT NOT NULL,
    email               TEXT NOT NULL,
    role                VARCHAR(30) NOT NULL CHECK (role IN ('L1_ANALYST', 'L2_ANALYST', 'SENIOR_ANALYST', 'TEAM_LEAD', 'COMPLIANCE_OFFICER', 'MLRO', 'ADMIN')),
    team                VARCHAR(50),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    preferences         JSONB NOT NULL DEFAULT '{}',   -- UI preferences, notification settings
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analyst_tenant ON analyst (tenant_id);
CREATE INDEX idx_analyst_role ON analyst (role) WHERE is_active = TRUE;
```

---

## Pros and Cons

### Pros

1. **Regulatory agility**: When a new jurisdiction is onboarded or a regulator changes their filing format (e.g., EU AMLA requirements in 2027), only the JSONB content structure needs updating. No schema migration is required. This dramatically reduces the time-to-compliance for new regulatory requirements.

2. **Payment rail flexibility**: Every payment rail has different metadata (SWIFT MT fields vs. ISO 20022 MX elements vs. ACH routing numbers vs. blockchain transaction hashes). The JSONB `payment_rail_metadata` column absorbs this variation without requiring a different schema per rail.

3. **Single technology stack**: PostgreSQL handles both relational and document workloads. No need for a separate MongoDB or Elasticsearch deployment for flexible data. This reduces operational complexity, infrastructure cost, and the number of technologies the team must master.

4. **Query performance where it matters**: Alert queue sorting, case assignment, party risk filtering -- the most performance-critical queries use relational columns with B-tree indexes. These queries perform identically to a fully normalized model. The JSONB columns are used for drill-down and context queries that are less latency-sensitive.

5. **GIN indexing for JSONB**: PostgreSQL GIN indexes on JSONB columns enable efficient containment queries (`@>` operator) for searching within flexible data structures. Querying "find all transactions where the intermediary BIC is DEUTDEFF" works efficiently even with millions of rows.

6. **Schema validation**: PostgreSQL CHECK constraints and JSON schema validation (via `pg_jsonschema` extension or application-layer validation) enforce structure on JSONB fields without requiring rigid column definitions. This provides a middle ground between "anything goes" and "must migrate for every change."

7. **Reduced table count**: The hybrid model has roughly 60% fewer tables than the fully normalized model (Suggestion 1). Fewer JOINs, simpler queries, faster development iteration.

### Cons

1. **JSONB query performance ceiling**: Queries that need to aggregate or sort by values inside JSONB columns are significantly slower than equivalent queries on indexed relational columns. If a JSONB field becomes a frequent filter criterion, it should be promoted to a relational column -- requiring a migration.

2. **No foreign key enforcement inside JSONB**: When `alert_details` contains `triggering_transactions` as an array of UUIDs, there is no database-level guarantee that those transactions exist. Application-layer validation must compensate, which is error-prone.

3. **Schema documentation burden**: The structure of each JSONB column must be documented and versioned separately since the database schema does not enforce it. Without disciplined documentation, different clients may write incompatible JSONB structures to the same column.

4. **Reporting complexity**: Ad-hoc reporting and BI queries against JSONB fields require PostgreSQL-specific JSON functions (`->`, `->>`, `jsonb_array_elements()`) that are less familiar to analysts and BI tool integrations than standard SQL column references.

5. **Data migration within JSONB**: When the internal structure of a JSONB field evolves (e.g., adding a required field to `report_content`), all existing rows must be updated -- which on large tables can be as expensive as a traditional schema migration, but without the safety net of NOT NULL constraints to catch incomplete updates.

6. **Testing complexity**: Validating JSONB field structures requires custom validation logic in tests and application code. There is no DDL-level safety net for "every SAR report_content must contain a narrative field."

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Database | PostgreSQL 16+ | JSONB with GIN indexing, partitioning, RLS |
| JSONB validation | pg_jsonschema extension | JSON Schema validation for critical JSONB columns |
| Fuzzy matching | pg_trgm | Trigram similarity for sanctions screening |
| Partitioning | pg_partman | Automated partition lifecycle management |
| Application ORM | Prisma, TypeORM, or SQLAlchemy with JSONB support | Must support JSONB query operators natively |
| API validation | JSON Schema (ajv or similar) | Validate JSONB payloads before database insertion |
| Migration tool | Flyway or Liquibase | DDL migrations for relational columns; data migrations for JSONB evolution |
| Monitoring | pg_stat_statements | Track JSONB query performance to identify candidates for column promotion |

---

## Migration and Scaling Considerations

### JSONB Column Promotion Pattern

When a value inside a JSONB column becomes frequently queried, promote it to a relational column:

```sql
-- Step 1: Add the relational column (nullable initially)
ALTER TABLE transaction ADD COLUMN intermediary_bic VARCHAR(11);

-- Step 2: Backfill from JSONB
UPDATE transaction SET intermediary_bic = intermediary_details->>'bic'
WHERE intermediary_details->>'bic' IS NOT NULL;

-- Step 3: Add index
CREATE INDEX idx_txn_intermediary_bic ON transaction (intermediary_bic) WHERE intermediary_bic IS NOT NULL;

-- Step 4: Update application code to write to both column and JSONB
-- Step 5: After validation period, switch queries to use the relational column
```

### JSONB Schema Evolution Strategy

1. **Versioning**: Include a `"_schema_version": 1` field in every JSONB payload
2. **Upcasting**: Application reads must handle all historical schema versions and upcast to current
3. **Background migration**: Run batch jobs to update old JSONB payloads to current schema version during low-traffic periods
4. **Validation on write**: Enforce current schema version on all new writes using JSON Schema validation

### Scaling Path

- **Phase 1 (< 5M txn/month)**: Single PostgreSQL instance with read replica
- **Phase 2 (5-50M txn/month)**: Add dedicated read replicas for reporting. Partition transaction and audit tables monthly. Monitor GIN index sizes.
- **Phase 3 (50M+ txn/month)**: Consider Citus for horizontal sharding by tenant_id. Evaluate promoting high-frequency JSONB query paths to relational columns. Consider separate Elasticsearch cluster for full-text search across case narratives.

### Storage Estimation

At 10M transactions/month with average 1.5 KB per transaction row (including JSONB):
- Transaction table: ~180 GB/year
- Alert table (assuming 5% alert rate): ~15 GB/year
- Audit log: ~50 GB/year
- JSONB GIN indexes: ~30% overhead on JSONB-indexed tables
- Total estimated: ~350-400 GB/year
