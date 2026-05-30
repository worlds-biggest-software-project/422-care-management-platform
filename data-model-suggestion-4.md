# Data Model Suggestion 4: OMOP CDM + Knowledge Graph Hybrid

> Project: Care Management Platform (Candidate #422)

## Summary

A domain-specific hybrid architecture combining the OHDSI OMOP Common Data Model (CDM) for standardized clinical and claims data analytics with a property graph layer (Neo4j or Apache AGE on PostgreSQL) for care coordination relationships, patient journey mapping, and social determinants network analysis. The OMOP CDM provides a proven, community-maintained schema designed explicitly for observational health data -- with standardized vocabularies (SNOMED CT, LOINC, ICD-10, RxNorm) and a rich ecosystem of open-source analytics tools. The graph layer models the inherently relationship-heavy aspects of care management: care team coordination, patient-provider networks, community resource referral chains, family relationships, and SDOH social networks. This approach is uniquely suited to a care management platform because it separates two fundamentally different data problems: population-level clinical analytics (OMOP's strength) and relationship-intensive care coordination (graph's strength).

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Data Ingestion Layer                       │
│   FHIR Bundles ──> ETL Pipeline ──> OMOP CDM + Graph Loader    │
│   Claims Data  ──>                                              │
│   SDOH Data    ──>                                              │
└──────────────────────┬──────────────────────┬───────────────────┘
                       │                      │
          ┌────────────▼────────────┐  ┌──────▼──────────────────┐
          │    OMOP CDM Layer       │  │   Graph Layer            │
          │    (PostgreSQL)         │  │   (Neo4j or Apache AGE)  │
          │                        │  │                          │
          │  - person              │  │  Nodes:                  │
          │  - condition_occurrence│  │  - Patient               │
          │  - drug_exposure       │  │  - CareTeamMember        │
          │  - procedure_occurrence│  │  - CareGap               │
          │  - measurement         │  │  - CarePlan              │
          │  - observation         │  │  - DiseaseProgram        │
          │  - visit_occurrence    │  │  - CommunityResource     │
          │  - payer_plan_period   │  │  - QualityMeasure        │
          │  - cost                │  │  - Organization          │
          │  - concept             │  │  - SdohNeed              │
          │  - concept_relationship│  │                          │
          │                        │  │  Relationships:          │
          │  + Care Management     │  │  - MANAGED_BY            │
          │    Extension Tables    │  │  - ENROLLED_IN           │
          │  - care_plan           │  │  - HAS_GAP              │
          │  - care_gap            │  │  - REFERRED_TO           │
          │  - outreach_attempt    │  │  - FAMILY_OF             │
          │  - program_enrollment  │  │  - COORDINATES_WITH      │
          │  - risk_score          │  │  - SCREENED_POSITIVE_FOR │
          │  - billing_time_entry  │  │  - LIVES_IN (census)     │
          │  - sdoh_screening      │  │  - OUTREACH_SEQUENCE     │
          └────────────────────────┘  └──────────────────────────┘
                       │                      │
          ┌────────────▼──────────────────────▼───────────────────┐
          │                  API / Query Layer                     │
          │  - OHDSI ATLAS (cohort builder, analytics)            │
          │  - Cypher/GQL queries (care coordination)             │
          │  - REST/GraphQL API (application)                     │
          │  - FHIR facade (interoperability)                     │
          └───────────────────────────────────────────────────────┘
```

---

## Key Entities and Relationships

### OMOP CDM Core Tables (Standardized Clinical Data)

The OMOP CDM v5.4 provides ~30 standardized tables. The most relevant for care management are:

```sql
-- OMOP CDM Person table (patient demographics)
CREATE TABLE person (
    person_id                   BIGINT PRIMARY KEY,
    gender_concept_id           INTEGER NOT NULL,       -- OMOP concept for sex/gender
    year_of_birth               INTEGER NOT NULL,
    month_of_birth              INTEGER,
    day_of_birth                INTEGER,
    birth_datetime              TIMESTAMP,
    race_concept_id             INTEGER NOT NULL,
    ethnicity_concept_id        INTEGER NOT NULL,
    location_id                 BIGINT,
    provider_id                 BIGINT,
    care_site_id                BIGINT,
    person_source_value         VARCHAR(50),            -- MRN or source identifier
    gender_source_value         VARCHAR(50),
    race_source_value           VARCHAR(50),
    ethnicity_source_value      VARCHAR(50)
);

-- Conditions (diagnoses) -- drives disease registry membership
CREATE TABLE condition_occurrence (
    condition_occurrence_id     BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    condition_concept_id        INTEGER NOT NULL,       -- SNOMED CT concept
    condition_start_date        DATE NOT NULL,
    condition_start_datetime    TIMESTAMP,
    condition_end_date          DATE,
    condition_end_datetime      TIMESTAMP,
    condition_type_concept_id   INTEGER NOT NULL,       -- Source type (EHR, claim, etc.)
    condition_status_concept_id INTEGER,
    stop_reason                 VARCHAR(20),
    provider_id                 BIGINT,
    visit_occurrence_id         BIGINT,
    condition_source_value      VARCHAR(50),            -- ICD-10 code as received
    condition_source_concept_id INTEGER
);

-- Measurements (labs, vitals) -- drives care gap detection
CREATE TABLE measurement (
    measurement_id              BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    measurement_concept_id      INTEGER NOT NULL,       -- LOINC concept
    measurement_date            DATE NOT NULL,
    measurement_datetime        TIMESTAMP,
    measurement_type_concept_id INTEGER NOT NULL,
    operator_concept_id         INTEGER,
    value_as_number             NUMERIC,
    value_as_concept_id         INTEGER,
    unit_concept_id             INTEGER,
    range_low                   NUMERIC,
    range_high                  NUMERIC,
    provider_id                 BIGINT,
    visit_occurrence_id         BIGINT,
    measurement_source_value    VARCHAR(50),            -- LOINC code as received
    unit_source_value           VARCHAR(50),
    value_source_value          VARCHAR(50)
);

-- Drug exposures -- drives medication adherence tracking
CREATE TABLE drug_exposure (
    drug_exposure_id            BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    drug_concept_id             INTEGER NOT NULL,       -- RxNorm concept
    drug_exposure_start_date    DATE NOT NULL,
    drug_exposure_start_datetime TIMESTAMP,
    drug_exposure_end_date      DATE,
    drug_exposure_end_datetime  TIMESTAMP,
    drug_type_concept_id        INTEGER NOT NULL,
    stop_reason                 VARCHAR(20),
    refills                     INTEGER,
    quantity                    NUMERIC,
    days_supply                 INTEGER,
    sig                         TEXT,
    route_concept_id            INTEGER,
    lot_number                  VARCHAR(50),
    provider_id                 BIGINT,
    visit_occurrence_id         BIGINT,
    drug_source_value           VARCHAR(50),
    drug_source_concept_id      INTEGER,
    route_source_value          VARCHAR(50),
    dose_unit_source_value      VARCHAR(50)
);

-- Visit occurrences -- encounter history
CREATE TABLE visit_occurrence (
    visit_occurrence_id         BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    visit_concept_id            INTEGER NOT NULL,       -- Inpatient, outpatient, ED, etc.
    visit_start_date            DATE NOT NULL,
    visit_start_datetime        TIMESTAMP,
    visit_end_date              DATE,
    visit_end_datetime          TIMESTAMP,
    visit_type_concept_id       INTEGER NOT NULL,
    provider_id                 BIGINT,
    care_site_id                BIGINT,
    visit_source_value          VARCHAR(50),
    admitted_from_concept_id    INTEGER,
    discharged_to_concept_id    INTEGER,
    preceding_visit_occurrence_id BIGINT
);

-- Payer plan periods -- insurance coverage and attribution
CREATE TABLE payer_plan_period (
    payer_plan_period_id        BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    payer_plan_period_start_date DATE NOT NULL,
    payer_plan_period_end_date  DATE NOT NULL,
    payer_concept_id            INTEGER,
    payer_source_value          VARCHAR(50),
    plan_concept_id             INTEGER,
    plan_source_value           VARCHAR(50),
    sponsor_concept_id          INTEGER,
    sponsor_source_value        VARCHAR(50),
    family_source_value         VARCHAR(50),
    stop_reason_concept_id      INTEGER,
    stop_reason_source_value    VARCHAR(50)
);

-- OMOP Concept table (standardized vocabulary)
CREATE TABLE concept (
    concept_id                  INTEGER PRIMARY KEY,
    concept_name                VARCHAR(255) NOT NULL,
    domain_id                   VARCHAR(20) NOT NULL,
    vocabulary_id               VARCHAR(20) NOT NULL,   -- SNOMED, LOINC, ICD10CM, RxNorm, etc.
    concept_class_id            VARCHAR(20) NOT NULL,
    standard_concept            CHAR(1),
    concept_code                VARCHAR(50) NOT NULL,
    valid_start_date            DATE NOT NULL,
    valid_end_date              DATE NOT NULL,
    invalid_reason              CHAR(1)
);

CREATE TABLE concept_relationship (
    concept_id_1                INTEGER NOT NULL,
    concept_id_2                INTEGER NOT NULL,
    relationship_id             VARCHAR(20) NOT NULL,
    valid_start_date            DATE NOT NULL,
    valid_end_date              DATE NOT NULL,
    invalid_reason              CHAR(1)
);
```

### Care Management Extension Tables (Beyond Standard OMOP)

```sql
-- Risk stratification (extends OMOP with care management specifics)
CREATE TABLE cm_risk_score (
    risk_score_id               BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    model_name                  VARCHAR(200) NOT NULL,
    model_version               VARCHAR(50) NOT NULL,
    score_value                 NUMERIC(8,4) NOT NULL,
    risk_tier                   VARCHAR(20) NOT NULL,
    calculated_at               TIMESTAMP NOT NULL,
    factors                     JSONB,
    equity_metrics              JSONB
);

-- Care plans
CREATE TABLE cm_care_plan (
    care_plan_id                BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    enrollment_id               BIGINT REFERENCES cm_program_enrollment(enrollment_id),
    title                       VARCHAR(300) NOT NULL,
    status                      VARCHAR(20) NOT NULL,
    category                    VARCHAR(50),
    period_start                DATE NOT NULL,
    period_end                  DATE,
    authored_by_provider_id     BIGINT REFERENCES provider(provider_id),
    goals                       JSONB DEFAULT '[]',
    interventions               JSONB DEFAULT '[]',
    problem_list                JSONB DEFAULT '[]',
    review_history              JSONB DEFAULT '[]'
);

-- Program enrollment
CREATE TABLE cm_program_enrollment (
    enrollment_id               BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    program_name                VARCHAR(200) NOT NULL,
    condition_concept_id        INTEGER REFERENCES concept(concept_id),
    status                      VARCHAR(20) NOT NULL,
    enrolled_date               DATE NOT NULL,
    disenrolled_date            DATE,
    enrolled_by_provider_id     BIGINT REFERENCES provider(provider_id)
);

-- Care gaps linked to quality measures
CREATE TABLE cm_care_gap (
    care_gap_id                 BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    measure_concept_id          INTEGER,                -- OMOP concept for the quality measure
    measure_code                VARCHAR(50) NOT NULL,
    measure_set                 VARCHAR(20) NOT NULL,
    gap_type                    VARCHAR(50) NOT NULL,
    status                      VARCHAR(20) NOT NULL,
    identified_date             DATE NOT NULL,
    due_date                    DATE,
    closed_date                 DATE,
    evidence                    JSONB DEFAULT '{}',
    actions_taken               JSONB DEFAULT '[]'
);

-- Outreach attempts
CREATE TABLE cm_outreach_attempt (
    outreach_id                 BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    channel                     VARCHAR(20) NOT NULL,
    direction                   VARCHAR(10) NOT NULL,
    status                      VARCHAR(20) NOT NULL,
    attempted_at                TIMESTAMP NOT NULL,
    attempted_by_provider_id    BIGINT REFERENCES provider(provider_id),
    duration_seconds            INTEGER,
    notes                       TEXT,
    ml_features                 JSONB
);

-- SDOH screening
CREATE TABLE cm_sdoh_screening (
    screening_id                BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    instrument_name             VARCHAR(100) NOT NULL,
    screening_date              DATE NOT NULL,
    status                      VARCHAR(20) NOT NULL,
    total_score                 NUMERIC(5,2),
    responses                   JSONB NOT NULL,
    identified_needs            JSONB DEFAULT '[]',
    referrals                   JSONB DEFAULT '[]'
);

-- CCM billing time tracking
CREATE TABLE cm_billing_time_entry (
    entry_id                    BIGINT PRIMARY KEY,
    person_id                   BIGINT NOT NULL REFERENCES person(person_id),
    care_plan_id                BIGINT REFERENCES cm_care_plan(care_plan_id),
    provider_id                 BIGINT REFERENCES provider(provider_id),
    activity_type               VARCHAR(50) NOT NULL,
    cpt_code                    VARCHAR(10),
    duration_minutes            INTEGER NOT NULL,
    service_date                DATE NOT NULL,
    billing_period              VARCHAR(7) NOT NULL
);
```

### Graph Layer (Care Coordination and Relationships)

```cypher
// -- Node Types --

// Patient node (synced from OMOP person table)
CREATE (p:Patient {
    person_id: 12345,
    mpi_id: "uuid-here",
    name: "Jane Smith",
    risk_tier: "HIGH",
    current_risk_score: 0.82,
    active_programs: ["Diabetes Management", "SDOH Support"],
    open_gap_count: 3,
    census_tract: "36061002300"
})

// Care team member
CREATE (c:CareTeamMember {
    provider_id: 456,
    name: "Dr. Sarah Johnson",
    role: "PCP",
    specialty: "Internal Medicine",
    caseload_count: 42,
    max_caseload: 50,
    organization_id: 789
})

// Community resource
CREATE (r:CommunityResource {
    resource_id: "cr-001",
    name: "Downtown Food Bank",
    type: "FOOD_ASSISTANCE",
    address: "123 Main St",
    capacity: "AVAILABLE",
    eligibility_criteria: ["income_below_200_fpl", "zip_10001_10010"],
    success_rate: 0.78
})

// SDOH Need (representing a social determinant need domain)
CREATE (n:SdohNeed {
    domain: "FOOD_INSECURITY",
    severity: "HIGH",
    screening_date: "2026-03-15"
})

// -- Relationship Types --

// Care coordination relationships
CREATE (p)-[:MANAGED_BY {role: "CARE_COORDINATOR", since: "2026-01-15"}]->(c)
CREATE (p)-[:MANAGED_BY {role: "PCP", since: "2025-06-01"}]->(c2)
CREATE (c)-[:COORDINATES_WITH {frequency: "WEEKLY", channel: "SECURE_MESSAGE"}]->(c2)

// Program enrollment
CREATE (p)-[:ENROLLED_IN {status: "ACTIVE", enrolled_date: "2026-01-15"}]->(prog:DiseaseProgram {name: "Diabetes Management"})

// Care gap relationships
CREATE (p)-[:HAS_GAP {status: "OPEN", due_date: "2026-06-01"}]->(gap:CareGap {type: "HBA1C_SCREENING", measure: "HEDIS-CDC"})

// SDOH and community referral chains
CREATE (p)-[:SCREENED_POSITIVE_FOR {date: "2026-03-15", instrument: "AHC-HRSN"}]->(n)
CREATE (p)-[:REFERRED_TO {date: "2026-03-16", status: "ACCEPTED", outcome: "RECEIVING_SERVICES"}]->(r)
CREATE (r)-[:ADDRESSES]->(n)

// Family relationships (enables family-level care plans)
CREATE (p)-[:FAMILY_OF {relationship: "SPOUSE"}]->(p2:Patient)
CREATE (p)-[:FAMILY_OF {relationship: "DEPENDENT_CHILD"}]->(p3:Patient)

// Geographic relationships (for population-level SDOH analysis)
CREATE (p)-[:LIVES_IN]->(ct:CensusTract {fips: "36061002300", svi_score: 0.85})
CREATE (ct)-[:PART_OF]->(z:ZipCode {zip: "10001"})

// Outreach sequence (temporal chain)
CREATE (o1:OutreachEvent {channel: "SMS", date: "2026-03-01", result: "NO_RESPONSE"})
CREATE (o2:OutreachEvent {channel: "PHONE", date: "2026-03-08", result: "REACHED"})
CREATE (p)-[:OUTREACH_SEQUENCE]->(o1)-[:FOLLOWED_BY]->(o2)

// Provider network (identifies high-performing provider combinations)
CREATE (c)-[:REFERS_TO {count: 15, avg_outcome_score: 0.82}]->(c3:CareTeamMember)
```

### Graph Query Examples

```cypher
-- Find all patients with food insecurity in a high-SVI census tract
-- who are not yet referred to a food assistance resource
MATCH (p:Patient)-[:SCREENED_POSITIVE_FOR]->(n:SdohNeed {domain: "FOOD_INSECURITY"}),
      (p)-[:LIVES_IN]->(ct:CensusTract)
WHERE ct.svi_score > 0.75
  AND NOT EXISTS {
    MATCH (p)-[:REFERRED_TO]->(r:CommunityResource {type: "FOOD_ASSISTANCE"})
  }
RETURN p.name, p.person_id, ct.fips, ct.svi_score

-- Find care coordinators with capacity who manage patients
-- in the same disease program
MATCH (c:CareTeamMember {role: "CARE_COORDINATOR"})
WHERE c.caseload_count < c.max_caseload
MATCH (c)-[:MANAGES]->(existing:Patient)-[:ENROLLED_IN]->(prog:DiseaseProgram)
RETURN c.name, c.caseload_count, c.max_caseload, prog.name, COUNT(existing)
ORDER BY c.caseload_count ASC

-- Identify family clusters with multiple high-risk members
-- (enables family-level care plan creation)
MATCH (p1:Patient)-[:FAMILY_OF*1..2]-(p2:Patient)
WHERE p1.risk_tier IN ['HIGH', 'VERY_HIGH']
  AND p2.risk_tier IN ['HIGH', 'VERY_HIGH']
  AND p1.person_id < p2.person_id
RETURN p1.name, p2.name, p1.risk_tier, p2.risk_tier

-- Trace outreach effectiveness by channel sequence
MATCH (p:Patient)-[:OUTREACH_SEQUENCE]->(o1:OutreachEvent)-[:FOLLOWED_BY*]->(oN:OutreachEvent)
WHERE oN.result = 'REACHED'
RETURN p.person_id, 
       COLLECT(o1.channel + ': ' + o1.result) AS outreach_chain,
       oN.channel AS successful_channel,
       SIZE(COLLECT(o1)) + 1 AS attempts_to_reach

-- Discover high-performing provider networks
MATCH (c1:CareTeamMember)-[r:REFERS_TO]->(c2:CareTeamMember)
WHERE r.avg_outcome_score > 0.8 AND r.count > 10
RETURN c1.name, c1.role, c2.name, c2.role, r.avg_outcome_score, r.count
ORDER BY r.avg_outcome_score DESC
```

---

## Pros

- **OMOP CDM is purpose-built for healthcare analytics**: The OMOP CDM was designed by OHDSI specifically for observational health data analysis. It comes with standardized vocabularies (SNOMED CT, LOINC, ICD-10, RxNorm) pre-mapped, eliminating the need to build vocabulary management from scratch. Over 400 institutions worldwide use OMOP, providing a large community of users and contributors.
- **Free analytics tooling**: OHDSI provides open-source tools that work directly against the OMOP CDM: ATLAS for cohort definition and characterization, Achilles for data quality assessment, PatientLevelPrediction for ML risk models, and CohortDiagnostics for phenotype validation. These tools are directly applicable to population health analytics and risk stratification.
- **Chronic disease analysis proven**: OMOP has been validated for treatment pathway analysis across chronic diseases (hypertension, type 2 diabetes, depression), making it directly relevant to disease management program design and evaluation.
- **Graph excels at care coordination**: The relationship-heavy aspects of care management -- care team composition, referral chains, family networks, SDOH resource navigation -- are naturally expressed in a property graph. Queries like "find all patients managed by this coordinator who also see this specialist and have food insecurity" are trivial in Cypher but require complex multi-table joins in SQL.
- **SDOH network analysis**: Graph databases enable unique SDOH capabilities: mapping community resource availability by geography, tracking referral chains and success rates, identifying underserved census tracts, and analyzing how social networks affect health outcomes.
- **Equity-aware analysis**: OMOP's standardized vocabulary and person table enable demographic stratification of risk scores, care gap closure rates, and program outcomes -- essential for detecting and correcting bias in risk models.
- **Family-level care plans**: Graph relationships naturally model family networks, enabling the "family-level care plan" feature identified as an underserved opportunity in the features analysis.
- **Research community alignment**: Using OMOP CDM positions the platform for federated research collaborations with OHDSI network sites -- a potential differentiator for academic medical centers and research-oriented health systems.

## Cons

- **Two databases to operate**: Running PostgreSQL (OMOP) alongside Neo4j (graph) doubles operational complexity: two backup strategies, two scaling approaches, two sets of monitoring, two query languages.
- **Data synchronization**: Patient data, risk scores, and care plan status must be synchronized between OMOP tables and graph nodes. Eventual consistency between the two stores requires careful event-driven synchronization.
- **OMOP ETL complexity**: Transforming source data (FHIR, HL7v2, claims) into OMOP CDM format is notoriously complex. The ETL requires mapping source codes to OMOP standard concepts, handling vocabulary versioning, and managing edge cases. OHDSI provides tools (WhiteRabbit, Rabbit-in-a-Hat, Usagi) but the process is labor-intensive.
- **OMOP is analytics-optimized, not transactional**: OMOP CDM is designed for analytical queries, not for transactional care management workflows. Care coordinator worklists, real-time task assignment, and care plan editing require the extension tables (cm_*) and potentially a separate application database for low-latency transactional operations.
- **Graph database expertise**: Cypher/GQL query skills are less common than SQL. The team needs graph modeling expertise for schema design, query optimization, and data loading.
- **Vocabulary management overhead**: OMOP requires loading and maintaining the Athena vocabulary bundle (~2GB), with regular updates as vocabularies evolve. Source-to-standard concept mapping must be maintained per integration.
- **Overbuilt for small deployments**: A community health center with 10,000 patients does not need a graph database or the full OMOP CDM infrastructure. This architecture targets mid-to-large deployments.
- **Apache AGE alternative**: If Neo4j licensing is a concern, Apache AGE (a PostgreSQL extension) provides graph query capabilities within PostgreSQL, eliminating the second database but with less mature graph query optimization and tooling.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| OMOP Store | PostgreSQL 16+ with OMOP CDM v5.4 schema |
| Graph Store | Neo4j Community Edition (open source); or Apache AGE extension for single-database deployment |
| Vocabulary | OHDSI Athena vocabulary download (SNOMED CT, LOINC, ICD-10, RxNorm, HCPCS) |
| ETL | OHDSI WhiteRabbit + Rabbit-in-a-Hat for source profiling; custom Python/Spark ETL for FHIR-to-OMOP transformation |
| Analytics | OHDSI ATLAS for cohort building; OHDSI Achilles for data quality; custom dashboards (Metabase/Superset) for operational reporting |
| Synchronization | Debezium CDC from PostgreSQL to Neo4j; or application-level dual-write with idempotent graph updates |
| FHIR Bridge | MENDS-on-FHIR pattern for FHIR-to-OMOP mapping; custom FHIR facade for API responses |
| Application DB | Extension tables (cm_*) in the same PostgreSQL instance as OMOP for transactional care management workflows |
| API | REST/GraphQL for application queries; Cypher HTTP endpoint for graph queries; OHDSI WebAPI for analytics |

---

## Migration and Scaling Considerations

- **Phased adoption**: Start with OMOP CDM and extension tables in PostgreSQL only. Add the graph layer in a second phase once care coordination features mature and the relationship-heavy queries justify the additional infrastructure.
- **Apache AGE as bridge**: Use Apache AGE (PostgreSQL extension) initially to run graph queries within PostgreSQL. Migrate to Neo4j only if graph query performance or feature needs exceed AGE capabilities.
- **OMOP ETL pipeline**: Build the FHIR-to-OMOP ETL pipeline incrementally: start with Condition, Measurement, and Drug Exposure (sufficient for chronic disease registries and care gaps). Add Visit, Procedure, and Observation as use cases require.
- **Vocabulary updates**: Schedule quarterly Athena vocabulary updates. Use OMOP concept_relationship table for concept mapping and hierarchy traversal.
- **Graph synchronization**: Use Debezium CDC to capture changes from OMOP/extension tables and propagate to the graph store. Key events: new patient registration, risk tier changes, program enrollment, care gap status changes, SDOH screening results.
- **Scaling the graph**: Neo4j scales reads via read replicas. For write-heavy workloads (high outreach volume), partition by tenant/organization using Neo4j Fabric or separate database instances.
- **OMOP partitioning**: Partition measurement and drug_exposure by person_id hash or measurement_date range. These are the largest tables in a population health OMOP instance.
- **Multi-tenancy**: OMOP CDM does not have a native tenant concept. Add a tenant_id column to person and extension tables, enforced via RLS. In the graph layer, use node labels or a tenant property for isolation.
- **Research collaboration**: The OMOP CDM enables federated research participation via the OHDSI network. Health systems using this platform can participate in network studies without exposing patient-level data.
