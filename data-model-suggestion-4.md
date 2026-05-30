# Data Model Suggestion 4: Graph Database with Polyglot Persistence

> Project: Anti-Money Laundering Platform (Candidate #432)
> Date: 2026-05-25

## Overview

This model uses a **graph database as the primary analytical engine** for the AML platform, combined with a relational database for operational workflows and a time-series database for transaction metrics. The graph database is not an auxiliary analytics layer -- it is the core where entity relationships, transaction flows, and network patterns are modeled, queried, and analyzed.

The rationale is domain-driven: money laundering is fundamentally a network problem. Criminals use layers of accounts, shell companies, intermediaries, and nominees to obscure the origin of funds. Traditional relational queries (even with recursive CTEs) cannot efficiently traverse these networks at the depth and speed required for real-time detection. Graph databases treat relationships as first-class citizens, making multi-hop traversals, pattern matching, and community detection operations that are natural and performant rather than bolted on.

This model uses **Neo4j** as the graph database (the de facto standard for financial crime graph analytics, used by ICIJ for the Panama Papers investigation and by multiple global banks for AML), **PostgreSQL** for operational state management (alert queues, case workflows, regulatory reports), and **TimescaleDB** (a PostgreSQL extension) for time-series transaction aggregations that feed the behavioural profiling engine.

---

## Architecture: Polyglot Persistence

```
+-------------------+     +-------------------+     +-------------------+
|   Neo4j           |     |   PostgreSQL      |     |   TimescaleDB     |
|   (Graph DB)      |     |   (Operational)   |     |   (Time-Series)   |
+-------------------+     +-------------------+     +-------------------+
| Entities          |     | Alert queue       |     | Transaction       |
| Relationships     |     | Case management   |     |   time-series     |
| Transaction flows |     | SAR/CTR reports   |     | Entity behavioral |
| Network patterns  |     | Analyst workflows |     |   profiles        |
| Entity resolution |     | Audit logs        |     | Aggregation       |
| Community detect. |     | System config     |     |   metrics         |
+-------------------+     +-------------------+     +-------------------+
         ^                        ^                        ^
         |                        |                        |
         +------------------------+------------------------+
                                  |
                    +-------------v-----------+
                    |   Event Bus (Kafka)     |
                    |   Synchronization Layer |
                    +-------------------------+
```

---

## Graph Database Schema (Neo4j)

### Node Types

```cypher
// ============================================================
// PARTY NODES
// ============================================================

// Individual party (customer, counterparty, beneficial owner)
CREATE CONSTRAINT party_id_unique FOR (p:Party) REQUIRE p.party_id IS UNIQUE;
CREATE CONSTRAINT party_id_exists FOR (p:Party) REQUIRE p.party_id IS NOT NULL;

// Example node creation:
CREATE (p:Party:Individual {
  party_id: 'uuid-1',
  tenant_id: 'tenant-uuid',
  legal_name: 'John Michael Doe',
  party_type: 'INDIVIDUAL',
  date_of_birth: date('1985-03-15'),
  country_of_residence: 'USA',
  nationality: 'USA',
  risk_rating: 'HIGH',
  cdd_level: 'ENHANCED',
  pep_status: true,
  pep_category: 'GOVERNMENT_OFFICIAL',
  onboarding_date: date('2024-06-15'),
  last_review_date: date('2026-05-01'),
  is_active: true
})

CREATE (o:Party:Organisation {
  party_id: 'uuid-2',
  tenant_id: 'tenant-uuid',
  legal_name: 'Shell Corp Holdings Ltd',
  party_type: 'ORGANISATION',
  country_of_incorporation: 'CYM',
  industry_code: '523999',
  lei: '549300ABC123DEF456',
  risk_rating: 'VERY_HIGH',
  cdd_level: 'ENHANCED',
  onboarding_date: date('2025-01-10'),
  is_active: true
})

// ============================================================
// ACCOUNT NODES
// ============================================================

CREATE CONSTRAINT account_id_unique FOR (a:Account) REQUIRE a.account_id IS UNIQUE;

CREATE (a:Account {
  account_id: 'acct-uuid-1',
  tenant_id: 'tenant-uuid',
  account_type: 'CURRENT',
  currency: 'USD',
  status: 'ACTIVE',
  risk_rating: 'HIGH',
  institution_bic: 'CHASUS33',
  opened_date: date('2024-06-15'),
  iban: 'US12345678901234',
  is_correspondent: false
})

// ============================================================
// TRANSACTION NODES
// ============================================================

CREATE CONSTRAINT txn_id_unique FOR (t:Transaction) REQUIRE t.transaction_id IS UNIQUE;

CREATE (t:Transaction {
  transaction_id: 'txn-uuid-1',
  tenant_id: 'tenant-uuid',
  transaction_date: datetime('2026-05-25T14:30:00Z'),
  transaction_type: 'WIRE_TRANSFER',
  direction: 'OUTBOUND',
  amount: 15000.00,
  currency: 'USD',
  amount_usd: 15000.00,
  payment_rail: 'SWIFT',
  payment_channel: 'ONLINE',
  risk_score: 0.82,
  source_system: 'core_banking_v3',
  end_to_end_id: 'E2E-20260525-001',
  remittance_info: 'Invoice 2026-0042'
})

// ============================================================
// WATCHLIST ENTRY NODES
// ============================================================

CREATE CONSTRAINT wl_entry_unique FOR (w:WatchlistEntry) REQUIRE w.entry_id IS UNIQUE;

CREATE (w:WatchlistEntry {
  entry_id: 'wl-uuid-1',
  source_code: 'OFAC_SDN',
  source_entry_id: '12345',
  entry_type: 'INDIVIDUAL',
  primary_name: 'JOHN DOE',
  program: 'SDGT',
  listed_date: date('2020-03-15'),
  is_active: true
})

// ============================================================
// JURISDICTION / COUNTRY NODES (reference data)
// ============================================================

CREATE CONSTRAINT country_code_unique FOR (c:Country) REQUIRE c.code IS UNIQUE;

CREATE (c:Country {
  code: 'CYM',
  name: 'Cayman Islands',
  risk_level: 'HIGH',
  fatf_status: 'GREY_LIST',
  is_tax_haven: true,
  is_sanctions_target: false
})

// ============================================================
// INSTITUTION NODES (banks, payment processors)
// ============================================================

CREATE CONSTRAINT institution_bic_unique FOR (i:Institution) REQUIRE i.bic IS UNIQUE;

CREATE (i:Institution {
  bic: 'CHASUS33',
  name: 'JPMorgan Chase',
  country: 'USA',
  institution_type: 'BANK',
  is_correspondent: true,
  risk_rating: 'LOW'
})
```

### Relationship Types

```cypher
// ============================================================
// PARTY-TO-ACCOUNT RELATIONSHIPS
// ============================================================

// Party holds account
MATCH (p:Party {party_id: 'uuid-1'}), (a:Account {account_id: 'acct-uuid-1'})
CREATE (p)-[:HOLDS_ACCOUNT {
  role: 'HOLDER',
  is_primary: true,
  since: date('2024-06-15')
}]->(a)

// Party is beneficial owner of account
MATCH (p:Party {party_id: 'uuid-3'}), (a:Account {account_id: 'acct-uuid-1'})
CREATE (p)-[:BENEFICIAL_OWNER_OF {
  ownership_percentage: 25.5,
  since: date('2025-01-10'),
  verified: true
}]->(a)

// ============================================================
// PARTY-TO-PARTY RELATIONSHIPS
// ============================================================

// Beneficial ownership
MATCH (owner:Party {party_id: 'uuid-1'}), (company:Party {party_id: 'uuid-2'})
CREATE (owner)-[:BENEFICIAL_OWNER_OF {
  ownership_percentage: 51.0,
  direct: true,
  since: date('2020-01-01'),
  verified: true,
  verification_date: date('2026-05-01')
}]->(company)

// Directorship
MATCH (director:Party {party_id: 'uuid-1'}), (company:Party {party_id: 'uuid-2'})
CREATE (director)-[:DIRECTOR_OF {
  position: 'Managing Director',
  since: date('2020-01-01'),
  is_active: true
}]->(company)

// Family relationships
MATCH (p1:Party {party_id: 'uuid-1'}), (p2:Party {party_id: 'uuid-4'})
CREATE (p1)-[:FAMILY_OF {
  relationship: 'SPOUSE',
  since: date('2010-06-15')
}]->(p2)

// Agent/nominee relationships
MATCH (nominee:Party {party_id: 'uuid-5'}), (principal:Party {party_id: 'uuid-2'})
CREATE (nominee)-[:NOMINEE_FOR {
  arrangement_type: 'NOMINEE_DIRECTOR',
  since: date('2022-03-01'),
  jurisdiction: 'CYM'
}]->(principal)

// ============================================================
// TRANSACTION FLOW RELATIONSHIPS
// ============================================================

// Originator sends transaction
MATCH (p:Party {party_id: 'uuid-1'}), (t:Transaction {transaction_id: 'txn-uuid-1'})
CREATE (p)-[:SENT {
  from_account: 'acct-uuid-1',
  channel: 'ONLINE'
}]->(t)

// Transaction received by beneficiary
MATCH (t:Transaction {transaction_id: 'txn-uuid-1'}), (p:Party {party_id: 'uuid-2'})
CREATE (t)-[:RECEIVED_BY {
  to_account: 'acct-uuid-2'
}]->(p)

// Transaction routed through intermediary
MATCH (t:Transaction {transaction_id: 'txn-uuid-1'}), (i:Institution {bic: 'DEUTDEFF'})
CREATE (t)-[:ROUTED_THROUGH {
  role: 'INTERMEDIARY',
  sequence: 1
}]->(i)

// Account-to-account fund flow (derived relationship for network analysis)
MATCH (a1:Account {account_id: 'acct-uuid-1'}), (a2:Account {account_id: 'acct-uuid-2'})
CREATE (a1)-[:FUNDS_FLOW {
  total_amount_usd: 87500.00,
  transaction_count: 12,
  first_transaction: datetime('2026-01-15T10:00:00Z'),
  last_transaction: datetime('2026-05-25T14:30:00Z'),
  avg_amount: 7291.67,
  distinct_currencies: ['USD', 'EUR']
}]->(a2)

// ============================================================
// PARTY-TO-COUNTRY RELATIONSHIPS
// ============================================================

MATCH (p:Party {party_id: 'uuid-1'}), (c:Country {code: 'USA'})
CREATE (p)-[:RESIDENT_OF {since: date('2010-01-01')}]->(c)

MATCH (p:Party {party_id: 'uuid-2'}), (c:Country {code: 'CYM'})
CREATE (p)-[:INCORPORATED_IN {since: date('2020-01-01')}]->(c)

// ============================================================
// SCREENING MATCH RELATIONSHIPS
// ============================================================

MATCH (p:Party {party_id: 'uuid-1'}), (w:WatchlistEntry {entry_id: 'wl-uuid-1'})
CREATE (p)-[:MATCHES_WATCHLIST {
  match_score: 0.97,
  match_algorithm: 'JARO_WINKLER',
  screened_at: datetime('2026-05-25T14:00:00Z'),
  status: 'PENDING',
  screened_name: 'John Doe',
  matched_name: 'JOHN DOE'
}]->(w)

// ============================================================
// ENTITY RESOLUTION RELATIONSHIPS
// ============================================================

// Potential same entity (identified by entity resolution algorithm)
MATCH (p1:Party {party_id: 'uuid-1'}), (p2:Party {party_id: 'uuid-6'})
CREATE (p1)-[:POTENTIAL_SAME_ENTITY {
  confidence: 0.89,
  matching_fields: ['name', 'dob', 'country'],
  algorithm: 'PROBABILISTIC_ER',
  identified_at: datetime('2026-05-20T10:00:00Z'),
  reviewed: false
}]->(p2)
```

### Graph Queries for AML Detection

```cypher
// ============================================================
// QUERY 1: STRUCTURING DETECTION
// Find parties making multiple transactions just below CTR threshold
// ============================================================

MATCH (p:Party)-[:SENT]->(t:Transaction)
WHERE t.transaction_date > datetime() - duration('P1D')
  AND t.amount_usd >= 8000
  AND t.amount_usd < 10000
  AND t.transaction_type IN ['CASH_DEPOSIT', 'CASH_WITHDRAWAL']
WITH p, collect(t) AS transactions, sum(t.amount_usd) AS total_amount
WHERE size(transactions) >= 2
  AND total_amount >= 10000
RETURN p.party_id, p.legal_name, p.risk_rating,
       size(transactions) AS txn_count, total_amount
ORDER BY total_amount DESC

// ============================================================
// QUERY 2: LAYERING DETECTION
// Find chains of transactions through multiple entities (3+ hops)
// ============================================================

MATCH path = (origin:Party)-[:SENT]->(t1:Transaction)-[:RECEIVED_BY]->
             (mid1:Party)-[:SENT]->(t2:Transaction)-[:RECEIVED_BY]->
             (mid2:Party)-[:SENT]->(t3:Transaction)-[:RECEIVED_BY]->
             (destination:Party)
WHERE t1.transaction_date > datetime() - duration('P7D')
  AND t2.transaction_date > t1.transaction_date
  AND t3.transaction_date > t2.transaction_date
  AND duration.between(t1.transaction_date, t3.transaction_date) < duration('P3D')
  AND t1.amount_usd > 10000
  AND abs(t1.amount_usd - t3.amount_usd) / t1.amount_usd < 0.15  // amounts within 15%
  AND origin <> destination
  AND mid1 <> mid2
RETURN origin.legal_name AS source,
       mid1.legal_name AS intermediary_1,
       mid2.legal_name AS intermediary_2,
       destination.legal_name AS final_recipient,
       t1.amount_usd AS initial_amount,
       t3.amount_usd AS final_amount,
       length(path) AS chain_length

// ============================================================
// QUERY 3: MULE ACCOUNT NETWORK DETECTION
// Find accounts receiving from many sources and rapidly forwarding
// ============================================================

MATCH (source:Party)-[:SENT]->(inbound:Transaction)-[:RECEIVED_BY]->(mule:Party)
WHERE inbound.transaction_date > datetime() - duration('P30D')
WITH mule, count(DISTINCT source) AS distinct_senders,
     sum(inbound.amount_usd) AS total_received
WHERE distinct_senders >= 5

MATCH (mule)-[:SENT]->(outbound:Transaction)-[:RECEIVED_BY]->(dest:Party)
WHERE outbound.transaction_date > datetime() - duration('P30D')
WITH mule, distinct_senders, total_received,
     count(DISTINCT dest) AS distinct_recipients,
     sum(outbound.amount_usd) AS total_sent
WHERE total_sent / total_received > 0.80  // forwarding 80%+ of received funds
  AND distinct_recipients <= 3            // concentrated destination
RETURN mule.party_id, mule.legal_name, mule.risk_rating,
       distinct_senders, total_received,
       distinct_recipients, total_sent,
       total_sent / total_received AS forwarding_ratio
ORDER BY total_received DESC

// ============================================================
// QUERY 4: ROUND-TRIPPING DETECTION
// Find circular fund flows returning to originator
// ============================================================

MATCH path = (origin:Party)-[:SENT]->(:Transaction)-[:RECEIVED_BY]->
             (:Party)-[:SENT]->(:Transaction)-[:RECEIVED_BY]->
             (:Party)-[:SENT]->(:Transaction)-[:RECEIVED_BY]->(origin)
WHERE ALL(t IN [n IN nodes(path) WHERE n:Transaction] WHERE
  t.transaction_date > datetime() - duration('P90D'))
WITH origin, path,
     [t IN nodes(path) WHERE t:Transaction | t.amount_usd] AS amounts,
     [p IN nodes(path) WHERE p:Party | p.legal_name] AS parties
WHERE reduce(s = 0, a IN amounts | s + a) > 50000
RETURN origin.legal_name,
       parties AS entities_in_loop,
       amounts AS amounts_in_loop,
       reduce(s = 0, a IN amounts | s + a) AS total_loop_amount,
       length(path) AS loop_length

// ============================================================
// QUERY 5: SHELL COMPANY NETWORK ANALYSIS
// Find clusters of entities sharing directors, addresses, or beneficial owners
// ============================================================

MATCH (company:Party:Organisation)-[:INCORPORATED_IN]->(c:Country)
WHERE c.risk_level IN ['HIGH', 'VERY_HIGH']
MATCH (company)<-[:DIRECTOR_OF|BENEFICIAL_OWNER_OF|NOMINEE_FOR]-(person:Party:Individual)
WITH person, collect(company) AS companies
WHERE size(companies) >= 3  // person linked to 3+ high-risk companies
UNWIND companies AS company
MATCH (company)-[:HOLDS_ACCOUNT]->(a:Account)
OPTIONAL MATCH (a)-[:FUNDS_FLOW]->(a2:Account)<-[:HOLDS_ACCOUNT]-(other_company)
WHERE other_company IN companies AND other_company <> company
RETURN person.legal_name AS controlling_person,
       [c IN companies | c.legal_name] AS shell_companies,
       size(companies) AS company_count,
       count(DISTINCT a2) AS inter_company_fund_flows

// ============================================================
// QUERY 6: COMMUNITY DETECTION (using Graph Data Science library)
// Identify clusters of tightly connected entities
// ============================================================

// Project a graph of fund flows for community detection
CALL gds.graph.project(
  'fundFlowGraph',
  'Party',
  {
    FUNDS_FLOW_TO: {
      type: 'SENT',
      orientation: 'NATURAL',
      properties: ['amount_usd']
    }
  }
)

// Run Louvain community detection
CALL gds.louvain.stream('fundFlowGraph')
YIELD nodeId, communityId
WITH gds.util.asNode(nodeId) AS party, communityId
WITH communityId, collect(party) AS members, count(*) AS size
WHERE size >= 5
RETURN communityId, size,
       [m IN members | m.legal_name] AS member_names,
       [m IN members | m.risk_rating] AS risk_ratings

// ============================================================
// QUERY 7: ENTITY 360-DEGREE VIEW
// Complete profile for investigation context
// ============================================================

MATCH (p:Party {party_id: $partyId})
OPTIONAL MATCH (p)-[r:HOLDS_ACCOUNT|BENEFICIAL_OWNER_OF]->(a:Account)
OPTIONAL MATCH (p)-[:SENT]->(sent:Transaction)
WHERE sent.transaction_date > datetime() - duration('P90D')
OPTIONAL MATCH (received:Transaction)-[:RECEIVED_BY]->(p)
WHERE received.transaction_date > datetime() - duration('P90D')
OPTIONAL MATCH (p)-[rel:DIRECTOR_OF|BENEFICIAL_OWNER_OF|FAMILY_OF|NOMINEE_FOR]-(related:Party)
OPTIONAL MATCH (p)-[wl:MATCHES_WATCHLIST]->(w:WatchlistEntry)
OPTIONAL MATCH (p)-[:RESIDENT_OF|INCORPORATED_IN]->(country:Country)
RETURN p,
       collect(DISTINCT {account: a, role: type(r)}) AS accounts,
       count(DISTINCT sent) AS sent_txn_count_90d,
       sum(DISTINCT sent.amount_usd) AS sent_amount_90d,
       count(DISTINCT received) AS received_txn_count_90d,
       collect(DISTINCT {party: related, relationship: type(rel)}) AS relationships,
       collect(DISTINCT {watchlist: w, score: wl.match_score, status: wl.status}) AS screening_matches,
       collect(DISTINCT country) AS jurisdictions
```

---

## PostgreSQL Operational Schema

The operational database handles state-machine workflows (alerts, cases, SARs) that require strict consistency, transactional integrity, and relational querying.

```sql
-- ============================================================
-- ALERT MANAGEMENT (operational state machine)
-- ============================================================

CREATE TABLE alert (
    alert_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    alert_number        BIGINT GENERATED ALWAYS AS IDENTITY,
    alert_date          TIMESTAMPTZ NOT NULL DEFAULT now(),
    alert_source        VARCHAR(20) NOT NULL CHECK (alert_source IN ('RULE', 'MODEL', 'GRAPH', 'SCREENING', 'MANUAL')),
    graph_query_ref     VARCHAR(100),                  -- reference to the Cypher query that generated this
    party_id            UUID NOT NULL,
    risk_score          DECIMAL(5,2) NOT NULL,
    severity            VARCHAR(10) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'NEW',
    typology            VARCHAR(50),
    summary             TEXT NOT NULL,
    explanation         TEXT,
    graph_context       JSONB,                         -- serialized subgraph relevant to this alert
    assigned_to         UUID,
    case_id             UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_queue ON alert (severity DESC, risk_score DESC)
    WHERE status IN ('NEW', 'ASSIGNED', 'UNDER_REVIEW');
CREATE INDEX idx_alert_tenant ON alert (tenant_id, status);
CREATE INDEX idx_alert_party ON alert (party_id);

-- ============================================================
-- CASE MANAGEMENT
-- ============================================================

CREATE TABLE investigation_case (
    case_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    case_number         BIGINT GENERATED ALWAYS AS IDENTITY,
    case_type           VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'OPEN',
    priority            VARCHAR(10) NOT NULL DEFAULT 'MEDIUM',
    subject_party_id    UUID NOT NULL,
    summary             TEXT NOT NULL,
    findings            TEXT,
    recommendation      TEXT,
    graph_analysis      JSONB,                         -- serialized network analysis results
    assigned_to         UUID,
    opened_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at           TIMESTAMPTZ,
    sar_id              UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL
);

CREATE INDEX idx_case_open ON investigation_case (priority DESC, opened_at)
    WHERE status NOT LIKE 'CLOSED%';
CREATE INDEX idx_case_tenant ON investigation_case (tenant_id, status);
CREATE INDEX idx_case_party ON investigation_case (subject_party_id);

-- ============================================================
-- CASE ACTIVITY LOG (immutable)
-- ============================================================

CREATE TABLE case_activity (
    activity_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_id             UUID NOT NULL REFERENCES investigation_case(case_id),
    activity_type       VARCHAR(30) NOT NULL,
    description         TEXT NOT NULL,
    activity_data       JSONB NOT NULL DEFAULT '{}',
    performed_by        UUID NOT NULL,
    performed_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_case ON case_activity (case_id, performed_at);

-- ============================================================
-- REGULATORY REPORTS
-- ============================================================

CREATE TABLE regulatory_report (
    report_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    report_number       BIGINT GENERATED ALWAYS AS IDENTITY,
    report_type         VARCHAR(20) NOT NULL,
    jurisdiction        VARCHAR(3) NOT NULL,
    regulator_code      VARCHAR(20) NOT NULL,
    filing_type         VARCHAR(20) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    case_id             UUID REFERENCES investigation_case(case_id),
    subject_party_id    UUID NOT NULL,
    report_content      JSONB NOT NULL DEFAULT '{}',
    submission_details  JSONB,
    narrative           TEXT,
    ai_generated_narrative TEXT,
    retention_until     DATE NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_report_status ON regulatory_report (status, regulator_code);
CREATE INDEX idx_report_tenant ON regulatory_report (tenant_id, status);

-- ============================================================
-- ANALYST MANAGEMENT
-- ============================================================

CREATE TABLE analyst (
    analyst_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    full_name           TEXT NOT NULL,
    email               TEXT NOT NULL,
    role                VARCHAR(30) NOT NULL,
    team                VARCHAR(50),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    log_id              UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    event_time          TIMESTAMPTZ NOT NULL DEFAULT now(),
    event_type          VARCHAR(50) NOT NULL,
    entity_type         VARCHAR(30) NOT NULL,
    entity_id           UUID NOT NULL,
    action              VARCHAR(20) NOT NULL,
    change_details      JSONB NOT NULL DEFAULT '{}',
    performed_by        UUID NOT NULL,
    PRIMARY KEY (log_id, event_time)
) PARTITION BY RANGE (event_time);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id, event_time);
CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, event_time);
```

---

## TimescaleDB Time-Series Schema

TimescaleDB (a PostgreSQL extension) handles the high-volume time-series data for transaction metrics and behavioural profiling.

```sql
-- Enable TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ============================================================
-- TRANSACTION METRICS (hypertable)
-- ============================================================

CREATE TABLE transaction_metrics (
    time                TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    party_id            UUID NOT NULL,
    account_id          UUID,
    metric_name         VARCHAR(50) NOT NULL,
    metric_value        DECIMAL(18,4) NOT NULL,
    dimensions          JSONB NOT NULL DEFAULT '{}'
    -- {"transaction_type": "WIRE_TRANSFER", "direction": "OUTBOUND",
    --  "counterparty_country": "CYM", "payment_rail": "SWIFT"}
);

SELECT create_hypertable('transaction_metrics', 'time');

-- Continuous aggregates for behavioural profiling
CREATE MATERIALIZED VIEW entity_hourly_profile
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    tenant_id,
    party_id,
    metric_name,
    count(*) AS event_count,
    sum(metric_value) AS total_value,
    avg(metric_value) AS avg_value,
    max(metric_value) AS max_value,
    min(metric_value) AS min_value,
    stddev(metric_value) AS stddev_value
FROM transaction_metrics
GROUP BY bucket, tenant_id, party_id, metric_name;

CREATE MATERIALIZED VIEW entity_daily_profile
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    tenant_id,
    party_id,
    metric_name,
    count(*) AS event_count,
    sum(metric_value) AS total_value,
    avg(metric_value) AS avg_value,
    max(metric_value) AS max_value,
    stddev(metric_value) AS stddev_value
FROM transaction_metrics
GROUP BY bucket, tenant_id, party_id, metric_name;

-- ============================================================
-- ENTITY BEHAVIORAL BASELINE
-- ============================================================

CREATE TABLE entity_behavioral_baseline (
    party_id            UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    metric_name         VARCHAR(50) NOT NULL,
    baseline_period     VARCHAR(20) NOT NULL,           -- '30D', '90D', '365D'
    mean_value          DECIMAL(18,4) NOT NULL,
    stddev_value        DECIMAL(18,4) NOT NULL,
    median_value        DECIMAL(18,4),
    p95_value           DECIMAL(18,4),
    p99_value           DECIMAL(18,4),
    sample_count        INTEGER NOT NULL,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (party_id, metric_name, baseline_period)
);

-- Example metric_names:
-- 'outbound_wire_count', 'outbound_wire_amount_usd', 'inbound_wire_count',
-- 'distinct_counterparties', 'distinct_destination_countries',
-- 'cash_deposit_count', 'cash_deposit_amount_usd',
-- 'max_single_transaction_usd', 'avg_transaction_amount_usd'

-- ============================================================
-- SCREENING METRICS (tracking screening throughput)
-- ============================================================

CREATE TABLE screening_metrics (
    time                TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    screening_type      VARCHAR(20) NOT NULL,
    list_code           VARCHAR(30) NOT NULL,
    names_screened      INTEGER NOT NULL,
    matches_found       INTEGER NOT NULL,
    true_matches        INTEGER NOT NULL DEFAULT 0,
    false_positives     INTEGER NOT NULL DEFAULT 0,
    avg_match_score     DECIMAL(5,4),
    p99_latency_ms      DECIMAL(10,2)
);

SELECT create_hypertable('screening_metrics', 'time');

-- ============================================================
-- ALERT AND CASE METRICS
-- ============================================================

CREATE TABLE compliance_metrics (
    time                TIMESTAMPTZ NOT NULL,
    tenant_id           UUID NOT NULL,
    metric_name         VARCHAR(50) NOT NULL,
    dimension           VARCHAR(50),
    metric_value        DECIMAL(18,4) NOT NULL
);

SELECT create_hypertable('compliance_metrics', 'time');

-- Example metric queries:
-- "What is the average alert disposition time for STRUCTURING alerts this month?"
-- "What is the false positive rate trend over the last 12 months?"
-- "How many SARs were filed per quarter by jurisdiction?"
```

---

## Data Synchronization Between Stores

```
Transaction Ingestion Pipeline:
1. Transaction event arrives via API/Kafka
2. Write to TimescaleDB transaction_metrics (time-series)
3. Write to Neo4j as Transaction node + SENT/RECEIVED_BY relationships
4. Detection engine queries Neo4j for pattern matches
5. If pattern matched, write Alert to PostgreSQL operational DB
6. Alert references the Neo4j graph context (serialized subgraph in JSONB)

Screening Pipeline:
1. Party onboarded or watchlist updated
2. Screening engine queries watchlist (can be PostgreSQL or Neo4j)
3. Match results written to both Neo4j (MATCHES_WATCHLIST relationship)
   and PostgreSQL (screening_result table)

Investigation Pipeline:
1. Analyst opens investigation in PostgreSQL case management
2. UI queries Neo4j for entity 360-degree view and network analysis
3. Graph visualizations rendered from Neo4j subgraph queries
4. Case activity logged to PostgreSQL (immutable audit trail)
5. SAR narrative generated from case data in PostgreSQL
```

---

## Pros and Cons

### Pros

1. **Network analysis is a first-class capability**: The graph database handles the most analytically valuable AML operations -- layering detection, mule network identification, shell company analysis, round-tripping detection, and entity resolution -- with queries that are natural, readable, and performant. A 5-hop layering chain query in Cypher is 3 lines; the equivalent recursive CTE in SQL is 30+ lines and orders of magnitude slower.

2. **Graph algorithms for advanced detection**: Neo4j's Graph Data Science library provides production-ready implementations of community detection (Louvain), centrality analysis (PageRank, betweenness), similarity algorithms, and pathfinding algorithms. These enable typology discovery (finding new money laundering patterns) that is practically impossible in relational databases.

3. **Entity resolution at scale**: Graph-native entity resolution identifies that "John Doe" at Bank A and "J. M. Doe" at Bank B are likely the same person by analyzing shared addresses, phone numbers, beneficial ownership chains, and transaction counterparties. The graph structure makes this a natural traversal operation rather than a cartesian join.

4. **Time-series optimized behavioral profiling**: TimescaleDB's continuous aggregates automatically maintain hourly and daily behavioral profiles without manual materialized view refreshes. Z-score calculations against baselines (for velocity and amount anomaly detection) are efficient against pre-aggregated time-series data.

5. **Investigation UX**: Graph visualizations of transaction networks and entity relationships are the most effective investigation tool for AML analysts. When the underlying data is already in a graph database, rendering interactive network diagrams is trivial compared to constructing them from relational JOINs.

6. **Polyglot best-of-breed**: Each database handles what it does best. Neo4j for relationships, PostgreSQL for transactional workflows, TimescaleDB for time-series aggregation. No single database is forced to do work it was not designed for.

### Cons

1. **Operational complexity**: Three database technologies (Neo4j, PostgreSQL, TimescaleDB) require three sets of operational expertise: backup procedures, monitoring, scaling, security hardening, and disaster recovery. This is the most operationally complex option.

2. **Data synchronization challenges**: Keeping data consistent across three stores requires a reliable event bus (Kafka) and careful handling of failure scenarios. If Neo4j is down when a transaction arrives, the system must queue and retry. Eventual consistency between stores is unavoidable.

3. **Neo4j licensing considerations**: Neo4j Community Edition is available under GPL, but the Graph Data Science library (which provides the most valuable algorithms for AML) requires Neo4j Enterprise or AuraDB, which are commercially licensed. ArangoDB (Apache 2.0) is an alternative but has a smaller ecosystem and fewer production-proven AML deployments.

4. **Neo4j transactional limitations**: Neo4j does not support distributed transactions across itself and PostgreSQL. There is no way to atomically write an alert to PostgreSQL and a relationship to Neo4j in a single transaction. The system must accept eventual consistency or implement saga patterns.

5. **Graph database scaling**: Neo4j scales reads well (with read replicas) but write scaling requires sharding, which Neo4j's built-in capabilities (Fabric) handle differently than relational sharding. For very high write throughput (100M+ transactions/month), ingestion pipeline design requires careful attention.

6. **Hiring difficulty**: Finding developers proficient in Cypher, graph data modeling, AND relational database design is significantly harder than finding PostgreSQL-only developers. The team needs graph-specific expertise.

7. **Cost**: Neo4j Enterprise + PostgreSQL + TimescaleDB (or three managed cloud instances) costs significantly more than a single PostgreSQL deployment. For a platform targeting mid-market institutions with accessible pricing, this infrastructure cost must be factored into the product economics.

---

## Technology Recommendations

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Graph database | Neo4j 5.x Enterprise or ArangoDB 3.12+ | Network analysis, entity resolution, pattern detection |
| Operational database | PostgreSQL 16+ | Alert/case/SAR workflows, audit trail |
| Time-series database | TimescaleDB 2.x (PostgreSQL extension) | Behavioral profiling, metric aggregation |
| Event bus | Apache Kafka | Reliable event streaming between stores |
| Graph visualization | Neo4j Bloom or custom D3.js/Cytoscape.js | Investigation network diagrams |
| Graph algorithms | Neo4j GDS library | Community detection, centrality, similarity |
| API gateway | Kong or similar | Unified API routing to appropriate backend |
| Monitoring | Prometheus + Grafana | Cross-store monitoring with Neo4j, PG, and TS exporters |

### Alternative: ArangoDB as Single Graph+Document Store

If licensing cost is a concern, ArangoDB (Apache 2.0) offers:
- Multi-model: graph + document + key-value in one database
- AQL query language for graph traversals
- Built-in graph algorithms (community detection, shortest path, centrality)
- Could replace both Neo4j and the JSONB aspects of PostgreSQL
- Trade-off: smaller community, fewer AML-specific case studies, less mature GDS equivalent

---

## Migration and Scaling Considerations

### Deployment Phases

**Phase 1: MVP (< 5M txn/month)**
- Neo4j Community Edition (single instance) or ArangoDB
- PostgreSQL (single instance with TimescaleDB extension)
- Kafka (single broker or managed Kafka)
- Total infrastructure: 3-4 servers/containers

**Phase 2: Growth (5-50M txn/month)**
- Neo4j Enterprise with read replicas (or ArangoDB cluster)
- PostgreSQL with read replicas
- Kafka cluster (3 brokers)
- Separate TimescaleDB instance for heavy aggregation workloads
- Total infrastructure: 8-12 servers/containers

**Phase 3: Scale (50M+ txn/month)**
- Neo4j Fabric for graph sharding by tenant
- PostgreSQL with Citus for operational sharding
- TimescaleDB multi-node for distributed hypertables
- Kafka cluster with topic partitioning by tenant
- Consider graph database warm/cold tier: recent 6 months in Neo4j, older data in archive with on-demand replay
- Total infrastructure: 20+ servers/containers

### Data Volume Estimates (10M txn/month)

| Store | Data growth | 1-year total |
|-------|------------|-------------|
| Neo4j (nodes + relationships) | ~30 GB/month | ~360 GB |
| PostgreSQL (operational) | ~10 GB/month | ~120 GB |
| TimescaleDB (time-series) | ~15 GB/month | ~180 GB |
| Kafka (retention) | ~20 GB (7-day retention) | ~20 GB |
| **Total** | **~75 GB/month** | **~680 GB** |

### Graph Data Lifecycle

- **Hot tier (Neo4j)**: Last 6-12 months of transaction nodes and all active party/account nodes
- **Warm tier (Neo4j archive or Parquet)**: 1-5 year old transaction data, queryable on demand
- **Cold tier (S3/object storage)**: 5+ year old data for regulatory retention
- **Relationship preservation**: Party and account relationships are never aged out of the hot tier; only transaction nodes are archived
- **Re-hydration**: Cold data can be re-loaded into Neo4j for regulatory examination or historical investigation

### Multi-Region Considerations

- Neo4j supports multi-data-center replication (Enterprise)
- Route tenants to region-appropriate clusters for data residency
- Kafka MirrorMaker for cross-region event replication of shared reference data (watchlists)
- Graph algorithms run locally in each region; results aggregated centrally only when needed
