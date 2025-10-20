# MCP Catalog Integration - Architecture Review Executive Summary

**Document:** Architecture Review for MCP Catalog Integration RFE
**Date:** 2025-10-20
**Architect:** System Architect (Archie)
**Full Review:** /workspace/sessions/agentic-session-1760968191/workspace/vTeam-specs/architecture-review.md

---

## Overview

The MCP Catalog Integration introduces enterprise-grade governance for Model Context Protocol (MCP) servers in OpenShift AI Dashboard. This architectural review evaluates the technical feasibility, integration patterns, and long-term implications of the proposed feature.

**Architecture Verdict:** **APPROVED with Recommendations**

The feature is architecturally sound and aligns with existing ODH Dashboard patterns. Implementation requires careful attention to integration contracts, policy enforcement, and data architecture.

---

## Key Architectural Decisions

### 1. Backend-for-Frontend (BFF) Pattern

**Decision:** Implement Go-based BFF layer following the proven pattern from gen-ai and model-registry packages.

**Why:**
- Consolidates Registry and Gateway integration logic
- Provides caching and performance optimization
- Enforces authorization before backend calls
- Shields UI from API changes

**Implementation:**
```
UI → BFF (Go) → Registry (metadata)
             → Gateway (deployments, policy)
             → Redis (cache)
             → TimescaleDB (audit logs)
```

### 2. Policy Architecture: Kubernetes CRDs + OPA

**Decision:** Store policies as Custom Resources, evaluate with Open Policy Agent (OPA).

**Why:**
- GitOps-friendly policy-as-code
- RBAC-controlled policy administration
- Flexible Rego language for complex rules (e.g., SBOM vulnerability checks)

**Example Policy CRD:**
```yaml
apiVersion: mcp.opendatahub.io/v1alpha1
kind: MCPCatalogPolicy
metadata:
  name: require-verified-publisher
spec:
  rules:
    - type: trustTier
      operator: in
      values: ["verified", "partner"]
      action: deny
```

### 3. Hybrid Policy Enforcement

**Decision:** Client-side validation (BFF) for UX + Server-side enforcement (Gateway) for security.

**Flow:**
1. BFF returns catalog with policy hints → UI grays out disallowed options
2. User selects server → BFF validates with OPA (pre-check)
3. BFF forwards to Gateway → Gateway validates with OPA (authoritative)

**Why:** Fast user feedback without sacrificing security.

### 4. Data Storage Strategy

| Data Type | Storage | Rationale |
|-----------|---------|-----------|
| Catalog Metadata | Registry (source of truth) + Redis cache (5 min TTL) | Read-heavy, caching sufficient |
| SBOMs | S3/Minio object storage | Large files (100KB-1MB+), cost-effective |
| Audit Logs | TimescaleDB (PostgreSQL) | Time-series optimized, SQL queries |
| Policies | Kubernetes CRDs | GitOps, version control, RBAC |

### 5. Federation: Hub-and-Spoke Pattern

**Decision:** BFF polls federated registries every 15 minutes, merges results with conflict resolution.

**Conflict Resolution:**
- Primary registry wins for duplicate MCP server IDs
- Federated sources tagged with origin
- Admin can override via policy

**Future:** Consider push-based webhooks for real-time sync.

---

## Critical Integration Contracts

### Registry API (Proposed)

```yaml
# GET /api/v1/mcpservers?lifecycle=stable&trustTier=verified&page=1&pageSize=50
Response:
  items: [MCPServerMetadata]
  metadata:
    totalCount: 150
    syncTimestamp: "2025-10-20T10:30:00Z"

# GET /api/v1/mcpservers/{id}
Response:
  id: "postgres-mcp-server"
  versions: [{ version: "2.0.0", lifecycle: "stable", sbomUrl: "..." }]
  trustTier: "verified"
  publisher: { verified: true }

# GET /api/v1/mcpservers/{id}/versions/{version}/sbom
Response: SPDX or CycloneDX JSON
```

**SLA:** <500ms (p95) for list, <200ms for detail with caching

### Gateway API (Proposed)

```yaml
# POST /api/v1/policy/validate
Request:
  mcpServerId: "postgres-mcp-server"
  version: "2.0.0"
  namespace: "ml-project-1"
Response:
  allowed: true/false
  policies: [{ policyId, result, message }]

# POST /api/v1/deployments
Request:
  mcpServerId: "postgres-mcp-server"
  version: "2.0.0"
  namespace: "ml-project-1"
Response:
  deploymentId: "uuid-123"
  status: "pending"
```

**SLA:** <500ms for policy validation, <2s for deployment initiation

---

## Scalability and Performance

### Caching Strategy

**4-Layer Cache:**
1. Browser cache (static assets: 1 year, API: 1 minute)
2. BFF in-memory cache (metadata: 5 minutes)
3. Redis distributed cache (SBOM: 1 hour, policy results: 1 minute)
4. Registry as source of truth

**Cache Invalidation:** Time-based TTL + Event-driven (Registry publishes change events)

### Performance Targets

- Catalog page load: <2 seconds (p95)
- API response time: <500ms list, <200ms detail
- Cache hit rate: >80%
- Concurrent users: 100+ platform engineers

### Scale Targets

- MCP servers in catalog: 500+ (MVP), 5000+ (Phase 3)
- Federated registries: 5-10 (MVP), 20+ (Phase 4)
- Audit events: 1M+ per month
- Deployments: 1000+ active deployments tracked

---

## Security Architecture

### Authentication/Authorization

- **Auth:** Leverage existing kube-rbac-proxy, extract user from `x-auth-request-user` header
- **RBAC:** Define ClusterRoles for catalog-user and catalog-admin
- **Authorization:** BFF performs SelfSubjectAccessReview before serving data

### Supply Chain Security

- **Publisher Verification:** GPG/Sigstore signing of MCP server packages
- **SBOM Required:** All MCP servers must provide SPDX or CycloneDX SBOM
- **Vulnerability Integration:** Display vulnerability summary from scanner (Clair, Trivy)
- **Audit Trail:** Immutable audit logs for all deployments and policy violations

### Trust Tiers (Proposed)

1. **Verified:** Red Hat verified publisher (GPG signed, security scanned)
2. **Partner:** Verified partner publisher (partner signing key)
3. **Community:** Community contribution with valid SBOM (self-signed)
4. **Unverified:** No verification (blocked by default policy)

---

## Deprecation and Lifecycle Management

### State Machine

```
Experimental → Stable → Deprecated → EOL
```

### Notification Timeline

- T-90 days: First notice (EOL in 90 days)
- T-30 days: Second notice
- T-7 days: Final warning
- T-0 days: EOL reached, deployment blocked

### Notification Channels

- Email to namespace owners (from RBAC groups)
- Dashboard persistent banner
- Slack/webhook (Phase 2)

### Event-Driven Architecture

```
Lifecycle change event → Message queue (Kafka/NATS)
    → Notification Service queries: "Which namespaces have this deployed?"
    → Send notifications to affected teams
```

---

## Federation and Disconnected Environments

### Federation Pattern (Hub-and-Spoke)

```
Primary Registry (Red Hat)
    ↓ BFF polls every 15 min
Partner Registry A ←─┐
Partner Registry B ←─┼─ BFF merges with conflict resolution
Internal Registry  ←─┘
```

### Disconnected Architecture (Mirror Pattern)

```
Connected Environment:
  mcp-catalog-mirror export → Bundle (metadata + SBOMs + images)
      ↓ Transfer via USB/secure channel
Air-Gapped Environment:
  mcp-catalog-mirror import → Local Mirror Registry
      ↓
  MCP Catalog reads from local mirror
```

**Tooling:** Extend `oc-mirror` pattern for MCP catalog bundles

---

## Implementation Phasing (9-13 Months)

### Phase 1: MVP Foundation (3-4 months)
- BFF + Repository pattern
- Registry integration (single primary)
- Catalog UI (browse, search, detail view)
- Gateway deployment integration
- Basic policy enforcement (lifecycle, trust tier)
- Audit logging (basic events)

**Deliverable:** Users can browse and deploy MCP servers with basic governance.

### Phase 2: Enhanced Governance (2-3 months)
- SBOM display and vulnerability summary
- OPA integration for flexible policies
- Policy CRDs and admin UI
- Deprecation notification system
- Migration guidance

**Deliverable:** Comprehensive governance with vulnerability visibility.

### Phase 3: Federation and Scale (2-3 months)
- Multi-registry federation (hub-and-spoke)
- Conflict resolution rules
- Performance optimization (500+ servers)
- Advanced audit query UI

**Deliverable:** Scale to 500+ MCP servers with federated sources.

### Phase 4: Enterprise Features (2-3 months)
- Disconnected environment support (mirror tooling)
- Multi-cluster catalog federation
- Grafana dashboards + OpenTelemetry tracing
- Approval workflows (ITSM integration)

**Deliverable:** Production-ready for air-gapped enterprise deployments.

---

## Technical Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Registry API instability** | High - Catalog unavailable | Circuit breaker pattern, serve stale cache, 5-minute TTL |
| **SBOM parsing performance** | Medium - Slow detail page | Lazy load SBOMs, async parsing, cache parsed results |
| **Policy evaluation latency** | Medium - Blocks deployments | Cache policy results (1-min TTL), 500ms timeout |
| **Audit log storage growth** | Medium - Cost/performance | Time-based partitioning, automated archival, 90-day hot storage |
| **Federation conflict resolution** | Low - Metadata conflicts | Clear rules (primary wins), admin overrides, source tagging |

---

## Architectural Decision Records Required

Before implementation begins, create and approve these ADRs:

1. **ADR-001:** BFF Pattern for MCP Catalog Integration
2. **ADR-002:** Policy Storage in Kubernetes CRDs
3. **ADR-003:** OPA for Policy Evaluation
4. **ADR-004:** Hybrid Client-Side and Server-Side Policy Enforcement
5. **ADR-005:** Redis for Distributed Caching
6. **ADR-006:** Time-Series Database for Audit Logs
7. **ADR-007:** SBOM Storage in Object Storage
8. **ADR-008:** Hub-and-Spoke Federation Pattern
9. **ADR-009:** Pull-Based Federation Sync
10. **ADR-010:** Event-Driven Deprecation Notifications

---

## Critical Questions to Resolve (Blockers)

### High Priority (Before Phase 1)

1. **Registry API Specification**
   - **Question:** What is the exact API contract? REST, gRPC, GraphQL?
   - **Recommendation:** REST with OpenAPI 3.0
   - **Owner:** Registry team
   - **Status:** ⚠️ UNRESOLVED

2. **Gateway Policy API**
   - **Question:** What API for policy validation?
   - **Recommendation:** `POST /api/v1/policy/validate` with sync response
   - **Owner:** Gateway team
   - **Status:** ⚠️ UNRESOLVED

3. **Trust Tier Definitions**
   - **Question:** What are the specific tiers and criteria?
   - **Recommendation:** verified, partner, community, unverified
   - **Owner:** Product Management + Security
   - **Status:** ⚠️ UNRESOLVED

4. **SBOM Storage Location**
   - **Question:** Where are SBOMs stored?
   - **Recommendation:** S3/Minio object storage
   - **Owner:** Registry team
   - **Status:** ⚠️ UNRESOLVED

5. **Audit Log Backend**
   - **Question:** What storage for audit logs?
   - **Recommendation:** TimescaleDB
   - **Owner:** Platform Engineering
   - **Status:** ⚠️ UNRESOLVED

---

## Success Metrics (KPIs)

**Adoption:**
- 80% of MCP deployments use catalog by Month 6
- 100+ unique platform engineers accessing catalog
- 500+ catalog page views per week

**Governance:**
- 95% policy violation prevention rate
- 100% of deployments have SBOMs (after Phase 2)
- 100% audit log coverage

**Performance:**
- <2 second catalog page load (p95)
- <500ms API response for list operations
- >80% cache hit rate
- 99.9% catalog availability

**Security:**
- <24 hours to detect deprecated server usage
- >70% verified publisher deployments
- <5 seconds for audit query response

---

## Recommended Next Steps

### Immediate (Week 1-2)
1. **API Contract Workshop:** Schedule joint session with Registry, Gateway, and Dashboard teams
2. **Trust Tier Definition:** Product + Security define trust tiers and verification criteria
3. **Stakeholder Review:** Present this architecture review to engineering leadership

### Short-Term (Week 3-6)
1. **ADR Creation:** Write and approve the 10 ADRs listed above
2. **Proof of Concept:** Build basic BFF + Registry integration to validate patterns
3. **UI Mockups:** Create detailed UI designs based on model catalog pattern

### Medium-Term (Week 7-12)
1. **Detailed Design:** API documentation, data schemas, deployment architecture
2. **Resource Planning:** Allocate 2-3 engineers for 6+ month engagement
3. **Phase 1 Kickoff:** Begin MVP implementation

---

## Conclusion

The MCP Catalog Integration is architecturally viable and strategically important for OpenShift AI's enterprise positioning. The proposed architecture leverages proven patterns (BFF, CRDs, OPA) while maintaining consistency with existing ODH Dashboard design.

**Critical Success Factors:**
1. Early agreement on Registry and Gateway API contracts
2. Clear ownership of trust tier definitions and policy governance
3. Robust caching and performance optimization from the start
4. Comprehensive audit logging for compliance requirements

**Recommended Resources:**
- 2-3 engineers for 9-13 months (phased delivery)
- Infrastructure: Redis cluster, TimescaleDB, S3/Minio
- Cross-team coordination with Registry and Gateway teams

**Timeline to Production:**
- Phase 1 MVP: 3-4 months (basic catalog + governance)
- Phase 2 Enhanced Governance: 6-7 months (SBOM + OPA)
- Phase 3 Federation: 9-10 months (multi-registry)
- Phase 4 Enterprise: 12-13 months (disconnected + multi-cluster)

This aligns with the north star architecture for OpenShift AI as an enterprise-grade platform. In 18 months, this feature will enable OpenShift AI to scale to hundreds of governed MCP server deployments, creating lasting impact across the AI platform portfolio.

---

**Architecture Review Status:** ✅ APPROVED with Recommendations
**Next Review:** After API contracts are defined (Week 3)
**Document Owner:** System Architect (Archie)
**Last Updated:** 2025-10-20
