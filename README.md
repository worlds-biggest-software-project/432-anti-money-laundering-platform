# Anti-Money Laundering (AML) Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source AML platform that gives financial institutions real-time transaction monitoring, intelligent alert triage, and automated SAR filing — without enterprise-only pricing or year-long implementation cycles.

Financial institutions of every size must detect, investigate, and report suspicious transactions to satisfy AML regulations enforced by bodies like FinCEN, the FCA, and AUSTRAC. This platform delivers the core compliance stack — transaction monitoring, case management, sanctions screening, and regulatory reporting — with AI capabilities that dramatically reduce false-positive rates and analyst workload. It targets fintechs, mid-market banks, and payment processors that are underserved by today's enterprise-only incumbents.

---

## Why Anti-Money Laundering (AML) Platform?

- **Enterprise lock-in and slow deployment.** Platforms like Feedzai, SymphonyAI, and NICE Actimize require months-long implementations and enterprise contracts, putting capable AML tooling out of reach for fintechs and smaller institutions.
- **High false-positive rates drain compliance teams.** Legacy rules-based systems generate alert volumes that overwhelm analysts, with the vast majority of alerts turning out to be benign. AI-driven triage is needed but only available in expensive commercial offerings.
- **No open-source alternative exists.** Every meaningful AML transaction monitoring solution on the market is proprietary SaaS. The core detection algorithms (peer group analysis, velocity checks, structuring detection) are published in FATF guidance and academic literature with no patent barriers — an open-source implementation is viable.
- **Crypto and digital asset coverage is bolted on, not native.** Even leading platforms like SymphonyAI require third-party integrations (Elliptic, Chainalysis) for blockchain transaction monitoring rather than offering it as a first-class capability.
- **Mid-market pricing gap.** Institutions processing under 500,000 monthly transactions lack an AML platform that combines ML-grade detection quality with accessible pricing and rapid deployment.

---

## Key Features

### Transaction Monitoring

- Real-time and batch analysis of financial transactions against configurable rule engines and ML behavioural models
- Velocity checks, structuring detection, and amount-threshold rules as baseline detection logic
- Entity-level behavioural profiling that tracks continuous patterns, not just individual events
- Coverage across payment rails: wire, ACH, SEPA, Faster Payments, and card transactions

### Alert Triage and Case Management

- Risk-scored alert queue with analyst claim/assign workflow and bulk disposition for high-volume low-risk patterns
- Structured investigation workflows with case opening, assignment, escalation, documentation, and closure
- Immutable audit trail of all analyst actions and decisions for regulatory examination readiness
- Entity profile 360-degree view combining transaction history, network connections, and screening results

### SAR and Regulatory Reporting

- Automated SAR/STR generation with editable narrative fields and regulatory submission workflow
- Multi-jurisdiction format support mapping a common template to country-specific regulator requirements (FinCEN, FCA, AUSTRAC)
- Currency Transaction Report (CTR) automated generation alongside SAR workflow
- AI-assisted narrative drafting from structured case evidence with analyst review before filing

### Sanctions and PEP Screening

- Real-time screening of counterparties against OFAC, UN, EU, and other consolidated sanctions lists
- Fuzzy matching for name variations, transliterations, and aliases
- Watchlist update automation and audit logging of screening decisions

### Network Analytics

- Graph-based visualisation of entity relationships and transaction flows
- Detection of layering structures and mule account networks not visible in individual-transaction analysis
- Investigation timeline showing the sequence of events leading to an alert

---

## AI-Native Advantage

AI transforms AML compliance at three critical points. First, ML-based behavioural anomaly detection augments static rules with entity-level profile scoring, catching sophisticated typologies like layering networks and mule accounts while dramatically reducing false positives. Second, LLM-powered SAR narrative generation — identified across the industry as the highest-value near-term AI application in AML — converts structured case data into publication-quality regulatory filings in seconds instead of hours. Third, unsupervised typology discovery identifies novel money laundering patterns that no existing rule covers, keeping detection ahead of evolving criminal methods. All AI decisions include explainable rationale to satisfy regulatory model validation requirements.

---

## Tech Stack & Deployment

- **API-first architecture**: REST API for transaction event ingestion from any upstream payment system, with webhook notifications for alert and case status changes
- **Graph database** for entity relationship and network analysis (open-source options include ArangoDB under Apache 2.0)
- **Deployment modes**: self-hosted, cloud, or hybrid, with data residency controls for multi-jurisdiction requirements
- **Integration targets**: core banking systems, payment processors (Stripe, Plaid), SWIFT, and regulatory submission gateways
- **Model governance**: drift monitoring, revalidation scheduling, and documentation aligned with SR 11-7 guidance

---

## Market Context

Financial crime costs the global economy an estimated 2-5% of GDP annually, driving sustained regulatory enforcement and compliance spending. Incumbent pricing is enterprise-tier, based on transaction volume or seat counts, with vendors like Feedzai, SymphonyAI, and NICE Actimize targeting large banks and payment networks. The primary buyers are compliance officers, heads of financial crime, and engineering teams at fintechs and mid-market banks seeking capable AML tooling without enterprise overhead.

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
