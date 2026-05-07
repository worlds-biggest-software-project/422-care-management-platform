# Care Management Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native platform for population-scale chronic disease management, care gap closure, and coordinated patient outreach.

Care Management Platform is a comprehensive tool for health systems, ACOs, FQHCs, and managed care organisations that need to identify at-risk patients, enroll them in evidence-based disease management programs, and systematically close care gaps across large populations. It replaces fragmented manual workflows with automated risk stratification, multi-channel outreach, collaborative care planning, and real-time quality measure reporting -- all designed from the ground up to leverage AI where it matters most.

---

## Why Care Management Platform?

- **Every incumbent is proprietary and expensive.** All seven major platforms analysed (Epic Healthy Planet, Azara, Innovaccer, Arcadia, Oracle Health, Persivia, Salesforce Health Cloud) are closed-source commercial products with no open-source alternative identified in the market.
- **Vendor lock-in is structural.** Epic Healthy Planet only works well inside Epic; Oracle HealtheIntent is optimised for the Oracle/Cerner ecosystem. Organisations running mixed EHR environments are forced into costly integration projects or second-best tools.
- **SDOH is bolted on, not built in.** Most platforms treat social determinants of health screening as an add-on feature. Validated instruments like AHC-HRSN and PRAPARE, community resource referral, and SDOH outcome tracking remain fragmented across separate tools.
- **Risk models carry hidden bias.** Commercial risk stratification models trained on historical claims data systematically underestimate risk in under-served populations who have historically used less care, leading to inequitable program enrollment.
- **Small organisations are priced out.** Most platforms target large health systems. Community health centres, small primary care practices, and resource-constrained organisations lack access to affordable, purpose-built care management software.

---

## Key Features

### Risk Stratification and Population Segmentation

- Multi-factor risk scoring combining claims, clinical, and demographic data
- Chronic disease registries for diabetes, hypertension, COPD, heart failure, and depression
- Explainable risk scores showing specific contributing factors for targeted interventions
- AI-driven predictive models that go beyond rule-based scoring
- Equity-aware risk model validation detecting and correcting bias against under-represented populations

### Care Gap Identification and Quality Measures

- Automated detection of overdue preventive services, missing labs, medication adherence gaps, and unmet screenings
- Mapping to HEDIS, MIPS, and value-based contract quality measures
- Real-time performance dashboards with drill-down to individual patient opportunities
- Predictive quality measure scoring identifying patients at risk of missing targets
- CMS CCM billing compliance with CPT 99490, 99491, 99487, and 99489 time tracking and documentation

### Care Plan Management and Coordination

- Collaborative care plan documentation with problem lists, goals, interventions, and patient-reported outcomes
- Care team task assignment and workflow routing across coordinators, nurses, social workers, and physicians
- Caseload management dashboards with capacity visibility and dynamic workload balancing
- Disease management program templates with configurable evidence-based protocols and escalation pathways
- Mobile access for field-based care coordinators

### Patient Outreach and Engagement

- Multi-channel outreach: automated call, SMS, email, and patient portal messaging
- Consent management and communication preference tracking
- ML-optimised outreach timing and modality selection per patient
- Contact rate tracking with engagement and ROI metrics
- Non-responder escalation strategies

### Data Integration and SDOH

- Bidirectional EHR integration (Epic, Cerner, Athena, and others) with claims, labs, and pharmacy data
- Master patient index for longitudinal patient record construction across disparate sources
- SDOH screening with validated instruments (AHC-HRSN, PRAPARE)
- Community resource referral integration with eligibility checking and outcome tracking
- Health information exchange (HIE) connectivity

---

## AI-Native Advantage

Unlike incumbents that have retrofitted AI onto legacy architectures, Care Management Platform is designed from the start to apply machine learning where it creates the most clinical and operational value. AI-driven intervention matching predicts which patients will respond to which programs based on characteristics, preferences, and past outcomes -- moving beyond one-size-fits-all care plans. Outreach timing optimisation learns each patient's engagement patterns to maximise contact rates. Readmission risk prediction and medication adherence models enable preventive action before complications occur. Equity-aware risk model validation continuously monitors for demographic bias and auto-recalibrates to ensure fair program enrollment.

---

## Tech Stack & Deployment

The platform targets self-hosted and cloud deployment, with a hybrid model for organisations that require on-premise data residency for PHI. EHR integration uses standard healthcare interoperability protocols (HL7 FHIR, HL7 v2) alongside direct API connectors for major EHR platforms. SDOH screening instruments (AHC-HRSN, PRAPARE) are publicly available and licence-free. CMS CCM billing requirements follow published regulatory specifications. The architecture supports multi-source data aggregation and normalisation across EHRs, claims systems, pharmacy benefit managers, and HIEs.

---

## Market Context

The chronic disease management application market is projected to reach $20.28 billion by 2032, growing at a CAGR of 10.32%. The CMS ACCESS model launching in July 2026 introduces performance-contingent payments of $15--$35 per member per month for enrolled chronic care management programs, creating strong new financial incentives for platform adoption. Target buyers include health systems with value-based contracts, Medicare Advantage plans, Medicaid managed care organisations, FQHCs, and ACOs.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
