# Master Vision Document v2.0: "Project Aegis"
## The Sovereign Banking Advisor with OWASP LLM Top 10 2025 Compliance

**Document Version**: 2.0  
**Last Updated**: December 11, 2025  
**Status**: Active Development  
**Classification**: Internal - Banking Grade

---

## Document Suite Structure

This Master Vision Document is organized into four interconnected files:

1. **vision.md** - Strategic Vision & Philosophy
2. **project-overview.md** - Governance & Roadmap
3. **architecture-master.md** - Technical Design & OWASP Integration
4. **blueprint.md** - Engineering Specs & Security NFRs

---

# File 1: vision.md

## 1. Executive Summary

Project Aegis aims to deploy a **Regulation-Grade GenAI Advisor** that empowers financial advisors with instant, accurate, and compliant retrieval of institutional knowledge. Unlike standard chatbots, Aegis is built on a "Zero-Trust" architecture where every token generated is traceable, auditable, and defensively engineered against hallucination, jailbreaks, and the **OWASP LLM Top 10 2025** vulnerabilities.

### Key Differentiators (v2.0)
- **OWASP LLM Top 10 2025 Native**: Architecture designed around the latest GenAI security framework
- **Dual Observability**: Grafana (infrastructure) + Arize Phoenix (AI-specific) monitoring
- **Defense-in-Depth**: Security controls at input, processing, and output layers
- **Automated Red Teaming**: Continuous attack simulation using Giskard/PyRIT

## 2. Core Philosophy

### The Aegis Principles

1. **Defensibility**: If we cannot explain *how* the model reached a conclusion, we do not show it.
2. **Safety First**: The system defaults to "I don't know" rather than hallucinating financial advice.
3. **Immutable Audit**: Every interaction, thought process, and tool execution is preserved for 5 years, immutable and searchable by compliance teams.
4. **Security by Design**: OWASP LLM Top 10 2025 risks mitigated at every architectural layer.
5. **Observability as Code**: Tracing, evaluation, and monitoring are first-class citizens, not afterthoughts.

## 3. Strategic Goals

### Business Objectives

1. **Advisor Efficiency**: Reduce research time by 40% using RAG over 100k+ banking documents.
2. **Risk Mitigation**: 100% automated redaction of MNPI (Material Non-Public Information) and PII before it touches the LLM.
3. **Regulatory Compliance**: Full adherence to:
   - **SR 11-7** (Model Risk Management)
   - **EU AI Act** (Transparency & High-Risk AI requirements)
   - **GDPR** (Right to be Forgotten via Crypto-Shredding)
   - **CCPA** (Data Privacy & Deletion Rights)
   - **OWASP LLM Top 10 2025** (GenAI Security Best Practices)

### Security Goals (NEW in v2.0)

| **OWASP Risk** | **Aegis Mitigation Strategy** | **Target Metric** |
|----------------|-------------------------------|-------------------|
| **LLM01: Prompt Injection** | NVIDIA NeMo Guardrails + Input Sanitization | 100% detection rate |
| **LLM02: Sensitive Info Disclosure** | Presidio PII Scrubber + Crypto-Shredding | 0% PII leakage |
| **LLM03: Supply Chain** | SBOM Generation + CVE Scanning | 0 high/critical CVEs |
| **LLM04: Data Poisoning** | Golden Set Validation + Embedding Drift Detection | <5% drift tolerance |
| **LLM05: Improper Output Handling** | DOMPurify + Output Schema Validation | 0 XSS vulnerabilities |
| **LLM06: Excessive Agency** | HITL Controls + Least Privilege Tools | 100% high-risk HITL |
| **LLM07: System Prompt Leakage** | Env Var Storage + Prompt Isolation | 0 leakage incidents |
| **LLM08: Vector/Embedding Weaknesses** | Phoenix Drift Monitoring + RBAC Filters | <0.3 drift score |
| **LLM09: Misinformation** | RAG Triad + LLM-as-a-Judge | >95% groundedness |
| **LLM10: Unbounded Consumption** | Circuit Breakers + Rate Limiting | <$0.05/turn avg cost |

## 4. Success Metrics (KPIs)

### Accuracy & Quality
- **Golden Set Pass Rate**: >98% on 500+ compliance-approved Q&A pairs
- **RAG Triad Scores**:
  - Context Relevance: >0.85
  - Groundedness: >0.90
  - QA Relevance: >0.85
- **Hallucination Rate**: <2% (measured via Arize Phoenix LLM-as-a-Judge)

### Performance
- **Latency**: <3 seconds time-to-first-token (TTFT) for standard queries
- **Availability**: 99.9% uptime (excluding planned maintenance)
- **Throughput**: Support 500 concurrent advisors

### Security
- **OWASP Compliance**: 100% coverage of LLM Top 10 2025 mitigations
- **PII Leakage**: 0% unmasked PII in logs or responses
- **Attack Detection**: >99% detection of red team attacks
- **Prompt Injection Block Rate**: 100% for known attack patterns

### Cost & Efficiency
- **Cost per Turn**: <$0.05 average (monitored via Phoenix + Grafana)
- **Research Time Reduction**: 40% (measured via time-tracking integration)
- **False Positive Rate**: <5% (answers flagged as incorrect but are actually correct)

## 5. Risk Management Framework

### Critical Risks & Mitigations

| **Risk** | **Likelihood** | **Impact** | **Mitigation** | **Owner** |
|----------|----------------|------------|----------------|-----------|
| LLM hallucinates material financial advice | High | Critical | Golden Set evals + Citation enforcement | AI Team |
| PII exposure in Vector DB | Medium | Critical | Presidio scrubbing before embedding | Security Team |
| Prompt injection bypasses guardrails | High | High | Multi-layer defense (NeMo + Phoenix monitoring) | Security Team |
| Model poisoning via fine-tuning data | Low | Critical | Baseline validation + Drift detection | AI Team |
| Cost overrun (>$10k/month) | Medium | Medium | Circuit breakers + Token budgets | Product Team |
| Regulatory audit failure | Low | Critical | Immutable audit logs + Quarterly attestation | Compliance Team |

### Incident Response Plan

1. **Detection**: Phoenix alerts + Grafana alarms
2. **Triage**: On-call engineer investigates within 15 minutes
3. **Containment**: Circuit breaker activates automatically for P0 issues
4. **Eradication**: Rollback to last known good version
5. **Recovery**: Phased re-deployment with increased monitoring
6. **Lessons Learned**: Post-mortem within 48 hours + JIRA ticket

---

# File 2: project-overview.md

## 1. Scope

### In-Scope
- **Primary Users**: Internal Financial Advisors (500-1000 users)
- **Data Sources**:
  - Unstructured: Policy PDFs, Investment Research, Market Commentaries (SharePoint/Blob Storage)
  - Structured: Client Portfolio Summaries (SQL Database)
  - Real-Time: Market data APIs (Bloomberg/Reuters)
- **Capabilities**:
  - RAG-powered Q&A over institutional knowledge
  - Multi-turn conversational memory
  - Citation and source attribution
  - Role-based access control (RBAC)
- **Integrations**:
  - User Entitlement System (Active Directory)
  - SIEM (Splunk) for audit logs
  - Ticketing system (JIRA) for flagged responses

### Out-of-Scope (Phase 1)
- Direct customer-facing automated advice (Phase 2)
- Execution of trades (Phase 3)
- Voice/speech interface (Phase 4)
- Mobile app (Phase 4)

## 2. Stakeholders & Roles

| **Role** | **Responsibility** | **Decision Authority** |
|----------|-------------------|------------------------|
| **Product Owner** | Prioritize features, define success criteria | Roadmap changes |
| **Technical Lead** | Architecture decisions, code review | Technology stack |
| **Compliance Officer** | Define "Red-Line" topics, approve Golden Set | Deployment go/no-go |
| **MRM (Model Risk Management)** | Validate LLM pipeline under SR 11-7 | Model approval |
| **Security Lead** | OWASP compliance, red team coordination | Security sign-off |
| **Phoenix Admin** | Configure evals, maintain dashboards | Observability changes |

## 3. Data Sources & Entitlements

### Document Classification

| **Classification** | **Access Control** | **Examples** |
|--------------------|-------------------|--------------|
| **Public** | All authenticated users | Market research, industry news |
| **Internal** | Role-based (e.g., "Senior Advisor") | Client success stories, sales playbooks |
| **Confidential** | Explicit group membership (e.g., "PrivateWealth") | UHNW estate strategies, M&A deal docs |
| **Restricted** | Named individuals only | Legal settlements, regulatory filings |

### Entitlement Model

**RBAC (Role-Based Access Control)** enforced at two layers:

1. **Vector DB Query Time**: Retriever filters documents based on `userContext.groups` before similarity search
2. **LLM Prompt Construction**: System prompt dynamically includes only sources user can access

**Implementation**:
```python
# Pseudo-code
def retrieve_documents(query: str, user_groups: List[str], top_k: i
