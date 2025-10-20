# MCP Catalog Integration - Architecture Review
**Review Date:** 2025-10-20
**Reviewer:** System Architect
**Status:** Draft for Discussion
**RFE Document:** /workspace/sessions/agentic-session-1760968191/workspace/vTeam-specs/rfe.md

## Executive Summary

The MCP Catalog Integration represents a significant architectural extension to the OpenShift AI Dashboard, introducing enterprise governance for Model Context Protocol (MCP) servers. This review evaluates the technical architecture from a systems thinking perspective, focusing on integration patterns, scalability, and long-term maintainability.

**Key Architectural Concerns:**
- **Integration Complexity:** Three-way integration (Registry, Gateway, Dashboard) requires careful API contract design
- **Policy Enforcement Pattern:** Client-side vs server-side policy enforcement has significant architectural implications
- **Data Synchronization:** Catalog metadata synchronization strategy impacts performance and consistency
- **Federation Architecture:** Multi-registry support introduces distributed systems challenges
- **Audit and Observability:** Comprehensive audit requirements impact storage and query architecture

**Recommended Approach:**
- Adopt Backend-for-Frontend (BFF) pattern consistent with existing ODH Dashboard architecture
- Implement server-side policy enforcement in Gateway with client-side validation for UX
- Use event-driven architecture for deprecation notifications and lifecycle management
- Design for eventual consistency in federated catalog scenarios
- Leverage Kubernetes-native patterns (CRDs, operators) for governance policy management

---

## 1. Architecture Assessment

### 1.1 Strengths

**Alignment with Existing Patterns:**
The RFE aligns well with existing ODH Dashboard architectural patterns, particularly:
- Follows the established model catalog UX pattern, reducing learning curve
- Leverages existing authentication/authorization via kube-rbac-proxy
- Uses PatternFly component library consistent with dashboard standards
- Fits within the modular architecture established by gen-ai and model-registry packages

**Clear Separation of Concerns:**
- **Registry:** Source of truth for MCP server metadata
- **Gateway:** Runtime policy enforcement and lifecycle management
- **Catalog UI:** Discovery, selection, and user interaction
- This separation enables independent evolution of each component

**Enterprise-Grade Governance:**
The architectural vision for governance (trust tiers, lifecycle states, SBOM integration) positions OpenShift AI as enterprise-ready, addressing a significant market gap.

**Kubernetes-Native Design:**
The implicit use of Kubernetes patterns (ConfigMaps, CRDs, RBAC) aligns with Red Hat's platform strengths.

### 1.2 Architectural Concerns

**Integration Orchestration Complexity:**
The three-way integration (Registry, Gateway, Dashboard) creates orchestration challenges:
- Who is responsible for maintaining consistency across components?
- What happens when Registry and Gateway have conflicting state?
- How are transactional semantics handled across distributed services?

**Policy Enforcement Architecture Ambiguity:**
The RFE is unclear on policy enforcement location:
- Client-side enforcement: Fast feedback but can be bypassed
- Server-side enforcement: Secure but adds latency
- **Recommendation:** Hybrid approach with client-side validation for UX and server-side enforcement for security

**Data Consistency Model:**
The RFE mentions "configurable refresh intervals" for Registry sync but doesn't address:
- What consistency guarantees are provided (eventual consistency vs strong consistency)?
- How are conflicts resolved when local cache and Registry diverge?
- What happens during network partitions?

**SBOM Storage and Parsing:**
SBOMs can be large (100s of KB to MBs for complex dependencies):
- Where are SBOMs stored? (Registry, separate object storage, inline in metadata?)
- How are they parsed and indexed for vulnerability queries?
- What performance impact does SBOM retrieval have on catalog browsing?

**Audit Log Scale:**
The audit requirements are comprehensive, but architectural implications are underspecified:
- Storage backend for audit logs (relational DB, time-series DB, object storage)?
- Query performance for "show all deployments by user X over 90 days"?
- Retention and archival strategy?

**Federation Architecture:**
Federation is listed as non-MVP but has first-order architectural implications:
- What is the trust model for federated catalogs?
- How are namespace collisions handled (two registries with same MCP server ID)?
- What is the synchronization protocol?

---

## 2. Integration Architecture Recommendations

### 2.1 Proposed Architecture Pattern: Backend-for-Frontend (BFF)

Following the proven pattern from gen-ai and model-registry packages, I recommend a BFF layer for MCP Catalog.

**Rationale:**
- **Consolidation:** BFF aggregates calls to Registry and Gateway, reducing client complexity
- **Security:** BFF enforces authorization before proxying to backend services
- **Caching:** BFF can cache Registry metadata to reduce load and latency
- **API Stability:** BFF shields UI from backend API changes
- **Observability:** Centralized instrumentation point for metrics and tracing

**Architecture Diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenShift AI Dashboard                        │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │              MCP Catalog UI (React/TypeScript)            │ │
│  │  - Browse, search, filter                                 │ │
│  │  - Detail views with metadata, versions, SBOM             │ │
│  │  - Policy violation display                               │ │
│  │  - Deprecation warnings                                   │ │
│  └───────────────────────────────────────────────────────────┘ │
│                           │                                      │
│                           │ JSON/HTTPS                           │
│                           ▼                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │         MCP Catalog BFF (Go Backend-for-Frontend)         │ │
│  │  - API: /mcp-catalog/api/v1/*                             │ │
│  │  - Registry aggregation & caching                         │ │
│  │  - Gateway policy enforcement orchestration               │ │
│  │  - SBOM parsing & vulnerability enrichment                │ │
│  │  - Audit log writing                                      │ │
│  │  - Federation coordinator                                 │ │
│  └───────────────────────────────────────────────────────────┘ │
│         │                    │                    │              │
└─────────┼────────────────────┼────────────────────┼──────────────┘
          │                    │                    │
          │ HTTP/REST          │ HTTP/REST          │ HTTP/REST
          ▼                    ▼                    ▼
    ┌──────────┐         ┌──────────┐        ┌────────────┐
    │   MCP    │         │   MCP    │        │   Audit    │
    │ Registry │         │ Gateway  │        │    Log     │
    │          │         │          │        │  Storage   │
    └──────────┘         └──────────┘        └────────────┘
```

### 2.2 API Contract Specifications

#### Registry Integration API

Based on industry patterns and the RFE requirements, I recommend the following API contract:

**Protocol:** REST over HTTPS
**Authentication:** Mutual TLS or Bearer Token (service account)
**Data Format:** JSON with OpenAPI 3.0 specification

**Core Endpoints:**

```yaml
# GET /api/v1/mcpservers - List all MCP servers
Request:
  Query Params:
    - lifecycle: string[] (experimental, stable, deprecated, eol)
    - trustTier: string[] (verified, partner, community, unverified)
    - capability: string[] (filter by capability tags)
    - page: int (pagination)
    - pageSize: int (default: 50, max: 200)
Response:
  {
    "items": [MCPServerMetadata],
    "metadata": {
      "totalCount": 150,
      "page": 1,
      "pageSize": 50,
      "syncTimestamp": "2025-10-20T10:30:00Z"
    }
  }

# GET /api/v1/mcpservers/{id} - Get MCP server details
Response:
  {
    "id": "postgres-mcp-server",
    "name": "Postgres MCP Server",
    "description": "...",
    "publisher": {
      "id": "acme-corp",
      "name": "ACME Corp",
      "verified": true,
      "verificationDate": "2025-09-15T00:00:00Z"
    },
    "versions": [
      {
        "version": "2.0.0",
        "lifecycle": "stable",
        "releaseDate": "2025-10-01T00:00:00Z",
        "changelog": "...",
        "sbomUrl": "/api/v1/mcpservers/{id}/versions/2.0.0/sbom",
        "resourceRequirements": {
          "cpu": "500m",
          "memory": "512Mi"
        },
        "dependencies": [...]
      }
    ],
    "trustTier": "verified",
    "capabilities": ["database", "sql", "postgres"],
    "documentation": "https://...",
    "metadata": {
      "created": "2025-01-01T00:00:00Z",
      "lastModified": "2025-10-15T00:00:00Z"
    }
  }

# GET /api/v1/mcpservers/{id}/versions/{version}/sbom - Get SBOM
Response:
  Content-Type: application/spdx+json OR application/vnd.cyclonedx+json
  {
    "spdxVersion": "SPDX-2.3",
    "dataLicense": "CC0-1.0",
    "SPDXID": "SPDXRef-DOCUMENT",
    "name": "postgres-mcp-server-2.0.0",
    "packages": [...]
  }

# GET /api/v1/federation/sources - List federated registry sources
Response:
  {
    "sources": [
      {
        "id": "primary-registry",
        "name": "Primary MCP Registry",
        "url": "https://registry.mcp.redhat.com",
        "type": "primary",
        "syncStatus": "healthy",
        "lastSync": "2025-10-20T10:00:00Z"
      }
    ]
  }
```

**Latency Expectations:**
- List operations: <500ms (p95), <200ms with caching
- Detail operations: <200ms (p95)
- SBOM retrieval: <1s (p95) - these can be large

**Caching Strategy:**
- Metadata cache TTL: 5 minutes (configurable)
- SBOM cache TTL: 1 hour (SBOMs are immutable per version)
- Invalidation: Event-driven via webhook or polling with ETag

#### Gateway Integration API

**Protocol:** REST over HTTPS
**Authentication:** Bearer Token (user or service account)

**Core Endpoints:**

```yaml
# POST /api/v1/policy/validate - Validate deployment against policies
Request:
  {
    "mcpServerId": "postgres-mcp-server",
    "version": "2.0.0",
    "namespace": "ml-project-1",
    "user": "platform-engineer-1"
  }
Response:
  {
    "allowed": false,
    "policies": [
      {
        "policyId": "require-verified-publisher",
        "result": "pass",
        "message": "Publisher is verified"
      },
      {
        "policyId": "require-stable-lifecycle",
        "result": "fail",
        "message": "Version lifecycle is 'experimental', policy requires 'stable' or 'deprecated'"
      }
    ],
    "decision": {
      "action": "deny",
      "reason": "Policy 'require-stable-lifecycle' failed"
    }
  }

# POST /api/v1/deployments - Deploy MCP server (Gateway orchestration)
Request:
  {
    "mcpServerId": "postgres-mcp-server",
    "version": "2.0.0",
    "namespace": "ml-project-1",
    "configuration": { ... }
  }
Response:
  {
    "deploymentId": "uuid-...",
    "status": "pending",
    "mcpServerId": "postgres-mcp-server",
    "version": "2.0.0",
    "namespace": "ml-project-1"
  }

# GET /api/v1/deployments/{namespace} - List deployments
Response:
  {
    "deployments": [
      {
        "deploymentId": "uuid-...",
        "mcpServerId": "postgres-mcp-server",
        "version": "2.0.0",
        "status": "running",
        "health": "healthy",
        "createdAt": "2025-10-20T09:00:00Z",
        "createdBy": "platform-engineer-1"
      }
    ]
  }
```

**Policy Evaluation:**
- **Synchronous:** Policy validation happens synchronously during deployment request
- **Response Time:** <500ms for policy evaluation
- **Policy Language:** Recommend OPA (Open Policy Agent) Rego for flexibility

### 2.3 BFF Implementation Specification

**Technology Stack:**
- **Language:** Go 1.23+ (consistent with gen-ai BFF)
- **HTTP Router:** httprouter or chi
- **OpenAPI Documentation:** Auto-generated from code annotations

**Repository Pattern:**
Following the domain-specific repository pattern from gen-ai:

```go
// MCPServersRepository - handles MCP server metadata operations
type MCPServersRepository interface {
    List(ctx context.Context, filters ListFilters) (*MCPServerList, error)
    Get(ctx context.Context, id string) (*MCPServerDetail, error)
    GetVersionSBOM(ctx context.Context, id string, version string) (*SBOM, error)
}

// PoliciesRepository - handles policy validation
type PoliciesRepository interface {
    Validate(ctx context.Context, req PolicyValidationRequest) (*PolicyValidationResponse, error)
    ListPolicies(ctx context.Context, namespace string) ([]Policy, error)
}

// DeploymentsRepository - handles deployment operations
type DeploymentsRepository interface {
    Deploy(ctx context.Context, req DeploymentRequest) (*Deployment, error)
    List(ctx context.Context, namespace string) ([]Deployment, error)
    Get(ctx context.Context, id string) (*Deployment, error)
}

// AuditRepository - handles audit logging
type AuditRepository interface {
    LogCatalogAccess(ctx context.Context, event AuditEvent) error
    LogDeployment(ctx context.Context, deployment Deployment) error
    Query(ctx context.Context, query AuditQuery) ([]AuditEvent, error)
}
```

**Caching Layer:**
```go
// Use go-redis or similar for distributed caching
type CacheStrategy struct {
    MetadataTTL: 5 * time.Minute
    SBOMTTL: 1 * time.Hour
    PolicyTTL: 1 * time.Minute
}
```

**API Endpoints:**
```
/mcp-catalog/api/v1/servers            - GET, list MCP servers
/mcp-catalog/api/v1/servers/{id}       - GET, server details
/mcp-catalog/api/v1/servers/{id}/versions/{version}/sbom - GET, SBOM
/mcp-catalog/api/v1/policies/validate  - POST, validate deployment
/mcp-catalog/api/v1/deployments        - POST, deploy server
/mcp-catalog/api/v1/deployments/{namespace} - GET, list deployments
/mcp-catalog/api/v1/audit              - GET, audit logs (admin only)
/mcp-catalog/api/v1/federation/sources - GET, federated sources
/mcp-catalog/api/v1/healthz            - GET, health check
/mcp-catalog/api/v1/openapi            - GET, OpenAPI spec
```

---

## 3. Governance Policy Architecture

### 3.1 Policy Storage: Kubernetes Custom Resources

**Recommendation:** Use Kubernetes Custom Resource Definitions (CRDs) for policy storage.

**Rationale:**
- Native Kubernetes integration
- RBAC-controlled policy administration
- GitOps-friendly (policies as code)
- Consistent with OpenShift platform patterns
- Version control via kubectl apply

**CRD Design:**

```yaml
apiVersion: mcp.opendatahub.io/v1alpha1
kind: MCPCatalogPolicy
metadata:
  name: require-verified-publisher
  namespace: opendatahub
spec:
  description: "Require verified publisher for all MCP server deployments"
  scope: cluster  # or 'namespace' for namespace-scoped policies
  rules:
    - type: trustTier
      operator: in
      values: ["verified", "partner"]
      action: deny
      message: "Only verified or partner publishers are allowed"
    - type: lifecycle
      operator: notIn
      values: ["eol"]
      action: deny
      message: "End-of-life versions cannot be deployed"
  enforcement: hard  # 'hard' blocks deployment, 'soft' logs warning only
  exemptions:
    namespaces: ["ml-sandbox"]  # sandbox namespace is exempt
---
apiVersion: mcp.opendatahub.io/v1alpha1
kind: MCPCatalogPolicy
metadata:
  name: sbom-required
  namespace: ml-production
spec:
  description: "Require SBOM for all deployments in production namespace"
  scope: namespace
  rules:
    - type: sbom
      operator: required
      action: deny
      message: "SBOM is required for production deployments"
  enforcement: hard
```

### 3.2 Policy Evaluation Engine: Open Policy Agent (OPA)

**Recommendation:** Integrate OPA for flexible policy evaluation.

**Rationale:**
- Industry standard for cloud-native policy enforcement
- Rego policy language is expressive and auditable
- Supports complex rules (e.g., "deny if SBOM contains vulnerability with CVSS > 7.0")
- Decoupled from application code
- Can evaluate policies against SBOM content, not just metadata

**OPA Integration Pattern:**

```
BFF receives deployment request
    ↓
BFF gathers policy input bundle:
  - MCP server metadata
  - User identity and groups
  - Namespace context
  - SBOM (if policy requires)
    ↓
BFF calls OPA HTTP API: POST /v1/data/mcp/catalog/allow
    ↓
OPA evaluates all applicable policies (CRDs converted to Rego)
    ↓
OPA returns decision: allow/deny + reasons
    ↓
BFF enforces decision
```

**Example Rego Policy:**

```rego
package mcp.catalog

import future.keywords.if
import future.keywords.in

# Default deny
default allow = false

# Allow if all rules pass
allow if {
    lifecycle_allowed
    trust_tier_allowed
    sbom_vulnerabilities_acceptable
}

lifecycle_allowed if {
    input.mcpServer.lifecycle in ["stable", "deprecated"]
}

trust_tier_allowed if {
    input.mcpServer.trustTier in ["verified", "partner"]
}

sbom_vulnerabilities_acceptable if {
    critical_vulns := [v | v := input.sbom.vulnerabilities[_]; v.severity == "CRITICAL"]
    count(critical_vulns) == 0
}

# Exemptions
allow if {
    input.namespace in data.exempted_namespaces
}
```

### 3.3 Policy Enforcement: Hybrid Client-Side + Server-Side

**Client-Side (BFF → UI):**
- BFF API returns policy constraints with metadata: `GET /mcp-catalog/api/v1/servers` includes policy hints
- UI filters/grays out disallowed options
- UI shows inline warnings for soft policies
- **Purpose:** Fast user feedback, reduced friction

**Server-Side (Gateway):**
- Gateway validates ALL deployment requests against OPA
- Gateway is authoritative enforcement point
- **Purpose:** Security enforcement, cannot be bypassed

**Flow:**

```
User browses catalog
    ↓
BFF returns servers with policy hints: { "servers": [...], "policyHints": { "blocked": ["server-x"], "warnings": ["server-y"] } }
    ↓
UI visually indicates blocked/warned servers
    ↓
User selects allowed server and clicks Deploy
    ↓
BFF validates with OPA (pre-check)
    ↓
BFF forwards to Gateway
    ↓
Gateway validates with OPA (authoritative check)
    ↓
Gateway deploys if allowed
```

---

## 4. Data Architecture and Schemas

### 4.1 Core Data Models

**MCPServerMetadata:**
```typescript
interface MCPServerMetadata {
  id: string;                    // Unique identifier (e.g., "postgres-mcp-server")
  name: string;                  // Display name
  description: string;
  publisher: Publisher;
  versions: MCPServerVersion[];
  trustTier: TrustTier;
  capabilities: string[];        // Tags for filtering (e.g., ["database", "sql"])
  documentation: string;         // URL to docs
  iconUrl?: string;              // Optional icon
  metadata: {
    created: string;             // ISO 8601
    lastModified: string;
    registrySource: string;      // Which registry this came from (for federation)
  };
}

interface Publisher {
  id: string;
  name: string;
  verified: boolean;
  verificationDate?: string;     // ISO 8601
  verificationAuthority?: string; // E.g., "Red Hat", "CNCF"
  website?: string;
}

interface MCPServerVersion {
  version: string;               // Semantic version (e.g., "2.0.0")
  lifecycle: LifecycleState;
  releaseDate: string;           // ISO 8601
  changelog?: string;
  deprecationDate?: string;      // ISO 8601, if lifecycle is deprecated
  eolDate?: string;              // ISO 8601, if lifecycle is eol
  sbomUrl: string;               // URL to fetch SBOM
  vulnerabilities?: VulnerabilitySummary;
  resourceRequirements: ResourceRequirements;
  dependencies: Dependency[];
  signature?: Signature;         // For supply chain verification
}

enum TrustTier {
  Verified = "verified",         // Red Hat verified
  Partner = "partner",           // Verified partner
  Community = "community",       // Community contribution, unverified
  Unverified = "unverified"      // No verification
}

enum LifecycleState {
  Experimental = "experimental", // Early stage, not production-ready
  Stable = "stable",             // Production-ready
  Deprecated = "deprecated",     // Discouraged, will reach EOL
  EOL = "eol"                    // End-of-life, should not be used
}

interface VulnerabilitySummary {
  critical: number;
  high: number;
  medium: number;
  low: number;
  lastScanned: string;           // ISO 8601
}

interface ResourceRequirements {
  cpu: string;                   // Kubernetes format (e.g., "500m")
  memory: string;                // Kubernetes format (e.g., "512Mi")
  gpu?: string;
  storage?: string;
}

interface Dependency {
  name: string;
  version: string;
  type: "runtime" | "build";
}

interface Signature {
  algorithm: string;             // E.g., "SHA256-RSA"
  value: string;
  signedBy: string;
}
```

**SBOM Models:**
```typescript
// Support both SPDX and CycloneDX
interface SBOM {
  format: "SPDX" | "CycloneDX";
  version: string;
  content: SPDXDocument | CycloneDXBOM; // Parsed JSON
  rawUrl: string;                // URL to raw SBOM file
}

// Minimal SPDX representation
interface SPDXDocument {
  spdxVersion: string;
  name: string;
  SPDXID: string;
  packages: SPDXPackage[];
}

interface SPDXPackage {
  SPDXID: string;
  name: string;
  versionInfo: string;
  licenseConcluded?: string;
  externalRefs?: SPDXExternalRef[];
}

interface SPDXExternalRef {
  referenceCategory: string;
  referenceType: string;
  referenceLocator: string;      // E.g., CPE or PURL
}
```

**Audit Event Schema:**
```typescript
interface AuditEvent {
  id: string;                    // UUID
  timestamp: string;             // ISO 8601
  eventType: AuditEventType;
  user: string;                  // Username
  userGroups: string[];
  namespace: string;
  mcpServerId?: string;
  version?: string;
  action: string;                // "viewed", "deployed", "deleted", "policy-violation"
  result: "success" | "failure";
  metadata: Record<string, any>; // Flexible metadata
  ipAddress?: string;
  userAgent?: string;
}

enum AuditEventType {
  CatalogView = "catalog.view",
  ServerView = "server.view",
  SBOMView = "sbom.view",
  Deployment = "deployment.create",
  PolicyViolation = "policy.violation",
  DeprecationNotification = "deprecation.notification"
}
```

### 4.2 Storage Backend Recommendations

**Catalog Metadata Storage:**
- **Option 1 (Recommended):** Use Registry as single source of truth, BFF caches in-memory with Redis
- **Option 2:** Replicate to PostgreSQL for advanced querying (joins, full-text search)
- **Rationale:** Catalog metadata is read-heavy, caching is sufficient for most use cases

**SBOM Storage:**
- **Option 1 (Recommended):** Store in object storage (S3, Minio) referenced by URL in metadata
- **Option 2:** Store inline in Registry if small (<100KB)
- **Rationale:** SBOMs can be large, object storage is cost-effective and scalable

**Audit Logs Storage:**
- **Option 1 (Recommended):** Time-series database (InfluxDB, TimescaleDB extension for PostgreSQL)
- **Option 2:** Elasticsearch for full-text search and complex queries
- **Option 3:** Object storage with Parquet format for long-term retention + analytics
- **Rationale:** Audit logs are append-only, time-series optimized storage is efficient

**Policy Storage:**
- **Kubernetes CRDs** (as discussed in section 3.1)

---

## 5. Security Architecture

### 5.1 Authentication and Authorization

**Authentication:**
- Leverage existing ODH Dashboard authentication via kube-rbac-proxy
- BFF extracts user identity from `x-auth-request-user` header
- BFF validates token with Kubernetes API (SelfSubjectReview)
- Service-to-service communication (BFF ↔ Registry, Gateway) uses mutual TLS or service account tokens

**Authorization (RBAC Model):**

Define Kubernetes RBAC for MCP Catalog operations:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mcp-catalog-user
rules:
  - apiGroups: ["mcp.opendatahub.io"]
    resources: ["mcpservers"]
    verbs: ["get", "list"]
  - apiGroups: ["mcp.opendatahub.io"]
    resources: ["mcpservers/deploy"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mcp-catalog-admin
rules:
  - apiGroups: ["mcp.opendatahub.io"]
    resources: ["mcpservers", "mcpcatalogpolicies"]
    verbs: ["*"]
  - apiGroups: ["mcp.opendatahub.io"]
    resources: ["auditlogs"]
    verbs: ["get", "list"]
```

**BFF Authorization Checks:**
- Before serving catalog data, BFF performs SelfSubjectAccessReview (SSAR) to verify user has `get` permission on `mcpservers`
- Before initiating deployment, BFF checks user has `create` permission on `mcpservers/deploy` in target namespace
- Admin routes (audit logs, policy management) check for cluster-admin or dedicated admin role

### 5.2 Supply Chain Security

**Publisher Verification:**
- Publishers sign MCP server packages with GPG keys
- Registry stores public keys and verifies signatures
- Catalog displays verification status with chain of custody
- **Implementation:** Use Sigstore/Cosign pattern for container signing

**SBOM Integration:**
- Require SBOMs in SPDX or CycloneDX format for all MCP servers
- Store SBOMs with immutable references (content-addressable by hash)
- Integrate with vulnerability scanning tools (Clair, Trivy, Snyk)
- Display vulnerability summary in catalog UI

**Signature Verification Flow:**
```
User selects MCP server version
    ↓
BFF fetches metadata including signature
    ↓
BFF (or UI) verifies signature against publisher's public key
    ↓
Display verification status: "Verified by Red Hat" or "Signature verification failed"
```

### 5.3 Audit Logging Requirements

**Comprehensive Audit Trail:**
- Log ALL catalog interactions (views, searches, deployments)
- Include user identity, timestamp, namespace, MCP server details
- Log policy violations (attempted deployments blocked by policy)
- Immutable log storage (append-only, tamper-evident)

**Audit Query Capabilities:**
- "Show all deployments by user X in last 90 days"
- "Show all policy violations for namespace Y"
- "Show all deployments of MCP server Z"
- "Show all catalog access for compliance period"

**Retention Policy:**
- Hot storage: 90 days (fast queries)
- Warm storage: 1 year (slower queries, compressed)
- Cold storage: 7 years (archival, compliance)

**Privacy Considerations:**
- PII in audit logs (usernames, IP addresses) - consider GDPR implications
- Provide data export/deletion capabilities for GDPR compliance

---

## 6. Scalability and Performance Architecture

### 6.1 Performance Requirements

Based on the RFE acceptance criteria:
- **Catalog load time:** <2 seconds
- **Pagination:** Support 100+ entries
- **Concurrent users:** Design for 100+ platform engineers accessing simultaneously

### 6.2 Caching Strategy

**Multi-Layer Caching:**

```
┌─────────────────────────────────────────────────┐
│ Layer 1: Browser Cache                          │
│ - Static assets (JS, CSS): 1 year              │
│ - API responses with ETag: 1 minute            │
└─────────────────────────────────────────────────┘
               ↓ Cache miss
┌─────────────────────────────────────────────────┐
│ Layer 2: BFF In-Memory Cache (go-cache)        │
│ - Metadata lists: 5 minutes                    │
│ - Individual server details: 5 minutes         │
└─────────────────────────────────────────────────┘
               ↓ Cache miss
┌─────────────────────────────────────────────────┐
│ Layer 3: Redis Distributed Cache               │
│ - Metadata lists: 5 minutes                    │
│ - SBOM: 1 hour (immutable per version)         │
│ - Policy evaluation results: 1 minute          │
└─────────────────────────────────────────────────┘
               ↓ Cache miss
┌─────────────────────────────────────────────────┐
│ Layer 4: Registry API                          │
│ - Source of truth                              │
└─────────────────────────────────────────────────┘
```

**Cache Invalidation:**
- **Time-based:** TTL expiration as specified above
- **Event-driven:** Registry publishes events (webhook or message queue) when metadata changes
- BFF subscribes to events and invalidates cache proactively

### 6.3 Pagination and Filtering

**Backend Pagination:**
- Registry API supports `page` and `pageSize` query parameters
- Default page size: 50, max: 200
- Return total count in metadata for client-side pagination controls

**Client-Side Filtering:**
- After fetching paginated results, UI applies additional filters (search text, capability tags)
- For large catalogs (1000+ entries), consider server-side filtering with query parameters

**Lazy Loading:**
- Initial catalog load: Fetch first page (50 entries) with minimal metadata
- Detail view: Fetch full metadata including SBOM on demand
- Infinite scroll or "Load More" button for additional pages

### 6.4 Federation Scalability

**Federated Registry Sync:**
- Background job syncs federated registries every 15 minutes (configurable)
- Store federated metadata with source identifier to prevent collisions
- Merge results with conflict resolution rules:
  - Primary registry wins for duplicate IDs
  - Federated sources are additive, not authoritative

**Conflict Resolution Example:**
```
Primary Registry: postgres-mcp-server v2.0.0 (verified)
Federated Registry: postgres-mcp-server v2.0.0 (community)
    ↓
Conflict Resolution Rule: Primary registry wins
    ↓
Catalog displays: postgres-mcp-server v2.0.0 (verified, source: primary-registry)
```

### 6.5 Horizontal Scaling

**BFF Scaling:**
- BFF is stateless (uses Redis for shared cache)
- Deploy multiple BFF replicas with Kubernetes Deployment
- Load balance with Kubernetes Service
- No sticky sessions required

**Database/Storage Scaling:**
- Redis: Use Redis Cluster for horizontal scaling
- Audit logs: Time-series DB sharding by timestamp
- SBOM storage: Object storage is inherently scalable

---

## 7. Federation Architecture

### 7.1 Federation Use Cases

1. **Multi-Organization:** Large enterprises with multiple subsidiaries, each with their own MCP registry
2. **Partner Ecosystems:** Red Hat partners publish MCP servers in their own registries
3. **Air-Gapped Environments:** Disconnected registries that sync periodically
4. **Regional Catalogs:** Different regions with different compliance requirements

### 7.2 Federation Patterns

**Pattern 1: Hub-and-Spoke (Recommended for MVP)**

```
Primary Registry (Hub)
    ↓ Pull/Sync
Partner Registry A ←─┐
Partner Registry B ←─┼─ BFF aggregates from all sources
Internal Registry  ←─┘
```

- BFF pulls metadata from all configured registries
- BFF merges and deduplicates results
- Conflict resolution rules defined by admin
- Simple to implement, suitable for 5-10 federated sources

**Pattern 2: Mesh Federation (Future)**

```
Registry A ←→ Registry B
    ↑  ↖        ↗  ↑
    ↓    ↘    ↙    ↓
Registry C ←→ Registry D
```

- Registries sync with each other peer-to-peer
- BFF queries local registry, which has replicated metadata
- More complex, suitable for 10+ sources with high availability requirements

### 7.3 Trust Model for Federation

**Trust Tiers with Source:**
```typescript
interface FederatedServer extends MCPServerMetadata {
  source: {
    registryId: string;
    registryName: string;
    trustLevel: "primary" | "partner" | "community";
  };
  effectiveTrustTier: TrustTier; // Computed based on publisher AND source
}
```

**Trust Computation Rules:**
- Verified publisher from primary registry: `verified`
- Verified publisher from partner registry: `partner`
- Community publisher from primary registry: `community`
- Any publisher from community registry: `unverified`

**Policy Enforcement with Federation:**
- Policies can specify allowed registry sources: `allowedSources: ["primary-registry", "partner-registry-A"]`
- Block deployments from untrusted sources

### 7.4 Synchronization Protocol

**Pull-Based Sync (Recommended):**
- BFF periodically polls federated registries (every 15 minutes)
- Use ETags or Last-Modified headers to detect changes
- Only fetch changed metadata (incremental sync)

**Push-Based Sync (Advanced):**
- Federated registries push changes to BFF via webhook
- Requires BFF to expose webhook endpoint
- More real-time but adds complexity

**Sync Pseudocode:**
```
every 15 minutes:
  for each federated registry:
    currentETag = cache.get("registry-{id}-etag")
    response = httpGet(registry.url + "/api/v1/mcpservers", headers: {"If-None-Match": currentETag})
    if response.status == 304:  // Not Modified
      continue
    newMetadata = response.body
    cache.set("registry-{id}-metadata", newMetadata)
    cache.set("registry-{id}-etag", response.headers["ETag"])
```

---

## 8. Disconnected/Air-Gapped Architecture

### 8.1 Requirements

- Enterprise customers operate in air-gapped environments with no internet access
- Catalog must function without connectivity to external registries
- Metadata and SBOMs must be available offline
- Periodic sync from external sources via "sneakernet" or bastion hosts

### 8.2 Architecture for Disconnected Environments

**Mirror Registry Pattern:**

```
┌──────────────────────────────────────────────────┐
│ Internet-Connected Environment                   │
│   ┌────────────────┐                             │
│   │ Public Registry│                             │
│   └────────┬───────┘                             │
│            │                                      │
│            ▼                                      │
│   ┌────────────────┐                             │
│   │ Sync Tool      │ Exports bundle:             │
│   │ (oc-mirror)    │ - Metadata JSON             │
│   │                │ - SBOMs                     │
│   │                │ - Container images          │
│   └────────┬───────┘                             │
└────────────┼────────────────────────────────────┘
             │ Bundle transferred via:
             │ - USB drive
             │ - Secure file transfer
             ▼
┌──────────────────────────────────────────────────┐
│ Air-Gapped Environment                           │
│   ┌────────────────┐                             │
│   │ Import Tool    │                             │
│   │                │ Imports bundle to:          │
│   └────────┬───────┘                             │
│            ▼                                      │
│   ┌────────────────┐                             │
│   │ Mirror Registry│                             │
│   │ (Disconnected) │                             │
│   └────────┬───────┘                             │
│            │                                      │
│            ▼                                      │
│   ┌────────────────┐                             │
│   │ MCP Catalog    │                             │
│   │ (reads from    │                             │
│   │  local mirror) │                             │
│   └────────────────┘                             │
└──────────────────────────────────────────────────┘
```

**Tooling Requirements:**
- Extend OpenShift `oc-mirror` tool or create MCP-specific mirror tool
- Bundle format: TAR archive with structured metadata and blobs
- Signature verification for bundle integrity

**Disconnected Registry Configuration:**
```yaml
apiVersion: mcp.opendatahub.io/v1alpha1
kind: MCPRegistryConfig
metadata:
  name: disconnected-registry
spec:
  type: disconnected
  url: https://mirror-registry.internal.corp/mcp
  authentication:
    type: mtls
    secretRef: mirror-registry-tls
  syncSchedule: manual  # No automatic sync, manual import only
```

### 8.3 Sync Workflow

**Export (Connected Environment):**
```bash
$ mcp-catalog-mirror export \
    --source https://registry.mcp.redhat.com \
    --output mcp-catalog-bundle-2025-10-20.tar.gz \
    --include-sboms \
    --include-images
```

**Transfer (via secure channel):**
```bash
$ scp mcp-catalog-bundle-2025-10-20.tar.gz bastion-host:/tmp/
```

**Import (Air-Gapped Environment):**
```bash
$ mcp-catalog-mirror import \
    --bundle /tmp/mcp-catalog-bundle-2025-10-20.tar.gz \
    --destination mirror-registry.internal.corp/mcp \
    --verify-signatures
```

---

## 9. Deprecation and Lifecycle Management

### 9.1 Lifecycle State Transitions

**State Machine:**

```
┌──────────────┐
│ Experimental │
└──────┬───────┘
       │ promote
       ▼
┌──────────────┐
│    Stable    │
└──────┬───────┘
       │ deprecate (set deprecation date)
       ▼
┌──────────────┐
│  Deprecated  │
└──────┬───────┘
       │ EOL date reached
       ▼
┌──────────────┐
│     EOL      │
└──────────────┘
```

**Transition Triggers:**
- Manual by publisher/admin
- Automatic by policy (e.g., "deprecate after 2 years with no updates")

### 9.2 Deprecation Notification Architecture

**Event-Driven Notifications:**

```
Lifecycle change event published to message queue (Kafka, NATS)
    ↓
Notification Service consumes event
    ↓
Notification Service queries: "Which namespaces have this MCP server deployed?"
    ↓
For each namespace with deployment:
    ↓
Send notification:
  - Email to namespace owners (from RBAC group)
  - Dashboard alert (persistent banner)
  - Slack/webhook (if configured)
```

**Notification Content:**
```
Subject: Action Required: MCP Server "postgres-mcp-server" version 1.5.0 is deprecated

The MCP server you have deployed is deprecated:
- Server: postgres-mcp-server
- Version: 1.5.0
- Namespace: ml-project-1
- Deprecation Date: 2025-11-01
- EOL Date: 2026-02-01 (90 days notice)

Recommended Action:
Upgrade to version 2.0.0 (stable)

Migration Guide: https://docs.example.com/mcp/postgres/migration-1.5-to-2.0
```

**Notification Timing:**
- T-90 days: First notice (EOL in 90 days)
- T-30 days: Second notice
- T-7 days: Final warning
- T-0 days: EOL reached, deployment blocked (if policy enforces)

**Dashboard Alerts:**
- Persistent banner on catalog page: "You have 3 deployments using deprecated MCP servers. Review now."
- Notification center with action items

### 9.3 Migration Assistance

**Migration Path Metadata:**
```typescript
interface DeprecationInfo {
  deprecatedVersion: string;
  recommendedVersion: string;
  migrationGuideUrl: string;
  breakingChanges: string[];      // List of breaking changes
  estimatedMigrationEffort: "low" | "medium" | "high";
  autoMigrationAvailable: boolean; // If true, one-click migration possible
}
```

**One-Click Migration (Future):**
- For compatible upgrades, provide "Upgrade Now" button in catalog
- BFF orchestrates: backup config → deploy new version → migrate config → delete old version

---

## 10. Observability Architecture

### 10.1 Metrics (Prometheus)

**Key Metrics to Collect:**

```
# Catalog usage metrics
mcp_catalog_page_views_total{namespace, user}
mcp_catalog_search_queries_total{namespace, user, search_term}
mcp_catalog_server_views_total{mcp_server_id, version}

# Deployment metrics
mcp_deployments_total{mcp_server_id, version, namespace, status="success|failure"}
mcp_deployments_active{mcp_server_id, version, namespace}

# Policy metrics
mcp_policy_evaluations_total{policy_id, result="allow|deny"}
mcp_policy_violations_total{policy_id, namespace}

# Performance metrics
mcp_catalog_api_request_duration_seconds{endpoint, method, status}
mcp_catalog_cache_hit_ratio{layer="memory|redis"}
mcp_registry_api_request_duration_seconds{registry_id}

# Health metrics
mcp_catalog_bff_up
mcp_registry_reachable{registry_id}
mcp_gateway_reachable
```

**Grafana Dashboards:**
- MCP Catalog Usage Dashboard: page views, popular servers, search trends
- MCP Deployments Dashboard: deployments over time, success/failure rates
- MCP Policy Dashboard: policy violations, top violated policies
- MCP Performance Dashboard: API latency, cache hit rates, error rates

### 10.2 Distributed Tracing (OpenTelemetry)

**Trace Spans:**

```
Request: User views MCP server detail page
    ├─ BFF: GET /mcp-catalog/api/v1/servers/{id}
    │   ├─ Cache: Check Redis cache
    │   ├─ Registry: GET /api/v1/mcpservers/{id} (cache miss)
    │   └─ Transform: Enrich with policy hints
    ├─ BFF: GET /mcp-catalog/api/v1/servers/{id}/versions/{version}/sbom
    │   ├─ Cache: Check Redis cache
    │   ├─ Object Storage: Fetch SBOM file (cache miss)
    │   └─ Parse: Parse SBOM JSON
    └─ UI: Render detail page
```

**Key Traces to Instrument:**
- Full request lifecycle (UI → BFF → Registry/Gateway)
- Policy evaluation path
- SBOM retrieval and parsing
- Federation sync jobs

### 10.3 Logging

**Structured Logging (JSON format):**

```json
{
  "timestamp": "2025-10-20T10:30:00Z",
  "level": "info",
  "service": "mcp-catalog-bff",
  "traceId": "abc123...",
  "spanId": "def456...",
  "user": "platform-engineer-1",
  "namespace": "ml-project-1",
  "message": "MCP server deployment initiated",
  "mcpServerId": "postgres-mcp-server",
  "version": "2.0.0",
  "duration_ms": 45
}
```

**Log Aggregation:**
- Use existing OpenShift logging stack (EFK or Loki)
- Retain logs for 30 days in hot storage, archive for compliance

---

## 11. Technical Risks and Mitigation

### 11.1 Critical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Registry API instability** | High - Catalog unavailable if Registry is down | Medium | Implement robust caching (5-minute TTL), serve stale data if Registry unreachable, circuit breaker pattern |
| **Gateway integration coupling** | High - Tight coupling makes changes difficult | Medium | Define clear API contracts with versioning, use interface-based design in BFF |
| **SBOM parsing performance** | Medium - Large SBOMs (>1MB) slow down detail page | High | Lazy load SBOMs, parse asynchronously, cache parsed results, consider SBOM summary API |
| **Policy evaluation latency** | Medium - Slow policy evaluation blocks deployments | Medium | Cache policy evaluation results (1-minute TTL), optimize OPA policy complexity, set 500ms timeout |
| **Audit log storage growth** | Medium - Logs grow unbounded, impact cost/performance | High | Implement time-based partitioning, automated archival, retention policies |
| **Federation conflict resolution** | Low - Conflicting metadata from federated sources | Medium | Define clear conflict resolution rules, prefer primary registry, allow admin overrides |
| **Disconnected sync complexity** | Medium - Mirror tooling is complex to implement | Medium | Start with manual export/import, automate incrementally, leverage existing oc-mirror patterns |

### 11.2 Risk Mitigation Strategies

**Registry Resilience:**
```go
// Circuit breaker pattern
func (r *MCPServersRepository) List(ctx context.Context) (*MCPServerList, error) {
    if r.circuitBreaker.IsOpen() {
        // Serve from cache (even if stale)
        return r.cache.Get("mcp-servers-list")
    }

    servers, err := r.registryClient.List(ctx)
    if err != nil {
        r.circuitBreaker.RecordFailure()
        // Fallback to cache
        return r.cache.Get("mcp-servers-list")
    }

    r.circuitBreaker.RecordSuccess()
    r.cache.Set("mcp-servers-list", servers, 5*time.Minute)
    return servers, nil
}
```

**SBOM Performance Optimization:**
```typescript
// Lazy load SBOM only when user expands "Dependencies" section
const SBOMViewer: React.FC = () => {
    const [sbom, setSBOM] = useState<SBOM | null>(null);
    const [expanded, setExpanded] = useState(false);

    useEffect(() => {
        if (expanded && !sbom) {
            // Fetch SBOM on demand
            fetchSBOM(mcpServerId, version).then(setSBOM);
        }
    }, [expanded]);

    return (
        <Accordion>
            <AccordionItem onClick={() => setExpanded(true)}>
                {sbom ? <SBOMDetails sbom={sbom} /> : <Spinner />}
            </AccordionItem>
        </Accordion>
    );
};
```

---

## 12. Architectural Decision Records (ADRs) Needed

The following ADRs should be created before implementation begins:

### ADR-001: BFF Pattern for MCP Catalog Integration
- **Decision:** Use Backend-for-Frontend pattern with Go BFF
- **Context:** Need to aggregate Registry and Gateway calls, enforce authorization, implement caching
- **Alternatives Considered:** Direct UI-to-Registry calls, GraphQL gateway, API gateway
- **Consequences:** Additional deployment unit, but improved security and performance

### ADR-002: Policy Storage in Kubernetes CRDs
- **Decision:** Store governance policies as Custom Resources
- **Context:** Need GitOps-friendly, RBAC-controlled policy management
- **Alternatives Considered:** ConfigMaps, external policy DB, OPA bundles
- **Consequences:** Native Kubernetes integration, version control, but requires CRD installation

### ADR-003: OPA for Policy Evaluation
- **Decision:** Use Open Policy Agent for policy evaluation
- **Context:** Need flexible, auditable policy language supporting complex rules
- **Alternatives Considered:** Custom policy engine, JavaScript-based policies, hardcoded rules
- **Consequences:** Industry standard, expressive Rego language, but adds dependency

### ADR-004: Hybrid Client-Side and Server-Side Policy Enforcement
- **Decision:** Validate policies client-side (BFF) for UX, enforce server-side (Gateway) for security
- **Context:** Balance between user experience (fast feedback) and security (authoritative enforcement)
- **Alternatives Considered:** Client-side only, server-side only
- **Consequences:** Best UX and security, but requires maintaining policy logic in two places

### ADR-005: Redis for Distributed Caching
- **Decision:** Use Redis for caching metadata, SBOM, and policy results
- **Context:** Need distributed cache shared across BFF replicas
- **Alternatives Considered:** In-memory cache only, Memcached, PostgreSQL cache
- **Consequences:** Scalable, low latency, but adds infrastructure dependency

### ADR-006: Time-Series Database for Audit Logs
- **Decision:** Use TimescaleDB (PostgreSQL extension) for audit logs
- **Context:** Need efficient time-based queries and retention policies
- **Alternatives Considered:** Elasticsearch, InfluxDB, object storage with Parquet
- **Consequences:** Efficient time-based queries, PostgreSQL compatibility, but requires TimescaleDB setup

### ADR-007: SBOM Storage in Object Storage
- **Decision:** Store SBOMs in S3-compatible object storage
- **Context:** SBOMs can be large (>1MB), need cost-effective storage
- **Alternatives Considered:** Store inline in Registry, PostgreSQL BLOB, filesystem
- **Consequences:** Scalable, cost-effective, but requires object storage service

### ADR-008: Hub-and-Spoke Federation Pattern
- **Decision:** Use hub-and-spoke federation with BFF as aggregator
- **Context:** Need to support 5-10 federated registries for MVP
- **Alternatives Considered:** Mesh federation, registry-to-registry sync
- **Consequences:** Simple to implement, suitable for MVP, but BFF is central point of failure

### ADR-009: Pull-Based Federation Sync
- **Decision:** BFF periodically polls federated registries (every 15 minutes)
- **Context:** Need to sync metadata from external registries
- **Alternatives Considered:** Push-based webhooks, event streaming
- **Consequences:** Simple, no webhook infrastructure needed, but less real-time

### ADR-010: Event-Driven Deprecation Notifications
- **Decision:** Use message queue (Kafka/NATS) for deprecation notifications
- **Context:** Need reliable, asynchronous notification delivery
- **Alternatives Considered:** Direct email from BFF, cron job polling
- **Consequences:** Decoupled, reliable, supports multiple notification channels, but adds message queue dependency

---

## 13. Reference Architecture Diagrams

### 13.1 Detailed Component Architecture (C4 Level 3)

```
┌───────────────────────────────────────────────────────────────────────────┐
│                      OpenShift AI Dashboard (Pod)                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                   MCP Catalog UI (React/TypeScript)                  │  │
│  │                                                                       │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐            │  │
│  │  │ Catalog     │  │ Server       │  │ Deployment     │            │  │
│  │  │ Browser     │  │ Detail View  │  │ Wizard         │            │  │
│  │  └─────────────┘  └──────────────┘  └────────────────┘            │  │
│  │                                                                       │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐            │  │
│  │  │ Search &    │  │ SBOM         │  │ Audit          │            │  │
│  │  │ Filter      │  │ Viewer       │  │ Dashboard      │            │  │
│  │  └─────────────┘  └──────────────┘  └────────────────┘            │  │
│  │                                                                       │  │
│  │  State Management: React Context + Custom Hooks                     │  │
│  │  API Client: Axios + OpenAPI Generated Types                        │  │
│  └───────────────────────────────────────────────────────────────────────┘
│                                     │                                      │
│                                     │ HTTPS /mcp-catalog/api/v1/*         │
│                                     ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │              MCP Catalog BFF (Go Backend-for-Frontend)              │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │                    HTTP Handlers Layer                       │   │  │
│  │  │  /servers → MCPServersHandler                               │   │  │
│  │  │  /servers/{id} → ServerDetailHandler                        │   │  │
│  │  │  /sbom/{id}/{ver} → SBOMHandler                            │   │  │
│  │  │  /deployments → DeploymentsHandler                          │   │  │
│  │  │  /audit → AuditHandler (admin only)                         │   │  │
│  │  │  /policies/validate → PolicyValidationHandler               │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  │                                │                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │               Repository Layer (Domain Logic)                │   │  │
│  │  │  - MCPServersRepository                                      │   │  │
│  │  │  - PoliciesRepository                                        │   │  │
│  │  │  - DeploymentsRepository                                     │   │  │
│  │  │  - AuditRepository                                           │   │  │
│  │  │  - FederationRepository                                      │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  │                                │                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────┐   │  │
│  │  │              Infrastructure Layer                            │   │  │
│  │  │  - RegistryClient (HTTP client)                              │   │  │
│  │  │  - GatewayClient (HTTP client)                               │   │  │
│  │  │  - CacheClient (Redis)                                       │   │  │
│  │  │  - AuditLogWriter (TimescaleDB)                              │   │  │
│  │  │  - OPAClient (Policy evaluation)                             │   │  │
│  │  └─────────────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────────┘
│            │              │              │              │                  │
└────────────┼──────────────┼──────────────┼──────────────┼──────────────────┘
             │              │              │              │
             │              │              │              │
   ┌─────────▼────────┐    │    ┌─────────▼────────┐    │
   │   MCP Registry   │    │    │   MCP Gateway    │    │
   │   (External)     │    │    │   (External)     │    │
   └──────────────────┘    │    └──────────────────┘    │
                            │                             │
                  ┌─────────▼────────┐        ┌─────────▼────────┐
                  │  Redis Cache     │        │  TimescaleDB     │
                  │  (Metadata,      │        │  (Audit Logs)    │
                  │   SBOM, Policy)  │        │                  │
                  └──────────────────┘        └──────────────────┘
```

### 13.2 Data Flow Diagram: Deploy MCP Server

```
┌──────────┐
│   User   │
└────┬─────┘
     │ 1. Clicks "Deploy" on postgres-mcp-server v2.0.0
     ▼
┌─────────────────┐
│   Catalog UI    │
└────┬────────────┘
     │ 2. POST /mcp-catalog/api/v1/deployments
     │    { mcpServerId: "postgres-mcp-server", version: "2.0.0", namespace: "ml-project-1" }
     ▼
┌─────────────────────────────────────────────────────────┐
│                     BFF                                  │
│                                                          │
│  3. Extract user identity from x-auth-request-user      │
│     user = "platform-engineer-1"                        │
│                                                          │
│  4. Check authorization (SSAR)                          │
│     → Kubernetes API: Can user create mcpservers/deploy │
│                       in namespace ml-project-1?        │
│     ← Yes                                               │
│                                                          │
│  5. Fetch MCP server metadata (cache or Registry)      │
│     → Cache: GET mcp-servers:postgres-mcp-server:2.0.0  │
│     ← Cache hit: { metadata, lifecycle: "stable", ... } │
│                                                          │
│  6. Prepare policy input bundle                        │
│     policyInput = {                                     │
│       mcpServer: { id, version, lifecycle, trustTier }, │
│       user: "platform-engineer-1",                      │
│       namespace: "ml-project-1"                         │
│     }                                                    │
│                                                          │
│  7. Validate against policies (OPA)                    │
│     → OPA: POST /v1/data/mcp/catalog/allow              │
│            { input: policyInput }                       │
│     ← { result: { allow: true, policies: [...] } }     │
│                                                          │
│  8. Forward deployment request to Gateway              │
│     → Gateway: POST /api/v1/deployments                 │
│                { mcpServerId, version, namespace, ... } │
│     ← { deploymentId: "uuid-123", status: "pending" }  │
│                                                          │
│  9. Write audit log                                     │
│     → TimescaleDB: INSERT INTO audit_events             │
│                    (timestamp, user, action, ...)       │
│                                                          │
│  10. Return success to UI                              │
│      { deploymentId: "uuid-123", status: "pending" }   │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────┐
│   Catalog UI    │ 11. Shows "Deployment initiated" success message
└─────────────────┘     Polls for deployment status updates
```

### 13.3 Federation Sync Flow

```
┌────────────────────────────────────────────────────────────┐
│              BFF Background Job (every 15 min)              │
└────────────────────────────────────────────────────────────┘
                          │
                          │ 1. Fetch list of federated sources
                          ▼
            ┌──────────────────────────────────────┐
            │  Read MCPFederationConfig CRD        │
            │  → Kubernetes API                    │
            │  ← sources: [primary, partner-A]     │
            └──────────────────────────────────────┘
                          │
          ┌───────────────┴───────────────┐
          │                               │
          ▼                               ▼
┌──────────────────┐          ┌──────────────────┐
│ Sync Primary     │          │ Sync Partner-A   │
│ Registry         │          │ Registry         │
└──────────────────┘          └──────────────────┘
          │                               │
          │ 2. Fetch metadata             │
          │    GET /api/v1/mcpservers     │
          │    with If-None-Match: ETag   │
          │                               │
          ▼                               ▼
┌────────────────────────────────────────────────┐
│  3. Merge Results                              │
│     - Primary registry: 50 servers             │
│     - Partner-A registry: 30 servers           │
│     - Duplicates: 5 servers                    │
│                                                 │
│  4. Apply Conflict Resolution Rules            │
│     For duplicate "postgres-mcp-server":       │
│       Primary (verified) > Partner (partner)   │
│       → Keep Primary version as authoritative  │
│       → Tag Partner version as "alternate"     │
│                                                 │
│  5. Update Cache                               │
│     → Redis: SET mcp-servers-merged-list       │
│              { items: [...75 servers] }        │
│                                                 │
│  6. Update Sync Status                         │
│     → TimescaleDB: INSERT INTO federation_sync │
│                    (timestamp, source, status) │
└────────────────────────────────────────────────┘
```

---

## 14. Implementation Phasing Recommendations

Given the complexity, I recommend a phased implementation approach:

### Phase 1: MVP Foundation (3-4 months)
**Goal:** Basic catalog browsing and deployment with governance

**Scope:**
- BFF implementation with Repository pattern
- Registry integration (read-only, single primary registry)
- UI: Catalog browser, search/filter, server detail view
- Gateway integration for deployment
- Basic policy enforcement (lifecycle, trust tier)
- Audit logging (basic events)
- Authentication/authorization via kube-rbac-proxy

**Out of Scope:**
- Federation
- SBOM parsing and vulnerability display
- Deprecation notifications
- Advanced policy language (OPA)

**Deliverables:**
- Deployed as module in ODH Dashboard
- Users can browse, search, deploy MCP servers
- Policies block experimental versions (configurable)
- Audit logs capture deployments

### Phase 2: Enhanced Governance (2-3 months)
**Goal:** SBOM integration, OPA policies, deprecation management

**Scope:**
- SBOM retrieval and display
- OPA integration for policy evaluation
- Policy CRDs and admin UI
- Deprecation notification system
- Vulnerability summary display (integrate with scanner)
- Migration guidance in catalog

**Deliverables:**
- SBOM visible in detail view
- Flexible policy configuration via CRDs
- Automated deprecation notifications
- Vulnerability counts in catalog list

### Phase 3: Federation and Scale (2-3 months)
**Goal:** Multi-registry support, performance optimization

**Scope:**
- Federation with hub-and-spoke pattern
- Federated source configuration UI
- Conflict resolution rules
- Performance optimization (caching, lazy loading)
- Pagination improvements
- Advanced audit query UI

**Deliverables:**
- Support for 5-10 federated registries
- Catalog scales to 500+ MCP servers
- <2 second page load time maintained
- Advanced audit reports for security teams

### Phase 4: Enterprise Features (2-3 months)
**Goal:** Disconnected environments, multi-cluster, observability

**Scope:**
- Mirror tooling for disconnected environments
- Multi-cluster catalog federation
- Grafana dashboards
- OpenTelemetry tracing
- Approval workflows (integrate with ITSM)
- Advanced RBAC (namespace-scoped catalogs)

**Deliverables:**
- Air-gapped deployment support
- Multi-cluster usage visibility
- Production-ready observability
- Enterprise workflow integrations

**Total Timeline:** 9-13 months for full feature set

---

## 15. Critical Architectural Questions to Resolve

The following questions from the RFE require immediate architectural decisions:

### High Priority (Blocking MVP)

1. **Registry API Contract:**
   - **Question:** What is the exact API specification for Registry? REST, gRPC, or GraphQL?
   - **Recommendation:** REST over HTTPS with OpenAPI 3.0 specification (aligns with industry standards)
   - **Owner:** Registry team
   - **Deadline:** Before Phase 1 starts

2. **Gateway Policy Enforcement API:**
   - **Question:** What API does Gateway expose for policy validation?
   - **Recommendation:** Synchronous REST API: `POST /api/v1/policy/validate` returning allow/deny + reasons
   - **Owner:** Gateway team
   - **Deadline:** Before Phase 1 starts

3. **Trust Tier Definition:**
   - **Question:** What are the specific trust tier levels and criteria?
   - **Recommendation:** Four tiers: verified (Red Hat), partner (verified partner), community (signed), unverified
   - **Owner:** Product Management + Security
   - **Deadline:** Before Phase 1 starts

4. **Policy Storage:**
   - **Question:** Where are policies stored? ConfigMaps, CRDs, or external DB?
   - **Recommendation:** Kubernetes CRDs for GitOps and RBAC integration
   - **Owner:** Architecture team
   - **Deadline:** Before Phase 2 starts

5. **SBOM Storage:**
   - **Question:** Where are SBOMs stored? Registry, object storage, or inline?
   - **Recommendation:** Object storage (S3/Minio) with URL references in Registry metadata
   - **Owner:** Registry team
   - **Deadline:** Before Phase 2 starts

### Medium Priority (Blocking Later Phases)

6. **Federation Protocol:**
   - **Question:** What protocol for federated catalog sync? REST polling, webhooks, or message queue?
   - **Recommendation:** REST polling with ETag for MVP, consider webhooks for Phase 3
   - **Owner:** Architecture team
   - **Deadline:** Before Phase 3 starts

7. **Audit Log Storage:**
   - **Question:** What storage backend for audit logs? DB, Elasticsearch, or object storage?
   - **Recommendation:** TimescaleDB for time-series efficiency and SQL compatibility
   - **Owner:** Platform Engineering
   - **Deadline:** Before Phase 1 starts

8. **Deprecation Notification Channels:**
   - **Question:** What notification channels? Email, Slack, webhook, or dashboard only?
   - **Recommendation:** MVP: Email + dashboard alerts; Later: Slack/webhook integration
   - **Owner:** Product Management
   - **Deadline:** Before Phase 2 starts

### Low Priority (Can be decided during implementation)

9. **UI Framework Details:**
   - **Question:** Specific PatternFly components for catalog list?
   - **Recommendation:** Reuse DataList or Table components from existing model catalog
   - **Owner:** UI/UX team

10. **Pagination Strategy:**
    - **Question:** Server-side or client-side pagination?
    - **Recommendation:** Server-side for scalability, client-side filtering on current page
    - **Owner:** Frontend team

---

## 16. Success Metrics and KPIs

To measure the success of this architecture, I recommend tracking:

**Adoption Metrics:**
- Number of MCP servers deployed via catalog (target: 80% of deployments use catalog by Month 6)
- Number of unique users accessing catalog (target: 100+ platform engineers)
- Catalog page views per week (target: 500+ views/week)

**Governance Metrics:**
- Policy violation prevention rate (target: Block 95% of policy-violating deployments)
- Percentage of deployments with SBOMs (target: 100% after Phase 2)
- Audit log coverage (target: 100% of deployments logged)

**Performance Metrics:**
- Catalog page load time (target: <2 seconds, p95)
- API response time (target: <500ms for list, <200ms for detail with cache)
- Cache hit rate (target: >80%)

**Reliability Metrics:**
- Catalog availability (target: 99.9% uptime)
- Registry API error rate (target: <0.1%)
- Gateway integration success rate (target: >99%)

**Security Metrics:**
- Time to detect deprecated server usage (target: <24 hours)
- Percentage of verified publisher deployments (target: >70%)
- Audit query response time (target: <5 seconds for common queries)

---

## 17. Conclusion and Next Steps

### Summary

The MCP Catalog Integration represents a well-conceived feature that addresses a real enterprise need for governed MCP server deployment. The architectural implications are significant, requiring careful integration with Registry and Gateway, robust policy enforcement, and scalable data storage.

**Key Architectural Recommendations:**
1. Adopt Backend-for-Frontend pattern with Go BFF
2. Use Kubernetes CRDs for policy storage with OPA evaluation
3. Implement hybrid client-side + server-side policy enforcement
4. Store SBOMs in object storage, audit logs in TimescaleDB
5. Start with hub-and-spoke federation pattern
6. Design for disconnected environments from Phase 1
7. Leverage existing ODH Dashboard authentication and modular architecture

**Critical Success Factors:**
- Clear API contracts between Dashboard, Registry, and Gateway teams
- Early agreement on trust tier definitions and policy language
- Performance optimization (caching, lazy loading) from the start
- Comprehensive audit logging architecture

### Immediate Next Steps

1. **Week 1-2: API Contract Definition**
   - Schedule joint working session with Registry and Gateway teams
   - Document OpenAPI specifications for all integration points
   - Agree on authentication/authorization mechanisms

2. **Week 3-4: ADR Review and Approval**
   - Publish the 10 ADRs listed in section 12
   - Schedule architecture review board meeting
   - Get sign-off from stakeholders

3. **Week 5-6: Proof of Concept**
   - Implement basic BFF with Repository pattern
   - Integrate with Registry (read-only)
   - Build simple catalog UI (list view only)
   - Validate performance and integration patterns

4. **Week 7-8: Detailed Design Documents**
   - Data model specification
   - API documentation
   - Policy CRD schema
   - Deployment architecture

5. **Week 9+: Phase 1 Implementation**
   - Begin MVP development following phased plan

### Open Questions for Product/Engineering Leadership

1. **Resource Allocation:** How many engineers will be dedicated to this feature? (Recommend: 2-3 engineers for 6+ months)
2. **Registry/Gateway Readiness:** Are Registry and Gateway APIs stable and documented? If not, what is their timeline?
3. **Policy Governance:** Who defines and maintains the default policy set? (Recommend: Security team owns policies)
4. **Federation Priority:** Is federation truly post-MVP, or do air-gapped customers need it sooner?
5. **Observability Stack:** What is the standard observability stack for ODH Dashboard? (Prometheus, Grafana, OpenTelemetry?)

---

**Document Version:** 1.0
**Last Updated:** 2025-10-20
**Reviewed By:** System Architect (Archie)
**Status:** Ready for Stakeholder Review

This architectural review aligns with the north star architecture for OpenShift AI as an enterprise-grade platform. The proposed patterns leverage industry best practices (BFF, OPA, CRDs) while maintaining consistency with existing ODH Dashboard architecture. The 18-month horizon for this feature positions OpenShift AI to scale to hundreds of MCP server deployments with enterprise governance that creates lasting impact across the AI platform.
