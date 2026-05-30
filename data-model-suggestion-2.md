# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Care Management Platform (Candidate #422)

## Summary

An event-sourced architecture with Command Query Responsibility Segregation (CQRS) where every state change in the care management workflow is captured as an immutable event in an append-only event store. Read-optimized projections (materialized views) are built from the event stream for different query needs: care coordinator dashboards, quality measure reporting, population analytics, and billing. This approach is particularly compelling for healthcare because the complete, tamper-proof history of every patient interaction, care plan change, risk score update, and outreach attempt is inherently preserved -- providing a natural audit trail, full temporal queryability, and the ability to reconstruct the state of any patient record at any point in time.

---

## Architecture Overview

```
                    ┌─────────────────┐
                    │   Command Side  │
                    │   (Write Model) │
                    └────────┬────────┘
                             │
                    Command Handlers
                    (validate, authorize)
                             │
                    ┌────────▼────────┐
                    │   Event Store   │
                    │  (append-only)  │
                    │  PostgreSQL /   │
                    │  EventStoreDB   │
                    └────────┬────────┘
                             │
              Event Bus (async projections)
              ┌──────────┬──────────────┬──────────────┐
              │          │              │              │
    ┌─────────▼──┐ ┌─────▼──────┐ ┌────▼────────┐ ┌──▼──────────┐
    │ Care Coord │ │ Quality    │ │ Population  │ │ Billing     │
    │ Dashboard  │ │ Measure    │ │ Analytics   │ │ & CCM Time  │
    │ Projection │ │ Projection │ │ Projection  │ │ Projection  │
    └────────────┘ └────────────┘ └─────────────┘ └─────────────┘
         (Query Side -- Read Models)
```

---

## Key Entities and Event Types

### Aggregate Roots

The event-sourced model organizes around aggregate roots -- the primary entities whose lifecycles are tracked through event streams:

1. **PatientAggregate** -- identity, demographics, consent
2. **RiskAssessmentAggregate** -- risk scoring lifecycle per patient
3. **ProgramEnrollmentAggregate** -- enrollment in disease management programs
4. **CarePlanAggregate** -- care plan creation, goals, interventions, reviews
5. **CareGapAggregate** -- gap identification, actions, closure
6. **OutreachAggregate** -- outreach campaigns and individual attempts
7. **SdohScreeningAggregate** -- SDOH assessment and referral lifecycle
8. **BillingAggregate** -- CCM time tracking and billing period management

### Event Store Schema

```sql
-- Core event store table (append-only)
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    event_version   INTEGER NOT NULL,        -- Aggregate version for optimistic concurrency
    event_data      JSONB NOT NULL,           -- Full event payload
    metadata        JSONB,                    -- Causation ID, correlation ID, user, IP
    tenant_id       UUID NOT NULL,            -- Organization/health system
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by      UUID NOT NULL,
    CONSTRAINT uq_aggregate_version UNIQUE (aggregate_id, event_version)
);

-- Index for replaying aggregate event streams
CREATE INDEX idx_event_store_aggregate 
    ON event_store (aggregate_id, event_version);

-- Index for projections that process events by type
CREATE INDEX idx_event_store_type_created 
    ON event_store (event_type, created_at);

-- Index for tenant-scoped queries
CREATE INDEX idx_event_store_tenant 
    ON event_store (tenant_id, created_at);

-- Snapshot table for performance (rebuild aggregate without full replay)
CREATE TABLE aggregate_snapshot (
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,
    snapshot_data   JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (aggregate_id, snapshot_version)
);
```

### Event Type Catalog

```
── Patient Events ──────────────────────────────────
PatientRegistered           { mpi_id, demographics, identifiers }
PatientDemographicsUpdated  { changed_fields }
PatientConsentGranted       { consent_type, granted_date }
PatientConsentRevoked       { consent_type, revoked_date, reason }
PatientContactUpdated       { contact_type, old_value, new_value }
PatientDeceased             { deceased_date }

── Risk Assessment Events ──────────────────────────
RiskScoreCalculated         { model_id, score, tier, factors[] }
RiskTierChanged             { old_tier, new_tier, reason }
RiskModelUpdated            { model_id, version, changes }
EquityAuditCompleted        { model_id, bias_metrics, adjustments }

── Program Enrollment Events ───────────────────────
PatientEnrolledInProgram    { program_id, enrolled_by, criteria_met }
PatientGraduatedFromProgram { program_id, outcomes }
PatientWithdrawnFromProgram { program_id, reason, withdrawn_by }
EnrollmentStatusChanged     { old_status, new_status, reason }

── Care Plan Events ────────────────────────────────
CarePlanCreated             { title, category, authored_by }
CarePlanActivated           { activated_by }
CarePlanGoalAdded           { goal_description, target, priority }
CarePlanGoalStatusChanged   { goal_id, old_status, new_status }
CarePlanInterventionAdded   { description, type, assigned_to }
CarePlanInterventionCompleted { intervention_id, outcome }
CarePlanReviewed            { reviewed_by, notes, next_review_date }
CarePlanCompleted           { outcome_summary }
CarePlanRevoked             { reason, revoked_by }

── Care Gap Events ─────────────────────────────────
CareGapIdentified           { measure_id, gap_type, due_date }
CareGapActionTaken          { action_type, performed_by }
CareGapClosed               { closed_by, evidence }
CareGapExcluded             { reason, excluded_by }
CareGapReopened             { reason }

── Outreach Events ─────────────────────────────────
OutreachCampaignCreated     { name, type, target_criteria }
OutreachAttempted           { channel, attempted_by }
OutreachPatientReached      { channel, duration, notes }
OutreachNoResponse          { channel, attempt_number }
OutreachPatientDeclined     { reason }
OutreachEscalated           { escalation_type, escalated_to }

── SDOH Events ─────────────────────────────────────
SdohScreeningStarted        { instrument, administered_by }
SdohScreeningCompleted      { responses[], total_score, needs[] }
SdohScreeningDeclined       { reason }
CommunityReferralMade       { resource_name, resource_type }
CommunityReferralAccepted   { accepted_date }
CommunityReferralCompleted  { outcome }

── Billing Events ──────────────────────────────────
BillingTimeRecorded         { activity_type, duration_minutes, cpt_code }
BillingPeriodClosed         { period, total_minutes, cpt_codes[] }
BillingClaimSubmitted       { claim_id, amount }
```

### Read Model Projections

#### Care Coordinator Dashboard Projection

```sql
-- Materialized read model for care coordinator worklist
CREATE TABLE projection_coordinator_worklist (
    patient_id          UUID NOT NULL,
    patient_name        VARCHAR(200),
    risk_tier           VARCHAR(20),
    current_risk_score  NUMERIC(8,4),
    active_programs     TEXT[],
    open_care_gaps      INTEGER,
    last_outreach_date  TIMESTAMPTZ,
    last_outreach_result VARCHAR(50),
    next_action_due     DATE,
    assigned_coordinator UUID,
    ccm_minutes_this_month INTEGER DEFAULT 0,
    sdoh_needs          TEXT[],
    updated_at          TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (patient_id)
);
```

#### Quality Measure Projection

```sql
-- Materialized read model for quality measure dashboards
CREATE TABLE projection_quality_measure (
    measure_id          UUID NOT NULL,
    measure_code        VARCHAR(50),
    measure_name        VARCHAR(300),
    measure_set         VARCHAR(20),
    denominator_count   INTEGER DEFAULT 0,
    numerator_count     INTEGER DEFAULT 0,
    exclusion_count     INTEGER DEFAULT 0,
    performance_rate    NUMERIC(5,2),
    reporting_period    VARCHAR(7),          -- YYYY-MM
    tenant_id           UUID NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (measure_id, reporting_period, tenant_id)
);
```

#### Population Analytics Projection

```sql
-- Materialized read model for population-level analytics
CREATE TABLE projection_population_summary (
    tenant_id           UUID NOT NULL,
    risk_tier           VARCHAR(20),
    condition_category  VARCHAR(100),
    total_patients      INTEGER DEFAULT 0,
    enrolled_in_program INTEGER DEFAULT 0,
    avg_risk_score      NUMERIC(8,4),
    open_gaps_total     INTEGER DEFAULT 0,
    outreach_contact_rate NUMERIC(5,2),
    avg_ccm_minutes     NUMERIC(8,2),
    snapshot_date       DATE NOT NULL,
    PRIMARY KEY (tenant_id, risk_tier, condition_category, snapshot_date)
);
```

### Command Handlers (Application Layer)

```python
# Pseudocode for care plan command handler
class CarePlanCommandHandler:
    def handle_create_care_plan(self, cmd: CreateCarePlanCommand):
        # 1. Validate: patient exists, user authorized, no duplicate active plan
        patient = self.repository.load("PatientAggregate", cmd.patient_id)
        
        # 2. Create aggregate and emit event
        care_plan = CarePlanAggregate.create(
            patient_id=cmd.patient_id,
            title=cmd.title,
            category=cmd.category,
            authored_by=cmd.user_id
        )
        # Emits CarePlanCreated event
        
        # 3. Persist events to event store
        self.event_store.append(care_plan.pending_events)
        
        # 4. Projectors asynchronously update read models
        # (coordinator worklist, patient timeline, etc.)

    def handle_add_goal(self, cmd: AddGoalCommand):
        care_plan = self.repository.load("CarePlanAggregate", cmd.care_plan_id)
        care_plan.add_goal(cmd.description, cmd.target, cmd.priority)
        # Emits CarePlanGoalAdded event
        self.event_store.append(care_plan.pending_events)
```

---

## Pros

- **Complete audit trail**: Every state change is an immutable event. HIPAA audit requirements are satisfied by the event store itself -- no separate audit logging needed. Investigators can reconstruct exactly who did what, when, and why.
- **Temporal queries**: "What was this patient's care plan on March 15?" is answered by replaying events up to that timestamp. This is invaluable for clinical reviews, billing disputes, and regulatory audits.
- **Regulatory compliance**: CMS CCM billing requires documented time, activities, and care plan elements. Events naturally capture this documentation trail without requiring care coordinators to fill out separate compliance forms.
- **Conflict-free concurrent editing**: Multiple care team members can concurrently add goals, log interventions, and record outreach on the same patient without write conflicts -- each action is a separate event.
- **Independent read model optimization**: Quality measure dashboards, coordinator worklists, and population analytics each get purpose-built projections optimized for their specific query patterns. No compromises between transactional and analytical workloads.
- **Event replay and correction**: If a risk model is found to be biased, scores can be recalculated and the affected events replayed through updated projections. Event sourcing naturally supports the equity-aware model validation requirement.
- **Integration-friendly**: Events map naturally to FHIR messaging patterns and can be published to external systems (EHRs, HIEs) as they occur.

## Cons

- **Complexity**: Event sourcing is significantly more complex than CRUD. The team must understand aggregates, event handlers, projectors, eventual consistency, and idempotency. Hiring is harder; fewer developers have production event sourcing experience.
- **Eventual consistency**: Read models are updated asynchronously. A care coordinator who records an outreach attempt may not see it reflected in their worklist for milliseconds to seconds. In care management this is usually acceptable, but requires careful UX design to avoid confusion.
- **Event schema evolution**: As requirements change (new SDOH domains, new quality measures, new outreach channels), event schemas must evolve without breaking existing event replay. Requires disciplined versioning and upcasting strategies.
- **Storage growth**: Every event is stored forever. A patient with 5 years of care management history may have thousands of events. Snapshots mitigate replay cost but add infrastructure complexity.
- **Debugging difficulty**: Diagnosing issues requires tracing through event sequences rather than inspecting current state in a table. Tooling for event store debugging is less mature than traditional database tooling.
- **Over-engineering risk**: For an MVP targeting community health centers with modest patient populations, event sourcing may be premature complexity. The full benefits emerge at scale with multiple consumers of the event stream.
- **Projection rebuild time**: If a projection needs to be rebuilt (schema change, bug fix), replaying millions of events can take hours. Requires careful operational planning.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event Store | PostgreSQL with append-only event_store table (simplest); or EventStoreDB for dedicated event store infrastructure |
| Event Bus | Apache Kafka or NATS JetStream for durable event distribution to projectors |
| Projections | Custom projector services per read model; or use Kafka Streams / Flink for stream processing |
| Read Models | PostgreSQL for transactional read models (worklists); ClickHouse or TimescaleDB for analytics projections |
| Snapshots | Stored in PostgreSQL alongside event store; rebuild threshold of ~500 events per aggregate |
| Framework | Axon Framework (Java), Eventuous (.NET), or custom Python/Node.js implementation |
| Serialization | JSON (JSONB in PostgreSQL) for event payloads; Avro or Protobuf for Kafka topics with schema registry |
| API | GraphQL for flexible read model queries; REST for command submission |

---

## Migration and Scaling Considerations

- **Event store partitioning**: Partition event_store by created_at (monthly) to manage table size. At 500K patients with active care management, expect ~100M events per year.
- **Snapshot strategy**: Take aggregate snapshots every 100-500 events to bound replay time. Snapshot the CarePlanAggregate after every major review cycle.
- **Projection scaling**: Each projection can be scaled independently. Quality measure projections can run on a schedule (hourly) while coordinator worklist projections update in near-real-time.
- **Event store compaction**: Never delete events (immutability is the point), but archive events older than the retention window to cold storage (S3/GCS) while keeping snapshots for fast aggregate loading.
- **Multi-tenancy**: Tenant ID in every event enables logical separation. Physical separation (separate event streams per tenant) for large health systems if needed.
- **Migration from CRUD**: If starting with a traditional CRUD model (Suggestion 1) and migrating to event sourcing later, introduce event publishing alongside CRUD writes (dual-write) then gradually build projections and shift reads. This is the "strangler fig" migration pattern.
- **FHIR event mapping**: Map care management events to FHIR Subscription notifications for real-time EHR integration. CarePlanCreated maps to FHIR CarePlan create notification; CareGapClosed maps to FHIR MeasureReport update.
- **Disaster recovery**: Event store replay from backup provides a natural disaster recovery mechanism. Rebuild all projections from the event store to recover from projection corruption.
