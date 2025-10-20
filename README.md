# vTeam-specs

Specification and architecture documents for OpenShift AI features.

## MCP Catalog Integration Feature

**Feature Status:** Architecture Review Complete
**Review Date:** 2025-10-20
**Reviewer:** System Architect (Archie)

### Documents

1. **[RFE Document](./rfe.md)** - Original Request for Enhancement
   - Feature overview, requirements, and use cases
   - Acceptance criteria and documentation considerations
   - Questions to be answered before implementation

2. **[Architecture Review](./architecture-review.md)** - Comprehensive Technical Architecture Analysis (17,000+ words)
   - Architecture assessment (strengths and concerns)
   - Integration architecture with BFF pattern
   - Data architecture and schemas
   - Security architecture (auth, RBAC, supply chain)
   - Scalability and performance considerations
   - Policy enforcement with Kubernetes CRDs and OPA
   - Federation architecture (hub-and-spoke)
   - Disconnected/air-gapped environment support
   - Technical risks and mitigation strategies
   - 10 Architectural Decision Records (ADRs) needed
   - Implementation phasing (9-13 months, 4 phases)
   - Critical questions to resolve before starting

3. **[Architecture Summary](./architecture-summary.md)** - Executive Summary for Leadership
   - Key architectural decisions (BFF, CRDs + OPA, hybrid policy enforcement)
   - Critical integration contracts (Registry API, Gateway API)
   - Scalability and performance targets
   - Security architecture highlights
   - Implementation phasing and timeline
   - Technical risks and success metrics
   - Status of critical blocking questions

4. **[Architecture Diagrams](./architecture-diagrams.md)** - Visual Reference Guide
   - High-level system architecture (BFF pattern)
   - Repository pattern design
   - Policy enforcement flow (hybrid client/server)
   - Policy architecture (CRD + OPA)
   - Data model architecture
   - 4-layer caching strategy
   - Federation hub-and-spoke pattern
   - Disconnected environment architecture
   - Deprecation notification flow
   - Audit log architecture (TimescaleDB)
   - Observability stack (Prometheus, Jaeger, Loki, Grafana)

### Key Architectural Decisions

**Architecture Verdict:** APPROVED with Recommendations

**Core Patterns:**
- Backend-for-Frontend (Go BFF) consistent with gen-ai and model-registry packages
- Kubernetes CRDs for policy storage with OPA (Rego) for evaluation
- Hybrid policy enforcement (client-side for UX, server-side for security)
- 4-layer caching (browser → BFF memory → Redis → Registry)
- Hub-and-spoke federation with conflict resolution
- TimescaleDB for time-series audit logs
- Event-driven deprecation notifications

**Technology Stack:**
- UI: React + TypeScript + PatternFly (consistent with ODH Dashboard)
- BFF: Go 1.23+ with httprouter and repository pattern
- Cache: Redis cluster for distributed caching
- Audit: TimescaleDB (PostgreSQL extension) for time-series queries
- Policy: OPA (Open Policy Agent) with Rego language
- SBOM Storage: S3/Minio object storage

**Integration Points:**
- Registry: REST API for metadata, SBOM retrieval
- Gateway: REST API for policy validation and deployment orchestration
- Kubernetes: CRDs for policies, RBAC for authorization
- kube-rbac-proxy: Authentication (consistent with ODH Dashboard v3.0+)

### Implementation Timeline

**Total Duration:** 9-13 months across 4 phases

- Phase 1 (MVP Foundation): 3-4 months - Basic catalog + governance
- Phase 2 (Enhanced Governance): 2-3 months - SBOM + OPA + deprecation
- Phase 3 (Federation & Scale): 2-3 months - Multi-registry + 500+ servers
- Phase 4 (Enterprise Features): 2-3 months - Air-gapped + multi-cluster

### Critical Blockers (Need Resolution Before Phase 1)

1. Registry API specification (REST, gRPC, GraphQL?)
2. Gateway policy enforcement API contract
3. Trust tier definitions and verification criteria
4. SBOM storage location (object storage vs inline)
5. Audit log backend decision (TimescaleDB recommended)

### Next Steps

1. **Week 1-2:** API contract definition with Registry and Gateway teams
2. **Week 3-4:** ADR review and approval (10 ADRs required)
3. **Week 5-6:** Proof of concept (basic BFF + Registry integration)
4. **Week 7-8:** Detailed design documents (data models, API specs)
5. **Week 9+:** Phase 1 MVP implementation

### Resources

**Team Size:** 2-3 engineers for 6+ months
**Infrastructure:** Redis cluster, TimescaleDB, S3/Minio, OPA
**Cross-Team Coordination:** Registry team, Gateway team, Security team

### Success Metrics

- Adoption: 80% of MCP deployments use catalog by Month 6
- Governance: 95% policy violation prevention rate
- Performance: <2s catalog load time, >80% cache hit rate
- Security: 100% SBOM coverage after Phase 2
- Reliability: 99.9% catalog availability

---

## Reference Architecture

The MCP Catalog Integration follows established patterns from the ODH Dashboard ecosystem:

**Alignment with ODH Dashboard Architecture:**
- Modular architecture (similar to gen-ai and model-registry packages)
- BFF pattern with Go backend
- Authentication via kube-rbac-proxy (v3.0+ multi-strategy)
- PatternFly component library for UI consistency
- Repository pattern for domain logic separation
- OpenAPI-documented REST APIs

**Industry Best Practices:**
- C4 model for architecture visualization
- Open Policy Agent (CNCF) for policy evaluation
- Kubernetes-native patterns (CRDs, operators)
- OpenTelemetry for distributed tracing
- Time-series databases for audit logs

This aligns with the north star architecture for OpenShift AI as an enterprise-grade platform, creating lasting impact across the AI platform portfolio.
