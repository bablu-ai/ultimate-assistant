# Project Aegis Constitution (Updated)

**Version**: 1.1
**Derived From**: Master Vision Document v3.0
**Purpose**: Enforceable architectural principles, constraints, and quality bars that govern all feature specifications across an agentic application stack (OAuth 2.0, LangGraph, Python/Java, MCP, A2A, MongoDB, Pinecone, React, Copilot-Kit, Phoenix, multi-vendor LLMs).[^1]

---

## 1. Core Architectural Principles

### 1.1 Multi-Vendor Architecture (Mandatory)

- **Rule**: No single-vendor lock-in. Every critical layer (LLM, vector DB, embeddings, guardrails, observability) MUST have a documented fallback option.[^1]
- **Requirement**: Architecture Decision Records (ADRs) required for all major technology choices, reviewed quarterly.[^1]
- **Failover**: Technical failover mechanisms MUST be implemented and tested for primary LLM and vector DB.[^1]


### 1.2 Evaluation-Driven Selection

- **Rule**: Technology choices MUST be made via explicit evaluation matrices covering: accuracy, latency (P95), cost, data residency, vendor risk.[^1]
- **Gate**: Selection gates with defined decision authority (CTO chair, AI Lead, Product, Finance, Compliance observer).[^1]
- **Re-evaluation**: Critical components (LLM, vector DB) MUST be re-evaluated quarterly.[^1]


### 1.3 Immutable Audit (Zero Exceptions)

- **Rule**: ALL interactions, internal reasoning, and tool calls MUST be logged immutably for 5 years.[^1]
- **Implementation**: Write-once logs; deletions via crypto-shredding only.[^1]
- **Retrieval**: Audit trails MUST be retrievable within 30 seconds.[^1]
- **Gate**: 100% audit trail completeness is a HARD GATE for go-live and phase progression.[^1]


### 1.4 Zero-Trust Security Model

- **Rule**: Every request MUST be authenticated; every token MUST be monitored.[^1]
- **Layers**: Input validation, processing isolation, output filtering—all enforced.[^1]
- **Access**: Role-based access with complete access audit trails and strict separation of duties.[^1]

---

## 2. Technology Constraints

### 2.1 Data Residency \& Privacy

- **Constraint**: All data processing MUST support US/EU data residency options.[^1]
- **PII**: PII detection and redaction MUST occur BEFORE data reaches LLMs.[^1]
- **Right to Delete**: Crypto-shredding process MANDATORY for GDPR/CCPA compliance.[^1]


### 2.2 Observability as Code

- **Rule**: Tracing, evaluation, and monitoring are first-class citizens, versioned alongside application code.[^1]
- **Requirement**: Every LLM call MUST emit a trace with: latency, groundedness, hallucination score, fairness indicators, cost.[^1]
- **Dashboards**: Alert thresholds, evaluation pipelines, and runbooks MUST be versioned artifacts.[^1]


### 2.3 Data Governance \& Staleness

- **Rule**: Vector database MUST sync daily with systems of record (within 24 hours).[^1]
- **Staleness Protocol**: Documents past "Review-By" date MUST be flagged and excluded from retrieval.[^1]
- **Purge**: Decommissioned policies MUST be removed from embeddings within 60 minutes.[^1]

---

## 3. Quality Bars (Non-Negotiable)

### 3.1 Accuracy \& Groundedness

- **Golden Set Pass Rate**: ≥98% (gates Phase 1 exit).[^1]
- **Groundedness Threshold**: ≥0.90 required to show answers; below triggers safe fallback.[^1]
- **Hallucination Rate**: ≤2% (gates Phase 1 exit; operational HITL escalation if exceeded).[^1]


### 3.2 Fairness \& Bias

- **Variance Across Protected Classes**: 0% (zero tolerance).[^1]
- **Validation**: Quarterly bias audits MANDATORY.[^1]
- **Kill Switch**: Brand sentiment or ethical alignment <0.70 triggers CRO-authorized "Safe-Mode".[^1]


### 3.3 Response Behavior

- **Default**: System MUST default to "I don't know" rather than hallucinate.[^1]
- **Low Confidence**: Answers below groundedness threshold MUST route to HITL or safe fallback.[^1]
- **Citations**: Every answer MUST include citations to underlying documents with unique interaction ID.[^1]

---

## 4. Security Bars (OWASP LLM Top 10)

### 4.1 Control Coverage

- **Rule**: ALL 10 OWASP LLM Top 10 (2025) risks MUST have implemented and tested controls.[^1]
- **Attack Block Rate**: ≥95% in quarterly red-team exercises (gates Phase 1 exit).[^1]
- **Testing**: Red-team scenarios MANDATORY with defined frequency and monitoring metrics.[^1]


### 4.2 Security Layers

- **Input Layer**: Prompt-injection defenses, PII detection, content-type validation.[^1]
- **Processing Layer**: Tenant/user isolation, strict tool permissions, rate limiting, context-window controls.[^1]
- **Output Layer**: Response validation, content filtering, guardrail enforcement before advisor delivery.[^1]


### 4.3 PII Protection

- **Zero Leakage**: 0 PII leakage in 1,000 test queries (gates Phase 1 exit).[^1]
- **Detection**: Automated PII detection (e.g., Presidio) at input layer.[^1]

---

## 5. Performance Bars

### 5.1 Latency

- **Time-to-First-Token (TTFT)**: ≤3 seconds P95 under expected load.[^1]
- **Failover Trigger**: Primary LLM latency threshold breach triggers automatic fallback routing.[^1]


### 5.2 Scale \& Availability

- **Concurrency**: 500 concurrent advisors, 100 queries/second at Phase 3 GA.[^1]
- **Availability**: ≥99.9% (gates Phase 2 exit).[^1]

---

## 6. Cost Bars \& Circuit Breakers

### 6.1 Cost Targets

- **Cost Per Query**: ≤\$0.05 average.[^1]
- **Monthly Variance**: ≤10% vs. forecast; remediation required if >20%.[^1]


### 6.2 Circuit Breakers

- **Threshold**: Cost per query >\$0.10 triggers "Safe-Mode" fallback.[^1]
- **Authority**: CFO approval required for continued operation above threshold.[^1]

---

## 7. Regulatory Compliance (Mandatory)

### 7.1 Regulatory Frameworks

- **SR 11-7**: Model inventory, validation, monitoring, escalation.[^1]
- **GDPR/CCPA**: Crypto-shredding, conversation export, right-to-be-forgotten.[^1]
- **EU AI Act**: Mapped controls with evidence sources and test cadence.[^1]


### 7.2 Compliance Checkpoints

- **Phase Gate**: 4-week Regulatory Review Gate (Compliance, Risk, Legal) cannot be parallelized.[^1]
- **Pre-Audit**: SR 11-7 pre-audit MUST pass with no red-line issues (gates Phase 3 exit).[^1]

---

## 8. Explicit Non-Goals (Phase 1-3)

### 8.1 Out of Scope

- Direct client-facing GenAI interactions.[^1]
- Automated trade execution or portfolio rebalancing.[^1]
- Cross-institution deployments.[^1]


### 8.2 Future Vision Only (Not Phase 1-3)

- Client-facing recommendations (Q3 2026 earliest).[^1]
- Multi-modal intelligence (Q4 2026 earliest).[^1]
- Execution integration with OMS (Q1 2027 earliest).[^1]
- Predictive analytics and cross-bank knowledge sharing (Beyond 2027).[^1]

---

## 9. Phase Exit Gates (Hard Requirements)

### 9.1 Phase 1 (Alpha) Exit

- Golden Set ≥98%.[^1]
- Hallucination rate ≤2%.[^1]
- 0 PII leakage in 1,000 test queries.[^1]
- OWASP attack block rate ≥95%.[^1]
- Compliance and Security sign-off documented.[^1]


### 9.2 Phase 2 (Beta) Exit

- NPS ≥4.5 from beta advisors.[^1]
- ≥70% weekly active beta advisors.[^1]
- Availability 99.9%.[^1]
- Cost within 10% of forecast.[^1]


### 9.3 Phase 3 (GA) Exit

- SR 11-7 pre-audit passed (no red-line issues).[^1]
- 100% training completion across advisors and support teams.[^1]
- 500 advisors active weekly at targeted usage levels.[^1]

---

## 10. Kill Switches \& Contingency (Mandatory)

### 10.1 Ethical Kill Switch

- **Trigger**: Brand sentiment or ethical alignment <0.70.[^1]
- **Authority**: CRO authorization required.[^1]
- **Action**: Revert to "Safe-Mode" (canned responses, Policy Portal guidance).[^1]
- **Recovery**: Staged rollout (Ring 0 → Ring 1 → Full) with CRO sign-off.[^1]


### 10.2 Technical Failover

- **LLM Failover**: Automatic routing to approved fallback LLM if latency thresholds exceeded.[^1]
- **Vector DB Failover**: Switch to alternate DB option if latency above threshold (~1 week impact expected).[^1]


### 10.3 Cost Circuit Breaker

- **Trigger**: Cost per query >\$0.10 or monthly variance >20%.[^1]
- **Action**: Activate "Safe-Mode" until remediation.[^1]

---

## 11. Governance Rules

### 11.1 Decision Authority

- **Technical Changes**: CTO approval.[^1]
- **Risk/Compliance Changes**: Compliance approval.[^1]
- **Scope/Timeline Changes**: Executive Sponsor approval.[^1]
- **Budget Increases**: CFO approval.[^1]
- **Kill Switch**: CRO authorization.[^1]


### 11.2 Change Management

- **Requirement**: All scope changes MUST include cost, timeline, fairness, and risk impact.[^1]
- **Documentation**: Changes logged in central change register.[^1]


### 11.3 Documentation Standards

- **ADRs**: Required for all major decisions; include context, options, rationale, trade-offs, review cadence.[^1]
- **Metrics Specification**: MUST include: definition, data source, frequency, target, alert thresholds, owner, phase gate dependency.[^1]

---

## 12. Rules for Future Specifications

All future feature specs MUST:[^1]

1. Align with Core Principles: Defensibility, Safety First, Immutable Audit, Security by Design, Observability as Code.[^1]
2. Preserve Quality Bars: Meet or exceed Golden Set (≥98%), groundedness (≥0.90), hallucination (≤2%), fairness (0% variance).[^1]
3. Maintain Security Posture: Uphold OWASP LLM Top 10 controls and ≥95% attack block rate.[^1]
4. Include Observability: Define traces, metrics, dashboards, and alerts as first-class deliverables.[^1]
5. Document Alternatives: Provide ADR with options, trade-offs, and review cadence.[^1]
6. Respect Exit Gates: Ensure changes do not compromise phase exit criteria.[^1]
7. Support Audit Requirements: Maintain 100% audit trail completeness and 5-year retention.[^1]
8. Include Cost Impact: Estimate impact on cost per query and monthly variance.[^1]
9. Consider Fairness: Include bias impact analysis for user-facing changes.[^1]
10. Enable Failover: Do not introduce single points of failure; maintain multi-vendor optionality.[^1]

---

## 13. Agent Governance \& Autonomy

### 13.1 Agent Lifecycle States

- **States**: Each agent MUST implement an explicit lifecycle: Registered → Approved → Enabled → Observed → Quarantined → Retired, with state stored in a governance registry.
- **Transitions**: Transitions MUST be gated by checks (security, evaluation score, regression history) and logged immutably with actor and rationale.


### 13.2 Autonomy Levels \& Override

- **Autonomy Classes**: Agents MUST be classified as Informational, Advisory, or Actioning, with clearly defined permissions and allowed tools per class.
- **Human Override**: Any Actioning behavior MUST be either HITL or HOTL with explicit human override and immediate kill switch at agent and capability level.


### 13.3 Scope \& Tool Boundary Control

- **Tool Whitelisting**: Each agent MUST declare a minimal tool whitelist with scoped permissions (resource, verb, tenant), enforced by runtime policy.
- **Output Constraints**: Agents MUST declare output contracts (schemas, PII rules, compliance constraints), enforced before delivery to downstream consumers.

---

## 14. MCP Server Security Model

### 14.1 MCP Server Trust Boundaries

- **Isolation**: MCP servers MUST be treated as separate trust domains with strict network and identity isolation from core application runtimes.
- **Identity**: Every MCP server MUST authenticate to the orchestrator using mutual TLS and short-lived credentials issued by a central identity provider.


### 14.2 Tool Capability Declarations

- **Capability Contracts**: Each MCP tool MUST declare capabilities, side effects, data domains, and maximum impact radius (e.g., read-only vs write vs external callouts).
- **Policy Enforcement**: Policy engines MUST enforce per-tenant, per-agent, and per-tool constraints before any MCP invocation is executed.


### 14.3 Auditing \& Secret Handling

- **Secret Minimization**: MCP servers MUST never store long-lived secrets in code or config; secrets MUST be retrieved at runtime from a centralized secret manager.
- **Audit Scope**: Every MCP call MUST be logged with tool, parameters (appropriately redacted), caller identity, tenant, and result classification (success/failure/risk-flagged).

---

## 15. A2A Interaction Constraints

### 15.1 A2A Communication Model

- **Explicit Channels**: Agent-to-agent communication MUST occur only through explicit, auditable channels (queues, topics, or orchestrator APIs), never via ad-hoc backdoors.
- **Typed Messages**: All A2A messages MUST conform to typed schemas (intent, inputs, expected outputs, confidence context) with versioning and validation.


### 15.2 Risk \& Escalation Rules

- **Risk Propagation**: Agents MUST propagate risk flags (low confidence, compliance risk, data sensitivity) in A2A messages, and downstream agents MUST honor these constraints.
- **Cycle Control**: Orchestrator MUST enforce cycle and breadth limits for A2A graphs (max depth, max fan-out) and kill any run breaching thresholds.


### 15.3 Cross-Tenant and Cross-Domain Isolation

- **Tenant Isolation**: A2A interactions MUST NEVER cross tenants; any cross-tenant pattern is prohibited unless explicitly approved by Compliance and documented in ADRs.
- **Domain Guardrails**: Cross-domain A2A flows (e.g., research → execution) MUST pass through policy checkpoints that enforce role, regulatory, and kill-switch constraints.

---

## 16. Agent Behavioral Evaluations (Phoenix)

### 16.1 Run-Level Evaluations

- **Trace Integration**: All agent runs (including A2A chains) MUST emit structured traces consumable by Phoenix for behavior-level evaluation.
- **Metrics**: Evaluations MUST include at minimum: task success, policy adherence, tool misuse, escalation correctness, and user satisfaction proxy.


### 16.2 Continuous Regression \& Canarying

- **Regression Suites**: Phoenix MUST host curated scenario suites for safety, compliance, and business KPIs, run on every model or policy change.
- **Canary Policy**: New agents or significant changes MUST be canaried on a small ring with Phoenix monitoring, with promotion gated on evaluation thresholds.


### 16.3 Incident Feedback Loop

- **Incident Backfill**: Post-incident analyses MUST result in new Phoenix scenarios capturing regressions and near-misses.
- **Ownership**: Each critical scenario MUST have a named owner responsible for thresholds, triage playbooks, and remediation timelines.

---

## 17. Memory \& State Classification

### 17.1 Memory Types \& Stores

- **Classification**: All memory MUST be classified into ephemeral context, session memory, long-term user memory, and global knowledge, each mapped to specific stores (e.g., MongoDB, Pinecone, cache).
- **Separation**: User-specific memory MUST never be stored in global shared indices; per-tenant and per-user isolation MUST be enforced at the storage layer.


### 17.2 Sensitivity \& Retention

- **Sensitivity Tiers**: Memory records MUST be tagged with sensitivity level (public, internal, confidential, regulated) driving storage, encryption, and retention policies.
- **Retention**: Each memory type and sensitivity tier MUST have explicit retention, right-to-forget, and crypto-shredding rules aligned with regulatory requirements.


### 17.3 Access Semantics \& Consent

- **Consent**: Persistent user memory MUST be opt-in with clear, explainable UI and revocation options exposed in the client.
- **Access Controls**: Agents MUST declare which memory classes they can access; memory lookups MUST be mediated via policy-aware APIs that enforce tenant, role, and purpose.

---

## 18. Client-Side Copilot Security

### 18.1 Frontend Trust Model

- **Zero-Trust Client**: React + Copilot-Kit clients MUST be treated as untrusted; no security-critical decision may rely solely on client-provided state.
- **Token Handling**: OAuth 2.0 and similar tokens MUST never be persisted in insecure storage (e.g., localStorage) and MUST use secure, HttpOnly cookies or secure in-memory strategies.


### 18.2 Prompt and Context Hardening

- **Client Controls**: Client-side controls (UI hints, labels) MUST NOT be treated as security boundaries and MUST be validated server-side.
- **Context Minimization**: Only strictly necessary context (documents, memory summaries, tool handles) may be sent to the client; sensitive reasoning and policy checks MUST remain server-side.


### 18.3 Browser and Supply-Chain Protections

- **Dependency Hygiene**: Frontend dependencies MUST be pinned, scanned for vulnerabilities, and monitored for supply-chain risks.
- **Clickjacking \& Injection**: Standard web controls (CSP, X-Frame-Options, sanitization) MUST be enforced to prevent injection and UI redressing attacks that influence copilot behavior.

---

## 19. Multi-Runtime Consistency

### 19.1 Contract Consistency Across Runtimes

- **Shared Contracts**: All runtimes (Python, Java, others) MUST consume shared, versioned schemas for messages, tools, and policies to guarantee consistent behavior.
- **Polyglot Governance**: Any change to cross-runtime contracts MUST go through a single governance workflow and be validated in integration tests across runtimes.


### 19.2 Policy and Guardrail Uniformity

- **Central Policy Engine**: Authorization, rate limiting, and guardrail policies MUST be implemented in a central engine invoked consistently by all runtimes and agents.
- **Drift Detection**: Automated checks MUST detect and block deployment of runtimes whose policy configuration diverges from approved baselines.


### 19.3 Observability \& Configuration

- **Unified Telemetry Schema**: Logs, metrics, and traces across runtimes MUST conform to a shared schema to enable unified analysis and Phoenix evaluations.
- **Configuration Management**: Runtime configuration (LLM providers, DBs, feature flags) MUST be centrally managed and environment-aware, with changes audited and rollbacks defined.

---

## 20. Enforcement

- **Pre-Implementation Review**: All specs MUST be reviewed against this constitution before implementation begins.[^1]
- **Phase Gate Reviews**: Constitution compliance MUST be verified at each phase exit gate.[^1]
- **Quarterly Audits**: Compliance, Security, and Architecture teams conduct quarterly constitution compliance audits.[^1]
- **Violation Escalation**: Any violation of HARD GATES or quality bars escalates immediately to CTO and Executive Sponsor.[^1]
- **Agentic Scope**: Any new agent, MCP server, A2A pattern, or memory class MUST not go live without explicit sign-off for sections 13–19.

If helpful, the next step can be to convert this into a diff against your v1.0 or into ADR stubs for each new section.

**Confidence: High** (Directly grounded in your provided constitution plus explicit alignment to the requested seven areas.)

<div align="center">⁂</div>

[^1]: constitution.md

