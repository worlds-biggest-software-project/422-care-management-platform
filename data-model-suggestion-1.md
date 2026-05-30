# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Care Management Platform (Candidate #422)

## Summary

A fully normalized relational data model built on PostgreSQL, following third normal form (3NF) for transactional operations with strategic denormalization for reporting workloads. This is the most conventional approach for healthcare applications, providing strong data integrity, ACID compliance, mature tooling, and well-understood patterns for HIPAA-compliant audit logging. The schema is designed around the core care management workflow: patients flow through risk stratification into disease management programs, where care teams create and execute care plans, close care gaps, and document outreach interactions.

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
Organization ──< Facility ──< CareTeam ──< CareTeamMember
                                              │
Patient ──< PatientIdentifier                 │
  │     ──< PatientDemographic                │
  │     ──< PatientConsent                    │
  │                                           │
  ├──< RiskScore ──> RiskModel                │
  │                                           │
  ├──< ProgramEnrollment ──> DiseaseProgram   │
  │       │                                   │
  │       └──< CarePlan                       │
  │              ├──< CarePlanGoal            │
  │              ├──< CarePlanIntervention     │
  │              └──< CarePlanReview          │
  │                                           │
  ├──< CareGap ──> QualityMeasure            │
  │       └──< CareGapAction                 │
  │                                           │
  ├──< OutreachAttempt ──> OutreachCampaign   │
  │                                           │
  ├──< CareTask ──────────────────────────────┘
  │
  ├──< Condition (clinical diagnoses)
  ├──< Observation (labs, vitals, screenings)
  ├──< Medication
  ├──< Encounter
  ├──< SdohScreening ──< SdohScreeningResponse
  │       └──< CommunityReferral
  └──< BillingTimeEntry (CCM time tracking)
```

### Core Schema Snippets

#### Master Patient Index

```sql
CREATE TABLE patient (
    patient_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mpi_id              UUID NOT NULL,           -- Master Patient Index ID
    mrn                 VARCHAR(50),
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    date_of_birth       DATE NOT NULL,
    sex                 VARCHAR(10),
    gender_identity     VARCHAR(50),
    preferred_language  VARCHAR(50),
    race                VARCHAR(50),
    ethnicity           VARCHAR(50),
    deceased_flag       BOOLEAN DEFAULT FALSE,
    deceased_date       DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_patient_mpi UNIQUE (mpi_id)
);

CREATE TABLE patient_identifier (
    identifier_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    identifier_type VARCHAR(50) NOT NULL,   -- MRN, SSN, INSURANCE_ID, etc.
    identifier_value VARCHAR(100) NOT NULL,
    source_system   VARCHAR(100) NOT NULL,
    assigning_authority VARCHAR(100),
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE patient_address (
    address_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    address_type    VARCHAR(20) NOT NULL,   -- HOME, WORK, TEMP
    line1           VARCHAR(200),
    line2           VARCHAR(200),
    city            VARCHAR(100),
    state           VARCHAR(50),
    postal_code     VARCHAR(20),
    county          VARCHAR(100),
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    period_start    DATE,
    period_end      DATE,
    active          BOOLEAN DEFAULT TRUE
);

CREATE TABLE patient_contact (
    contact_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    contact_type    VARCHAR(20) NOT NULL,   -- PHONE, EMAIL, SMS
    contact_value   VARCHAR(200) NOT NULL,
    preferred       BOOLEAN DEFAULT FALSE,
    active          BOOLEAN DEFAULT TRUE
);

CREATE TABLE patient_consent (
    consent_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    consent_type    VARCHAR(50) NOT NULL,   -- CCM_ENROLLMENT, SMS_OUTREACH, etc.
    status          VARCHAR(20) NOT NULL,   -- ACTIVE, REVOKED, EXPIRED
    granted_date    DATE NOT NULL,
    revoked_date    DATE,
    documented_by   UUID REFERENCES care_team_member(member_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Risk Stratification

```sql
CREATE TABLE risk_model (
    model_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name      VARCHAR(200) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,   -- RULE_BASED, ML_PREDICTIVE
    description     TEXT,
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_risk_model UNIQUE (model_name, model_version)
);

CREATE TABLE risk_score (
    score_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    model_id        UUID NOT NULL REFERENCES risk_model(model_id),
    score_value     NUMERIC(8,4) NOT NULL,
    risk_tier       VARCHAR(20) NOT NULL,   -- LOW, MODERATE, HIGH, VERY_HIGH
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_from      TIMESTAMPTZ NOT NULL,
    valid_to        TIMESTAMPTZ,
    CONSTRAINT uq_risk_score_current UNIQUE (patient_id, model_id, valid_from)
);

CREATE TABLE risk_score_factor (
    factor_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    score_id        UUID NOT NULL REFERENCES risk_score(score_id),
    factor_name     VARCHAR(200) NOT NULL,
    factor_value    VARCHAR(200),
    contribution    NUMERIC(8,4),           -- Weight/contribution to overall score
    description     TEXT
);
```

#### Disease Programs and Care Plans

```sql
CREATE TABLE disease_program (
    program_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    program_name    VARCHAR(200) NOT NULL,
    condition_code  VARCHAR(20),            -- ICD-10 or SNOMED code
    condition_system VARCHAR(50),           -- ICD10, SNOMED_CT
    description     TEXT,
    protocol_version VARCHAR(50),
    active          BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE program_enrollment (
    enrollment_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    program_id      UUID NOT NULL REFERENCES disease_program(program_id),
    status          VARCHAR(20) NOT NULL,   -- ENROLLED, COMPLETED, WITHDRAWN, GRADUATED
    enrolled_date   DATE NOT NULL,
    disenrolled_date DATE,
    enrolled_by     UUID REFERENCES care_team_member(member_id),
    disenroll_reason VARCHAR(200),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE care_plan (
    care_plan_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    enrollment_id   UUID REFERENCES program_enrollment(enrollment_id),
    title           VARCHAR(300) NOT NULL,
    status          VARCHAR(20) NOT NULL,   -- DRAFT, ACTIVE, COMPLETED, REVOKED
    category        VARCHAR(50),            -- DISEASE_MGMT, PREVENTIVE, COMPLEX_CASE
    period_start    DATE NOT NULL,
    period_end      DATE,
    authored_by     UUID REFERENCES care_team_member(member_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE care_plan_goal (
    goal_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    care_plan_id    UUID NOT NULL REFERENCES care_plan(care_plan_id),
    description     TEXT NOT NULL,
    target_value    VARCHAR(200),
    target_date     DATE,
    status          VARCHAR(20) NOT NULL,   -- PROPOSED, ACCEPTED, IN_PROGRESS, ACHIEVED, CANCELLED
    priority        VARCHAR(20),            -- HIGH, MEDIUM, LOW
    outcome_code    VARCHAR(20),            -- LOINC or custom code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE care_plan_intervention (
    intervention_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    care_plan_id    UUID NOT NULL REFERENCES care_plan(care_plan_id),
    goal_id         UUID REFERENCES care_plan_goal(goal_id),
    description     TEXT NOT NULL,
    intervention_type VARCHAR(50),          -- EDUCATION, MEDICATION, REFERRAL, SCREENING
    status          VARCHAR(20) NOT NULL,   -- PLANNED, IN_PROGRESS, COMPLETED, CANCELLED
    scheduled_date  DATE,
    completed_date  DATE,
    assigned_to     UUID REFERENCES care_team_member(member_id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Care Gaps and Quality Measures

```sql
CREATE TABLE quality_measure (
    measure_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    measure_code    VARCHAR(50) NOT NULL,
    measure_set     VARCHAR(20) NOT NULL,   -- HEDIS, MIPS, CUSTOM
    measure_name    VARCHAR(300) NOT NULL,
    description     TEXT,
    numerator_desc  TEXT,
    denominator_desc TEXT,
    measure_year    INTEGER,
    active          BOOLEAN DEFAULT TRUE,
    CONSTRAINT uq_quality_measure UNIQUE (measure_code, measure_set, measure_year)
);

CREATE TABLE care_gap (
    gap_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    measure_id      UUID NOT NULL REFERENCES quality_measure(measure_id),
    gap_type        VARCHAR(50) NOT NULL,   -- SCREENING, LAB, MEDICATION, VISIT
    description     TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL,   -- OPEN, IN_PROGRESS, CLOSED, EXCLUDED
    identified_date DATE NOT NULL,
    due_date        DATE,
    closed_date     DATE,
    closed_by       UUID REFERENCES care_team_member(member_id),
    source_system   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Outreach

```sql
CREATE TABLE outreach_campaign (
    campaign_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_name   VARCHAR(200) NOT NULL,
    campaign_type   VARCHAR(50) NOT NULL,   -- CARE_GAP, WELLNESS, ENROLLMENT, FOLLOW_UP
    target_criteria TEXT,
    start_date      DATE NOT NULL,
    end_date        DATE,
    status          VARCHAR(20) NOT NULL,   -- DRAFT, ACTIVE, PAUSED, COMPLETED
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE outreach_attempt (
    attempt_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    campaign_id     UUID REFERENCES outreach_campaign(campaign_id),
    channel         VARCHAR(20) NOT NULL,   -- PHONE, SMS, EMAIL, PORTAL, MAIL
    direction       VARCHAR(10) NOT NULL,   -- OUTBOUND, INBOUND
    status          VARCHAR(20) NOT NULL,   -- ATTEMPTED, REACHED, NO_ANSWER, LEFT_MESSAGE, DECLINED
    attempted_at    TIMESTAMPTZ NOT NULL,
    attempted_by    UUID REFERENCES care_team_member(member_id),
    duration_seconds INTEGER,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### SDOH Screening

```sql
CREATE TABLE sdoh_instrument (
    instrument_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_name VARCHAR(200) NOT NULL,  -- AHC-HRSN, PRAPARE
    version         VARCHAR(50),
    active          BOOLEAN DEFAULT TRUE
);

CREATE TABLE sdoh_screening (
    screening_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    instrument_id   UUID NOT NULL REFERENCES sdoh_instrument(instrument_id),
    screening_date  DATE NOT NULL,
    administered_by UUID REFERENCES care_team_member(member_id),
    total_score     NUMERIC(5,2),
    status          VARCHAR(20) NOT NULL,   -- COMPLETED, PARTIAL, DECLINED
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE sdoh_screening_response (
    response_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    screening_id    UUID NOT NULL REFERENCES sdoh_screening(screening_id),
    domain          VARCHAR(50) NOT NULL,   -- HOUSING, FOOD, TRANSPORTATION, UTILITIES, SAFETY
    question_code   VARCHAR(50) NOT NULL,
    answer_code     VARCHAR(50),
    answer_text     TEXT,
    need_identified BOOLEAN DEFAULT FALSE
);

CREATE TABLE community_referral (
    referral_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    screening_id    UUID REFERENCES sdoh_screening(screening_id),
    resource_name   VARCHAR(200) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,   -- FOOD_BANK, HOUSING_AID, TRANSPORTATION, etc.
    status          VARCHAR(20) NOT NULL,   -- REFERRED, ACCEPTED, COMPLETED, DECLINED, NOT_ELIGIBLE
    referred_date   DATE NOT NULL,
    outcome_date    DATE,
    outcome_notes   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### CCM Billing and Time Tracking

```sql
CREATE TABLE billing_time_entry (
    entry_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    patient_id      UUID NOT NULL REFERENCES patient(patient_id),
    care_plan_id    UUID REFERENCES care_plan(care_plan_id),
    member_id       UUID NOT NULL REFERENCES care_team_member(member_id),
    activity_type   VARCHAR(50) NOT NULL,   -- PHONE_CALL, CARE_PLAN_REVIEW, DOCUMENTATION, COORDINATION
    cpt_code        VARCHAR(10),            -- 99490, 99491, 99487, 99489
    duration_minutes INTEGER NOT NULL,
    service_date    DATE NOT NULL,
    billing_period  VARCHAR(7) NOT NULL,    -- YYYY-MM
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### Audit Trail

```sql
CREATE TABLE audit_log (
    audit_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,   -- INSERT, UPDATE, DELETE
    changed_by      UUID,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT
);
```

---

## Pros

- **Data integrity**: Foreign keys, constraints, and transactions enforce referential integrity across care plans, enrollments, risk scores, and care gaps -- critical for clinical accuracy.
- **HIPAA compliance**: Row-level security (RLS) in PostgreSQL natively supports role-based access control for PHI. Audit logging via triggers is well-understood.
- **Mature ecosystem**: PostgreSQL has decades of production use in healthcare. ORM support (SQLAlchemy, Prisma, TypeORM), migration tools (Flyway, Alembic), and backup/restore workflows are robust.
- **Complex query support**: Joins across care plans, risk scores, quality measures, and outreach are natural in SQL. Reporting queries for HEDIS/MIPS dashboards are straightforward.
- **Temporal queries**: Using valid_from/valid_to columns on risk scores and other time-varying data supports point-in-time queries without separate temporal infrastructure.
- **Standards alignment**: Schema maps cleanly to FHIR resources (Patient, CarePlan, Goal, Condition, Observation) for interoperability.
- **Hiring and team scalability**: PostgreSQL is the most widely known open-source RDBMS. Finding developers is straightforward.

## Cons

- **Schema rigidity**: Adding new SDOH domains, disease programs, or quality measures requires DDL changes and migrations. Healthcare requirements change frequently.
- **EHR data impedance mismatch**: FHIR resources are deeply nested JSON; flattening them into normalized tables requires a complex ETL layer with ongoing maintenance as FHIR profiles evolve.
- **Query performance at scale**: Joins across 20+ tables for population-level analytics (e.g., "all diabetic patients with open HbA1c gaps and high SDOH need scores") can become expensive at 500K+ patient populations without careful indexing and query optimization.
- **Document-like data**: SDOH screening responses, care plan narratives, and clinical notes fit awkwardly in rigid relational columns. Many fields end up as TEXT or require supplementary tables.
- **Versioning complexity**: Tracking historical versions of care plans, risk scores, and quality measures requires either temporal tables (PostgreSQL has no native bitemporal support) or manual versioning patterns that add schema complexity.
- **Reporting layer needed**: Transactional 3NF schema is suboptimal for analytical dashboards. A separate reporting layer (materialized views, data warehouse, or OLAP) is typically needed.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with pg_partman for time-series partitioning |
| ORM | SQLAlchemy (Python) or Prisma (Node.js/TypeScript) |
| Migrations | Flyway or Alembic with version-controlled migration scripts |
| Audit | pg_audit extension + trigger-based audit_log table |
| Row-Level Security | PostgreSQL RLS policies per organization/care team |
| Search | pg_trgm extension for patient name fuzzy matching; full-text search for care plan narratives |
| Reporting | Materialized views for quality measure dashboards; consider dbt for transformation layer |
| Backup | pg_basebackup with WAL archiving for point-in-time recovery |

---

## Migration and Scaling Considerations

- **Partitioning**: Partition audit_log, outreach_attempt, and billing_time_entry by month using PostgreSQL declarative partitioning. Risk scores can be partitioned by calculated_at.
- **Read replicas**: Use streaming replication to offload reporting queries to read replicas, keeping the primary database responsive for care coordinator workflows.
- **Connection pooling**: PgBouncer or pgpool-II to manage connection limits, especially with many concurrent care coordinator sessions.
- **Indexing strategy**: Composite indexes on (patient_id, status) for care gaps, (patient_id, model_id, valid_from) for risk scores, and (patient_id, billing_period) for CCM time entries. GIN indexes on any JSONB columns used for flexible attributes.
- **Multi-tenancy**: Use a tenant_id (organization_id) column on all tables with RLS policies for multi-tenant deployments serving multiple health systems.
- **Data growth**: At 500K patients with 5 years of history, expect ~50M rows in observation, ~10M in outreach_attempt, ~5M in care_gap. PostgreSQL handles this scale well with proper indexing and partitioning.
- **FHIR ETL**: Build a dedicated ingest pipeline that transforms incoming FHIR bundles into normalized rows. Use staging tables to validate data before inserting into production tables. Consider a FHIR facade layer that reconstructs FHIR resources from normalized tables for API responses.
