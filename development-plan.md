# Care Management Platform — Phased Development Plan

> Project: 422-care-management-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased implementation specification for an open-source, AI-native population-scale care management platform.

The product identifies at-risk patients (risk stratification), enrols them in evidence-based disease management programs, systematically closes care gaps mapped to HEDIS/MIPS quality measures, coordinates care teams, runs multi-channel patient outreach, screens for social determinants of health, and tracks CMS CCM billable time — all built around HL7 FHIR R4 interoperability and HIPAA-grade access control and audit.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The platform is ML/analytics-heavy (risk stratification, equity auditing, outreach optimisation, readmission prediction). Python has the strongest healthcare-data and ML ecosystem (`scikit-learn`, `pandas`, `numpy`, `fhir.resources`) and the OHDSI/clinical-informatics community is Python-first. |
| API framework | **FastAPI** | Async-native (needed for FHIR fan-out and outreach webhooks), generates an **OpenAPI 3.1** spec automatically (a `standards.md` requirement), and integrates Pydantic v2 for request/response validation. |
| Data validation | **Pydantic v2** | Single source of truth for API schemas and JSONB document validation (addresses the JSONB "schema-on-read" risk in data-model-suggestion-3). |
| Database | **PostgreSQL 16** | Chosen with the **Hybrid Relational + JSONB** model (data-model-suggestion-3): normalized tables for workflow entities (patient, care_gap, enrollment, billing) + JSONB for FHIR resources, SDOH responses, risk factors, and care-plan content. Single-database operational simplicity is decisive for the FQHC/SMB target segment identified in `features.md`. Suggestions 2 (event sourcing) and 4 (OMOP+graph) are explicitly deferred as over-engineering for the MVP; the schema is forward-compatible with both (see Phase 11). |
| ORM / DB access | **SQLAlchemy 2.0 (async)** with custom JSONB column types | Mature async support, fine-grained control over JSONB operators and GIN-index queries that Prisma handles less elegantly. |
| Migrations | **Alembic** | Standard SQLAlchemy migration tool; version-controlled DDL. |
| Task queue | **Celery + Redis** | Async workloads: outreach dispatch, FHIR ingestion ETL, risk-score batch recomputation, materialized-view refresh, quality-measure evaluation. Redis doubles as cache and broker. |
| FHIR handling | **`fhir.resources` (Pydantic FHIR R4 models)** | Validates and (de)serialises FHIR R4 Patient, Condition, Observation, CarePlan, Goal, CareTeam, QuestionnaireResponse, MeasureReport resources. |
| Outreach providers | **Twilio (SMS/voice)**, **SMTP/SendGrid (email)** behind a provider abstraction | Pluggable `OutreachProvider` interface so self-hosters can swap vendors. |
| LLM / AI | **Pluggable LLM gateway** (OpenAI-compatible API by default; configurable base URL) | Care-plan auto-generation and intervention matching. Abstracted so on-prem deployments can point at a local model for PHI residency. |
| ML runtime | **scikit-learn** (risk, readmission, adherence, outreach-timing models); **`fairlearn`** for equity auditing | Interpretable, lightweight, no GPU requirement — fits self-hosted FQHC deployments. Explainability via SHAP for the "explainable risk scoring" feature. |
| Frontend | **Next.js 15 (App Router) + TypeScript + shadcn/ui + TanStack Query** | Care-coordinator worklist dashboard, care-plan editor, quality-measure dashboards, patient timeline. Server components for fast first paint; React Query against the FastAPI REST API. |
| Auth | **OAuth 2.0 / OIDC** (RFC 6749) via `authlib`; SMART-on-FHIR launch support for EHR embedding | `standards.md` requires OAuth 2.0; SMART-on-FHIR enables Epic/Oracle embedding. Internal sessions use short-lived JWTs. |
| Authorization | **Role-based access control (RBAC)** + PostgreSQL **Row-Level Security** keyed on `tenant_id` | HIPAA Security Rule minimum-necessary access; multi-tenant isolation per organization. |
| Audit | **Trigger-based `audit_log` (JSONB)** + application-level access logging | HIPAA audit-trail requirement; every PHI read/write logged. |
| Containerisation | **Docker + docker-compose** | Self-hosted deployment target (README). One compose file brings up API, worker, Postgres, Redis, web. |
| Testing | **pytest** + **pytest-asyncio** + **testcontainers** (real Postgres/Redis) + **Playwright** (web e2e) | Healthcare correctness demands integration tests against a real database. |
| Code quality | **ruff** (lint+format), **mypy** (strict typing), **bandit** (security) | Type safety and security scanning are non-negotiable for a PHI system. |
| Package manager | **uv** (Python), **pnpm** (web) | Fast, reproducible installs. |
| Terminology | Bundled **ICD-10-CM**, **SNOMED CT** (subset), **LOINC**, **RxNorm**, **CPT** code reference loaders | Required for condition matching, lab/care-gap detection, and CCM billing codes. |

### Project Structure

```
care-management-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── README.md
├── migrations/                      # Alembic migrations
│   └── versions/
├── src/
│   └── cmp/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory + router registration
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py           # Async engine + session factory
│       │   ├── base.py              # DeclarativeBase, JSONB types, mixins (tenant, timestamps)
│       │   ├── rls.py               # Row-Level Security session context
│       │   └── models/              # SQLAlchemy models (one file per domain)
│       │       ├── patient.py
│       │       ├── risk.py
│       │       ├── program.py
│       │       ├── care_plan.py
│       │       ├── care_gap.py
│       │       ├── outreach.py
│       │       ├── sdoh.py
│       │       ├── billing.py
│       │       ├── care_team.py
│       │       ├── clinical.py      # fhir_resource, clinical_observation
│       │       └── audit.py
│       ├── schemas/                 # Pydantic request/response + JSONB document schemas
│       ├── domain/                  # Business logic (no DB/HTTP concerns)
│       │   ├── risk/                # Risk engine: rule-based + ML scorers, equity audit
│       │   ├── caregaps/            # Quality-measure evaluation engine
│       │   ├── careplan/            # Care plan lifecycle + AI draft generation
│       │   ├── outreach/            # Campaign orchestration, channel routing, ML timing
│       │   ├── sdoh/                # Screening scoring, need detection, referral matching
│       │   ├── billing/             # CCM CPT rules + time aggregation
│       │   ├── caseload/            # Caseload balancing
│       │   └── mpi/                 # Master patient index matching
│       ├── integrations/
│       │   ├── fhir/                # FHIR client, ingest ETL, FHIR facade serialiser
│       │   ├── ehr/                 # EHR connectors (Epic, Cerner via SMART-on-FHIR)
│       │   ├── outreach_providers/  # Twilio, SMTP/SendGrid adapters
│       │   └── llm/                 # LLM gateway client
│       ├── api/
│       │   ├── deps.py              # Auth, tenant, RBAC dependencies
│       │   └── routers/             # FastAPI routers (patients, risk, caregaps, ...)
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/               # ingest, scoring, outreach, measure_eval, mv_refresh
│       ├── terminology/             # Code-system loaders (ICD-10, SNOMED, LOINC, RxNorm, CPT)
│       └── security/                # OAuth/OIDC, JWT, RBAC policy, audit middleware
├── tests/
│   ├── conftest.py                  # testcontainers fixtures (Postgres, Redis)
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/                    # Sample FHIR bundles, screening instruments, measures
├── seed/                            # Seed data: disease programs, quality measures, SDOH instruments
└── web/
    ├── package.json
    ├── app/                         # Next.js App Router pages
    ├── components/
    ├── lib/                         # API client, auth
    └── e2e/                         # Playwright specs
```

---

## Phase 1: Foundation & Core Data Layer

### Purpose
Establish the project skeleton, configuration, async database layer, multi-tenant isolation, audit logging, and the core patient/MPI/care-team schema. After this phase the platform can store tenants, users, care-team members, and patients with HIPAA-grade RLS and audit, and exposes a health-checked, OpenAPI-documented API shell. Everything else builds on this.

### Tasks

#### 1.1 — Project Scaffold & Configuration

**What**: Create the repo structure, dependency management, Docker stack, and environment-driven configuration.

**Design**:
- `pyproject.toml` with dependencies grouped (`api`, `workers`, `ml`, `dev`). Tooling config for `ruff`, `mypy --strict`, `pytest`.
- `config.py` using Pydantic `BaseSettings`:

```python
class Settings(BaseSettings):
    app_env: Literal["dev", "test", "prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    jwt_secret: SecretStr
    oidc_issuer: str | None = None
    oidc_client_id: str | None = None
    llm_base_url: str = "https://api.openai.com/v1"
    llm_api_key: SecretStr | None = None
    twilio_account_sid: str | None = None
    twilio_auth_token: SecretStr | None = None
    smtp_url: str | None = None
    phi_encryption_key: SecretStr           # column-level encryption for sensitive identifiers
    audit_enabled: bool = True
    model_config = SettingsConfigDict(env_file=".env", env_prefix="CMP_")
```
- `docker-compose.yml` services: `api`, `worker`, `web`, `postgres:16`, `redis:7`. Postgres init script enables `pgcrypto`, `pg_trgm`, `pg_partman` extensions.
- `main.py`: FastAPI app factory registering routers, exception handlers, CORS, audit middleware; `/health` and `/health/db` endpoints.

**Testing**:
- `Unit: Settings loads from env with prefix CMP_ → correct typed values`
- `Unit: missing required DATABASE_URL → ValidationError naming the field`
- `Integration: GET /health → 200 {"status":"ok"}`
- `Integration: GET /health/db with Postgres up → 200; with DB down → 503`
- `E2E: docker compose up → all 5 services healthy within 60s`

#### 1.2 — Async DB Layer, Base Mixins & Tenancy

**What**: SQLAlchemy 2.0 async engine, session factory, declarative base, and reusable mixins.

**Design**:
- `db/base.py`:

```python
class Base(DeclarativeBase): ...

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

class TenantMixin:
    tenant_id: Mapped[uuid.UUID] = mapped_column(index=True)
```
- `db/session.py`: `create_async_engine`, `async_sessionmaker`, FastAPI dependency `get_session()`.
- Custom typed JSONB column helper (`MutableJSONB`) wrapping `postgresql.JSONB` for Pydantic round-tripping.

**Testing**:
- `Integration: session commits and rolls back correctly (testcontainers Postgres)`
- `Unit: TimestampMixin sets created_at/updated_at on insert/update`
- `Unit: MutableJSONB serialises a Pydantic model to JSONB and back losslessly`

#### 1.3 — Row-Level Security & Tenant Context

**What**: Enforce per-tenant data isolation at the database level.

**Design**:
- Every tenant-scoped table gets an RLS policy: `USING (tenant_id = current_setting('app.current_tenant')::uuid)`.
- `db/rls.py`: a context manager that runs `SET LOCAL app.current_tenant = :tid` and `SET LOCAL app.current_user = :uid` per request transaction.
- API dependency `tenant_session()` derives tenant from the authenticated principal and yields an RLS-scoped session.

**Testing**:
- `Integration: query as tenant A cannot read tenant B rows (RLS denies)`
- `Integration: SET LOCAL leaks nothing across pooled connections (verify reset per txn)`
- `Integration: superuser/migration role bypasses RLS as configured`

#### 1.4 — Patient / MPI / Care Team Schema

**What**: Implement the core relational tables from data-model-suggestion-3 (hybrid).

**Design** — key tables (`patient` with JSONB extensions, plus care-team structure):

```sql
CREATE TABLE organization (
    organization_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    tenant_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE TABLE facility (
    facility_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organization(organization_id),
    name VARCHAR(200) NOT NULL, tenant_id UUID NOT NULL
);
CREATE TABLE care_team (
    care_team_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    facility_id UUID REFERENCES facility(facility_id),
    name VARCHAR(200) NOT NULL, tenant_id UUID NOT NULL
);
CREATE TABLE care_team_member (
    member_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    care_team_id UUID REFERENCES care_team(care_team_id),
    user_id UUID NOT NULL,                      -- links to auth user
    role VARCHAR(50) NOT NULL,                  -- CARE_COORDINATOR, RN, SOCIAL_WORKER, PHYSICIAN, CHW
    max_caseload INTEGER, tenant_id UUID NOT NULL
);
CREATE TABLE patient (
    patient_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mpi_id UUID NOT NULL UNIQUE,
    mrn VARCHAR(50), first_name VARCHAR(100) NOT NULL, last_name VARCHAR(100) NOT NULL,
    date_of_birth DATE NOT NULL, sex VARCHAR(10), preferred_language VARCHAR(50),
    race VARCHAR(50), ethnicity VARCHAR(50), deceased_flag BOOLEAN DEFAULT FALSE,
    tenant_id UUID NOT NULL,
    identifiers JSONB DEFAULT '[]', addresses JSONB DEFAULT '[]', contacts JSONB DEFAULT '[]',
    extensions JSONB DEFAULT '{}', fhir_resource JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_patient_identifiers ON patient USING GIN (identifiers);
CREATE INDEX idx_patient_name_trgm ON patient USING GIN ((first_name || ' ' || last_name) gin_trgm_ops);
```
- Pydantic schemas for `PatientCreate`, `PatientRead`, embedded `Identifier`, `Address`, `Contact` (validate JSONB content).

**Testing**:
- `Unit: PatientCreate rejects future date_of_birth`
- `Unit: Identifier list validation rejects unknown identifier_type`
- `Integration: insert patient with identifiers JSONB → GIN-indexed query by identifier value returns it`
- `Integration: fuzzy name search via pg_trgm returns near-matches`

#### 1.5 — Audit Logging

**What**: HIPAA-compliant change and access audit.

**Design**:
- `audit_log` table (table_name, record_id, action, changed_by, old_values JSONB, new_values JSONB, ip_address, user_agent, accessed_at).
- Generic Postgres trigger function `fn_audit()` attached to all PHI tables for INSERT/UPDATE/DELETE.
- Application middleware logs PHI **reads** (GET on patient-scoped endpoints) with principal + resource id.

**Testing**:
- `Integration: UPDATE patient → audit_log row with old/new JSONB diff and changed_by`
- `Integration: GET /patients/{id} → read-audit entry with user + ip`
- `Integration: DELETE → audit row with action=DELETE before row removal`

### Definition of Done
Standard checklist (see end of document).

---

## Phase 2: AuthN/AuthZ & FHIR Resource Store

### Purpose
Add identity, OAuth 2.0 / OIDC authentication, RBAC, SMART-on-FHIR launch support, and the FHIR resource ingestion store. After this phase, real users authenticate, every endpoint enforces role + tenant, and the system can receive and persist FHIR R4 resources from EHRs — the substrate for risk scoring and care-gap detection.

### Tasks

#### 2.1 — Authentication (OAuth 2.0 / OIDC + SMART-on-FHIR)

**What**: Authenticate care-team users and support EHR-launched SMART sessions.

**Design**:
- `security/oauth.py` using `authlib`: authorization-code flow against a configurable OIDC issuer; local username/password fallback for self-hosted dev.
- SMART-on-FHIR launch: handle `launch` + `iss` params, perform the SMART authorization handshake, obtain a FHIR access token, map the launched user to a `care_team_member`.
- Short-lived access JWT (15 min) + refresh token; JWT carries `sub`, `tenant_id`, `roles[]`, `member_id`.
- `api/deps.py: get_principal()` validates JWT and returns `Principal(user_id, tenant_id, roles, member_id)`.

**Testing**:
- `Integration (mocked OIDC): valid code → JWT issued with correct claims`
- `Integration: expired JWT → 401`
- `Integration: SMART launch with valid iss/launch → session bound to FHIR server + member`
- `Unit: JWT with tampered signature → rejected`

#### 2.2 — RBAC & Authorization Policy

**What**: Enforce role-based permissions on every route.

**Design**:
- Roles: `ADMIN`, `CARE_COORDINATOR`, `RN`, `SOCIAL_WORKER`, `PHYSICIAN`, `CHW`, `READ_ONLY`, `BILLING`.
- `security/policy.py`: permission matrix mapping `(role, action, resource)` → allow/deny. Action verbs: `read`, `write`, `enroll`, `close_gap`, `bill`, `admin`.
- FastAPI dependency `require(permission: str)` raising 403 on deny; logs denied attempts to audit.

**Testing**:
- `Unit: BILLING role cannot write care_plan → deny`
- `Unit: CARE_COORDINATOR can close_gap → allow`
- `Integration: READ_ONLY POST /patients → 403 + audit entry`

#### 2.3 — FHIR Resource Store & Facade

**What**: Persist raw FHIR R4 resources and serve them back FHIR-compliantly.

**Design**:
- `fhir_resource` table (from suggestion-3): `resource_type`, `fhir_id`, `patient_id`, `source_system`, `resource_data JSONB`, `version_id`, GIN index + partial indexes for Condition/Observation/MedicationRequest.
- `integrations/fhir/facade.py`: validates inbound payloads with `fhir.resources` R4 models; reconstructs FHIR resources from stored JSONB for `GET /fhir/{type}/{id}`.
- Supported inbound types: Patient, Condition, Observation, MedicationRequest, Encounter, CarePlan, Goal, CareTeam, QuestionnaireResponse.

**Testing**:
- `Unit: invalid Condition resource (missing subject) → 422 with FHIR-style OperationOutcome`
- `Integration: POST /fhir/Condition → stored; GET /fhir/Condition/{id} → byte-equivalent resource`
- `Integration: GIN query "all Conditions for patient X" returns expected set`
- `Fixture: load sample Synthea FHIR bundle → all resources persisted`

### Definition of Done
Standard checklist.

---

## Phase 3: FHIR Ingestion ETL & Clinical Normalisation

### Purpose
Turn raw FHIR resources into the queryable clinical substrate (conditions, observations, medications, encounters) and resolve patient identity (MPI). This is the data foundation the risk engine and care-gap engine depend on. After this phase, a FHIR bundle dropped in becomes a normalized, MPI-resolved longitudinal record.

### Tasks

#### 3.1 — Master Patient Index (MPI) Matching

**What**: Resolve incoming patient records to a single canonical patient across source systems.

**Design**:
- `domain/mpi/matcher.py`: deterministic + probabilistic matching. Deterministic on (SSN, or MRN+assigning_authority). Probabilistic (Fellegi-Sunter style) on name (Jaro-Winkler), DOB, sex, address — weighted score with `auto_link` and `manual_review` thresholds.
- On match → attach to existing `mpi_id`; on no match → mint new; on ambiguous → queue a `mpi_review_task`.

```python
@dataclass
class MatchCandidate:
    patient_id: uuid.UUID
    score: float
    matched_fields: list[str]

def match(incoming: PatientCreate, session) -> MatchResult:  # AUTO_LINK | NEW | REVIEW
    ...
```

**Testing**:
- `Unit: exact SSN match → AUTO_LINK`
- `Unit: same name+DOB, different MRN, high prob score → AUTO_LINK`
- `Unit: borderline score in [0.6,0.85) → REVIEW (task created)`
- `Unit: typo in last name still links via Jaro-Winkler above threshold`

#### 3.2 — Clinical Observation Extraction

**What**: Extract indexed clinical values from FHIR Observations/Conditions/MedicationRequests.

**Design**:
- `clinical_observation` table (suggestion-3): `observation_type`, `loinc_code`, `snomed_code`, `value_numeric`, `value_unit`, `effective_date`, `components JSONB`, `reference_range JSONB`.
- ETL maps FHIR Observation → relational row: pull `code.coding[LOINC]`, `valueQuantity`, multi-component (BP) into `components`.
- `condition` extraction populates a derived `patient_condition` table keyed by ICD-10/SNOMED for registry membership.

**Testing**:
- `Unit: FHIR Observation HbA1c (LOINC 4548-4) → row value_numeric=7.2, unit=%`
- `Unit: BP Observation with systolic/diastolic components → components JSONB has both`
- `Integration: ingest bundle → clinical_observation rows queryable by loinc_code + date`

#### 3.3 — Ingestion Pipeline & Idempotency

**What**: Celery-driven ingestion with dedup and re-processing safety.

**Design**:
- `workers/tasks/ingest.py`: `ingest_fhir_bundle(bundle_id)` → validate → MPI resolve → upsert fhir_resource (keyed on fhir_id+type+source) → extract clinical rows.
- Idempotent on `(fhir_id, resource_type, source_system, version_id)`: re-ingesting the same version is a no-op; newer `meta.lastUpdated` supersedes.
- Staging table `fhir_ingest_staging` for validation before promotion.

**Testing**:
- `Integration: ingest same bundle twice → no duplicate clinical_observation rows`
- `Integration: ingest newer version of Observation → supersedes prior, old retained as version`
- `Integration: malformed entry in bundle → that entry quarantined, rest processed`

### Definition of Done
Standard checklist.

---

## Phase 4: Risk Stratification Engine

### Purpose
The clinical heart of the product. Compute multi-factor risk scores, assign risk tiers, and produce explainable factor breakdowns. Supports both rule-based scoring (MVP, transparent) and ML predictive scoring (extensible). After this phase, populations are segmented into actionable risk tiers with auditable reasoning — driving program enrollment and worklist prioritisation.

### Tasks

#### 4.1 — Risk Score Schema & Scorer Interface

**What**: Persist risk scores with JSONB factor breakdowns and define a pluggable scorer.

**Design**:
- `risk_score` table (suggestion-3): `model_name`, `model_version`, `score_value`, `risk_tier`, `calculated_at`, `factors JSONB`, `equity_metrics JSONB`. GIN index on `factors`.
- Scorer interface:

```python
class RiskFactor(BaseModel):
    name: str; value: str | float; contribution: float; description: str

class RiskResult(BaseModel):
    score_value: float; risk_tier: Literal["LOW","MODERATE","HIGH","VERY_HIGH"]
    factors: list[RiskFactor]; model_name: str; model_version: str

class RiskScorer(Protocol):
    def score(self, patient_features: PatientFeatures) -> RiskResult: ...
```
- `PatientFeatures` assembled from clinical_observation, patient_condition, encounters, claims, SDOH.

**Testing**:
- `Unit: RiskResult tier derives correctly from score thresholds`
- `Integration: persisted factors JSONB queryable via GIN (e.g., patients where factor 'recent_ed_visit')`

#### 4.2 — Rule-Based Scorer (MVP)

**What**: Transparent, configurable rule-based risk model.

**Design**:
- `domain/risk/rule_based.py`: weighted additive model. Configurable YAML ruleset: condition counts (diabetes, CHF, COPD, HTN, depression), recent ED visits/admissions, polypharmacy, abnormal labs (HbA1c>9, eGFR<45), age, SDOH need count.
- Each rule contributes to `factors[]` with `contribution` weight → total → tier thresholds (LOW<0.25, MODERATE<0.5, HIGH<0.75, else VERY_HIGH).

**Testing**:
- `Unit: patient with diabetes + 2 ED visits + HbA1c 10.5 → HIGH tier with 3 named factors`
- `Unit: healthy patient → LOW with empty/low factors`
- `Unit: changing YAML weight changes contribution proportionally`

#### 4.3 — Explainable Output & Risk API

**What**: Expose risk scores and per-factor explanations.

**Design**:
- `GET /patients/{id}/risk` → latest RiskResult with factors.
- `GET /populations/risk?tier=HIGH&condition=diabetes` → paginated cohort.
- `POST /risk/recompute` (admin) → enqueue batch recompute.

**Testing**:
- `Integration: GET /patients/{id}/risk → latest score + ordered factors by contribution`
- `Integration: population query filters by tier and condition`
- `Integration: recompute job updates scores, supersedes prior (valid_to set)`

#### 4.4 — Batch Scoring Job

**What**: Recompute scores across a population on schedule.

**Design**:
- Celery beat task `recompute_population_risk(tenant_id)` chunked over patients; writes new `risk_score` rows; emits `RiskTierChanged` notifications when tier changes.

**Testing**:
- `Integration: batch over 1000 patients completes; tier-change set matches expected`
- `Integration: idempotent re-run same day produces no spurious tier-change events`

### Definition of Done
Standard checklist.

---

## Phase 5: Quality Measures & Care Gap Engine

### Purpose
Detect care gaps by evaluating HEDIS/MIPS quality-measure logic against each patient's clinical data, track gap closure, and compute measure performance. This delivers the core value-based-care reporting the target buyers need and feeds CMS ACCESS-model accountability. After this phase, the platform answers "which patients have which open gaps, and what is our HEDIS performance?"

### Tasks

#### 5.1 — Quality Measure Definition Model

**What**: Represent quality measures with structured, year-versioned logic.

**Design**:
- `quality_measure` table (suggestion-3): `measure_code`, `measure_set` (HEDIS/MIPS/CUSTOM), `measure_year`, `numerator_logic JSONB`, `denominator_logic JSONB`, `exclusion_logic JSONB`.
- Logic expressed as a small declarative DSL (JSON): predicates over conditions (value-set OIDs / code lists), observations (LOINC + value comparison + lookback window), age, sex, enrollment.

```json
{
  "denominator": {"all": [{"condition_in": "diabetes_vs"}, {"age_between": [18, 75]}]},
  "numerator":   {"observation": {"loinc": "4548-4", "within_months": 12}},
  "exclusion":   {"any": [{"condition_in": "hospice_vs"}, {"deceased": true}]}
}
```
- Seed common measures: CDC (diabetes HbA1c), CBP (controlling high BP), BCS (breast cancer screening), COL (colorectal), CDC-eye, SPD (statin), AMR (asthma med ratio).

**Testing**:
- `Unit: DSL evaluates denominator predicate over a fixture patient correctly`
- `Unit: lookback window excludes observation older than window`
- `Unit: unknown value-set reference → config error at load`

#### 5.2 — Care Gap Evaluation Engine

**What**: Evaluate measures → create/close care gaps.

**Design**:
- `care_gap` table (suggestion-3): `measure_id`, `gap_type`, `status` (OPEN/IN_PROGRESS/CLOSED/EXCLUDED), `identified_date`, `due_date`, `evidence JSONB`, `actions_taken JSONB`, `closure_evidence JSONB`.
- `domain/caregaps/evaluator.py`: for each patient in a measure denominator and not excluded, if numerator unmet → OPEN gap (idempotent: don't duplicate open gaps); if numerator becomes met → auto-close with `closure_evidence`.

**Testing**:
- `Unit: diabetic patient, no HbA1c in 12mo → OPEN gap for CDC`
- `Unit: same patient gets HbA1c → gap auto-closes with evidence`
- `Unit: hospice patient → EXCLUDED, no gap`
- `Integration: re-evaluation does not duplicate an existing OPEN gap`

#### 5.3 — Measure Performance Reporting & Dashboards

**What**: Compute and serve measure performance.

**Design**:
- Materialized view `mv_quality_measure_performance` (numerator/denominator/exclusions/rate per measure per tenant per month), refreshed by Celery beat.
- `GET /measures/performance?set=HEDIS&period=2026-05` → rates + counts.
- `GET /measures/{code}/opportunities` → patient-level open gaps (drill-down).

**Testing**:
- `Integration: mv refresh computes rate = numerator/denominator*100 correctly`
- `Integration: drill-down lists exactly the open-gap patients`
- `Integration: SARIF-style is N/A; verify MeasureReport FHIR export shape (5.4)`

#### 5.4 — FHIR MeasureReport Export

**What**: Export performance as FHIR R4 MeasureReport for interoperability (per `standards.md`).

**Design**:
- `integrations/fhir/measure_report.py`: build `MeasureReport` (type=summary) per measure/period with population counts and `measureScore`.

**Testing**:
- `Unit: MeasureReport validates against fhir.resources R4 model`
- `Integration: GET /fhir/MeasureReport?measure=CDC&period=2026-05 → valid resource`

### Definition of Done
Standard checklist.

---

## Phase 6: Disease Programs, Enrollment & Care Plans

### Purpose
Enable enrollment of risk-stratified patients into evidence-based disease management programs and collaborative care-plan authoring (problem list, goals, interventions, reviews). This is where care actually gets coordinated. After this phase, coordinators enroll patients, author care plans, and track goal/intervention progress — mapped to FHIR CarePlan/Goal.

### Tasks

#### 6.1 — Disease Program Templates & Enrollment

**What**: Configurable program templates + enrollment lifecycle.

**Design**:
- `disease_program` (program_name, condition_code/system, protocol_version, template JSONB of default goals/interventions/escalations).
- `program_enrollment` table (status: ENROLLED/COMPLETED/WITHDRAWN/GRADUATED, enrolled_date, enrolled_by, disenroll_reason).
- Seed templates: Diabetes, Hypertension, COPD, CHF, Depression — each with evidence-based default goals and intervention checklists.
- `POST /patients/{id}/enrollments {program_id}` enforces consent (Phase 9 link) + risk eligibility.

**Testing**:
- `Unit: enrollment status transitions validated (ENROLLED→GRADUATED ok; WITHDRAWN→ENROLLED rejected)`
- `Integration: enroll diabetic patient in Diabetes program → care plan seeded from template`
- `Integration: enroll without consent → 409`

#### 6.2 — Care Plan Authoring

**What**: CRUD for care plans with goals, interventions, problem list, reviews.

**Design**:
- `care_plan` table (suggestion-3, hybrid): relational status/period/author + JSONB `goals`, `interventions`, `problem_list`, `review_history`, `fhir_care_plan`.
- Pydantic schemas validate JSONB sub-documents (`Goal`, `Intervention` with `goal_id` referential check at app layer — noted con of JSONB approach).
- Endpoints: `POST/GET/PATCH /careplans`, `POST /careplans/{id}/goals`, `/interventions`, `/reviews`.

**Testing**:
- `Unit: intervention referencing non-existent goal_id → 422 (app-layer integrity)`
- `Integration: add goal then intervention → both in JSONB, GIN-queryable`
- `Integration: PATCH status DRAFT→ACTIVE allowed; ACTIVE→DRAFT rejected`

#### 6.3 — FHIR CarePlan / Goal Sync

**What**: Bidirectional FHIR CarePlan/Goal representation.

**Design**:
- `careplan/fhir_sync.py`: serialise internal care plan → FHIR CarePlan + contained Goal resources; ingest external FHIR CarePlan → internal representation.

**Testing**:
- `Unit: internal care plan → valid FHIR CarePlan with linked Goals`
- `Integration: round-trip (export then re-import) preserves goals/interventions`

#### 6.4 — AI Care-Plan Draft Generation

**What**: LLM-drafted initial care plan for coordinator review (AI-native differentiator).

**Design**:
- `integrations/llm/careplan_drafter.py`: prompt template embedding patient diagnoses, risk factors, open gaps, SDOH needs, program template → returns structured draft goals/interventions. PHI-minimised prompt (no direct identifiers). Output strictly validated against the `Goal`/`Intervention` schema; never auto-activated — `status=DRAFT`, `authored_by=AI_ASSIST` flag for human review.

System prompt structure:
```
You are a clinical care-plan assistant. Given the patient's conditions, risk factors,
open care gaps, and SDOH needs, propose SMART goals and evidence-based interventions
for the {program_name} program. Output JSON matching the provided schema. Do not invent
diagnoses. Flag anything requiring physician review.
```

**Testing**:
- `Integration (mocked LLM): draft returns schema-valid goals → persisted as DRAFT/AI_ASSIST`
- `Unit: malformed LLM JSON → rejected, no partial write`
- `Unit: prompt contains no direct patient identifiers (PHI minimisation check)`

### Definition of Done
Standard checklist.

---

## Phase 7: Care Team Workflows, Tasks & Caseload

### Purpose
Operationalise day-to-day coordinator work: task assignment, the coordinator worklist, and caseload visibility. This is the primary daily UI surface. After this phase, coordinators have a prioritised worklist and managers can see and balance caseloads.

### Tasks

#### 7.1 — Care Tasks & Assignment

**What**: Task entity with assignment and workflow states.

**Design**:
- `care_task` (patient_id, assigned_to member_id, task_type, status NEW/IN_PROGRESS/DONE/CANCELLED, due_date, priority, source — e.g. gap, outreach, review). Auto-generated tasks from open gaps and overdue care-plan reviews.

**Testing**:
- `Unit: closing a care gap auto-completes its linked task`
- `Integration: assign task → appears on assignee worklist`

#### 7.2 — Coordinator Worklist Projection

**What**: Aggregated per-patient worklist (from suggestion-3 `mv_coordinator_worklist`).

**Design**:
- Materialized view joining latest risk tier, open gap count, last outreach, CCM minutes this month, active care-plan status, SDOH needs; refreshed incrementally on relevant events + periodic full refresh.
- `GET /worklist?coordinator={id}&sort=risk_desc` paginated.

**Testing**:
- `Integration: mv reflects open_gaps, last_outreach, ccm_minutes accurately`
- `Integration: worklist sorted by risk tier then due date`

#### 7.3 — Caseload Dashboard & Manual Balancing

**What**: Caseload capacity visibility and reassignment.

**Design**:
- `GET /caseloads` → per-coordinator counts vs `max_caseload`, overdue counts.
- `POST /caseloads/rebalance` (manual reassignment of patients between coordinators).

**Testing**:
- `Integration: caseload counts match assigned patients`
- `Integration: rebalance moves patients and updates worklists + audit`

### Definition of Done
Standard checklist.

---

## Phase 8: Multi-Channel Outreach

### Purpose
Reach patients across SMS, email, voice, and portal with consent enforcement and result tracking. Outreach is how care gaps actually get closed and patients stay engaged. After this phase, coordinators run campaigns and log/automate contacts with full consent compliance.

### Tasks

#### 8.1 — Consent & Communication Preferences

**What**: Track consent per channel/purpose (HIPAA + TCPA for SMS/voice).

**Design**:
- `patient_consent` (consent_type: CCM_ENROLLMENT, SMS_OUTREACH, VOICE_OUTREACH, EMAIL_OUTREACH, status ACTIVE/REVOKED/EXPIRED, granted/revoked dates). Outreach dispatch checks active consent for the channel or refuses.

**Testing**:
- `Unit: SMS dispatch with no SMS_OUTREACH consent → blocked`
- `Integration: revoke consent → subsequent dispatch on that channel refused + audited`

#### 8.2 — Outreach Provider Abstraction

**What**: Pluggable channel providers.

**Design**:
```python
class OutreachProvider(Protocol):
    channel: Literal["SMS","EMAIL","VOICE","PORTAL"]
    async def send(self, to: Contact, message: RenderedMessage) -> DeliveryResult: ...
```
- Twilio adapter (SMS/voice), SMTP/SendGrid adapter (email), in-app portal adapter. Webhook endpoints ingest delivery/response status.

**Testing**:
- `Unit (mocked Twilio): send SMS → DeliveryResult(status=SENT, provider_id)`
- `Integration: Twilio status webhook → outreach_attempt status updated`
- `Integration: invalid webhook signature → 401, no update`

#### 8.3 — Campaigns & Attempt Tracking

**What**: Campaign orchestration and per-attempt records.

**Design**:
- `outreach_campaign` (type CARE_GAP/WELLNESS/ENROLLMENT/FOLLOW_UP, target_criteria, status) and `outreach_attempt` (channel, direction, status, attempted_at, duration, notes, `ml_features JSONB`, `response_data JSONB`).
- Celery task `run_campaign` resolves target cohort (e.g., open-gap patients), renders templated messages, dispatches via providers respecting consent + quiet hours.
- Non-responder escalation: after N failed attempts, escalate channel/create task.

**Testing**:
- `Integration: campaign targets open-gap cohort → attempts created for consented patients only`
- `Unit: 3 NO_RESPONSE attempts → escalation task created`
- `Integration: contact-rate metric computed per campaign`

### Definition of Done
Standard checklist.

---

## Phase 9: SDOH Screening & Community Referral

### Purpose
Make social determinants of health a first-class capability (the key differentiator from `features.md`): validated screening instruments, need detection, and community-resource referral with outcome tracking. After this phase, the platform screens with AHC-HRSN/PRAPARE, detects needs, and tracks referrals to closure.

### Tasks

#### 9.1 — Screening Instruments (AHC-HRSN, PRAPARE)

**What**: Encode validated, licence-free instruments and capture responses.

**Design**:
- `sdoh_instrument` definitions (questions, domains, scoring) loaded from seed JSON; `sdoh_screening` table (suggestion-3) with `responses JSONB`, `identified_needs JSONB`, `referrals JSONB`, `fhir_questionnaire_response JSONB`.
- Domains: HOUSING, FOOD, TRANSPORTATION, UTILITIES, SAFETY, FINANCIAL.
- Need detection rules per instrument (e.g., AHC-HRSN core domains positive thresholds).

**Testing**:
- `Unit: AHC-HRSN responses with food-insecurity positive → identified_need FOOD/HIGH`
- `Integration: persist screening → GIN query "patients with FOOD need" returns it`
- `Unit: export FHIR QuestionnaireResponse validates`

#### 9.2 — Community Resource Directory & Referral

**What**: Resource directory with eligibility + referral lifecycle.

**Design**:
- `community_resource` (name, type, address, eligibility_criteria JSONB, geo) and referral records (status REFERRED/ACCEPTED/COMPLETED/DECLINED/NOT_ELIGIBLE, referred/outcome dates).
- `domain/sdoh/matcher.py`: match identified need → eligible resources by type + geography + eligibility criteria.

**Testing**:
- `Unit: food need + patient in zip 10001 → eligible food banks ranked`
- `Integration: create referral → status transitions tracked; outcome recorded`
- `Unit: ineligible patient excluded from matches`

#### 9.3 — SDOH Outcome Tracking

**What**: Track whether referrals resolved needs and feed back into risk.

**Design**:
- Link referral outcomes to need resolution; expose `GET /sdoh/outcomes` aggregations (referral acceptance rate, resolution rate by domain). SDOH need count feeds the risk scorer (Phase 4 feature).

**Testing**:
- `Integration: completed referral marks need resolved; outcome metrics update`

### Definition of Done
Standard checklist.

---

## Phase 10: CCM Billing & Time Tracking

### Purpose
Capture billable care-management time and enforce CMS CCM compliance (CPT 99490, 99491, 99487, 99489), enabling the fee-for-service and CMS ACCESS-model revenue that justifies platform ROI. After this phase, organizations can substantiate and export CCM claims.

### Tasks

#### 10.1 — Time Entry & CPT Rules

**What**: Record per-activity time and derive billable CPT codes.

**Design**:
- `billing_time_entry` table (suggestion-3): `activity_type`, `cpt_code`, `duration_minutes`, `service_date`, `billing_period` (YYYY-MM), `compliance_data JSONB` (care_plan_elements_addressed, team_composition_documented, consent_verified).
- `domain/billing/ccm_rules.py`: aggregate monthly minutes per patient → determine eligible CPT:
  - 99490: ≥20 min clinical staff, non-complex
  - 99439: each additional 20 min (add-on)
  - 99491: ≥30 min physician/QHP
  - 99487: ≥60 min complex CCM; 99489 add-on each additional 30 min
  - Enforce: active consent, established care plan, eligible chronic conditions (≥2), within calendar month.

**Testing**:
- `Unit: 22 staff minutes + consent + care plan + 2 conditions → 99490 eligible`
- `Unit: 18 minutes → not billable`
- `Unit: no consent → not billable regardless of minutes`
- `Unit: 65 complex minutes → 99487`

#### 10.2 — Billing Period Close & Compliance Check

**What**: Close a billing period and produce a compliance-validated claim list.

**Design**:
- `POST /billing/periods/{period}/close` → for each patient, run CCM rules, assert compliance documentation present, produce claimable lines; flag deficiencies (missing consent/care-plan element) instead of silently dropping.
- `GET /billing/claims?period=2026-05` export (CSV + JSON).

**Testing**:
- `Integration: close period → claim lines match rule outcomes`
- `Integration: patient missing documented care plan → flagged, not claimed`
- `Integration: CSV export columns match billing schema`

### Definition of Done
Standard checklist.

---

## Phase 11: AI/ML Layer — Predictive Models & Equity Auditing

### Purpose
Deliver the AI-native advantages: ML predictive risk, equity-aware model validation (a core differentiator addressing bias against under-served populations), readmission and medication-adherence prediction, and outreach-timing optimisation. After this phase, the platform moves beyond rule-based scoring to learned, continuously-audited models.

### Tasks

#### 11.1 — ML Risk Scorer & Feature Store

**What**: Train and serve an ML predictive risk model implementing the `RiskScorer` interface.

**Design**:
- `domain/risk/ml_scorer.py`: scikit-learn gradient-boosted classifier predicting 12-month high-utilisation/admission. Feature pipeline from clinical_observation, conditions, encounters, claims, SDOH. SHAP values populate `factors[]` for explainability (reuses Phase 4 schema and API).
- Versioned model artifacts in `model_registry` table (model_name, version, metrics JSONB, artifact path).

**Testing**:
- `Unit: ml_scorer.score() returns RiskResult with SHAP-derived factors summing to score`
- `Integration: ML scorer swappable for rule-based via config without API change`
- `Fixture: synthetic cohort → AUC above baseline threshold`

#### 11.2 — Equity-Aware Validation (`fairlearn`)

**What**: Continuously detect and report risk-model bias across demographic groups.

**Design**:
- `domain/risk/equity_audit.py`: compute group fairness metrics (selection rate, calibration, false-negative rate) across race/ethnicity/sex/language using `fairlearn`. Write `equity_metrics JSONB` on scores and an `equity_audit` report; alert when disparity exceeds threshold and recommend recalibration.

```python
@dataclass
class EquityReport:
    model_version: str
    group_metrics: dict[str, GroupMetric]   # by demographic group
    max_disparity: float
    flagged: bool
```

**Testing**:
- `Unit: synthetic biased model (under-flags group B) → flagged=True with disparity`
- `Unit: balanced model → flagged=False`
- `Integration: equity report persisted and exposed at GET /risk/equity`

#### 11.3 — Readmission & Adherence Models

**What**: Predictive readmission risk and medication non-adherence.

**Design**:
- Readmission model (post-discharge 30-day) and adherence model (PDC/MPR from drug_exposure gaps). Outputs surface as worklist flags and intervention recommendations.

**Testing**:
- `Unit: PDC computed correctly from refill intervals`
- `Integration: recent inpatient discharge → readmission flag on worklist`

#### 11.4 — Outreach Timing & Modality Optimisation

**What**: Predict best channel and time per patient.

**Design**:
- `domain/outreach/optimiser.py`: model trained on historical `outreach_attempt` outcomes (channel, hour, result) → per-patient channel/time recommendation written to `ml_features` and used by campaign dispatch ordering.

**Testing**:
- `Unit: patient with history of evening SMS success → recommends SMS evening`
- `Integration: campaign uses optimiser ordering when enabled`

### Definition of Done
Standard checklist.

---

## Phase 12: Web Frontend, Reporting & Deployment Hardening

### Purpose
Deliver the care-coordinator and manager UI, analytics/outcomes reporting, and production-grade deployment. After this phase the platform is usable end-to-end by non-technical care teams and deployable by self-hosters.

### Tasks

#### 12.1 — Coordinator Web App

**What**: Next.js app: worklist, patient timeline, care-plan editor, gap closure, outreach logging, SDOH screening, CCM timer.

**Design**:
- App Router pages: `/worklist`, `/patients/[id]` (timeline, risk factors, gaps, care plan, SDOH, billing), `/measures`, `/caseloads`, `/campaigns`. shadcn/ui components, TanStack Query against FastAPI. A live CCM timer widget posts `billing_time_entry`. Auth via OIDC; tenant context from JWT.

**Testing**:
- `E2E (Playwright): login → worklist loads sorted by risk`
- `E2E: open patient → close a care gap → gap disappears, task completes`
- `E2E: start CCM timer, stop → time entry recorded`
- `E2E: complete AHC-HRSN screening → need + referral suggestion shown`

#### 12.2 — Quality & Population Dashboards

**What**: HEDIS/MIPS performance and population analytics dashboards.

**Design**:
- `/measures` performance cards with drill-down to opportunity lists; `/population` risk-tier and condition distributions from `projection_population_summary`-style view; pre/post intervention outcome charts.

**Testing**:
- `E2E: measures dashboard rate matches API; drill-down lists gap patients`
- `Integration: population summary view aggregates correctly`

#### 12.3 — Outcomes & Export Reporting

**What**: Program outcome analysis and exports for payer contracts.

**Design**:
- `GET /reports/outcomes?program=Diabetes` pre/post metrics (e.g., HbA1c control, ED reduction); CSV/FHIR MeasureReport exports.

**Testing**:
- `Integration: pre/post cohort metric computed against fixture data`
- `Integration: export produces well-formed CSV + valid MeasureReport`

#### 12.4 — Deployment, Security & Ops Hardening

**What**: Production deployment artifacts and security posture.

**Design**:
- Multi-stage Dockerfiles (api, worker, web); docker-compose.prod with TLS termination (TLS 1.2+/1.3 per `standards.md`), Postgres with WAL archiving for PITR, Redis persistence, partitioning (audit_log, outreach_attempt, billing_time_entry by month).
- Security: `bandit` + dependency scanning in CI; column-level encryption (pgcrypto) for sensitive identifiers; rate limiting; secrets via env/secret manager. CI runs ruff, mypy, pytest (with testcontainers), Playwright.

**Testing**:
- `E2E: docker compose -f docker-compose.prod.yml up → full stack healthy with TLS`
- `Integration: PITR restore from base backup + WAL recovers to point in time`
- `CI: bandit reports no high-severity findings; mypy strict passes`

### Definition of Done
Standard checklist.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Core Data Layer        ─── required by everything
    │
Phase 2: AuthN/AuthZ & FHIR Resource Store   ─── requires 1
    │
Phase 3: FHIR Ingestion ETL & Normalisation  ─── requires 2
    │
    ├── Phase 4: Risk Stratification Engine   ─── requires 3
    │       │
    │       └── Phase 11: AI/ML & Equity      ─── requires 4 (and benefits from 5,8)
    │
    └── Phase 5: Quality Measures & Care Gaps ─── requires 3 (can parallel with 4)
            │
            ├── Phase 6: Programs & Care Plans ─── requires 4 + 5
            │       │
            │       └── Phase 10: CCM Billing  ─── requires 6 (+ consent from 8/9)
            │
            ├── Phase 7: Care Team Workflows   ─── requires 4 + 5 (can parallel with 6)
            │
            ├── Phase 8: Multi-Channel Outreach─── requires 5 (can parallel with 6,7)
            │
            └── Phase 9: SDOH & Referral       ─── requires 3 (can parallel with 6,7,8)

Phase 12: Web Frontend, Reporting & Deploy   ─── requires all functional phases (4–11)
```

**Parallelism opportunities:**
- After Phase 3: **Phase 4** and **Phase 5** can be developed concurrently.
- After Phases 4+5: **Phase 6**, **Phase 7**, **Phase 8**, and **Phase 9** can largely proceed in parallel by separate contributors (shared touch-point is the worklist projection in Phase 7).
- **Phase 11** (AI/ML) is additive and can be developed alongside Phases 6–10 once Phase 4's `RiskScorer` interface exists.
- **Phase 12** frontend work for each domain can begin as soon as that domain's API stabilises (incremental UI delivery rather than a single big-bang phase).

**Deferred architecture (post-v1):** Event sourcing (data-model-suggestion-2) and OMOP CDM + knowledge-graph (data-model-suggestion-4) are intentionally not in scope. The hybrid schema keeps `fhir_resource` raw payloads and emits domain notifications, so a future migration to event sourcing (strangler-fig dual-write) or an OMOP analytics mirror + Apache AGE graph layer remains open without rework.

---

## Definition of Done (per phase)

Every phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pytest`), including testcontainers-backed integration tests against real Postgres/Redis.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy --strict` on `src/cmp`).
5. Security scan passes (`bandit` — no high-severity findings; no plaintext PHI in logs).
6. Docker build succeeds and the relevant services come up healthy via docker-compose.
7. The feature works end-to-end (CLI/API and, where applicable, Playwright e2e).
8. New configuration options are documented in `.env.example` and README.
9. New or changed API endpoints appear correctly in the auto-generated OpenAPI 3.1 spec.
10. Database changes ship with an Alembic migration that applies and rolls back cleanly.
11. RLS and audit coverage verified for any new PHI-bearing table (tenant isolation test + audit test).
12. Any FHIR-facing output validates against the relevant `fhir.resources` R4 model.
```
