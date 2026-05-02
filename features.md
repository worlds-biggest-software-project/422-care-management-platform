# Care Management Platform — Feature & Functionality Survey

> Candidate #422 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Epic Healthy Planet | Commercial (embedded in Epic EHR) | Proprietary | https://www.epic.com/software/population-health/ |
| Azara Healthcare | Commercial SaaS | Proprietary | https://www.azarahealthcare.com/ |
| Innovaccer | Commercial SaaS | Proprietary | https://innovaccer.com/ |
| Arcadia | Commercial SaaS | Proprietary | https://arcadia.io/ |
| Oracle Health HealtheIntent | Commercial (part of Oracle Health) | Proprietary | https://www.oracle.com/health/ |
| Persivia CareSpace | Commercial SaaS | Proprietary | https://persivia.com/ |
| Salesforce Health Cloud | Commercial SaaS | Proprietary | https://www.salesforce.com/products/health-cloud/ |

## Feature Analysis by Solution

### Epic Healthy Planet

**Core features**
- Risk stratification with multi-factor scoring identifying high-risk, high-utilization, chronically ill patients
- Automated chronic disease registries for diabetes, heart failure, COPD, depression, hypertension
- Care gap identification with automated alerts when patients are due for preventive screenings or follow-ups
- Quality measure reporting for HEDIS, MIPS, and value-based contract measures
- Patient outreach workflows with automated communication
- Care coordination tools with provider communication across multiple settings
- Longitudinal patient data aggregation from EHR and connected systems
- Integration with Epic's broader suite enabling seamless clinical workflow connection

**Differentiating features**
- Deep embedded integration within Epic EHR eliminates data synchronisation friction
- Holds approximately 42% of acute-care market, providing significant installed base
- Risk scoring with explainability showing specific factors contributing to patient risk
- Native connection to Epic's clinical workflows and provider documentation

**UX patterns**
- EHR-integrated experience where population health insights appear within clinical workflow
- Risk scores with factor breakdown enabling targeted intervention selection
- Automated outreach reduces manual care coordinator overhead
- Performance dashboards visible within Epic interface

**Integration points**
- Native Epic ecosystem integration (no external integrations needed for Epic customers)
- Limited external integration (optimized for Epic customers)
- Health information exchange (HIE) connections for external clinical data
- Claims data connections where available

**Known gaps**
- Limited flexibility for non-Epic health systems (proprietary integration)
- Smaller feature set for independent health systems compared to dedicated platforms
- Less emphasis on social determinants of health (SDOH) screening compared to standalone platforms
- Less sophisticated outreach capabilities compared to specialised platforms

**Licence / IP notes**
- Proprietary commercial licence (part of Epic EHR bundle)
- Risk stratification algorithms are proprietary

---

### Azara Healthcare

**Core features**
- Named Best in KLAS for Population Health Management four consecutive years (2023–2026)
- Risk stratification and patient segmentation
- Chronic disease registries for high-priority conditions
- Care gap identification and closure tracking
- Disease management program templates with evidence-based protocols
- Patient outreach across multiple channels (automated call, SMS, email, portal messaging)
- Care plan development and management workflows
- Quality measure reporting and tracking
- Post-merger integration with i2i Population Health serving 25M+ Americans across 1,000+ community health centers

**Differentiating features**
- Dominant platform in Federally Qualified Health Center (FQHC) segment with 1,000+ center coverage
- Consistent KLAS recognition across four consecutive years demonstrates sustained excellence
- Strong focus on community health center workflows and resource-constrained settings
- i2i merger (2025) expanded capabilities in primary care and underserved populations
- Serves 25M+ Americans providing population-scale evidence of platform effectiveness

**UX patterns**
- Community health center-focused design optimized for resource-constrained environments
- Workflow optimisation for smaller, leaner care teams
- Emphasis on accessibility for diverse patient populations
- Integration-friendly architecture for multi-source data

**Integration points**
- EHR integrations (Epic, Cerner, Medidata, Athena, others)
- Health information exchange connections
- Claims data integrations
- Third-party API integrations

**Known gaps**
- Less emphasis on large health system integration complexity
- SDOH screening capabilities not prominently featured
- Analytics depth may be less sophisticated than enterprise-focused platforms
- Limited public information on AI-powered risk modelling

**Licence / IP notes**
- Proprietary commercial licence
- Risk stratification and disease management algorithms proprietary

---

### Innovaccer

**Core features**
- Ranked by Black Book Research as top AI-driven population health platform (2025, 2026)
- AI-driven risk models with multi-source data aggregation
- Care gap identification and intervention activation
- Unified data platform combining clinical, claims, and other data sources
- Disease management programs with configurable protocols
- Patient engagement and outreach tools
- Care team workflow management
- Quality measure reporting and performance tracking
- Advanced analytics and outcomes reporting

**Differentiating features**
- AI-first approach distinguishes from traditional care management platforms
- Black Book Research top ranking two consecutive years reflects market recognition
- Sophisticated multi-source data aggregation and normalization
- Focus on evidence-based intervention activation
- Strong analytics and outcomes measurement capabilities

**UX patterns**
- AI-powered insights integrated throughout platform
- Predictive recommendations guiding care coordinator actions
- Data-driven prioritisation of patients and interventions
- Advanced reporting with drill-down analytics

**Integration points**
- Multi-source data integrations (EHR, claims, labs, pharmacy, SDOH)
- APIs for custom integrations
- EHR platform connectors (Epic, Cerner, Athena, others)
- Payer and claims system connections

**Known gaps**
- Limited information on SDOH-specific screening instruments
- Outreach capabilities less detailed than specialised outreach platforms
- Smaller presence in FQHC/community health segment than Azara
- AI-driven approach may create black-box concerns requiring more transparency

**Licence / IP notes**
- Proprietary commercial licence
- AI-driven risk models and data aggregation algorithms proprietary

---

### Arcadia

**Core features**
- Data aggregation and analytics platform for population health
- Demonstrated outcomes: 41.5% reduction in ED visits for COPD patients in one Northeast ACO
- Risk stratification and patient segmentation
- Care gap identification aligned to quality measures
- Disease management program support
- Quality measure reporting and performance tracking
- Longitudinal patient record construction
- Outcomes analysis and reporting

**Differentiating features**
- Strong outcomes evidence: documented 41.5% ED reduction for specific disease cohort
- Data aggregation focus differentiates from full-service platforms
- Emphasis on outcomes measurement and validation
- Flexible deployment model suitable for various health system sizes

**UX patterns**
- Analytics-first interface emphasizing data insights
- Outcomes-focused reporting showing intervention effectiveness
- Modular approach allowing phased implementation
- Integration-friendly architecture

**Integration points**
- Multi-source data integrations (EHR, claims, labs, pharmacy)
- Health information exchange connections
- Claims data integrations
- APIs for custom integrations

**Known gaps**
- Less detailed information on patient outreach capabilities
- SDOH screening and community resource referral not emphasized
- Care team workflow tools less detailed than full-service platforms
- Smaller installed base compared to Epic or Azara

**Licence / IP notes**
- Proprietary commercial licence
- Analytics and outcomes measurement algorithms proprietary

---

### Oracle Health HealtheIntent

**Core features**
- Longitudinal patient record aggregation across health system
- Risk stratification and patient segmentation
- Care gap identification and closure tracking
- Quality measure reporting for HEDIS, MIPS, and value-based contracts
- Disease management program templates
- Care coordination workflows
- Multi-source data integration (EHR, claims, labs, pharmacy)
- Performance analytics and outcomes tracking
- Part of broader Oracle Health ecosystem enabling integration with other modules

**Differentiating features**
- Enterprise-scale platform from Oracle with significant infrastructure investment
- Integration with broader Oracle Health (formerly Cerner) ecosystem
- Strong data aggregation and master patient index capabilities
- Comprehensive quality measure coverage

**UX patterns**
- Enterprise data warehouse architecture ensuring scalability
- Integrated workflows across Oracle Health modules
- Advanced analytics and reporting capabilities
- Role-based interfaces for different user types

**Integration points**
- Deep integration with Oracle Health/Cerner EHR
- Connections to other Oracle Health modules
- External EHR integration (varies by deployment)
- Claims and HIE connections

**Known gaps**
- Limited standalone value for non-Oracle health systems
- Less emphasis on community health center workflows
- SDOH screening and outreach capabilities not prominently featured
- Smaller market presence in independent health systems

**Licence / IP notes**
- Proprietary commercial licence (part of Oracle Health)

---

### Persivia CareSpace

**Core features**
- Modular disease management programs for high-priority conditions
- Predictive risk scoring identifying patients for enrollment
- Care team workflow tools for coordinators, nurses, social workers
- Caseload management dashboards and capacity visibility
- Care plan authoring and management
- Patient engagement and outreach capabilities
- Quality measure reporting
- Integration with EHR and claims systems

**Differentiating features**
- Modular design enabling selective program deployment
- Ranked among top five PHM platforms (2025)
- Strong focus on care team workflows and caseload optimisation
- Emphasis on user experience for front-line care coordinators

**UX patterns**
- Care coordinator-centric design simplifying day-to-day workflows
- Task-oriented interface for managing patient assignments
- Caseload visibility enabling workload balancing
- Mobile-friendly design for field-based coordinators

**Integration points**
- EHR integrations (Epic, Cerner, Athena, others)
- Claims data connections
- HIE integrations
- APIs for custom integrations

**Known gaps**
- Smaller market presence compared to major competitors
- SDOH screening capabilities not prominently featured
- Analytics and outcomes reporting less detailed than data-focused platforms
- Limited information on AI-powered risk modelling

**Licence / IP notes**
- Proprietary commercial licence
- Risk scoring and disease management algorithms proprietary

---

### Salesforce Health Cloud

**Core features**
- CRM-rooted platform adapted for care management
- Patient engagement and communication tools
- Care plan and documentation management
- Care team coordination workflows
- Task assignment and tracking
- Integration with Salesforce ecosystem
- Customizable workflows and data models
- Reporting and dashboard capabilities

**Differentiating features**
- CRM-rooted approach emphasising patient relationship management
- Highly customizable platform enabling industry-specific configuration
- Strong integration with Salesforce ecosystem (Einstein AI, etc.)
- Increasingly adopted for payer-side care management
- Expertise in complex case coordination workflows

**UX patterns**
- Familiar Salesforce interface enabling easy adoption for Salesforce users
- Customizable dashboards and workflows
- Mobile-first design
- Einstein AI integration for intelligent recommendations

**Integration points**
- Deep Salesforce ecosystem integration
- Health Cloud-specific healthcare integrations
- APIs for custom integrations
- Broad third-party connection support

**Known gaps**
- Requires significant customisation for healthcare-specific workflows
- SDOH screening and community resource integration not native
- Risk stratification and disease management less sophisticated than dedicated platforms
- Requires strong configuration and customisation expertise

**Licence / IP notes**
- Proprietary commercial licence (Salesforce)

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

These capabilities are present in nearly every care management platform and any new entrant must match them:

- **Risk stratification engine**: Multi-factor scoring (claims, clinical, SDOH) identifying patients for care management
- **Chronic disease registries**: Condition-specific registries for diabetes, hypertension, COPD, heart failure, depression, others
- **Care gap identification**: Automated detection of overdue preventive services, missing labs, medication gaps, unmet screenings
- **Patient outreach**: Multi-channel communication (SMS, email, phone, portal) with consent management
- **Care plan management**: Collaborative documentation of patient problems, goals, interventions, outcomes
- **EHR and claims integration**: Bidirectional data from primary EHR and claims data sources
- **Quality measure reporting**: HEDIS, MIPS, value-based contract measure performance dashboards
- **Care team workflows**: Task assignment, caseload management, provider communication

### Differentiating Features

Capabilities present in some solutions providing competitive advantage:

- **AI-driven risk models**: ML models predicting high-risk patients beyond rule-based scoring (Innovaccer, some Oracle Health, Salesforce Einstein)
- **SDOH screening integration**: Validated screening instruments (AHC-HRSN, PRAPARE) with community resource referral (less common; opportunity area)
- **Multi-source data aggregation**: Normalizing and consolidating data from disparate EHRs, claims, labs, pharmacy, SDOH
- **Explainable risk scoring**: Showing specific factors contributing to patient risk scores enabling targeted interventions
- **Outreach effectiveness metrics**: Tracking contact rates, engagement, intervention uptake, and ROI
- **Dynamic caseload balancing**: Algorithms matching patients to coordinators based on skill and capacity
- **Value-based care integration**: CMS CCM billing compliance (CPT 99490, 99491, 99487, 99489) with time tracking and documentation
- **Outcomes measurement**: Pre/post intervention analysis and program-level effectiveness reporting
- **Mobile care coordination**: Apps enabling field-based coordinators to access and update patient information
- **Predictive intervention matching**: AI recommending which interventions each patient is most likely to respond to

### Underserved Areas / Opportunities

Gaps that represent genuine opportunities for a new entrant:

- **Lightweight, SMB-focused care management**: Most platforms target large health systems. Simpler, lower-cost offering for community health centers, small primary care practices would serve large market.
- **SDOH as core, not bolted-on**: Most platforms add SDOH screening as feature. Platform designed from ground up around SDOH identification, resource navigation, and outcome tracking could differentiate.
- **Community resource management integrated**: Social workers and community health workers need community resource databases with eligibility checking, referral tracking, and outcomes. Currently fragmented across separate tools.
- **Patient experience transparency**: Most platforms optimise care coordinator efficiency. Tools centred on patient experience (showing care plan progress, enabling goal setting, celebrating wins) less common.
- **Outreach effectiveness optimization**: Which outreach modality (SMS vs. call vs. email) works best for which patient? When is best time to reach out? Currently manual decisions; ML-optimised outreach targeting could differentiate.
- **Social network effects**: Care coordinators working with the same patient families should collaborate. Platforms enabling family-level care plans not common.
- **Financial transparency for patients**: Tools showing patients how their care translates to reduced costs, better outcomes, improved quality measures less common.
- **Behavioral health integration**: Mental health and substance use highly prevalent in high-risk populations; integration with behavioral health EHRs and care management less common.
- **Predictive intervention matching**: Which patients most likely to respond to which interventions? Currently care plan design is standardised; ML-optimised programs could improve outcomes.
- **Workload prediction and surge capacity**: Predicting care coordinator workload weeks in advance enabling better scheduling. Most systems reactive.

### AI-Augmentation Candidates

Features currently implemented with manual/rule-based approaches but where AI could meaningfully excel:

- **Risk model validation and equity assessment**: Detecting bias in risk models against under-represented populations; auto-recalibrating to correct for disparities
- **Outreach timing optimization**: ML models predicting optimal time to contact each patient (based on historical response patterns)
- **Intervention matching**: Predicting which patients will respond to which interventions based on patient characteristics, preferences, past responses
- **Care plan auto-generation**: Drafting initial care plans based on patient diagnosis, risk factors, goals; care coordinator reviews and personalizes
- **Contact rate improvement**: Predicting which patients are at risk of disengaging; proactive intervention recommendations
- **Quality measure prediction**: Showing coordinators which patients are at risk of missing quality measures; recommending targeted interventions
- **Burnout prediction and prevention**: Identifying care coordinators at risk of burnout; recommending workload adjustments, peer support
- **Readmission risk prediction**: ML models trained on historical readmission patterns identifying high-risk patients for preventive intervention
- **Medication adherence prediction**: Identifying patients likely to be non-adherent; recommending adherence support strategies
- **Social determinant impact on health**: Quantifying impact of SDOH interventions (housing, food assistance, transportation) on health outcomes
- **Network analysis**: Identifying which providers/health systems/coordinators have best outcomes with specific patient populations
- **Cost-effectiveness analysis**: Recommending most cost-effective interventions for each patient based on claims and outcomes data

## Legal & IP Summary

All platforms analysed are proprietary commercial services with no open-source alternatives identified. Risk stratification algorithms, disease management program designs, and data aggregation/normalisation approaches are proprietary to each vendor. Several platforms (particularly Epic, Oracle, Innovaccer) have patent portfolios related to risk stratification, care gap identification, and data normalisation, but no specific patents were flagged as impeding new development. SDOH screening instruments (AHC-HRSN, PRAPARE) used across platforms are publicly available and licence-free. CMS CCM billing requirements (CPT codes, documentation standards) are public regulatory requirements, not IP-encumbered. No material was omitted due to IP uncertainty.

## Recommended Feature Scope

Based on the above analysis, here is a prioritised feature scope for a new Care Management Platform:

**Must-have (MVP)**
- Risk stratification engine combining claims, clinical, and demographic factors
- Chronic disease registries for diabetes, hypertension, COPD, heart failure, depression
- Care gap identification aligned to quality measures (HEDIS, MIPS, value-based contracts)
- Patient outreach with multi-channel communication (SMS, email, portal, phone)
- Care plan documentation with problem list, goals, interventions, outcomes tracking
- Care team workflow tools with task assignment and caseload visibility
- EHR and claims data integration for longitudinal patient records
- Quality measure performance reporting and drill-down analytics
- CMS CCM billing compliance with time tracking and documentation requirements

**Should-have (v1.1)**
- SDOH screening with validated instruments and community resource referral
- Mobile app for field-based care coordinators enabling real-time updates
- Explainable risk scoring showing factors contributing to patient risk
- AI-driven intervention recommendations (not just risk identification)
- Outcomes measurement and pre/post intervention analysis
- Outreach effectiveness metrics (contact rates, engagement, ROI)
- Dynamic caseload balancing matching patients to coordinator skills/capacity
- Predictive readmission risk scoring and intervention recommendations
- Integration with behavioral health EHRs and substance use programs

**Nice-to-have (backlog)**
- AI-optimised outreach timing and modality selection per patient
- Family-level care plans enabling multi-family member coordination
- Patient-facing portal showing care plan progress and celebrating wins
- Financial transparency showing care cost reduction and ROI
- Burnout prediction and prevention for care coordinator workforce
- Medication adherence prediction with support strategy recommendations
- Social determinant impact quantification (housing/food/transportation outcomes)
- Network analysis identifying high-performing provider/coordinator combinations
- Advanced cost-effectiveness analysis recommending optimal interventions per patient
- Workload forecasting enabling surge capacity planning
