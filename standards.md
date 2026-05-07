# Standards & API Reference

> Project: Anti-Money Laundering Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### Regulatory Frameworks

**FATF 40 Recommendations (Financial Action Task Force)**
- URL: https://www.fatf-gafi.org/en/publications/Fatfrecommendations/Fatf-recommendations.html
- The global baseline standard for AML/CFT/CPF compliance. Covers risk-based approach, customer due diligence (CDD), suspicious transaction reporting, correspondent banking, and new technologies including virtual assets. Amended in February 2025 to improve financial inclusion proportionality. Any AML platform must map its detection coverage and reporting workflows to these 40 recommendations. The April 2026 Ministerial Declaration prioritises fraud and payment transparency.

**FATF Methodology for Mutual Evaluations**
- URL: https://www.fatf-gafi.org/en/publications/Mutualevaluations/Fatf-methodology.html
- The 2022 assessment methodology against which national AML regimes are measured. Compliance teams use it to benchmark their programme effectiveness. An AML platform should align its risk assessment outputs with the effectiveness metrics in this methodology.

**EU AML Package — Regulation (EU) 2024/1620 establishing AMLA, Regulation (EU) 2024/1624 (Single Rulebook), Directive (EU) 2024/1640**
- URL: https://www.amla.europa.eu/about-amla_en and https://eur-lex.europa.eu/EN/legal-content/summary/authority-for-anti-money-laundering-and-countering-the-financing-of-terrorism.html
- The EU's unified AML/CFT framework. The Single Rulebook (applies from 10 July 2027) replaces fragmented national rules with harmonised CDD, transaction monitoring, and reporting obligations. The Authority for Anti-Money Laundering (AMLA) became operational 1 July 2025 and will directly supervise 40 large high-risk institutions from 2028. An AML platform targeting EU deployment must support Single Rulebook CDD tiers and AMLA reporting requirements.

**US Bank Secrecy Act (BSA) / FinCEN Regulations**
- URL: https://www.fincen.gov/resources/filing-information
- The primary US AML statute. Requires financial institutions to maintain AML programmes, file Suspicious Activity Reports (SARs) and Currency Transaction Reports (CTRs), and conduct Customer Due Diligence. FinCEN maintains the BSA E-Filing System for electronic SAR/CTR submission.

**FFIEC BSA/AML Examination Manual**
- URL: https://bsaaml.ffiec.gov/manual
- The authoritative US regulatory examination guide for BSA/AML programmes. Covers transaction monitoring system adequacy, SAR filing processes, and examiner procedures. Updated February 2026. AML platforms must produce audit trails, alert disposition histories, and SAR documentation that satisfy FFIEC examiner review. October 2025 SAR FAQ update provides current guidance on continuing-activity filing and relationship maintenance decisions.

**Federal Reserve SR 11-7 / SR 21-8 — Model Risk Management Guidance**
- URL: https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm
- SR 11-7 (2011) establishes the model validation framework applied to all quantitative models at US-regulated institutions, including ML-based AML detection. SR 21-8 (2021) explicitly extended SR 11-7 to BSA/AML models. AML platforms must support model documentation (conceptual soundness, data integrity, outcome analysis), independent validation evidence, and model performance monitoring to satisfy these requirements.

**Wolfsberg Group Financial Crime Principles**
- URL: https://wolfsberg-group.org/resources/
- Industry principles developed by 12 global banks covering correspondent banking due diligence, payment transparency, PEP risk management, and trade finance. The Correspondent Banking Due Diligence Questionnaire (CBDDQ) and Financial Crime Compliance Questionnaire (FCCQ) are the global standard for interbank AML due diligence. Platforms targeting correspondent banking use cases should support CBDDQ data exchange.

### Financial Messaging Standards

**ISO 20022 — Universal Financial Industry Message Scheme**
- URL: https://www.swift.com/standards/iso-20022/iso-20022-standards
- The global standard for structured financial messaging, replacing SWIFT MT formats (migration completed November 2025). ISO 20022 MX messages carry richer structured data — originator/beneficiary full names, addresses, and LEIs — enabling more accurate sanctions screening, name matching, and transaction pattern analysis. AML platforms ingesting ISO 20022 messages can reduce false positives through richer context. CBPR+ usage guidelines available on SWIFT MyStandards.

**ISO 15022 — SWIFT MT Legacy Formats**
- URL: https://www.swift.com/standards
- The predecessor messaging standard (FIN/MT messages). Although ISO 20022 is now mandatory for cross-border payments, many domestic and bilateral connections still operate on MT formats during transition. AML platforms serving established banks should support both MT and MX ingestion.

**SWIFT Standards Developer Kit (SDK)**
- URL: https://www.swift.com/ordering-support/developer-resource-centre/design/standards
- Developer resources and XML Schema files for ISO 20022 implementation. Provides CBPR+ XML schemas, PDF and Excel specification downloads for payment message types relevant to correspondent banking monitoring.

### Reporting & Taxonomy Standards

**UNODC goAML XML Schema (v5.0.1)**
- URL: https://www.unodc.org/unodc/en/global-it-products/goaml.html and https://unite.un.org/goaml/
- goAML is UNODC's standard software solution for Financial Intelligence Units (FIUs) used in 60+ countries. Regulated institutions submit Suspicious Transaction Reports (STRs), Cash Transaction Reports (CTRs), and Electronic Funds Transfer (EFT) reports as XML files conforming to the goAML XSD schema (current version 5.0.1). An AML platform targeting non-US markets must generate goAML-compliant XML exports for regulator submission.

**FinCEN SAR/CTR Electronic Filing Specifications**
- URL: https://www.fincen.gov/news/news-releases/important-notices-e-filers-fincen-releases-technical-e-filing-specifications-new and https://www.fincen.gov/system/files/shared/e-filing_GENspecs.pdf
- Technical specifications for batch electronic filing of SARs and CTRs with the BSA E-Filing System. Fixed-length ASCII record format. Institutions with high filing volumes use batch submission; AML platforms must generate conformant batch files. FinCEN also provides BSA Direct e-filing for direct programmatic submission.

### Data & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The standard for describing REST APIs, fully compatible with JSON Schema 2020-12. Leading AML API providers (ComplyAdvantage, Unit21) publish OpenAPI 3.x specifications. An AML platform should expose its transaction ingestion, alert, and case management APIs as OpenAPI 3.1 documents to maximise integration ease.

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification.html
- Standard for validating JSON data structures. Used to define and validate AML event payloads, transaction objects, entity profiles, and alert structures. Enables strict contract testing between transaction source systems and the AML platform.

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL: https://datatracker.ietf.org/doc/html/rfc6749 and https://openid.net/specs/openid-connect-core-1_0.html
- Industry-standard authentication and authorisation protocols for API access. AML platform APIs should require OAuth 2.0 bearer tokens (preferably with PKCE for public clients) and support OpenID Connect for identity federation with financial institution identity providers.

**GDPR (Regulation (EU) 2016/679)**
- URL: https://gdpr-info.eu/
- Governs processing of personal data in the EU, including transaction and customer data held in AML systems. Impacts data retention schedules, subject access rights, and cross-border data transfer restrictions. AML platforms must support configurable data residency, retention policies, and audit logs of personal data access to satisfy GDPR obligations.

**EU Digital Operational Resilience Act (DORA) — Regulation (EU) 2022/2554**
- URL: https://www.digital-operational-resilience-act.com/
- Applies from January 2025. Imposes ICT risk management, incident reporting, resilience testing, and third-party risk requirements on EU financial institutions and their critical ICT providers. AML platforms sold to EU institutions may qualify as critical third-party ICT providers and face DORA oversight. Platforms must support DORA-compliant incident classification and reporting.

### Graph Technology Standards

**Cypher Query Language / openCypher**
- URL: https://opencypher.org/
- The declarative query language for graph databases (Neo4j and others). AML platforms using graph databases for network analytics should use Cypher (or its openCypher open standard) for entity relationship queries, pattern detection (ring structures, layering paths), and community detection. openCypher is vendor-neutral and is being standardised under ISO GQL.

**ISO/IEC 39075 — GQL (Graph Query Language)**
- URL: https://www.iso.org/standard/76120.html
- Emerging ISO standard for graph database querying, building on Cypher/SQL-PGQ. Relevant to long-term portability of AML graph queries across graph database backends.

---

## Similar Products — Developer Documentation & APIs

### Unit21

- **Description:** API-first AML and fraud operations platform targeting fintechs and mid-market financial institutions. Supports transaction ingestion, rule-based and ML-based alert generation, case management, and automated SAR/goAML filing.
- **API Documentation:** https://docs.unit21.ai/docs
- **SDKs/Libraries:** REST/JSON; official SDKs not publicly listed but API is documented with OpenAPI-compatible schemas at docs.unit21.ai
- **Developer Guide:** https://docs.unit21.ai/
- **Standards:** REST/JSON, OpenAPI-compatible
- **Authentication:** API key with Bearer token; OAuth 2.0 for enterprise integrations
- **Key Integration Notes:** Ingests entity (individual/business), transaction, and instrument objects. Supports webhooks for alert and case status events. Auto-files FinCEN SARs and goAML STRs.

### ComplyAdvantage (Mesh API)

- **Description:** AI-driven sanctions, PEP, and adverse media screening with ongoing monitoring. REST API with JSON responses following the OpenAPI specification.
- **API Documentation:** https://docs.complyadvantage.com/api-docs
- **SDKs/Libraries:** REST/JSON; integration guides for JavaScript and Python available through the developer portal
- **Developer Guide:** https://complyadvantage.com/integration/
- **Standards:** OpenAPI 3.x, REST/JSON
- **Authentication:** API key
- **Key Integration Notes:** Screening API supports batch customer onboarding and real-time event-driven checks. Ongoing monitoring webhooks trigger on watchlist changes. Fuzzy name matching with configurable thresholds.

### Feedzai

- **Description:** Enterprise ML-based financial crime and fraud platform. Open ML API (feedzai-openml) allows integration of custom ML models into the Feedzai runtime environment.
- **API Documentation:** http://dev.feedzai.com/rest-api/ (requires portal registration); Open ML API: https://github.com/feedzai/feedzai-openml
- **SDKs/Libraries:** Java client library (https://github.com/killbill/feedzai-client); Maven dependencies for OpenML API
- **Developer Guide:** https://support.feedzai.com/hc/en-us
- **Standards:** REST/JSON with Swagger documentation; OpenML API uses Java/Maven
- **Authentication:** OAuth 2.0 / Access Key / OpenID Connect (per NICE Actimize-documented pattern for enterprise AML platforms)
- **Key Integration Notes:** Developer portal is open to all; OpenML library is Apache 2.0 licensed allowing custom model integration. Feedzai publishes REST API guidelines using Swagger.

### NICE Actimize

- **Description:** Enterprise financial crime and compliance platform covering AML, fraud, and market surveillance. Provides REST API library for alert, investigation, and case management customisation.
- **API Documentation:** https://docs.niceactimize.com/
- **SDKs/Libraries:** REST/JSON APIs; NICE Actimize ETL documentation at https://nice-actimize-etl.readthedocs.io/
- **Developer Guide:** https://expert-help.nice.com/Integrations_and_Extending_Content/API
- **Standards:** REST/JSON; OpenAPI-described endpoints
- **Authentication:** OAuth 2.0, Access Key, or OpenID Connect (per HTTPS bearer token model)
- **Key Integration Notes:** ActOne Extend API supports custom workflow extensions. ETL pipeline documentation covers data ingestion from core banking and payment systems.

### Elliptic (Crypto AML)

- **Description:** Blockchain analytics and crypto AML compliance platform. API supports wallet and transaction risk scoring, batch AML submissions, and workflow automation for crypto-asset businesses.
- **API Documentation:** https://developers.elliptic.co/docs
- **SDKs/Libraries:** Node.js, PHP, Python SDKs — https://developers.elliptic.co/docs/quick-start-sdks
- **Developer Guide:** https://developers.elliptic.co/docs/getting-started
- **Standards:** REST/JSON
- **Authentication:** HMAC API key and secret (per-request signing)
- **Key Integration Notes:** Single Analysis endpoint returns synchronous risk score. Batch endpoint handles bulk wallet/transaction submissions. SDK abstracts authentication signing. Supports automated workflow actions (label, assign, archive customers) via API.

### Chainalysis (Crypto AML / KYT)

- **Description:** Know Your Transaction (KYT) platform for blockchain analytics. Provides real-time crypto transaction risk scoring and address screening.
- **API Documentation:** https://docs.chainalysis.com (access requires business registration)
- **SDKs/Libraries:** Available via Chainalysis developer portal; integrations documented through Dfns (https://docs.dfns.co/integrations/aml-kyt/chainalysis) and Sumsub (https://docs.sumsub.com/docs/chainalysis)
- **Developer Guide:** Available at Chainalysis developer portal upon agreement
- **Standards:** REST/JSON
- **Authentication:** API key; access requires verified business compliance use case
- **Key Integration Notes:** Free crypto sanctions screening API available publicly. KYT supports integration with Fireblocks, Sumsub, and custody platforms. Covers 100+ blockchains. De facto industry standard for crypto AML.

### Sumsub

- **Description:** Identity verification, KYC/KYB, and AML screening platform. REST API covering applicant onboarding, document verification, sanctions/PEP screening, and transaction monitoring with integrated ComplyAdvantage AI risk intelligence (added March 2026).
- **API Documentation:** https://docs.sumsub.com/reference/about-sumsub-api
- **SDKs/Libraries:** Available at https://docs.sumsub.com/; mobile SDKs (iOS, Android) for in-app verification flows
- **Developer Guide:** https://docs.sumsub.com/reference/get-started-with-api
- **Standards:** REST/JSON
- **Authentication:** App token (HMAC signed requests)
- **Key Integration Notes:** Covers full KYC-to-AML pipeline in a single API. Transaction monitoring integrated with KYC applicant profiles. March 2026 ComplyAdvantage integration adds AI-driven adverse media risk scoring.

### OFAC-API (SDN Screening)

- **Description:** Third-party API wrapping the US Treasury OFAC Specially Designated Nationals (SDN) list and other sanctions lists. Provides consolidated screening with fuzzy matching, refreshed every 2 minutes.
- **API Documentation:** https://docs.ofac-api.com/
- **SDKs/Libraries:** REST/JSON; no official SDK but straightforward HTTP integration
- **Developer Guide:** https://www.ofac-api.com/developers
- **Standards:** REST/JSON; OpenAPI-described (https://sdn-openapi.netlify.app/)
- **Authentication:** API key
- **Key Integration Notes:** Supports SDN, EU Financial Sanctions, and 30+ other lists in a single request. Batch Job API handles up to 1 million records for ongoing screening. Jaro-Winkler and Soundex phonetic fuzzy matching built in. Free tier available.

---

## Notes

**Emerging standards to monitor:**
- ISO GQL (ISO/IEC 39075) is expected to standardise graph database queries across vendors, which will affect AML network analytics layer portability.
- FATF's ongoing work on virtual asset guidance (Recommendation 15 / Travel Rule) is evolving rapidly. The FATF Travel Rule (requiring originator/beneficiary information on crypto transfers) is implemented unevenly across jurisdictions; platforms targeting crypto-asset businesses must track FATF VASP guidance updates.
- AMLA's technical regulatory standards (from the EU Single Rulebook, applying 2027) will specify mandatory transaction monitoring thresholds and SAR filing formats for EU institutions — these are still being drafted as of May 2026.

**Data licensing note:**
Sanctions list data (OFAC SDN, UN Consolidated List, EU Financial Sanctions) is publicly available without licence fees. However, commercial PEP database providers (Dow Jones, Refinitiv World-Check, LexisNexis) are proprietary and require data licensing agreements. An open-source AML platform should rely on publicly available sanctions lists and clearly document which PEP/adverse media sources require third-party licensing.
