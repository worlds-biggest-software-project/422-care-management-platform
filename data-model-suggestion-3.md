# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Care Management Platform (Candidate #422)

## Summary

A pragmatic hybrid architecture that uses PostgreSQL as a single database engine, combining traditional normalized relational tables for core entities with stable schemas (patients, organizations, care teams, quality measures) and PostgreSQL JSONB columns for entities with variable, evolving, or deeply nested structures (FHIR resource payloads, SDOH screening responses, risk score factor breakdowns, clinical observations, care plan narratives). This approach directly addresses the fundamental tension in healthcare data modeling: the need for referential integrity and SQL queryability on structured care management workflows, while accommodating the highly variable and rapidly evolving clinical data that arrives in FHIR JSON format from EHRs, claims systems, and HIEs.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    PostgreSQL 16+                             │
│                                                              │
│  ┌─────────────────────┐    ┌─────────────────────────────┐  │
│  │  Relational Core    │    │  JSONB Document Layer        │  │
│  │  (3NF, FK, indexes) │    │  (flexible, schema-on-read) │  │
│  │                     │    │                             │  │
│  │  - patient          │    │  - fhir_resource            │  │
│  │  - organization     │    │  - clinical_observation     │  │
│  │  - care_team        │    │  - sdoh_response_data       │  │
│  │  - disease_program  │    │  - risk_factor_details      │  │
│  │  - quality_measure  │    │  - care_plan_narrative      │  │
│  │  - care_gap         │    │  - outreach_ml_features     │  │
│  │  - program_enroll   │    │  - external_claims_data     │  │
│  │  - outreach_attempt │    │  - integration_payload      │  │
│  │  - billing_entry    │    │                             │  │
│  └─────────┬───────────┘    └──────────┬──────────────────┘  │
│            │          JOIN / reference  │                     │
│            └───────────────────────────┘                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────────┐│
│  │  Materialized Views (reporting & dashboards)             ││
│  │  - mv_quality_measure_performance                        ││
│  │  - mv_population_risk_summary                            ││
│  │  - mv_coordinator_worklist                               ││
│  │  - mv_ccm_billing_summary                                ││
│  └──────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

---

## Key Entities and Relationships

### Design Principle: Relational Where Stable, JSONB Where Variable

The split follows a clear rule:
- **Relational columns**: Fields that are queried frequently, used in JOINs, participate in foreign keys, or must be constrained (patient demographics, care gap status, enrollment dates, billing codes).
- **JSONB columns**: Fields that vary by source system, change shape over time, arrive as nested JSON from FHIR APIs, or contain complex structures that would require dozens of auxiliary tables if fully normalized.

### Core Schema Snippets

#### Patient (Hybrid: Relational Core + JSONB Extensions)

```sql
CREATE TABLE patient (
    patient_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mpi_id              UUID NOT NULL UNIQUE,
    mrn                 VARCHAR(50),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    date_of_birth       DATE NOT NULL,
    sex                 VARCHAR(10),
    preferred_language  VARCHAR(50),
    race                VARCHAR(50),
    ethnicity           VARCHAR(50),
    deceased_flag       BOOLEAN DEFAULT FALSE,
    tenant_id           UUID NOT NULL,
    -- JSONB for variable data
    identifiers         JSONB DEFAULT '[]',     -- Array of {type, value, source, authority}
    addresses           JSONB DEFAULT '[]',     -- Array of {type, line1, city, state, zip, geo}
    contacts            JSONB DEFAULT '[]',     -- Array of {type, value, preferred}
    extensions          JSONB DEFAULT '{}',     -- Arbitrary extensions per deployment
    fhir_resource       JSONB,                  -- Complete FHIR Patient resource as-received
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- GIN index on JSONB for flexible querying
CREATE INDEX idx_patient_identifiers ON patient USING GIN (identifiers);
CREATE INDEX idx_patient_extensions ON patient USING GIN (extensions);

-- Functional index for searching within JSONB
CREATE INDEX idx_patient_zip ON patient 
    ((addresses->0->>'postal_code')) WHERE addresses IS NOT NULL;
```

#### FHIR Resource Store (Document Layer)

```sql
-- Stores raw FHIR resources as-received from EHRs and HIEs
CREATE TABLE fhir_resource (
    resource_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fhir_id             VARCHAR(200),           -- FHIR resource.id
    resource_type       VARCHAR(50) NOT NULL,    -- Patient, Condition, Observation, etc.
    patient_id          UUID REFERENCES patient(patient_id),
    source_system       VARCHAR(100) NOT NULL,
    resource_data       JSONB NOT NULL,          -- Complete FHIR resource
    version_id          INTEGER DEFAULT 1,
    last_updated        TIMESTAMPTZ,
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_fhir_resource UNIQUE (fhir_id, resource_type, source_system)
);

-- GIN index for querying any element within FHIR resources
CREATE INDEX idx_fhir_resource_data ON fhir_resource USING GIN (resource_data);

-- Partial indexes for common FHIR queries
CREATE INDEX idx_fhir_conditions ON fhir_resource (patient_id, last_updated)
    WHERE resource_type = 'Condition';
CREATE INDEX idx_fhir_observations ON fhir_resource (patient_id, last_updated)
    WHERE resource_type = 'Observation';
CREATE INDEX idx_fhir_medications ON fhir_resource (patient_id, last_updated)
    WHERE resource_type = 'MedicationRequest';
```

#### Clinical Observations (Hybrid)

```sql
-- Extracted and indexed clinical data from FHIR Observations
CREATE TABLE clinical_observation (
    observation_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(patient_id),
    fhir_resource_id    UUID REFERENCES fhir_resource(resource_id),
    observation_type    VARCHAR(50) NOT NULL,    -- LAB, VITAL, SCREENING, ASSESSMENT
    loinc_code          VARCHAR(20),
    snomed_code         VARCHAR(20),
    display_name        VARCHAR(200),
    -- Relational for queryable values
    value_numeric       NUMERIC(12,4),
    value_text          VARCHAR(500),
    value_unit          VARCHAR(50),
    effective_date      TIMESTAMPTZ NOT NULL,
    status              VARCHAR(20) NOT NULL,    -- FINAL, PRELIMINARY, AMENDED
    -- JSONB for variable components (reference ranges, interpretations, components)
    components          JSONB,                   -- Multi-component observations (e.g., BP systolic/diastolic)
    reference_range     JSONB,                   -- {low, high, unit, interpretation}
    interpretation      JSONB,                   -- Array of interpretation codes
    source_system       VARCHAR(100),
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clinical_obs_patient_type 
    ON clinical_observation (patient_id, observation_type, effective_date DESC);
CREATE INDEX idx_clinical_obs_loinc 
    ON clinical_observation (loinc_code, effective_date DESC);
```

#### Risk Stratification (Hybrid)

```sql
CREATE TABLE risk_score (
    score_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(patient_id),
    model_name          VARCHAR(200) NOT NULL,
    model_version       VARCHAR(50) NOT NULL,
    -- Relational for quick filtering and sorting
    score_value         NUMERIC(8,4) NOT NULL,
    risk_tier           VARCHAR(20) NOT NULL,
    calculated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- JSONB for variable factor breakdowns per model
    factors             JSONB NOT NULL,          -- [{name, value, contribution, description}]
    model_metadata      JSONB,                   -- Model parameters, thresholds, version info
    equity_metrics      JSONB,                   -- Bias assessment results per demographic group
    tenant_id           UUID NOT NULL
);

CREATE INDEX idx_risk_score_patient 
    ON risk_score (patient_id, calculated_at DESC);
CREATE INDEX idx_risk_score_tier 
    ON risk_score (tenant_id, risk_tier, calculated_at DESC);
-- GIN index for querying within factor breakdowns
CREATE INDEX idx_risk_score_factors 
    ON risk_score USING GIN (factors);
```

#### Care Plan (Hybrid)

```sql
CREATE TABLE care_plan (
    care_plan_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(patient_id),
    enrollment_id       UUID REFERENCES program_enrollment(enrollment_id),
    -- Relational for workflow queries
    title               VARCHAR(300) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    category            VARCHAR(50),
    period_start        DATE NOT NULL,
    period_end          DATE,
    authored_by         UUID,
    -- JSONB for flexible, deeply nested care plan content
    goals               JSONB DEFAULT '[]',      -- [{id, description, target, status, priority, milestones[]}]
    interventions       JSONB DEFAULT '[]',      -- [{id, goal_id, description, type, status, assigned_to, schedule}]
    problem_list        JSONB DEFAULT '[]',      -- [{code, system, display, onset_date, status}]
    patient_preferences JSONB DEFAULT '{}',      -- Communication preferences, language, cultural considerations
    narrative           JSONB DEFAULT '{}',      -- Rich-text care plan narrative sections
    review_history      JSONB DEFAULT '[]',      -- [{date, reviewer, notes, changes_made}]
    fhir_care_plan      JSONB,                   -- FHIR CarePlan resource for interoperability
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_care_plan_patient 
    ON care_plan (patient_id, status);
-- GIN index for searching within goals and interventions
CREATE INDEX idx_care_plan_goals 
    ON care_plan USING GIN (goals);
CREATE INDEX idx_care_plan_interventions 
    ON care_plan USING GIN (interventions);
```

#### Care Gaps and Quality Measures (Primarily Relational)

```sql
CREATE TABLE quality_measure (
    measure_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    measure_code        VARCHAR(50) NOT NULL,
    measure_set         VARCHAR(20) NOT NULL,
    measure_name        VARCHAR(300) NOT NULL,
    measure_year        INTEGER,
    -- JSONB for complex measure logic that varies year to year
    numerator_logic     JSONB,                   -- Structured definition of numerator criteria
    denominator_logic   JSONB,                   -- Structured definition of denominator criteria
    exclusion_logic     JSONB,                   -- Structured definition of exclusion criteria
    metadata            JSONB DEFAULT '{}',
    active              BOOLEAN DEFAULT TRUE,
    CONSTRAINT uq_quality_measure UNIQUE (measure_code, measure_set, measure_year)
);

CREATE TABLE care_gap (
    gap_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(patient_id),
    measure_id          UUID NOT NULL REFERENCES quality_measure(measure_id),
    gap_type            VARCHAR(50) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    identified_date     DATE NOT NULL,
    due_date            DATE,
    closed_date         DATE,
    closed_by           UUID,
    -- JSONB for gap-specific evidence and actions
    evidence            JSONB DEFAULT '{}',      -- Supporting data for gap identification
    actions_taken       JSONB DEFAULT '[]',      -- [{date, action_type, performed_by, notes, outcome}]
    closure_evidence    JSONB,                   -- Data proving gap was closed
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_care_gap_patient_status 
    ON care_gap (patient_id, status);
CREATE INDEX idx_care_gap_measure 
    ON care_gap (measure_id, status, tenant_id);
```

#### SDOH Screening (Hybrid)

```sql
CREATE TABLE sdoh_screening (
    screening_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(patient_id),
    instrument_name     VARCHAR(100) NOT NULL,   -- AHC-HRSN, PRAPARE
    instrument_version  VARCHAR(50),
    screening_date      DATE NOT NULL,
    administered_by     UUID,
    status              VARCHAR(20) NOT NULL,
    total_score         NUMERIC(5,2),
    -- JSONB for variable screening responses (different instruments have different questions)
    responses           JSONB NOT NULL,          -- [{domain, question_code, question_text, answer, score, need_identified}]
    identified_needs    JSONB DEFAULT '[]',      -- [{domain, severity, description}]
    -- JSONB for referral tracking within the screening context
    referrals           JSONB DEFAULT '[]',      -- [{resource_name, type, status, referred_date, outcome}]
    fhir_questionnaire_response JSONB,           -- FHIR QuestionnaireResponse resource
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sdoh_patient 
    ON sdoh_screening (patient_id, screening_date DESC);
-- GIN index to query identified needs across patients
CREATE INDEX idx_sdoh_needs 
    ON sdoh_screening USING GIN (identified_needs);
```

#### Outreach (Primarily Relational)

```sql
CREATE TABLE outreach_attempt (
    attempt_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(patient_id),
    campaign_id         UUID REFERENCES outreach_campaign(campaign_id),
    channel             VARCHAR(20) NOT NULL,
    direction           VARCHAR(10) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    attempted_at        TIMESTAMPTZ NOT NULL,
    attempted_by        UUID,
    duration_seconds    INTEGER,
    notes               TEXT,
    -- JSONB for ML features and outreach optimization data
    ml_features         JSONB,                   -- {predicted_best_time, channel_preference_score, engagement_probability}
    response_data       JSONB,                   -- {sentiment, topics_discussed, follow_up_needed, patient_reported_barriers}
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### CCM Billing (Primarily Relational)

```sql
CREATE TABLE billing_time_entry (
    entry_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id          UUID NOT NULL REFERENCES patient(patient_id),
    care_plan_id        UUID REFERENCES care_plan(care_plan_id),
    member_id           UUID NOT NULL,
    activity_type       VARCHAR(50) NOT NULL,
    cpt_code            VARCHAR(10),
    duration_minutes    INTEGER NOT NULL,
    service_date        DATE NOT NULL,
    billing_period      VARCHAR(7) NOT NULL,
    notes               TEXT,
    -- JSONB for compliance documentation
    compliance_data     JSONB DEFAULT '{}',      -- {care_plan_elements_addressed[], team_composition_documented, consent_verified}
    tenant_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Materialized Views for Reporting

```sql
-- Quality measure performance dashboard
CREATE MATERIALIZED VIEW mv_quality_measure_performance AS
SELECT
    qm.measure_code,
    qm.measure_name,
    qm.measure_set,
    cg.tenant_id,
    COUNT(*) FILTER (WHERE cg.status IN ('OPEN', 'IN_PROGRESS', 'CLOSED')) AS denominator,
    COUNT(*) FILTER (WHERE cg.status = 'CLOSED') AS numerator,
    COUNT(*) FILTER (WHERE cg.status = 'EXCLUDED') AS exclusions,
    ROUND(
        COUNT(*) FILTER (WHERE cg.status = 'CLOSED')::NUMERIC / 
        NULLIF(COUNT(*) FILTER (WHERE cg.status IN ('OPEN', 'IN_PROGRESS', 'CLOSED')), 0) * 100, 2
    ) AS performance_rate,
    DATE_TRUNC('month', CURRENT_DATE)::DATE AS reporting_month
FROM care_gap cg
JOIN quality_measure qm ON cg.measure_id = qm.measure_id
GROUP BY qm.measure_code, qm.measure_name, qm.measure_set, cg.tenant_id;

-- Coordinator worklist with aggregated patient data
CREATE MATERIALIZED VIEW mv_coordinator_worklist AS
SELECT
    p.patient_id,
    p.first_name || ' ' || p.last_name AS patient_name,
    rs.risk_tier,
    rs.score_value AS current_risk_score,
    (SELECT COUNT(*) FROM care_gap cg WHERE cg.patient_id = p.patient_id AND cg.status = 'OPEN') AS open_gaps,
    (SELECT MAX(attempted_at) FROM outreach_attempt oa WHERE oa.patient_id = p.patient_id) AS last_outreach,
    (SELECT SUM(duration_minutes) FROM billing_time_entry b 
     WHERE b.patient_id = p.patient_id AND b.billing_period = TO_CHAR(CURRENT_DATE, 'YYYY-MM')) AS ccm_minutes_this_month,
    cp.status AS care_plan_status,
    p.tenant_id
FROM patient p
LEFT JOIN LATERAL (
    SELECT score_value, risk_tier FROM risk_score 
    WHERE patient_id = p.patient_id ORDER BY calculated_at DESC LIMIT 1
) rs ON TRUE
LEFT JOIN LATERAL (
    SELECT status FROM care_plan 
    WHERE patient_id = p.patient_id AND status = 'ACTIVE' ORDER BY updated_at DESC LIMIT 1
) cp ON TRUE;
```

---

## Pros

- **Single database**: PostgreSQL handles both relational and document workloads. No need to synchronize between separate relational and document databases. Operational simplicity is enormous for small-to-medium deployments targeting FQHCs and community health centers.
- **FHIR native storage**: Incoming FHIR resources can be stored as-is in JSONB without a complex ETL flattening step. Key fields are extracted to relational columns for indexing; the full resource is preserved for interoperability and audit.
- **Schema evolution without migrations**: Adding a new SDOH screening instrument, new risk factor types, or new outreach channels requires no DDL changes -- just new keys in JSONB columns. Quality measure logic can be expressed as structured JSON that evolves independently of the schema.
- **Query flexibility**: GIN indexes on JSONB enable queries like "find all patients with identified food insecurity needs" across variable-structure screening responses, without predicting every possible query pattern at design time.
- **Relational integrity where it matters**: Foreign keys between patients, care plans, care gaps, and quality measures prevent orphaned records. Enrollment status transitions are constrained. Billing entries reference valid care plans.
- **PostgreSQL ecosystem**: All the benefits of PostgreSQL (ACID, RLS, pg_audit, replication, partitioning) apply to both relational and JSONB data in a single database.
- **Gradual normalization**: Start with JSONB for uncertain structures; extract high-value fields to relational columns as query patterns stabilize. This is a low-risk evolutionary approach.
- **FHIR facade**: The fhir_resource table directly supports a FHIR-compliant REST API -- store resources as JSONB, serve them back as JSON. No serialization/deserialization overhead.

## Cons

- **JSONB query performance**: While GIN indexes help, complex queries within deeply nested JSONB are slower than equivalent relational queries. Population-level analytics that filter on JSONB fields across 500K+ patients require careful index design.
- **No JSONB foreign keys**: Relationships expressed within JSONB (e.g., a goal_id referenced in an intervention within the same care plan JSONB column) cannot be enforced by the database. Application-layer validation is required.
- **Schema discipline required**: The flexibility of JSONB can lead to inconsistent data if not governed by application-layer validation (JSON Schema, Zod, or similar). "Schema-on-read" means bad data can silently enter the system.
- **Indexing complexity**: GIN indexes on large JSONB columns can become expensive to maintain. Functional indexes on specific JSONB paths provide better performance but require knowing query patterns in advance.
- **ORM friction**: ORMs (SQLAlchemy, Prisma) handle JSONB less elegantly than relational columns. Custom query builders or raw SQL are often needed for JSONB operations.
- **Reporting limitations**: Materialized views that join relational and JSONB data are more complex to write and maintain. Business intelligence tools (Metabase, Superset) have varying levels of JSONB support.
- **Data validation gap**: Unlike relational CHECK constraints and NOT NULL, JSONB content validation depends entirely on application code. A missing field in a care plan's goals JSONB goes undetected by the database.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ (native JSONB, GIN indexes, jsonpath queries) |
| JSONB Validation | JSON Schema validation at application layer; pg_jsonschema extension for database-level validation |
| ORM | Prisma (good JSONB support in recent versions) or SQLAlchemy with custom JSONB column types |
| FHIR Facade | HAPI FHIR server backed by PostgreSQL JSONB store; or custom REST API using FHIR resource types |
| Search | PostgreSQL full-text search for care plan narratives; GIN indexes for JSONB path queries |
| Migrations | Flyway or Alembic for relational schema changes; JSONB structure changes managed via application versioning |
| Analytics | dbt with PostgreSQL adapter for JSONB extraction and transformation; materialized views for dashboards |
| Monitoring | pg_stat_statements for query performance; monitor GIN index sizes |

---

## Migration and Scaling Considerations

- **Incremental FHIR ingestion**: When a new EHR integration is added, store raw FHIR bundles in fhir_resource immediately. Extract key relational fields (conditions, observations) into dedicated tables asynchronously. The JSONB store acts as a staging area that is always queryable.
- **JSONB to relational extraction**: As query patterns stabilize, extract high-frequency JSONB fields to dedicated relational columns. For example, if "food insecurity" becomes a frequent filter, add a `food_insecurity_flag BOOLEAN` column derived from SDOH screening JSONB. Use generated columns or triggers to keep in sync.
- **Partitioning strategy**: Partition fhir_resource by resource_type (list partitioning) and clinical_observation by effective_date (range partitioning). Partition billing_time_entry by billing_period.
- **Read replica offloading**: Use streaming replication to offload materialized view refreshes and analytics queries to read replicas.
- **JSONB compression**: PostgreSQL TOAST automatically compresses large JSONB values. For very large FHIR resources (e.g., Bundle resources with hundreds of entries), consider storing only the relevant extracted resources rather than the full bundle.
- **Multi-tenancy**: tenant_id on all tables with RLS policies. JSONB data inherits tenant isolation through the same row-level security.
- **Scaling thresholds**: This approach works well up to ~2M patients on a single PostgreSQL instance with proper indexing. Beyond that, consider Citus for horizontal sharding or migrating analytics workloads to a dedicated OLAP system (ClickHouse, BigQuery).
- **FHIR API compatibility**: The fhir_resource table enables a FHIR-compliant search API using PostgreSQL jsonpath queries, supporting _has, _include, and chained search parameters directly against JSONB data.
