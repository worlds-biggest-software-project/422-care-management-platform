# Standards & API Reference

> Project: Care Management Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### FHIR & HL7 Standards

**HL7 FHIR R4 (Fast Healthcare Interoperability Resources)**
- Standard reference: HL7 FHIR Release 4.0.1
- Official URL: https://www.hl7.org/fhir/
- Relevance: Modern healthcare data exchange standard providing RESTful APIs for patient records, care plans, goals, and care team coordination. Foundation for CMS interoperability mandates and population health data aggregation across EHRs and payers.

**HL7 FHIR Care Plan Resource**
- Official URL: https://www.hl7.org/fhir/careteam.html
- Relevance: Standardizes care plan documentation, goals, activities, and care team composition. Enables structured care plan sharing across multiple EHRs and care management systems participating in coordinated care workflows.

**HL7 FHIR Condition Resource**
- Official URL: https://www.hl7.org/fhir/condition.html
- Relevance: Standardizes patient problem lists and diagnoses with clinical relevance ratings. Facilitates disease management program selection and chronic condition tracking across care settings.

**HL7 CDS Hooks**
- Official URL: https://cds-hooks.hl7.org/
- Relevance: Framework for delivering clinical decision support at point of care, including care gap alerts, risk stratification recommendations, and intervention suggestions directly within EHR workflows.

### CMS Regulatory Standards

**CMS Interoperability and Patient Access Final Rule (CMS-9115-F)**
- Official reference: 21st Century Cures Act Implementation Final Rule
- Official URL: https://www.cms.gov/priorities/burden-reduction/overview/interoperability/policies-regulations/cms-interoperability-patient-access-final-rule-cms-9115-f
- Relevance: Mandates FHIR-based patient access APIs by health plans and providers, enabling care management platforms to access complete longitudinal patient records for risk stratification and care gap identification.

**HL7 Graduated Structured Technical Format (gSTF) for Quality Measure Reporting**
- Relevance: Standardizes reporting of quality measures (HEDIS, MIPS, NCQA) required for value-based contracts and population health accountability.

### Quality & Outcome Measurement Standards

**HEDIS (Healthcare Effectiveness Data and Information Set)**
- Official URL: https://www.ncqa.org/hedis/
- Relevance: National quality measure set used by health plans and health systems to assess care management outcomes. Care management platforms track and report HEDIS measures (e.g., diabetes care, hypertension management, preventive screenings).

**MIPS (Merit-based Incentive Payment System) Quality Measures**
- Official reference: CMS Physician Quality Reporting System (PQRS)
- Official URL: https://www.cms.gov/medicare/payment-models/merit-based-incentive-payment-system
- Relevance: CMS quality measure framework for physician practices; care management platforms report performance against MIPS measures tied to Medicare reimbursement.

**ISO 28591-1: Personal health record (PHR) - Information and services**
- Standard reference: ISO/IEC 28591-1:2023
- Relevance: International standard for personal health record structure and functionality; supports patient engagement features within care management platforms.

### Social Determinants of Health (SDOH) Standards

**AHC-HRSN (Accountable Health Communities - Health-Related Social Needs)**
- Official URL: https://www.cms.gov/priorities/innovation/models/ahc
- Relevance: CMS-validated SDOH screening instrument for identifying housing, food, transportation, and utility needs. Used in population stratification and resource referral within care management programs.

**PRAPARE Screening Tool**
- Official reference: National Association of Community Health Centers (NACHC)
- Relevance: Standardized SDOH assessment tool covering social, economic, environmental, and behavioral factors. Integrated into care assessment workflows for holistic patient understanding.

### Data Model & API Specifications

**OpenAPI Specification (v3.1+)**
- Official URL: https://spec.openapis.org/oas/v3.1.0.html
- Relevance: Standard for documenting REST APIs used by care management platforms to integrate with EHRs, payers, and specialty care systems.

**Continuity of Care Document (CCD) / Consolidated Clinical Document Architecture (C-CDA)**
- Standard reference: HL7 CDA Release 2.0
- Official URL: https://www.hl7.org/implement/standards/product_brief.cfm?product_id=7
- Relevance: XML-based clinical document interchange standard enabling care transitions documentation between acute care, ambulatory, and care management systems.

**SNOMED CT (Systematized Nomenclature of Medicine Clinical Terms)**
- Official URL: https://www.snomed.org/
- Relevance: Clinical terminology standard for diagnoses, problems, medications, and procedures. Ensures consistent data representation across heterogeneous EHR and care management systems.

**LOINC (Logical Observation Identifiers Names and Codes)**
- Official URL: https://loinc.org/
- Relevance: International standard for laboratory, radiology, and clinical observations. Essential for aggregating lab results, vital signs, and biometric measurements used in population risk stratification.

### Security & Privacy Standards

**HIPAA Privacy Rule**
- Standard reference: 45 CFR Parts 160 and 164
- Official URL: https://www.hhs.gov/hipaa/for-professionals/privacy/index.html
- Relevance: Establishes protected health information (PHI) safeguards; care management platforms must implement role-based access and audit logging.

**HIPAA Security Rule**
- Standard reference: 45 CFR Parts 160 and 164, Subpart C
- Official URL: https://www.hhs.gov/hipaa/for-professionals/security/index.html
- Relevance: Technical and administrative controls for electronic PHI (ePHI) in care management systems; requires encryption, access controls, and intrusion detection.

**OAuth 2.0 Authorization Framework**
- Standard reference: RFC 6749
- Official URL: https://tools.ietf.org/html/rfc6749
- Relevance: Secure API authentication and delegation mechanism used in FHIR API integrations and care team member authorization within platforms.

**TLS 1.2+ (IETF RFC 5246)**
- Standard reference: RFC 5246 (TLS 1.2), RFC 8446 (TLS 1.3)
- Relevance: Encryption standard for ePHI transmission between care management system and upstream EHRs, payer systems, and HIE networks.

## Similar Products — Developer Documentation & APIs

### Epic Healthy Planet

- **Description:** Care management and population health module integrated within Epic EHR, offering risk stratification, chronic disease registries, care gap detection, and quality measure reporting for Epic-native organizations.
- **API Documentation:** https://fhir.epic.com/
- **SDKs/Libraries:** SMART on FHIR; FHIR R4 APIs for patient data, care plans, and goals
- **Developer Guide:** Epic developer portal with sandbox access; care management app integration guides
- **Standards:** HL7 FHIR R4; SMART on FHIR; CDS Hooks for care gap alerting
- **Authentication:** OAuth 2.0 via SMART on FHIR launch; API credentials for backend integrations

### Azara Healthcare

- **Description:** Leading population health management platform (Best in KLAS 2023–2026), dominant in FQHC segment. Aggregates multi-source data, applies AI-driven risk stratification, and activates disease-specific care interventions.
- **API Documentation:** Custom enterprise API integration
- **SDKs/Libraries:** Multi-source data aggregation APIs; EHR-agnostic data models
- **Developer Guide:** Enterprise integration partnership program; data feed specifications
- **Standards:** Supports HL7 v2.x, FHIR R4, proprietary data models for multi-source normalization
- **Authentication:** Custom OAuth-style integration; role-based API keys for partner access

### Innovaccer Health Cloud

- **Description:** AI-driven population health platform ranked #1 for 2025–2026 by Black Book Research. Aggregates claims, clinical, and social data with predictive risk modeling and care gap automation.
- **API Documentation:** https://innovaccer.com/platform/apis
- **SDKs/Libraries:** Care management APIs; risk model APIs; FHIR-aligned data integration
- **Developer Guide:** Platform documentation for care program automation and analytics
- **Standards:** FHIR R4; OpenAPI specification; custom normalizer layer for multi-source data
- **Authentication:** OAuth 2.0; API key-based access for enterprise integrations

### Oracle Health HealtheIntent

- **Description:** Care management and population health module within Oracle Health (formerly Cerner) ecosystem. Provides longitudinal patient aggregation, risk stratification, and care gap management across large health systems.
- **API Documentation:** https://developer.oracle.com/en/learn/technical-articles/oracle-health-api-guide
- **SDKs/Libraries:** HL7 FHIR R4 APIs; CDS Hooks for clinical decision support
- **Developer Guide:** Oracle Health developer portal with sandbox and integration guides
- **Standards:** HL7 FHIR R4; SMART on FHIR; X12 EDI for claims data feeds
- **Authentication:** OAuth 2.0; SMART on FHIR launch; custom API credentials

### Arcadia Health Analytics

- **Description:** Data aggregation and population health analytics platform with demonstrated outcomes (e.g., 41.5% ED reduction for COPD). Focuses on longitudinal data integration and analytics-driven care management.
- **API Documentation:** Enterprise data integration APIs
- **SDKs/Libraries:** Multi-source data connectors; analytics export APIs
- **Developer Guide:** Custom integration via health information exchange (HIE) partners
- **Standards:** HL7 v2.x, FHIR R4, proprietary analytics data models
- **Authentication:** Custom API authentication; role-based access for analytics reporting

### Persivia CareSpace

- **Description:** Modular disease management and care coordination platform ranked top five for PHM in 2025. Offers evidence-based disease programs, predictive risk scoring, and care team workflow tools.
- **API Documentation:** Custom enterprise APIs for care team integration
- **SDKs/Libraries:** Disease program APIs; risk scoring APIs; workflow orchestration
- **Developer Guide:** Enterprise integration partnership; EHR connector documentation
- **Standards:** FHIR R4-aligned data models; OpenAPI specifications for integrations
- **Authentication:** OAuth 2.0; API key management for partner ecosystem

### Salesforce Health Cloud

- **Description:** CRM-rooted platform increasingly adopted for payer-side care management and complex case coordination. Provides care team workflows, patient engagement, and outcomes tracking.
- **API Documentation:** https://developer.salesforce.com/docs/platform/latest/index.html
- **SDKs/Libraries:** Salesforce APIs for care programs, patient engagement, and team workflows
- **Developer Guide:** Salesforce Health Cloud developer documentation and sandbox
- **Standards:** REST APIs; FHIR R4 data model alignment; custom metadata objects for care programs
- **Authentication:** OAuth 2.0; Salesforce identity and access management (IAM)

## Notes

### Emerging Standards & Areas

1. **TEFCA (Trusted Exchange Framework and Common Agreement):** CMS is operationalizing TEFCA to enable nation-wide, standards-based health information exchange. Care management platforms will increasingly leverage TEFCA networks to access patient data from out-of-network providers and payers.

2. **Predictive Risk Scoring Standards:** No formal standard exists for risk stratification algorithms or risk model reporting. Industry is moving toward transparency guidelines (e.g., FDA AI/ML regulation) for clinical decision support integration.

3. **Care Plan Interoperability:** While FHIR defines CarePlan and Goal resources, standardized exchange protocols for real-time care plan synchronization across multiple participating providers remain evolving.

### Gaps

- **Social Determinants Data Standards:** While SDOH screening instruments are standardized (AHC-HRSN, PRAPARE), integration of SDOH data into longitudinal EHR records remains proprietary; HL7 FHIR extensions for SDOH are still in development.
- **Population Outcomes Measurement:** No unified standard for population-level outcome reporting across health plans and provider networks; HEDIS and MIPS remain separate reporting frameworks.
- **Care Coordination Messaging:** Real-time care team communication and task routing protocols remain largely proprietary or using general workflows (email, SMS) rather than standardized healthcare APIs.
