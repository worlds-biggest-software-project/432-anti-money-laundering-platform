# Anti-Money Laundering (AML) Platform — Phased Development Plan

> Project: 432-anti-money-laundering-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents. The operational data model follows **data-model-suggestion-1** (normalised PostgreSQL), and the network-analytics layer follows **data-model-suggestion-4** (Neo4j graph as a synchronised projection). Event-sourcing concepts (suggestion 2) are borrowed selectively for the immutable audit trail.

---

## Product Summary

**What it does.** An AI-native, open-source AML platform that ingests financial transaction events over a REST API, monitors them in real time and in batch against a configurable rule engine plus ML behavioural models, screens counterparties against sanctions/PEP lists, raises risk-scored alerts, drives case investigations through an immutable workflow, and generates SAR/CTR regulatory filings (including AI-drafted narratives) in jurisdiction-specific formats.

**Who uses it.** Compliance officers, MLROs, and L1/L2 financial-crime analysts at fintechs, mid-market banks, and payment processors (under ~500k monthly transactions); plus the engineering teams that integrate the platform with core banking and payment rails.

**Key differentiators.** (1) Open-source and self-hostable — no enterprise lock-in. (2) ML triage that cuts false positives, with explainable rationale for every alert. (3) LLM-generated SAR narratives. (4) First-class graph network analytics for layering/mule detection. (5) Built-in model governance (SR 11-7).

**Deployment model.** API-first, self-hosted / cloud / hybrid via Docker Compose (single-node) and Helm (Kubernetes). Data-residency controls per tenant.

**Standards the build must honour.** ISO 20022 (transaction ingestion), OpenAPI 3.1 + JSON Schema 2020-12 (API contract), OAuth 2.0 / OIDC (auth), FinCEN SAR/CTR e-filing spec + UNODC goAML 5.0.1 XSD (reporting outputs), openCypher (graph queries), FATF/FFIEC/SR 11-7 (workflow, audit, model governance), GDPR/DORA (residency, retention, incident reporting).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The platform is ML- and LLM-heavy (anomaly detection, SAR narrative generation, entity resolution). Python has the strongest ecosystem for these (scikit-learn, XGBoost, the OpenAI/Anthropic SDKs, `rapidfuzz`, `jellyfish`). Domain logic (rules, screening) is I/O-bound, not CPU-bound, so the GIL is not a blocker. |
| API framework | **FastAPI** | Native OpenAPI 3.1 generation (a required standard), Pydantic v2 request/response validation (gives JSON Schema 2020-12 contracts for free), async I/O for high-throughput ingestion, first-class dependency injection for auth. |
| Operational database | **PostgreSQL 16** | Per data-model-suggestion-1: ACID, range partitioning for billion-row transaction tables, `pg_trgm`/`fuzzystrmatch` for fuzzy name matching, `pgcrypto` for PII encryption, Row-Level Security for multi-tenancy, `pg_partman` for partition lifecycle. |
| Graph database | **Neo4j 5 (Community, GPLv3)** | Per data-model-suggestion-4: money laundering is a network problem; multi-hop layering/mule traversals are natural in Cypher and intractable in SQL recursive CTEs at scale. openCypher gives long-term portability (ISO GQL). Kept as a **projection** synchronised from Postgres, never the system of record. |
| Async task queue | **Celery + Redis** | Webhook delivery, batch screening, ML scoring fan-out, SAR submission, and watchlist refresh are async. Celery is the mature Python standard; Redis doubles as broker and result backend. |
| Streaming / event bus | **Redis Streams (MVP) → Kafka (scale)** | Transaction ingestion feeds the scoring pipeline and the Neo4j projection. Redis Streams keep the MVP single-dependency; the projection consumer is written against an abstract `EventBus` interface so Kafka is a drop-in at scale. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature async ORM; Alembic gives version-controlled, reversible migrations (regulatory schema-change auditability). |
| LLM access | **`litellm` over Anthropic/OpenAI** | Provider-agnostic SAR narrative generation; lets self-hosters point at a local model. All prompts/outputs logged for model governance. |
| Fuzzy matching | **`rapidfuzz` + `jellyfish`** | Jaro-Winkler, Levenshtein, Soundex/Metaphone — the algorithms named in the OFAC-API spec — in pure-Python/C with high throughput. |
| Frontend | **React 18 + TypeScript + Vite, TanStack Query, shadcn/ui** | Analyst workbench is a data-dense SPA (alert queue, case workbench, entity 360, graph viewer). Graph rendering via **Cytoscape.js**. |
| Auth | **OAuth 2.0 / OIDC via Authlib**, RBAC enforced in API | Standard required by `standards.md`; federates with institution IdPs. Roles map to the `analyst.role` enum. |
| Containerisation | **Docker + docker-compose (dev/single-node), Helm (k8s)** | Self-hosted is a core deployment mode. |
| Testing | **pytest + pytest-asyncio + testcontainers + Schemathesis** | Unit + integration against real Postgres/Neo4j/Redis containers; Schemathesis fuzzes the OpenAPI contract; Playwright for frontend e2e. |
| Code quality | **Ruff (lint+format), Mypy (strict), pre-commit** | Single fast toolchain. |
| Package manager | **uv** | Fast, lockfile-based, reproducible builds. |
| Observability | **OpenTelemetry → Prometheus + Grafana; structlog JSON logs** | Scoring latency SLOs, projection lag, queue depth. DORA incident evidence. |

### Project Structure

```
aml-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml                 # postgres, neo4j, redis, api, worker, web
├── helm/                              # k8s chart for production
├── alembic/
│   ├── env.py
│   └── versions/                      # one migration per schema change
├── openapi/
│   └── aml-platform.json              # exported OpenAPI 3.1 (CI-checked)
├── seed/
│   ├── rules/                         # built-in detection rules (YAML)
│   ├── watchlists/                    # OFAC/UN/EU loader fixtures
│   └── jurisdictions/                 # SAR/CTR field mappings (FinCEN, FCA, AUSTRAC)
├── src/
│   └── aml/
│       ├── main.py                    # FastAPI app factory
│       ├── config.py                  # pydantic-settings, env-driven
│       ├── deps.py                    # DI: db session, current_user, tenant
│       ├── db/
│       │   ├── base.py                # SQLAlchemy engine/session
│       │   ├── models/                # ORM models per domain
│       │   └── rls.py                 # row-level-security helpers
│       ├── domain/                    # pure domain logic (no I/O)
│       │   ├── parties/
│       │   ├── transactions/
│       │   ├── monitoring/            # rule engine + ML scoring
│       │   ├── screening/             # sanctions/PEP fuzzy matching
│       │   ├── alerts/
│       │   ├── cases/
│       │   ├── reporting/             # SAR/CTR generation + formats
│       │   └── network/               # graph projection + analytics
│       ├── api/
│       │   ├── routers/               # one router per domain
│       │   ├── schemas/               # Pydantic request/response models
│       │   └── auth.py                # OAuth2/OIDC + RBAC
│       ├── events/
│       │   ├── bus.py                 # EventBus interface (Redis Streams / Kafka)
│       │   └── events.py              # event dataclasses
│       ├── workers/
│       │   ├── celery_app.py
│       │   ├── scoring.py
│       │   ├── screening.py
│       │   ├── projection.py          # Postgres → Neo4j sync
│       │   ├── webhooks.py
│       │   └── reporting.py
│       ├── integrations/
│       │   ├── iso20022.py            # pain.001 / pacs.008 parsing
│       │   ├── llm.py                 # litellm wrapper + prompt logging
│       │   └── watchlist_sources/     # OFAC SDN, UN, EU loaders
│       └── audit/
│           └── log.py                 # append-only audit writer
├── web/                               # React + TS analyst workbench
│   ├── package.json
│   └── src/
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                      # sample ISO 20022, OFAC SDN, transactions
```

Group is by **concern** (domain → api → workers), so each phase adds modules without restructuring.

---

## Phase 1: Foundation & Platform Skeleton

### Purpose
Establish the runnable scaffold: project config, containerised dependencies, database connection with multi-tenancy, OpenAPI-emitting FastAPI app, auth, append-only audit logging, and CI. After this phase the platform boots, authenticates a request, writes an audit record, and exposes a health endpoint with a versioned OpenAPI document.

### Tasks

#### 1.1 — Project scaffold, config, and tooling

**What**: Create the repository skeleton, `pyproject.toml` (uv), Ruff/Mypy/pre-commit, Dockerfile, and `docker-compose.yml` for postgres + neo4j + redis + api + worker + web.

**Design**:
- `config.py` uses `pydantic-settings`:
```python
class Settings(BaseSettings):
    database_url: PostgresDsn
    neo4j_uri: str = "bolt://neo4j:7687"
    neo4j_user: str = "neo4j"
    neo4j_password: SecretStr
    redis_url: str = "redis://redis:6379/0"
    oidc_issuer: str
    oidc_audience: str
    jwt_jwks_url: str
    pii_encryption_key: SecretStr          # AES-256, pgcrypto
    llm_provider: str = "anthropic"
    llm_model: str = "claude-sonnet-4.6"
    default_tenant_ctr_threshold: Decimal = Decimal("10000.00")
    environment: Literal["dev", "staging", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="AML_", env_file=".env")
```
- `docker-compose.yml` services: `postgres:16` (with `pg_trgm`, `fuzzystrmatch`, `pgcrypto` enabled via init SQL), `neo4j:5-community`, `redis:7`, `api`, `worker`, `web`.
- Healthcheck endpoint `GET /healthz` returns `{"status":"ok","db":bool,"neo4j":bool,"redis":bool}`.

**Testing**:
- `Unit: Settings loads from env with AML_ prefix → typed values, SecretStr masks PII key in repr.`
- `Unit: missing required env (database_url) → ValidationError naming the field.`
- `Integration: docker-compose up → GET /healthz returns 200 with all dependencies true.`
- `CI: ruff check, ruff format --check, mypy --strict all pass.`

#### 1.2 — Database engine, session DI, and tenancy

**What**: Async SQLAlchemy engine, request-scoped session dependency, and tenant-context propagation that sets the Postgres RLS GUC.

**Design**:
- `db/base.py`: `create_async_engine`, `async_sessionmaker`.
- `deps.py` `get_session()` opens a transaction, sets `SET LOCAL app.tenant_id = :tenant` (read in RLS policies), yields, commits/rolls back.
- `tenant` ORM model from suggestion-1 Domain 8 (`tenant_id`, `tenant_code`, `jurisdiction`, `timezone`, `ctr_threshold`, `default_currency`, `data_retention_years`).
- RLS policy template applied to every tenant-scoped table:
```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON <t>
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

**Testing**:
- `Integration (real PG): two tenants insert parties; query under tenant A returns only A's rows.`
- `Integration: session rolls back on exception, commits on success.`
- `Unit: get_session sets and clears the tenant GUC.`

#### 1.3 — OAuth 2.0 / OIDC authentication and RBAC

**What**: Bearer-token validation against an OIDC issuer (JWKS), mapping the token to an `analyst` with a role, and a `require_role(...)` dependency.

**Design**:
- `api/auth.py`: validate JWT signature/aud/exp via cached JWKS; extract `sub`→`user_id`, resolve/create `analyst` row.
- Roles (from suggestion-1 `analyst.role`): `L1_ANALYST, L2_ANALYST, SENIOR_ANALYST, TEAM_LEAD, COMPLIANCE_OFFICER, MLRO, ADMIN`.
- `Depends(require_role(Role.COMPLIANCE_OFFICER))` returns 403 on insufficient role.

**Testing**:
- `Integration (mocked JWKS): valid token → 200, analyst resolved.`
- `Integration: expired/wrong-aud/bad-signature token → 401.`
- `Integration: L1 analyst calls SAR-approve endpoint → 403.`

#### 1.4 — Append-only audit log

**What**: Partitioned, INSERT-only `audit_log` and a writer used by every state-changing operation.

**Design**:
- Table from suggestion-1 Domain 8 (`event_type`, `entity_type`, `entity_id`, `action`, `old_values`/`new_values` JSONB, `performed_by`, `ip_address`, `session_id`), `PARTITION BY RANGE (event_time)`.
- `audit/log.py`: `async def record(session, *, event_type, entity_type, entity_id, action, old, new, actor, request)`.
- DB role used by the app is `GRANT INSERT, SELECT` only on `audit_log` (no UPDATE/DELETE) — enforced at the grant level.

**Testing**:
- `Integration: record() inserts a row with correct JSON diff.`
- `Integration (real PG): UPDATE/DELETE on audit_log raises insufficient-privilege.`
- `Unit: diff helper produces only changed keys in old/new.`

---

## Phase 2: Core Domain Data Model & CRUD

### Purpose
Implement the operational schema (suggestion-1 Domains 1–2) and the parties/accounts APIs. This is the reference data every later phase depends on: transactions reference parties and accounts; screening screens parties; cases have a subject party.

### Tasks

#### 2.1 — Party, account & relationship schema + migrations

**What**: Alembic migrations and ORM models for `party`, `party_alias`, `party_address`, `party_identification`, `party_relationship`, `account`, `account_party`.

**Design**:
- Adopt suggestion-1 DDL verbatim, including: bitemporal columns (`valid_from`/`valid_to`/`recorded_at`), `gin_trgm_ops` indexes on names/aliases, `pep_status`, `risk_rating`/`cdd_level` enums.
- PII columns (`tax_id`, `id_number`, `account_number`) stored via `pgcrypto` `pgp_sym_encrypt` using `pii_encryption_key`; ORM exposes transparent encrypt/decrypt hybrid properties.
- Add `tenant_id UUID NOT NULL` to all tables for RLS.

**Testing**:
- `Integration: alembic upgrade then downgrade leaves an empty schema (reversibility).`
- `Integration: insert party with tax_id → ciphertext on disk, plaintext via ORM property.`
- `Unit: party_type/risk_rating CHECK violations rejected.`

#### 2.2 — Party API (KYC/CDD)

**What**: REST CRUD for parties with aliases, addresses, IDs, and relationships.

**Design**:
- Endpoints:
  - `POST /v1/parties` → 201 `{party_id}`
  - `GET /v1/parties/{id}` → full profile
  - `PATCH /v1/parties/{id}` → updates `risk_rating`, `cdd_level`, etc. (audited)
  - `GET /v1/parties?name=&risk_rating=&pep=` → trigram-ranked search
  - `POST /v1/parties/{id}/relationships`
- Pydantic `PartyCreate`/`PartyOut` models → OpenAPI/JSON Schema.
- Every mutation calls `audit.record(...)`.

**Testing**:
- `Integration: POST party then GET → round-trips; audit row written.`
- `Integration: GET /v1/parties?name=Jon → fuzzy-matches "John" via pg_trgm, ranked by similarity.`
- `Schemathesis: contract fuzzing of party endpoints passes.`

#### 2.3 — Account API

**What**: CRUD for accounts and account-party links.

**Design**:
- `POST /v1/accounts`, `GET /v1/accounts/{id}`, `POST /v1/accounts/{id}/parties` (role: HOLDER/JOINT_HOLDER/...).
- Reject account linked to non-existent party (FK).

**Testing**:
- `Integration: link party to account with role HOLDER → 201; duplicate (account,party,role) → 409.`
- `Integration: account with bad currency code → 422.`

---

## Phase 3: Transaction Ingestion & the Monitoring Engine (Core Value — Part 1)

### Purpose
The heart of the product: ingest transactions over the API (and ISO 20022 files), persist them in the partitioned table, and run them through a configurable rule engine that emits findings. Real-time scoring with sub-second latency is the primary SLO. After this phase the platform detects structuring, velocity, and threshold typologies on live transactions.

### Tasks

#### 3.1 — Transaction schema, ingestion API, and ISO 20022 parsing

**What**: Partitioned `transaction` table, `POST /v1/transactions` (single + batch), and an ISO 20022 ingester.

**Design**:
- Adopt suggestion-1 transaction DDL: `PARTITION BY RANGE (transaction_date)`, originator/beneficiary/intermediary blocks, `end_to_end_id`/`instruction_id` (ISO 20022), `amount_usd` normalisation, `risk_score`/`risk_factors[]`.
- `pg_partman` configured for monthly partitions + retention = tenant `data_retention_years`.
- Endpoints:
  - `POST /v1/transactions` body = `TransactionIngest` (JSON Schema 2020-12) → 202, returns `transaction_id`; publishes `TransactionIngested` to the EventBus.
  - `POST /v1/transactions/batch` (ndjson, up to 10k) → `batch_id`.
  - `POST /v1/transactions/iso20022` (`pacs.008`/`pain.001` XML) → parsed via `integrations/iso20022.py` (lxml + SWIFT CBPR+ schema), mapped to `TransactionIngest`.
- `amount_usd` computed from an FX rate table (seeded; pluggable provider).

**Testing**:
- `Integration: POST valid transaction → 202, row in correct monthly partition, TransactionIngested event published.`
- `Fixture: parse sample pacs.008 → originator/beneficiary/amount/E2E id mapped correctly.`
- `Integration: batch of 10k ndjson → all persisted; partition pruning verified on date-range query.`
- `Unit: malformed ISO 20022 → 422 with element path in error.`

#### 3.2 — Rule engine core

**What**: A declarative, versioned rule engine that evaluates each transaction (and rolling entity windows) and emits findings.

**Design**:
- Rules defined in YAML (seeded in `seed/rules/`), persisted in `detection_rule` (suggestion-1 Domain 4: `typology`, `severity`, `rule_logic`, `parameters` JSONB, `effective_from/to`, `approved_by`).
- A `Rule` is a pure function `evaluate(ctx: EvalContext) -> Finding | None`:
```python
@dataclass(frozen=True)
class EvalContext:
    txn: Transaction
    entity_window: EntityWindow      # rolling aggregates for the party
    params: dict[str, Any]

@dataclass(frozen=True)
class Finding:
    rule_code: str
    typology: Typology
    severity: Severity
    score: float                     # 0..1
    explanation: str                 # human-readable, regulator-defensible
    triggering_txn_ids: list[UUID]
```
- Built-in rules (FATF/published, no IP risk):
  - `THRESHOLD`: `amount_usd >= params.threshold`.
  - `VELOCITY`: count/sum over `params.window_hours` exceeds `params.max_count`/`max_sum`.
  - `STRUCTURING`: ≥`params.min_txns` cash txns each below CTR threshold summing above it within `params.window_hours`.
  - `HIGH_RISK_JURISDICTION`: counterparty country in `params.jurisdictions` (FATF grey/black list).
  - `DORMANT_ACTIVATION`: txn after `params.dormant_days` of inactivity.
- `EntityWindow` is read from a Redis-backed rolling store updated on each ingest (sorted set keyed by party_id, scored by timestamp) to keep evaluation O(window) and sub-second.
- `rule_logic` supports a safe expression DSL (`simpleeval`) for custom rules beyond built-ins; no arbitrary code execution.

**Testing**:
- `Unit: STRUCTURING — three $9,500/$9,700/$9,600 cash deposits in 24h → Finding(STRUCTURING, HIGH) with all three txn ids; two deposits → None.`
- `Unit: VELOCITY — 14 wires/48h over max_count=6 → Finding; explanation cites count & window.`
- `Unit: custom DSL rule with a forbidden builtin → rejected at load time.`
- `Integration: rule effective_to in the past → not evaluated.`

#### 3.3 — Real-time scoring pipeline

**What**: An EventBus consumer that scores each ingested transaction against active rules and persists `risk_score`/`risk_factors`, emitting `TransactionRiskScored`/`TransactionFlagged`.

**Design**:
- Worker subscribes to `TransactionIngested`; loads active rules (cached, invalidated on rule change); builds `EntityWindow` from Redis; runs all rules; aggregates findings into a transaction `risk_score` (max-of normalised severities, configurable).
- Writes `risk_score`/`risk_factors[]` back to the transaction row; publishes `TransactionFlagged` per finding (consumed by Phase 5 alerting).
- SLO: p95 scoring latency < 250 ms for a single transaction; measured via OTel span `scoring.latency_ms`.

**Testing**:
- `Integration (real PG+Redis): ingest a structuring sequence → transaction flagged, risk_factors=["STRUCTURING"], TransactionFlagged emitted.`
- `Integration: rule disabled → no flag on a previously-flagging txn.`
- `Perf: 1,000 txns scored, p95 latency asserted < 250 ms.`

---

## Phase 4: Sanctions & PEP Screening (Core Value — Part 2)

### Purpose
Add real-time and batch screening of parties and transaction counterparties against OFAC, UN, and EU consolidated lists with fuzzy matching. Sanctions hits are the highest-severity findings and must be detectable at onboarding, on every transaction, and on watchlist updates.

### Tasks

#### 4.1 — Watchlist schema, loaders, and refresh

**What**: Schema for watchlist sources/entries/aliases/identifiers and loaders for OFAC SDN, UN Consolidated, and EU Financial Sanctions (all licence-free per `standards.md`).

**Design**:
- Adopt suggestion-1 Domain 6 DDL (`watchlist_source`, `watchlist_entry`, `watchlist_alias`, `watchlist_identifier`) with `gin_trgm_ops` on names/aliases.
- `integrations/watchlist_sources/`: per-source parser (OFAC SDN XML, UN XML, EU XML) → normalised entries; idempotent upsert keyed on `(source_id, source_entry_id)`.
- Celery beat task refreshes daily; emits `WatchlistUpdated` (added/removed/modified counts) and triggers re-screening of active parties whose names now match.

**Testing**:
- `Fixture: load sample OFAC SDN XML → entries + aliases + identifiers populated; re-load is idempotent.`
- `Integration: refresh that adds an entry matching an existing party → re-screen triggered, screening_result created.`

#### 4.2 — Fuzzy matching engine

**What**: A name-screening function returning scored candidate matches.

**Design**:
```python
@dataclass(frozen=True)
class ScreenMatch:
    entry_id: UUID
    matched_name: str
    source_code: str
    program: str | None
    score: float                     # 0..1
    algorithm: Literal["EXACT","JARO_WINKLER","LEVENSHTEIN","SOUNDEX"]

def screen_name(name: str, *, threshold: float, lists: list[str]) -> list[ScreenMatch]: ...
```
- Candidate generation: `pg_trgm` similarity prefilter (cheap), then `rapidfuzz` Jaro-Winkler + `jellyfish` Soundex re-scoring (the algorithms named for OFAC-API). Normalisation: case-fold, strip diacritics, transliterate, drop honorifics.
- Configurable per-tenant threshold (default 0.85).

**Testing**:
- `Unit: "Vladimir Putin" vs "Vladmir Putin" → JARO_WINKLER score ≥ 0.9.`
- `Unit: transliteration "Mohammed" vs "Muhammad" → matches above threshold.`
- `Unit: common-name false-positive below threshold → excluded.`

#### 4.3 — Screening API and result workflow

**What**: Onboarding, transaction, and batch screening endpoints plus the `screening_result` disposition workflow.

**Design**:
- Adopt suggestion-1 `screening_result` table.
- `POST /v1/screening` `{party_id|name, type}` → list of `ScreenMatch`, each persisted `PENDING`.
- Transaction scoring pipeline (3.3) screens counterparty names inline; a `TRUE_MATCH`-eligible hit raises a `SCREENING`-source alert (Phase 5).
- `PATCH /v1/screening/results/{id}` → `TRUE_MATCH | FALSE_POSITIVE` with `review_notes` (audited).

**Testing**:
- `Integration: onboard a party whose name is on OFAC → PENDING screening_result + CRITICAL alert.`
- `Integration: analyst dispositions FALSE_POSITIVE → status updated, audit row written.`
- `Integration: transaction to a sanctioned beneficiary → SCREENING alert with the matched program.`

---

## Phase 5: Alerts & Case Management

### Purpose
Turn findings and screening hits into a triageable, risk-prioritised alert queue, and provide the structured investigation workflow (open → assign → escalate → document → close) with an immutable activity trail. This is the analyst-facing core required for regulatory examination readiness (FFIEC).

### Tasks

#### 5.1 — Alert generation & queue

**What**: Consume `TransactionFlagged`/screening hits, deduplicate into alerts, and expose the prioritised queue with claim/assign and bulk disposition.

**Design**:
- Adopt suggestion-1 `alert` + `alert_transaction` tables (status state machine: `NEW → ASSIGNED → UNDER_REVIEW → ESCALATED → CLOSED_* / SAR_FILED`).
- Dedup: findings for the same party+typology within a configurable window roll into one alert (append `alert_transaction` rows) rather than spawning duplicates.
- Endpoints:
  - `GET /v1/alerts?status=&severity=&assigned_to=` ordered by `severity DESC, risk_score DESC`.
  - `POST /v1/alerts/{id}/claim`, `POST /v1/alerts/{id}/assign`.
  - `POST /v1/alerts/{id}/disposition` `{disposition, reason}`.
  - `POST /v1/alerts/bulk-disposition` `{alert_ids[], disposition, reason, bulk_rule}` for high-volume low-risk patterns.
- `explanation` carried from the Finding to satisfy explainable-AI/regulatory rationale.

**Testing**:
- `Integration: two flags same party/typology in window → single alert with 2 alert_transaction rows.`
- `Integration: queue ordered CRITICAL>HIGH then by risk_score.`
- `Integration: bulk-disposition 50 alerts FALSE_POSITIVE → all closed, one audit row per alert.`

#### 5.2 — Investigation case workflow

**What**: Cases with assignment, escalation, notes, evidence, and an immutable `case_activity` log.

**Design**:
- Adopt suggestion-1 `investigation_case`, `case_activity`, `case_evidence` tables.
- Endpoints: `POST /v1/cases` (link alerts), `POST /v1/cases/{id}/assign|escalate|notes|evidence`, `PATCH /v1/cases/{id}` (status), `GET /v1/cases/{id}` (full dashboard view).
- Every transition writes a `case_activity` row (`old_value`/`new_value`); evidence uploads store SHA-256 + object-store path for integrity.
- Closing a case as `CLOSED_SAR_FILED` requires a linked SAR (enforced in Phase 6).

**Testing**:
- `Integration: open case from 3 alerts → alerts move to status reflecting case linkage; case_activity has CREATED + ALERT_LINKED rows.`
- `Integration: escalate L1→L2 → escalated_to set, activity logged.`
- `Integration: upload evidence → sha256 stored; tamper (changed bytes) detected on verify.`

---

## Phase 6: Regulatory Reporting (SAR / CTR)

### Purpose
Generate, review, and submit Suspicious Activity Reports and Currency Transaction Reports in jurisdiction-specific formats, mapping a common internal template to FinCEN, goAML, and FCA outputs. This closes the compliance loop from detection to filing.

### Tasks

#### 6.1 — SAR/CTR schema and common template

**What**: `sar_report` and `ctr_report` tables and an internal common report model that maps to multiple regulators.

**Design**:
- Adopt suggestion-1 Domain 7 DDL (status state machine `DRAFT → NARRATIVE_GENERATED → UNDER_REVIEW → APPROVED → SUBMITTED → ACCEPTED/REJECTED`; `retention_until`).
- `CommonReport` Pydantic model holds jurisdiction-neutral fields; per-jurisdiction mappers (`seed/jurisdictions/`) project it to a target format.
- CTR auto-generation: a daily worker aggregates cash-in/out per party per day; if `> tenant.ctr_threshold`, create a `PENDING` CTR.

**Testing**:
- `Integration: cash deposits summing $11,000 in a day → CTR auto-created at default $10k threshold.`
- `Unit: SAR state machine rejects SUBMITTED without prior APPROVED.`

#### 6.2 — Multi-jurisdiction format generators

**What**: Serialisers producing FinCEN SAR/CTR e-filing batch records, UNODC goAML 5.0.1 XML, and an FCA mapping.

**Design**:
- `reporting/formats/fincen.py`: fixed-length ASCII batch per the FinCEN GENspecs (Phase reads the spec; field map in `seed/jurisdictions/fincen.yaml`).
- `reporting/formats/goaml.py`: XML validated against the bundled goAML 5.0.1 XSD (lxml schema validation).
- `GET /v1/reports/{id}/export?format=fincen|goaml|fca` returns the conformant artefact.

**Testing**:
- `Fixture: build SAR → goAML XML validates against goAML 5.0.1 XSD.`
- `Fixture: FinCEN batch record has correct fixed-length field offsets.`
- `Unit: missing mandatory FinCEN field → validation error before export.`

#### 6.3 — AI-assisted SAR narrative generation

**What**: LLM drafting of the SAR narrative from structured case evidence, with analyst review/edit before filing — the highest-value AI feature per research.

**Design**:
- `integrations/llm.py` builds a structured prompt from case data (subject, linked transactions, typologies, timeline, findings) and returns a draft narrative.
- Prompt template (system): *"You are an AML compliance analyst. Draft a factual, chronological SAR narrative covering the five elements (who, what, when, where, why suspicious). Use only the structured facts provided. Do not speculate or invent figures. Output plain prose."* User message = JSON case bundle.
- Result stored in `sar_report.ai_generated_narrative`; analyst edits land in `narrative`; `narrative_reviewed` gate required before `APPROVED`.
- **Governance**: every prompt + response + `model_id`/`model_version` logged (feeds Phase 8). Emits `SARNarrativeGenerated`.

**Testing**:
- `Integration (mocked LLM): draft from a case bundle → narrative stored, SARNarrativeGenerated emitted, prompt+response logged.`
- `Unit: narrative bundle excludes encrypted PII raw values (only references).`
- `Integration: attempt APPROVED while narrative_reviewed=false → 409.`

---

## Phase 7: Network Analytics (Graph)

### Purpose
Project parties, accounts, and transaction flows into Neo4j and expose graph analytics — layering-chain and mule-network detection, plus a visual entity-relationship explorer — that are infeasible in SQL. This delivers the README's first-class graph differentiator.

### Tasks

#### 7.1 — Postgres→Neo4j projection

**What**: An EventBus consumer that maintains a Neo4j graph mirror of operational data.

**Design**:
- Consumer subscribes to `PartyOnboarded`, `account` links, and `TransactionIngested`; upserts `:Party`, `:Account`, `:Transaction` nodes and `:SENT`/`:RECEIVED`/`:OWNS`/`:RELATED_TO` edges per suggestion-4 schema (idempotent `MERGE` on `*_id`).
- Tracks projection lag via a `projection_checkpoint` (last processed event time); exposed on `/healthz`.
- Postgres remains system of record; Neo4j is rebuildable by replaying events.

**Testing**:
- `Integration (real Neo4j): ingest 3 transactions A→B→C → graph has the path; re-running the consumer is idempotent (no duplicate edges).`
- `Integration: projection lag reported on /healthz.`

#### 7.2 — Graph analytics queries

**What**: Cypher-backed detection of layering chains, circular flows (round-tripping), and mule fan-in/fan-out.

**Design**:
- `domain/network/queries.py` (openCypher):
  - **Layering chain**: `MATCH path=(a:Party)-[:SENT*3..6]->(z:Party) WHERE total flow conserved within tolerance RETURN path`.
  - **Round-tripping**: cycles returning to origin within N hops.
  - **Mule fan-in/out**: nodes with in/out degree over threshold within a window.
- Results that exceed configured thresholds emit `TransactionFlagged` with typology `LAYERING`/`MULE_ACCOUNT`/`ROUND_TRIPPING`, feeding the alert pipeline (Phase 5) — closing the loop so graph findings become alerts.

**Testing**:
- `Fixture: seed a 4-hop layering structure → layering query returns the chain; a non-laundering star topology does not.`
- `Fixture: seed a 3-account cycle → round-tripping query detects it.`
- `Integration: mule fan-in over threshold → MULE_ACCOUNT alert raised.`

#### 7.3 — Network API & entity 360

**What**: Endpoints powering the graph viewer and the entity 360 profile.

**Design**:
- `GET /v1/network/{party_id}?depth=2` → nodes+edges JSON for Cytoscape.js.
- `GET /v1/parties/{id}/profile` → entity-360 aggregate (transaction stats 30/90d, alert/case/SAR history, screening status, relationships) per suggestion-2 `proj_entity_profile` shape, computed from Postgres + Neo4j.

**Testing**:
- `Integration: GET network depth=2 → bounded node/edge set, sanctioned nodes flagged.`
- `Integration: entity 360 aggregates match underlying transaction/alert counts.`

---

## Phase 8: ML Behavioural Detection & Model Governance

### Purpose
Augment the rule engine with ML-based entity-level anomaly scoring to cut false positives, and add the SR 11-7 model-governance scaffolding (registry, validation, drift monitoring) that regulators require for any ML used in AML decisions.

### Tasks

#### 8.1 — Behavioural feature store & anomaly model

**What**: Per-entity behavioural features and an unsupervised anomaly scorer that contributes to transaction risk.

**Design**:
- Features from `mv_entity_daily_stats` (suggestion-1) + rolling Redis windows: txn count/sum/avg per 30/90d, distinct counterparties/countries, velocity z-scores.
- Model: IsolationForest / XGBoost (per `ml_model.framework`), serialised to `artifact_path`; `score_entity(features) -> (anomaly_score, top_factors)`.
- Anomaly score blends into the transaction `risk_score`; `top_factors` populate the alert `explanation` (explainability).

**Testing**:
- `Unit: entity with a sudden 10x volume spike → high anomaly_score; stable entity → low.`
- `Integration: anomaly contributes a MODEL-source finding with explanatory factors.`

#### 8.2 — Model registry & governance (SR 11-7)

**What**: Register, validate, deploy, and monitor models with documentation and drift detection.

**Design**:
- Adopt suggestion-1 `ml_model` table (`validation_status`, `performance_metrics`, `next_validation_date`).
- Endpoints (role MLRO): `POST /v1/models` (register + metrics), `POST /v1/models/{id}/validate`, `POST /v1/models/{id}/deploy` (only one active per type).
- Drift monitor (scheduled): compares live score distribution vs training baseline (PSI); breach → alert to MLRO + `next_validation_date` advanced.
- LLM SAR usage from 6.3 also recorded against a `NLP_NARRATIVE` model record for auditability.

**Testing**:
- `Integration: deploy unvalidated model → 409.`
- `Integration: PSI over threshold → governance alert raised, validation rescheduled.`
- `Integration: only one ANOMALY_DETECTION model active after deploying a second.`

---

## Phase 9: Webhooks, Integrations & Public API Hardening

### Purpose
Make the platform embeddable: outbound webhooks for alert/case/SAR status changes, robust batch ingestion, rate limiting, idempotency, and a frozen, CI-verified OpenAPI 3.1 contract — matching the Unit21-style API-first integration surface.

### Tasks

#### 9.1 — Outbound webhooks

**What**: Tenant-registered webhook endpoints receiving signed event notifications with retry.

**Design**:
- `webhook_subscription` table (`url`, `secret`, `event_types[]`, `is_active`).
- Worker delivers `alert.created`, `alert.dispositioned`, `case.status_changed`, `sar.submitted`; HMAC-SHA256 signature header; exponential-backoff retries → dead-letter after N.

**Testing**:
- `Integration (mock receiver): alert created → POST with valid HMAC signature received.`
- `Integration: receiver returns 500 thrice → retried then dead-lettered.`

#### 9.2 — API hardening: idempotency, rate limiting, OpenAPI freeze

**What**: Idempotency keys on ingestion, per-tenant rate limits, and an OpenAPI contract check in CI.

**Design**:
- `Idempotency-Key` header on `POST /v1/transactions*`; dedup via Redis with TTL.
- Token-bucket rate limit per tenant (Redis).
- CI step exports OpenAPI and diffs against committed `openapi/aml-platform.json`; Schemathesis runs the full contract suite.

**Testing**:
- `Integration: same Idempotency-Key twice → one transaction, same response.`
- `Integration: exceed rate limit → 429 with Retry-After.`
- `CI: OpenAPI diff clean; Schemathesis suite passes.`

---

## Phase 10: Analyst Workbench (Frontend)

### Purpose
Deliver the React analyst UI: alert queue, case workbench (with AI SAR narrative review), entity 360, graph viewer, and a compliance KPI dashboard — the surfaces compliance teams use daily.

### Tasks

#### 10.1 — App shell, auth, and alert queue

**What**: Vite+React+TS shell with OIDC login, role-aware nav, and the prioritised alert queue with claim/assign/bulk-disposition.

**Design**:
- TanStack Query against the v1 API; shadcn/ui table; severity/risk sorting; bulk-select disposition.

**Testing**:
- `Playwright e2e: log in, claim an alert, disposition it → status updates; backend audit row exists.`
- `Playwright: bulk-select + false-positive → all rows closed.`

#### 10.2 — Case workbench, SAR review, entity 360 & graph viewer

**What**: Case screen combining evidence/notes/linked transactions, AI-narrative accept/edit/reject controls, entity 360 panel, and a Cytoscape.js graph explorer.

**Design**:
- SAR review surfaces `ai_generated_narrative` with edit-to-`narrative` and `narrative_reviewed` gate before submit.
- Graph viewer consumes `GET /v1/network/{party_id}`; sanctioned/high-risk nodes styled distinctly.
- KPI dashboard reads `proj_compliance_metrics` (false-positive rate, avg disposition time, SARs filed).

**Testing**:
- `Playwright: open case, generate narrative (mocked LLM), edit, mark reviewed, submit → SAR moves APPROVED→SUBMITTED.`
- `Playwright: graph viewer renders a 4-hop layering structure with the sanctioned node highlighted.`

---

## Phase 11: Multi-Tenancy, Data Residency, Retention & Compliance Ops

### Purpose
Production-hardening for regulated multi-jurisdiction operation: enforce tenant isolation end-to-end, configurable retention/purge, GDPR data-subject and access-audit support, and DORA-aligned incident reporting.

### Tasks

#### 11.1 — Residency, retention & purge

**What**: Per-tenant data-residency routing and retention/purge jobs.

**Design**:
- Per-region Postgres/Neo4j clusters; tenant→region routing in config; shared reference data (rules, watchlists) replicated.
- Scheduled purge: drop transaction/audit partitions and soft-delete parties past `data_retention_years` (jurisdiction-aware); purge actions audited.

**Testing**:
- `Integration: tenant routed to its region's DB; cross-region read denied.`
- `Integration: partition older than retention dropped by purge job; audit recorded.`

#### 11.2 — GDPR & DORA support

**What**: Data-subject access export, PII-access audit reporting, and DORA incident classification/reporting.

**Design**:
- `GET /v1/gdpr/subject/{party_id}/export` → all held personal data (decrypted, access audited).
- PII-access audit report endpoint for examiners.
- Incident model with DORA severity classification + report export.

**Testing**:
- `Integration: subject export returns all PII; the export itself writes an audit row.`
- `Integration: classify an incident → DORA report generated with required fields.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Skeleton        ─── required by everything
    │
Phase 2: Core Data Model & CRUD       ─── requires 1
    │
Phase 3: Ingestion & Monitoring       ─── requires 2   (CORE VALUE)
    ├── Phase 4: Sanctions Screening   ─── requires 2; parallel with 3
    │
Phase 5: Alerts & Case Management     ─── requires 3 + 4
    │
Phase 6: Regulatory Reporting (SAR/CTR)─── requires 5
    │
    ├── Phase 7: Network Analytics     ─── requires 3 (graph); feeds 5
    ├── Phase 8: ML & Model Governance ─── requires 3; feeds 5/6; parallel with 7
    ├── Phase 9: Webhooks & API Harden ─── requires 5; parallel with 7/8
    │
Phase 10: Analyst Workbench (Frontend)─── requires 5,6,7 (consumes their APIs)
    │
Phase 11: Multi-Tenancy/Residency/Ops ─── requires 1–6; finalised last
```

**Parallelism opportunities:**
- **Phases 3 and 4** can be built concurrently once Phase 2 lands (both depend only on parties/accounts).
- **Phases 7, 8, and 9** can be built concurrently once Phase 5 (and Phase 3 for 7/8) is complete.
- **Phase 10** (frontend) can begin against mocked APIs as soon as the Phase 3/5 OpenAPI contracts are frozen.

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; new code has tests for happy-path **and** edge cases.
3. `ruff check`, `ruff format --check`, and `mypy --strict` pass.
4. New/changed Alembic migrations apply **and** reverse cleanly.
5. `docker-compose up` builds and boots; `GET /healthz` is green.
6. The phase's primary capability works end-to-end (demonstrated by an integration or e2e test).
7. New configuration options are documented in the README and have safe defaults.
8. New API endpoints appear in the exported `openapi/aml-platform.json`, and the CI OpenAPI-diff + Schemathesis suite pass.
9. Every state-changing operation writes an `audit_log` row (verified by test).
10. RLS isolation holds for any new tenant-scoped table (verified by a two-tenant test).
11. For ML/LLM changes (Phases 6, 8): prompts/inputs/outputs and `model_id`/`model_version` are logged for governance.
```
