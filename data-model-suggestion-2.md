# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Anti-Money Laundering Platform (Candidate #432)
> Date: 2026-05-25

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the AML platform. Every state change in the system -- a transaction ingested, an alert generated, an analyst disposition, a SAR filed -- is captured as an immutable event in an append-only event store. Read-optimized projections (materialized views) are built from the event stream to serve queries for alert queues, case dashboards, entity profiles, and regulatory reports.

This architecture is a natural fit for AML platforms because the domain has an inherent requirement for complete, immutable, tamper-evident audit trails. Regulators (FFIEC, FCA, AUSTRAC) require the ability to reconstruct the exact state of any investigation at any point in time. Event sourcing delivers this by design rather than as an afterthought bolted onto a CRUD model.

The write side uses PostgreSQL as the event store (leveraging its ACID guarantees and partitioning), while the read side uses purpose-built projections in PostgreSQL, Redis, and optionally Elasticsearch for full-text search across case narratives and screening results.

---

## Architecture Overview

```
                    +-----------------+
                    |  Command Bus    |
                    | (Write Side)    |
                    +--------+--------+
                             |
                    +--------v--------+
                    |   Event Store   |
                    | (PostgreSQL)    |
                    +--------+--------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v------+ +----v------+ +-----v--------+
     | Alert Queue   | | Entity    | | Regulatory   |
     | Projection    | | Profile   | | Reporting    |
     | (PostgreSQL)  | | Projection| | Projection   |
     +---------------+ | (PG+Redis)| | (PostgreSQL) |
                        +-----------+ +--------------+
              |              |              |
              +--------------+--------------+
                             |
                    +--------v--------+
                    |   Query Bus     |
                    |  (Read Side)    |
                    +-----------------+
```

---

## Event Store Schema

### Core Event Store Tables

```sql
-- The append-only event store: the single source of truth
CREATE TABLE event_store (
    event_id            UUID NOT NULL DEFAULT gen_random_uuid(),
    stream_id           UUID NOT NULL,                 -- aggregate root ID
    stream_type         VARCHAR(50) NOT NULL,          -- aggregate type name
    event_type          VARCHAR(100) NOT NULL,         -- fully qualified event type
    event_version       INTEGER NOT NULL,              -- position within the stream
    event_data          JSONB NOT NULL,                -- event payload
    metadata            JSONB NOT NULL DEFAULT '{}',   -- correlation IDs, causation, user context
    occurred_at         TIMESTAMPTZ NOT NULL,          -- business time: when the event happened
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(), -- system time: when stored
    tenant_id           UUID NOT NULL,
    causation_id        UUID,                          -- ID of the command or event that caused this
    correlation_id      UUID,                          -- end-to-end trace ID

    PRIMARY KEY (event_id, recorded_at),
    UNIQUE (stream_id, event_version, recorded_at)     -- optimistic concurrency control
) PARTITION BY RANGE (recorded_at);

-- Monthly partitions
CREATE TABLE event_store_2026_01 PARTITION OF event_store FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE event_store_2026_02 PARTITION OF event_store FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... managed by pg_partman

CREATE INDEX idx_event_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_event_type ON event_store (event_type, recorded_at);
CREATE INDEX idx_event_correlation ON event_store (correlation_id);
CREATE INDEX idx_event_tenant ON event_store (tenant_id, recorded_at);
CREATE INDEX idx_event_causation ON event_store (causation_id) WHERE causation_id IS NOT NULL;

-- Snapshot store for aggregates with long event histories
CREATE TABLE snapshot_store (
    snapshot_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(50) NOT NULL,
    snapshot_version    INTEGER NOT NULL,              -- event_version at snapshot time
    snapshot_data       JSONB NOT NULL,                -- serialized aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON snapshot_store (stream_id, snapshot_version DESC);

-- Subscription checkpoints for projection rebuilds
CREATE TABLE projection_checkpoint (
    projection_name     VARCHAR(100) NOT NULL,
    partition_key       VARCHAR(100) NOT NULL DEFAULT 'default',
    last_event_id       UUID NOT NULL,
    last_event_time     TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, partition_key)
);

-- Dead letter queue for failed event processing
CREATE TABLE dead_letter_event (
    dead_letter_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id            UUID NOT NULL,
    projection_name     VARCHAR(100) NOT NULL,
    error_message       TEXT NOT NULL,
    error_stack         TEXT,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    next_retry_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at         TIMESTAMPTZ
);

CREATE INDEX idx_dead_letter_retry ON dead_letter_event (next_retry_at)
    WHERE resolved_at IS NULL;
```

---

## Event Type Catalogue

### Transaction Domain Events

```sql
-- Example event_data payloads stored in event_store.event_data

-- stream_type: 'Transaction'
-- event_type: 'TransactionIngested'
-- {
--   "transaction_id": "uuid",
--   "external_ref": "SWIFT-REF-123",
--   "transaction_type": "WIRE_TRANSFER",
--   "direction": "OUTBOUND",
--   "amount": 15000.00,
--   "currency": "USD",
--   "amount_usd": 15000.00,
--   "originator": {
--     "party_id": "uuid",
--     "account_id": "uuid",
--     "name": "John Doe",
--     "country": "USA",
--     "bic": "CHASUS33"
--   },
--   "beneficiary": {
--     "party_id": "uuid",
--     "account_id": "uuid",
--     "name": "Shell Corp Ltd",
--     "country": "CYM",
--     "bic": "BARCKYKY"
--   },
--   "payment_rail": "SWIFT",
--   "payment_channel": "ONLINE",
--   "remittance_info": "Invoice 2026-0042",
--   "end_to_end_id": "E2E-20260525-001",
--   "source_system": "core_banking_v3"
-- }

-- event_type: 'TransactionRiskScored'
-- {
--   "transaction_id": "uuid",
--   "risk_score": 0.82,
--   "risk_factors": ["HIGH_RISK_JURISDICTION", "VELOCITY_BREACH", "AMOUNT_THRESHOLD"],
--   "model_id": "uuid",
--   "model_version": "2.3.1",
--   "scoring_latency_ms": 12
-- }

-- event_type: 'TransactionFlagged'
-- {
--   "transaction_id": "uuid",
--   "rule_id": "uuid",
--   "rule_code": "STRUCT-001",
--   "typology": "STRUCTURING",
--   "severity": "HIGH",
--   "explanation": "Three cash deposits totaling $9,800 within 24 hours, each below $10,000 CTR threshold"
-- }
```

### Party (KYC/CDD) Domain Events

```sql
-- stream_type: 'Party'

-- event_type: 'PartyOnboarded'
-- {
--   "party_type": "INDIVIDUAL",
--   "legal_name": "John Michael Doe",
--   "date_of_birth": "1985-03-15",
--   "country_of_residence": "USA",
--   "nationality": "USA",
--   "risk_rating": "MEDIUM",
--   "cdd_level": "STANDARD",
--   "identification": {
--     "type": "PASSPORT",
--     "number": "encrypted:...",
--     "issuing_country": "USA",
--     "expiry_date": "2030-06-20"
--   }
-- }

-- event_type: 'PartyRiskRatingChanged'
-- {
--   "old_rating": "MEDIUM",
--   "new_rating": "HIGH",
--   "reason": "Matched PEP list - Government Official, Country X",
--   "changed_by": "analyst-uuid",
--   "effective_date": "2026-05-25"
-- }

-- event_type: 'PartyPEPStatusIdentified'
-- {
--   "pep_category": "GOVERNMENT_OFFICIAL",
--   "pep_level": "NATIONAL",
--   "source": "WORLD_CHECK",
--   "match_score": 0.95,
--   "identified_by": "SCREENING_ENGINE"
-- }

-- event_type: 'PartyCDDReviewCompleted'
-- {
--   "review_type": "PERIODIC",
--   "old_cdd_level": "STANDARD",
--   "new_cdd_level": "ENHANCED",
--   "findings": "Increased transaction volume to high-risk jurisdictions",
--   "next_review_date": "2027-05-25",
--   "reviewed_by": "analyst-uuid"
-- }

-- event_type: 'PartyRelationshipEstablished'
-- {
--   "related_party_id": "uuid",
--   "relationship_type": "BENEFICIAL_OWNER",
--   "ownership_percentage": 25.5,
--   "effective_date": "2026-01-15"
-- }
```

### Alert Domain Events

```sql
-- stream_type: 'Alert'

-- event_type: 'AlertGenerated'
-- {
--   "alert_source": "RULE",
--   "rule_id": "uuid",
--   "rule_code": "VEL-003",
--   "party_id": "uuid",
--   "account_id": "uuid",
--   "risk_score": 0.78,
--   "severity": "HIGH",
--   "typology": "VELOCITY",
--   "summary": "14 outbound wire transfers totaling $127,000 in 48 hours to 8 distinct beneficiaries in 4 countries",
--   "triggering_transactions": ["txn-uuid-1", "txn-uuid-2", "..."],
--   "explanation": "Entity average is 2 outbound wires per week. Current 48-hour count exceeds 6-sigma threshold."
-- }

-- event_type: 'AlertAssigned'
-- {
--   "assigned_to": "analyst-uuid",
--   "assigned_by": "system",
--   "assignment_reason": "AUTO_ROUND_ROBIN",
--   "due_date": "2026-05-28T17:00:00Z"
-- }

-- event_type: 'AlertEscalated'
-- {
--   "escalated_from": "analyst-uuid-l1",
--   "escalated_to": "analyst-uuid-l2",
--   "escalation_reason": "Complex layering structure identified requiring senior review",
--   "new_severity": "CRITICAL"
-- }

-- event_type: 'AlertDispositioned'
-- {
--   "disposition": "TRUE_POSITIVE",
--   "disposition_reason": "Confirmed structuring pattern. Three deposits of $9,500, $9,700, and $9,600 within 24 hours, all cash, all same branch.",
--   "dispositioned_by": "analyst-uuid",
--   "case_id": "uuid",
--   "time_to_disposition_minutes": 42
-- }

-- event_type: 'AlertBulkDispositioned'
-- {
--   "alert_ids": ["uuid-1", "uuid-2", "..."],
--   "disposition": "FALSE_POSITIVE",
--   "disposition_reason": "Batch of payroll transfers from verified corporate account matching established pattern",
--   "dispositioned_by": "analyst-uuid",
--   "bulk_rule": "PAYROLL_PATTERN_MATCH"
-- }
```

### Case Management Domain Events

```sql
-- stream_type: 'Case'

-- event_type: 'CaseOpened'
-- {
--   "case_type": "SUSPICIOUS_ACTIVITY",
--   "priority": "HIGH",
--   "subject_party_id": "uuid",
--   "summary": "Structuring pattern detected across multiple accounts",
--   "triggering_alert_ids": ["uuid-1", "uuid-2"],
--   "opened_by": "analyst-uuid"
-- }

-- event_type: 'CaseAssigned'
-- {
--   "assigned_to": "analyst-uuid",
--   "assigned_by": "team-lead-uuid",
--   "due_date": "2026-06-01T17:00:00Z"
-- }

-- event_type: 'CaseNoteAdded'
-- {
--   "note_text": "Reviewed transaction flow. Funds received from 3 shell companies in Cyprus, immediately transferred to personal account in Switzerland. Classic layering pattern.",
--   "note_type": "INVESTIGATION_FINDING",
--   "added_by": "analyst-uuid"
-- }

-- event_type: 'CaseEvidenceAttached'
-- {
--   "evidence_type": "TRANSACTION_EXTRACT",
--   "file_name": "txn_extract_2026Q1.csv",
--   "file_path": "s3://aml-evidence/cases/uuid/txn_extract_2026Q1.csv",
--   "file_size_bytes": 45230,
--   "mime_type": "text/csv",
--   "sha256_hash": "a3f2...",
--   "uploaded_by": "analyst-uuid"
-- }

-- event_type: 'CaseEscalated'
-- {
--   "escalated_to": "compliance-officer-uuid",
--   "escalation_reason": "Total suspicious amount exceeds $500,000. SAR filing recommended.",
--   "new_priority": "CRITICAL",
--   "escalated_by": "senior-analyst-uuid"
-- }

-- event_type: 'CaseClosed'
-- {
--   "closure_status": "CLOSED_SAR_FILED",
--   "closure_reason": "SAR filed with FinCEN. Case documentation complete.",
--   "sar_id": "uuid",
--   "closed_by": "compliance-officer-uuid",
--   "total_investigation_hours": 8.5
-- }
```

### Sanctions Screening Domain Events

```sql
-- stream_type: 'ScreeningSession'

-- event_type: 'ScreeningInitiated'
-- {
--   "screening_type": "ONBOARDING",
--   "screened_party_id": "uuid",
--   "screened_names": ["John Doe", "J. Doe", "Jon Doe"],
--   "lists_screened": ["OFAC_SDN", "UN_CONSOLIDATED", "EU_SANCTIONS", "UK_SANCTIONS"],
--   "initiated_by": "system"
-- }

-- event_type: 'ScreeningMatchFound'
-- {
--   "screened_name": "John Doe",
--   "matched_entry_id": "ofac-uid-12345",
--   "matched_name": "JOHN DOE",
--   "matched_list": "OFAC_SDN",
--   "match_score": 0.97,
--   "match_algorithm": "JARO_WINKLER",
--   "program": "SDGT"
-- }

-- event_type: 'ScreeningMatchDispositioned'
-- {
--   "match_id": "uuid",
--   "disposition": "FALSE_POSITIVE",
--   "reason": "Different date of birth. OFAC entry DOB 1960-01-01, our customer DOB 1985-03-15.",
--   "dispositioned_by": "analyst-uuid"
-- }

-- event_type: 'WatchlistUpdated'
-- {
--   "source_code": "OFAC_SDN",
--   "entries_added": 12,
--   "entries_removed": 3,
--   "entries_modified": 7,
--   "update_timestamp": "2026-05-25T14:30:00Z",
--   "total_entries": 12847
-- }
```

### Regulatory Reporting Domain Events

```sql
-- stream_type: 'SARReport'

-- event_type: 'SARDrafted'
-- {
--   "case_id": "uuid",
--   "report_type": "SAR",
--   "jurisdiction": "USA",
--   "regulator_code": "FINCEN",
--   "filing_type": "INITIAL",
--   "subject_party_id": "uuid",
--   "activity_start_date": "2026-03-01",
--   "activity_end_date": "2026-05-20",
--   "total_amount": 287500.00,
--   "suspicious_activity_types": ["STRUCTURING", "LAYERING"],
--   "prepared_by": "analyst-uuid"
-- }

-- event_type: 'SARNarrativeGenerated'
-- {
--   "sar_id": "uuid",
--   "narrative_source": "AI_GENERATED",
--   "model_id": "uuid",
--   "model_version": "1.2.0",
--   "narrative_length_chars": 3240,
--   "narrative_preview": "Between March 1, 2026 and May 20, 2026, the subject conducted..."
-- }

-- event_type: 'SARNarrativeReviewed'
-- {
--   "sar_id": "uuid",
--   "reviewed_by": "analyst-uuid",
--   "changes_made": true,
--   "change_summary": "Added detail about beneficiary account in Switzerland. Corrected transaction count from 14 to 16."
-- }

-- event_type: 'SARApproved'
-- {
--   "sar_id": "uuid",
--   "approved_by": "compliance-officer-uuid",
--   "approval_notes": "Reviewed narrative and supporting evidence. Approved for filing."
-- }

-- event_type: 'SARSubmitted'
-- {
--   "sar_id": "uuid",
--   "submission_method": "BSA_EFILING_BATCH",
--   "submission_ref": "BSA-2026-0525-001",
--   "submitted_at": "2026-05-25T16:00:00Z",
--   "submitted_by": "system"
-- }

-- event_type: 'SARAccepted'
-- {
--   "sar_id": "uuid",
--   "regulator_ack_ref": "FINCEN-ACK-2026-12345",
--   "accepted_at": "2026-05-25T18:30:00Z"
-- }
```

---

## Read Model Projections

### Projection 1: Alert Queue (for analyst workbench)

```sql
-- Materialized read model rebuilt from alert events
CREATE TABLE proj_alert_queue (
    alert_id            UUID PRIMARY KEY,
    alert_number        BIGINT NOT NULL,
    alert_date          TIMESTAMPTZ NOT NULL,
    alert_source        VARCHAR(20) NOT NULL,
    rule_code           VARCHAR(50),
    model_version       VARCHAR(20),
    party_id            UUID NOT NULL,
    party_name          TEXT NOT NULL,
    party_risk_rating   VARCHAR(10) NOT NULL,
    account_id          UUID,
    risk_score          DECIMAL(5,2) NOT NULL,
    severity            VARCHAR(10) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    typology            VARCHAR(50),
    summary             TEXT NOT NULL,
    explanation         TEXT,
    assigned_to         UUID,
    assigned_to_name    TEXT,
    assigned_at         TIMESTAMPTZ,
    due_date            TIMESTAMPTZ,
    case_id             UUID,
    case_number         BIGINT,
    transaction_count   INTEGER NOT NULL DEFAULT 0,
    total_amount_usd    DECIMAL(18,4),
    tenant_id           UUID NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL,
    version             BIGINT NOT NULL DEFAULT 0       -- optimistic concurrency for projection
);

CREATE INDEX idx_proj_alert_queue_open ON proj_alert_queue (severity DESC, risk_score DESC)
    WHERE status IN ('NEW', 'ASSIGNED', 'UNDER_REVIEW');
CREATE INDEX idx_proj_alert_assigned ON proj_alert_queue (assigned_to, status);
CREATE INDEX idx_proj_alert_tenant ON proj_alert_queue (tenant_id, status);
CREATE INDEX idx_proj_alert_party ON proj_alert_queue (party_id);
```

### Projection 2: Entity 360-Degree Profile

```sql
-- Denormalized entity profile for investigation context
CREATE TABLE proj_entity_profile (
    party_id            UUID PRIMARY KEY,
    party_type          VARCHAR(20) NOT NULL,
    legal_name          TEXT NOT NULL,
    aliases             TEXT[] NOT NULL DEFAULT '{}',
    risk_rating         VARCHAR(10) NOT NULL,
    cdd_level           VARCHAR(20) NOT NULL,
    pep_status          BOOLEAN NOT NULL DEFAULT FALSE,
    pep_category        VARCHAR(50),
    country_of_residence VARCHAR(3),
    onboarding_date     DATE NOT NULL,

    -- Aggregated statistics
    total_accounts      INTEGER NOT NULL DEFAULT 0,
    active_accounts     INTEGER NOT NULL DEFAULT 0,
    total_txn_count_30d INTEGER NOT NULL DEFAULT 0,
    total_txn_amount_30d DECIMAL(18,4) NOT NULL DEFAULT 0,
    total_txn_count_90d INTEGER NOT NULL DEFAULT 0,
    total_txn_amount_90d DECIMAL(18,4) NOT NULL DEFAULT 0,
    distinct_counterparties_30d INTEGER NOT NULL DEFAULT 0,
    distinct_countries_30d INTEGER NOT NULL DEFAULT 0,
    avg_txn_amount_90d  DECIMAL(18,4),
    max_txn_amount_90d  DECIMAL(18,4),

    -- Alert and case history
    total_alerts        INTEGER NOT NULL DEFAULT 0,
    open_alerts         INTEGER NOT NULL DEFAULT 0,
    total_cases         INTEGER NOT NULL DEFAULT 0,
    open_cases          INTEGER NOT NULL DEFAULT 0,
    total_sars_filed    INTEGER NOT NULL DEFAULT 0,
    last_alert_date     TIMESTAMPTZ,
    last_sar_date       TIMESTAMPTZ,

    -- Screening status
    last_screening_date TIMESTAMPTZ,
    pending_screening_matches INTEGER NOT NULL DEFAULT 0,
    sanctions_hit       BOOLEAN NOT NULL DEFAULT FALSE,

    -- Relationships
    related_party_count INTEGER NOT NULL DEFAULT 0,
    beneficial_owners   JSONB NOT NULL DEFAULT '[]',

    tenant_id           UUID NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL,
    version             BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_proj_entity_risk ON proj_entity_profile (risk_rating, tenant_id);
CREATE INDEX idx_proj_entity_name_trgm ON proj_entity_profile USING gin (legal_name gin_trgm_ops);
CREATE INDEX idx_proj_entity_pep ON proj_entity_profile (pep_status) WHERE pep_status = TRUE;
```

### Projection 3: Case Dashboard

```sql
-- Denormalized case view for investigation workbench
CREATE TABLE proj_case_dashboard (
    case_id             UUID PRIMARY KEY,
    case_number         BIGINT NOT NULL,
    case_type           VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    priority            VARCHAR(10) NOT NULL,

    -- Subject
    subject_party_id    UUID NOT NULL,
    subject_name        TEXT NOT NULL,
    subject_risk_rating VARCHAR(10) NOT NULL,

    -- Assignment
    assigned_to         UUID,
    assigned_to_name    TEXT,
    assigned_at         TIMESTAMPTZ,
    escalated_to        UUID,
    escalated_to_name   TEXT,

    -- Content
    summary             TEXT NOT NULL,
    findings            TEXT,
    recommendation      TEXT,

    -- Linked alerts and transactions
    linked_alert_count  INTEGER NOT NULL DEFAULT 0,
    linked_transaction_count INTEGER NOT NULL DEFAULT 0,
    total_suspicious_amount DECIMAL(18,4),

    -- Evidence
    evidence_count      INTEGER NOT NULL DEFAULT 0,
    note_count          INTEGER NOT NULL DEFAULT 0,

    -- SAR
    sar_id              UUID,
    sar_status          VARCHAR(20),

    -- Timing
    opened_at           TIMESTAMPTZ NOT NULL,
    due_date            TIMESTAMPTZ,
    closed_at           TIMESTAMPTZ,
    age_hours           DECIMAL(10,1),

    tenant_id           UUID NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL,
    version             BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_proj_case_open ON proj_case_dashboard (priority DESC, opened_at)
    WHERE status NOT LIKE 'CLOSED%';
CREATE INDEX idx_proj_case_assigned ON proj_case_dashboard (assigned_to, status);
CREATE INDEX idx_proj_case_tenant ON proj_case_dashboard (tenant_id, status);
```

### Projection 4: Regulatory Reporting

```sql
-- SAR/CTR filing status tracking
CREATE TABLE proj_regulatory_filings (
    report_id           UUID PRIMARY KEY,
    report_type         VARCHAR(20) NOT NULL,
    report_number       BIGINT NOT NULL,
    jurisdiction        VARCHAR(3) NOT NULL,
    regulator_code      VARCHAR(20) NOT NULL,
    filing_type         VARCHAR(20) NOT NULL,
    status              VARCHAR(20) NOT NULL,

    -- Subject
    subject_party_id    UUID NOT NULL,
    subject_name        TEXT NOT NULL,

    -- Activity
    activity_start_date DATE,
    activity_end_date   DATE,
    total_amount        DECIMAL(18,4),
    suspicious_activity_types TEXT[],

    -- Workflow
    case_id             UUID,
    prepared_by_name    TEXT,
    reviewed_by_name    TEXT,
    approved_by_name    TEXT,
    approved_at         TIMESTAMPTZ,
    submitted_at        TIMESTAMPTZ,
    submission_ref      TEXT,
    regulator_ack_ref   TEXT,

    -- Narrative status
    narrative_source    VARCHAR(20),                   -- 'AI_GENERATED', 'MANUAL'
    narrative_reviewed  BOOLEAN NOT NULL DEFAULT FALSE,

    tenant_id           UUID NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL,
    version             BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_proj_filing_status ON proj_regulatory_filings (status, regulator_code);
CREATE INDEX idx_proj_filing_jurisdiction ON proj_regulatory_filings (jurisdiction, report_type);
CREATE INDEX idx_proj_filing_submitted ON proj_regulatory_filings (submitted_at)
    WHERE submitted_at IS NOT NULL;
```

### Projection 5: Compliance Analytics (for dashboards and KPIs)

```sql
-- Pre-aggregated metrics for compliance dashboards
CREATE TABLE proj_compliance_metrics (
    metric_date         DATE NOT NULL,
    tenant_id           UUID NOT NULL,
    metric_name         VARCHAR(50) NOT NULL,
    metric_value        DECIMAL(18,4) NOT NULL,
    dimension_1         VARCHAR(50),                   -- e.g. typology, severity, jurisdiction
    dimension_2         VARCHAR(50),                   -- e.g. analyst, team
    last_updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (metric_date, tenant_id, metric_name, dimension_1, dimension_2)
);

-- Example metrics:
-- 'alerts_generated', 'alerts_dispositioned', 'false_positive_rate',
-- 'avg_disposition_time_minutes', 'cases_opened', 'cases_closed',
-- 'sars_filed', 'sars_accepted', 'transactions_monitored',
-- 'screening_matches_pending', 'high_risk_entities_count'
```

---

## Event Processing Infrastructure

### Projection Rebuild Script

```sql
-- Function to replay events for a specific projection
CREATE OR REPLACE FUNCTION rebuild_projection(
    p_projection_name VARCHAR(100),
    p_from_event_time TIMESTAMPTZ DEFAULT '1970-01-01'::TIMESTAMPTZ
) RETURNS TABLE(events_processed BIGINT, duration_ms BIGINT) AS $$
DECLARE
    v_start TIMESTAMPTZ;
    v_count BIGINT := 0;
BEGIN
    v_start := clock_timestamp();

    -- Reset checkpoint
    DELETE FROM projection_checkpoint
    WHERE projection_name = p_projection_name;

    -- The actual replay logic is handled by the application-layer
    -- event processor, which reads events in order and applies
    -- the projection handler for each event type.

    -- This function records the rebuild metadata.
    INSERT INTO projection_rebuild_log (
        projection_name, rebuild_started_at, from_event_time
    ) VALUES (
        p_projection_name, v_start, p_from_event_time
    );

    events_processed := v_count;
    duration_ms := EXTRACT(EPOCH FROM (clock_timestamp() - v_start)) * 1000;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;

-- Rebuild tracking
CREATE TABLE projection_rebuild_log (
    rebuild_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name     VARCHAR(100) NOT NULL,
    rebuild_started_at  TIMESTAMPTZ NOT NULL,
    rebuild_completed_at TIMESTAMPTZ,
    from_event_time     TIMESTAMPTZ NOT NULL,
    events_processed    BIGINT,
    status              VARCHAR(20) NOT NULL DEFAULT 'RUNNING' CHECK (status IN ('RUNNING', 'COMPLETED', 'FAILED')),
    error_message       TEXT
);
```

### Outbox Pattern for Reliable Event Publishing

```sql
-- Transactional outbox for reliable event publishing to message broker
CREATE TABLE event_outbox (
    outbox_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id            UUID NOT NULL,
    destination_topic   VARCHAR(100) NOT NULL,
    event_type          VARCHAR(100) NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at        TIMESTAMPTZ,
    is_published        BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE INDEX idx_outbox_unpublished ON event_outbox (created_at)
    WHERE is_published = FALSE;
```

---

## Pros and Cons

### Pros

1. **Perfect audit trail by design**: Every state change is an immutable event. The event store IS the audit trail. There is no separate audit table to maintain, no triggers to forget, no risk of audit gaps. This directly satisfies FFIEC examination requirements for complete investigation history reconstruction.

2. **Point-in-time state reconstruction**: Replaying events up to any timestamp reconstructs the exact state of any entity, alert, case, or screening result at that moment. Regulators frequently ask "What did you know about this customer on the date of this transaction?" -- event sourcing answers this question definitively.

3. **Independent read model optimization**: Each projection is optimized for its specific query pattern. The alert queue projection is optimized for analyst workbench queries. The entity profile projection is optimized for 360-degree views. The regulatory reporting projection is optimized for SAR/CTR tracking. None compromises the others.

4. **Projection rebuilds for new requirements**: When a new regulatory requirement demands a new view of the data (e.g., AMLA reporting requirements in 2027), a new projection can be built by replaying the entire event history without modifying the source data or existing projections.

5. **Natural event-driven architecture**: The event store feeds downstream consumers (ML model training, analytics, real-time alerting, cross-system notifications) through a single event stream. This eliminates the CDC (change data capture) complexity that CRUD-based systems require for downstream integration.

6. **Causation and correlation tracking**: The `causation_id` and `correlation_id` fields enable full traceability from transaction ingestion through risk scoring, alert generation, case investigation, and SAR filing -- the complete lifecycle chain that compliance officers need for regulatory reporting.

### Cons

1. **Eventual consistency in read models**: Projections lag behind the event store. An analyst might disposition an alert but see the stale status for a few seconds until the projection updates. For a compliance workbench this is usually acceptable but must be communicated in the UI.

2. **Increased storage requirements**: Events are never deleted, and projections duplicate data. A system handling 10M transactions/month will generate 50-100M events/month across all domains. At an average of 2KB per event, that is 100-200 GB/month of event data alone, plus projection storage.

3. **Complexity of event schema evolution**: As the domain evolves, event schemas change. An `AlertGenerated` event from 2026 may have different fields than one from 2028. The system must handle upcasting (transforming old events to new schemas during replay) without data loss.

4. **Projection rebuild time**: For a system with billions of events, rebuilding a projection from scratch can take hours or days. Snapshots help, but the rebuild window is a significant operational concern that must be planned for.

5. **Steeper learning curve**: CQRS/ES is architecturally more complex than CRUD. Development teams need to understand event modeling, aggregate boundaries, eventual consistency, idempotent event handlers, and projection management. Hiring for this skillset is harder than for traditional RDBMS development.

6. **Debugging complexity**: When a projection shows incorrect data, the root cause must be traced through the event stream. This requires tooling for event stream inspection and replay that does not exist out of the box.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Event store | PostgreSQL 16+ | ACID guarantees for event append, partitioning, proven at scale |
| Message broker | Apache Kafka or NATS JetStream | Ordered event streaming to projections and downstream consumers |
| Projection database | PostgreSQL 16+ | Same technology as event store reduces operational overhead |
| Cache layer | Redis 7+ | Low-latency entity profile cache and alert queue sorting |
| Search projection | Elasticsearch / OpenSearch | Full-text search across case narratives and screening results |
| Event processing | Custom application code (Rust/Go/Java) | Event handlers applying events to projections |
| Outbox publisher | Debezium (CDC from outbox table) | Reliable event publishing from PostgreSQL outbox to Kafka |
| Schema registry | Confluent Schema Registry or Buf | Event schema versioning and compatibility checking |
| Monitoring | Prometheus + Grafana | Projection lag monitoring, event throughput, dead letter tracking |

---

## Migration and Scaling Considerations

### Event Store Growth Management

- **Partitioning**: Monthly partitions on `recorded_at` using pg_partman
- **Archival**: Events older than the regulatory retention period (5-7 years) can be archived to cold storage (S3 + Parquet) while maintaining the ability to replay for regulatory requests
- **Snapshots**: Take aggregate snapshots every 100-1000 events to bound replay time for frequently-accessed aggregates (entity profiles with long transaction histories)
- **Compression**: PostgreSQL TOAST compression for JSONB event payloads reduces storage by 40-60%

### Projection Scaling

- **Independent scaling**: Each projection can run on its own database instance if query load demands it
- **Parallel rebuilds**: Projections can be rebuilt in parallel since they read from the same event store independently
- **Blue-green projection updates**: Deploy new projection version alongside old, switch traffic once caught up
- **Read replicas**: PostgreSQL streaming replication for read-heavy projections (alert queue, entity profile)

### Multi-Region Deployment

- **Event store replication**: Use PostgreSQL logical replication to replicate the event store to regional clusters
- **Regional projections**: Build projections locally in each region from the replicated event stream
- **Data residency**: Partition the event store by `tenant_id` and route tenants to region-appropriate clusters
- **Conflict resolution**: Since events are append-only and immutable, there are no write conflicts to resolve in multi-region setups (as long as each aggregate is owned by a single region)

### Migration from CRUD to Event Sourcing

If starting with a CRUD model and migrating to event sourcing:

1. **Dual-write phase**: Write to both CRUD tables and event store simultaneously
2. **Event generation**: Generate synthetic "snapshot" events from current CRUD state for each aggregate
3. **Projection validation**: Build projections from events and compare with CRUD tables to verify correctness
4. **Cutover**: Switch reads to projections, writes to event store
5. **CRUD retirement**: Decommission CRUD tables after validation period

### Estimated Infrastructure (10M transactions/month)

| Component | Size | Instances |
|-----------|------|-----------|
| Event store (PostgreSQL) | 2 TB/year | 1 primary + 1 standby |
| Projection databases | 500 GB | 1 primary + 2 read replicas |
| Kafka cluster | 1 TB retention | 3 brokers |
| Redis cache | 32 GB | 1 primary + 1 replica |
| Elasticsearch | 500 GB | 3-node cluster |
