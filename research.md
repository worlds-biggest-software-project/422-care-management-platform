# Project 422 – Care Management Platform

_Research date: 2026-05-02_

---

## 1. Problem Statement

Chronic disease is the dominant driver of healthcare cost and poor population health outcomes in the United States and comparable high-income countries. Conditions such as diabetes, heart failure, COPD, hypertension, and depression are responsible for roughly 90% of the $4.5 trillion in annual US healthcare spending, yet a large proportion of that spending is attributable to avoidable complications and care gaps—missed preventive services, fragmented follow-up, and patients who fall through the cracks between acute episodes.

Care management programs—disease management, chronic care management (CCM), and complex case management—were designed to bridge these gaps by proactively identifying at-risk patients, enrolling them in structured support programs, and coordinating care across providers. The challenge is operational scale: identifying which patients need what intervention, outreaching them effectively across a diverse and often digitally variable population, documenting care-plan progress, and measuring improvement against quality benchmarks.

Manual care management processes break down rapidly at population scale. Without software to stratify risk, close care gaps systematically, and automate routine outreach, programs remain reactive rather than preventive, and quality-measure performance suffers accordingly.

---

## 2. Existing Landscape

The population health and care management software market is mature, competitive, and rapidly incorporating AI:

- **Epic Healthy Planet** – Embedded within the Epic EHR and holding approximately 42% of the acute-care market, Healthy Planet provides risk stratification, chronic disease registries, care gap alerts, and quality measure tracking directly within the patient chart. Its deep integration with Epic clinical workflows gives it a structural advantage in Epic-heavy health systems.
- **Azara Healthcare** – Named Best in KLAS for Population Health Management four consecutive years (2023–2026), with a score of 93.5 in 2026. Following a 2025 merger with i2i Population Health, Azara now covers 25 million-plus Americans across more than 1,000 community health centers—making it the dominant platform in the Federally Qualified Health Center (FQHC) segment.
- **Innovaccer** – Ranked by Black Book Research as the top AI-driven population health platform in both 2025 and 2026, Innovaccer aggregates multi-source data, applies AI-driven risk models, and activates care gap interventions within a unified data platform.
- **Arcadia** – A data aggregation and analytics platform with demonstrated outcomes: one Northeast ACO using Arcadia's tools achieved a 41.5% reduction in ED visits for COPD patients.
- **Oracle Health HealtheIntent** – Part of the Oracle Health (formerly Cerner) ecosystem, offering longitudinal patient record aggregation, risk stratification, and care gap management across large health systems.
- **Persivia CareSpace** – Ranked among the top five PHM platforms for 2025, offering modular disease management programs, predictive risk scoring, and care team workflow tools.
- **Salesforce Health Cloud** – CRM-rooted platform increasingly adopted for care management workflows, particularly for payer-side care management and complex case coordination.

---

## 3. Key Functional Requirements

A comprehensive Care Management Platform must support:

1. **Risk stratification engine** – Multi-factor scoring models (claims, clinical, social determinants of health) that segment the population by risk tier and identify candidates for care management enrollment.
2. **Disease management program templates** – Configurable evidence-based protocols for high-priority conditions (diabetes, hypertension, COPD, CHF, depression), including goal setting, intervention checklists, and escalation pathways.
3. **Care gap identification and closure tracking** – Automated detection of overdue preventive services, missing lab values, medication adherence gaps, and unmet screening targets, mapped to HEDIS and other quality measures.
4. **Population outreach tools** – Multi-channel patient outreach (automated call, SMS, email, patient portal message) with configurable messaging and consent management.
5. **Care plan authoring and management** – Collaborative care plan documentation accessible to all care team members, with problem list, goal tracking, intervention history, and patient-reported outcomes.
6. **Care team coordination** – Task assignment and workflow routing among care coordinators, nurses, social workers, and physicians, with caseload management dashboards and capacity visibility.
7. **EHR and claims integration** – Bidirectional data feeds from primary EHRs, health information exchanges (HIEs), pharmacy benefit managers, and payer claims data to build longitudinal patient records.
8. **Quality measure reporting** – Real-time performance dashboards for HEDIS, MIPS, and value-based contract measures, with drill-down to individual patient opportunities.
9. **Social determinants screening** – Validated SDOH screening instrument support (AHC-HRSN, PRAPARE), community resource referral integration, and SDOH data capture in the longitudinal record.
10. **Analytics and outcomes reporting** – Cohort-level and program-level outcome tracking, pre/post intervention analysis, and exportable reporting for payer performance contracts.

---

## 4. Technical Challenges

- **Data aggregation and normalization** – Care management depends on a complete longitudinal patient view, which requires integrating disparate sources (multiple EHRs, claims, labs, pharmacy, SDOH) with inconsistent formats, coding systems, and patient-matching logic. Master patient index accuracy is critical.
- **Risk model validity and equity** – Many commercial risk models are trained on claims data that reflects historical care-utilization patterns, which can underestimate risk in under-served populations who have used less care. Bias in risk stratification leads to inequitable program enrollment.
- **Outreach effectiveness** – Achieving meaningful contact rates with chronically ill patients, especially those with language barriers, low health literacy, or limited digital access, requires multilingual, multi-modal outreach and a strategy for non-responders.
- **Care coordinator caseload management** – Matching patients to appropriately skilled coordinators and sustaining manageable caseloads as program enrollment scales requires dynamic workload-balancing algorithms.
- **Regulatory and billing compliance** – CMS CCM billing codes (CPT 99490, 99491, 99487, 99489) require specific documentation of time, consent, care plan elements, and care team composition. Software must enforce these requirements without creating excessive documentation burden.
- **Interoperability at scale** – As the CMS ACCESS model (launching July 2026) ties $15–$35 PMPM CCM payments to clinical improvement targets, platforms must demonstrate attributable outcomes—a significant data infrastructure and analytics challenge.

---

## 5. Market Opportunity

The chronic disease management application market is projected to reach $20.28 billion by 2032, growing at a CAGR of 10.32%, driven by aging populations, the rise of value-based payment models, and expanding CMS chronic care management reimbursement structures. The CMS ACCESS model launching in July 2026 introduces performance-contingent payments of $15–$35 per member per month for enrolled chronic care management programs, creating strong financial incentives for health systems and payer-sponsored care management programs to invest in sophisticated platforms.

Target buyers span health systems with value-based contracts, Medicare Advantage plans, Medicaid managed care organizations, Federally Qualified Health Centers, and Accountable Care Organizations. The convergence of fee-for-service CCM billing, value-based performance bonuses, and rising chronic disease prevalence creates multiple revenue streams that make the ROI case for platform investment compelling.

---

## Sources

- [Top Population Health Management Software Vendors In 2025 – Innovaccer](https://innovaccer.com/blogs/top-population-health-management-software)
- [9 Leading Chronic Care Management Software Picks of 2026 – Arcadia](https://arcadia.io/resources/chronic-care-management-software)
- [Best Healthcare Provider PHM Solutions Reviews 2026 – Gartner Peer Insights](https://www.gartner.com/reviews/market/healthcare-provider-population-health-management-platforms)
- [Chronic Disease Management Software: 2026 Guide – SCNSoft](https://www.scnsoft.com/healthcare/chronic-disease-management-software)
- [Best Population Health Management Software: 2026 Edition – Folio3 Digital Health](https://digitalhealth.folio3.com/blog/best-population-health-management-software/)
- [Top Digital Solutions for Disease Management in 2025 – Mahalo Health](https://www.mahalo.health/insights/22-top-disease-management-alternatives-in-2025)
- [Care Management Software Overview and 10 Options for 2026 – Arcadia](https://arcadia.io/resources/care-management-software)
- [Top 5 Population Health Management Software in 2025 – Persivia](https://persivia.com/2025/10/01/population-health-management-software/)
