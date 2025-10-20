# MCP Catalog Integration - Architecture Diagrams

**Quick Reference:** Visual architecture diagrams for the MCP Catalog Integration
**Full Review:** /workspace/sessions/agentic-session-1760968191/workspace/vTeam-specs/architecture-review.md
**Date:** 2025-10-20

---

## 1. High-Level System Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        OpenShift AI Dashboard                             │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                  MCP Catalog UI (React/TS)                        │   │
│  │  - Browse catalog                                                 │   │
│  │  - Search & filter                                                │   │
│  │  - Server detail view                                             │   │
│  │  - SBOM viewer                                                    │   │
│  │  - Deployment wizard                                              │   │
│  └────────────────────────────┬─────────────────────────────────────┘   │
│                                │                                          │
│                                │ JSON/HTTPS                               │
│                                │ /mcp-catalog/api/v1/*                    │
│                                ▼                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │            MCP Catalog BFF (Go Backend-for-Frontend)             │   │
│  │  - Aggregate Registry + Gateway calls                            │   │
│  │  - Cache metadata, SBOM, policy results                          │   │
│  │  - Authorization enforcement                                     │   │
│  │  - Audit log writing                                             │   │
│  │  - Federation coordinator                                        │   │
│  └────┬──────────┬──────────┬──────────┬──────────┬─────────────────┘   │
│       │          │          │          │          │                      │
└───────┼──────────┼──────────┼──────────┼──────────┼──────────────────────┘
        │          │          │          │          │
        │          │          │          │          │
        ▼          ▼          ▼          ▼          ▼
   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
   │  MCP   │ │  MCP   │ │ Redis  │ │ Times  │ │  OPA   │
   │Registry│ │Gateway │ │ Cache  │ │caleDB  │ │Policy  │
   │        │ │        │ │        │ │ Audit  │ │Engine  │
   └────────┘ └────────┘ └────────┘ └────────┘ └────────┘
   Metadata   Deploy      Metadata   Logs       Policy
   Source     Orchestrate Cache      Storage    Eval
```

---

## 2. BFF Repository Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│                         HTTP Handlers Layer                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │ /servers     │  │ /deployments │  │ /audit       │             │
│  │ Handler      │  │ Handler      │  │ Handler      │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
└─────────┼──────────────────┼──────────────────┼──────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Repository Layer (Domain Logic)                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │ MCPServers       │  │ Deployments      │  │ Audit            │ │
│  │ Repository       │  │ Repository       │  │ Repository       │ │
│  │                  │  │                  │  │                  │ │
│  │ - List()         │  │ - Deploy()       │  │ - Log()          │ │
│  │ - Get()          │  │ - List()         │  │ - Query()        │ │
│  │ - GetSBOM()      │  │ - Get()          │  │                  │ │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘ │
└───────────┼────────────────────┼────────────────────┼──────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer (Clients)                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │ RegistryClient   │  │ GatewayClient    │  │ CacheClient      │ │
│  │ (HTTP)           │  │ (HTTP)           │  │ (Redis)          │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Policy Enforcement Flow

```
┌────────────┐
│    User    │ 1. Selects MCP server "postgres-mcp-server v2.0.0"
└─────┬──────┘    Clicks "Deploy"
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Catalog UI                                 │
│  2. POST /mcp-catalog/api/v1/deployments                            │
│     { mcpServerId: "postgres-mcp-server", version: "2.0.0", ... }  │
└─────┬───────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                              BFF                                     │
│                                                                      │
│  3. Extract user identity                                           │
│     user = "platform-engineer-1" (from x-auth-request-user)        │
│                                                                      │
│  4. Check authorization (SelfSubjectAccessReview)                   │
│     → Kubernetes API: Can user deploy in namespace?                 │
│     ← Yes                                                            │
│                                                                      │
│  5. Fetch MCP server metadata (cache or Registry)                  │
│     → Cache hit: { lifecycle: "stable", trustTier: "verified" }    │
│                                                                      │
│  6. Prepare policy input                                            │
│     policyInput = {                                                 │
│       mcpServer: { lifecycle: "stable", trustTier: "verified" },   │
│       user: "platform-engineer-1",                                  │
│       namespace: "ml-project-1"                                     │
│     }                                                                │
│                                                                      │
│  7. Validate policies (CLIENT-SIDE check for UX)                   │
│     → OPA: POST /v1/data/mcp/catalog/allow                          │
│     ← { result: { allow: true } }                                   │
│                                                                      │
│  8. Forward to Gateway                                              │
│     → Gateway: POST /api/v1/deployments                             │
└─────┬───────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          MCP Gateway                                 │
│                                                                      │
│  9. Validate policies (SERVER-SIDE authoritative check)            │
│     → OPA: POST /v1/data/mcp/catalog/allow                          │
│     ← { result: { allow: true } }                                   │
│                                                                      │
│  10. Deploy MCP server to runtime                                   │
│      Create Kubernetes resources (Deployment, Service, ...)        │
│      ← { deploymentId: "uuid-123", status: "pending" }             │
└─────┬───────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                              BFF                                     │
│  11. Write audit log                                                │
│      → TimescaleDB: INSERT INTO audit_events                        │
│                                                                      │
│  12. Return success                                                 │
│      ← { deploymentId: "uuid-123", status: "pending" }             │
└─────┬───────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           Catalog UI                                 │
│  13. Display "Deployment initiated successfully"                    │
│      Show deployment ID and status                                  │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Points:**
- Client-side policy check (step 7): Fast UX feedback, can be bypassed
- Server-side policy check (step 9): Authoritative security enforcement
- Both checks use same OPA policy engine for consistency

---

## 4. Policy Architecture (CRD + OPA)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                                 │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │              MCPCatalogPolicy CRDs                             │ │
│  │                                                                 │ │
│  │  apiVersion: mcp.opendatahub.io/v1alpha1                      │ │
│  │  kind: MCPCatalogPolicy                                        │ │
│  │  metadata:                                                     │ │
│  │    name: require-verified-publisher                           │ │
│  │  spec:                                                         │ │
│  │    rules:                                                      │ │
│  │      - type: trustTier                                         │ │
│  │        operator: in                                            │ │
│  │        values: ["verified", "partner"]                        │ │
│  │        action: deny                                            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                 │                                    │
│                                 │ Watch for changes                  │
│                                 ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │              Policy Sync Controller                            │ │
│  │  - Watches MCPCatalogPolicy CRDs                              │ │
│  │  - Converts CRDs to OPA Rego policies                         │ │
│  │  - Pushes to OPA as bundle                                    │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                 │                                    │
└─────────────────────────────────┼────────────────────────────────────┘
                                  │
                                  │ Update policy bundle
                                  ▼
                    ┌──────────────────────────┐
                    │      OPA Server          │
                    │                          │
                    │  - Loads Rego policies   │
                    │  - Evaluates: allow/deny │
                    │  - Returns decision      │
                    └──────────────────────────┘
                                  ▲
                                  │ Query: allow?
                                  │
                    ┌──────────────────────────┐
                    │      BFF / Gateway       │
                    │  POST /v1/data/mcp/      │
                    │       catalog/allow      │
                    └──────────────────────────┘
```

**Example Rego Policy:**
```rego
package mcp.catalog

default allow = false

# Allow if lifecycle is stable or deprecated
allow if {
    input.mcpServer.lifecycle in ["stable", "deprecated"]
    input.mcpServer.trustTier in ["verified", "partner"]
}

# Block experimental versions
deny["Experimental versions are not allowed in production"] if {
    input.mcpServer.lifecycle == "experimental"
    input.namespace != "ml-sandbox"
}
```

---

## 5. Data Model Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                     MCPServerMetadata                               │
├────────────────────────────────────────────────────────────────────┤
│  id: string                     ◄─┐                                │
│  name: string                     │ Unique identifier              │
│  description: string              │                                │
│  publisher: Publisher          ───┤                                │
│  versions: MCPServerVersion[]  ───┤                                │
│  trustTier: TrustTier          ───┤                                │
│  capabilities: string[]           │                                │
│  documentation: string            │                                │
│  metadata: {                      │                                │
│    created: ISO8601               │                                │
│    lastModified: ISO8601          │                                │
│    registrySource: string      ───┘ For federation tracking       │
│  }                                                                 │
└────────────────────────────────────────────────────────────────────┘
                      │
                      │ 1:N
                      ▼
┌────────────────────────────────────────────────────────────────────┐
│                   MCPServerVersion                                  │
├────────────────────────────────────────────────────────────────────┤
│  version: string (semver)                                          │
│  lifecycle: LifecycleState     ─────► experimental | stable |     │
│                                        deprecated | eol            │
│  releaseDate: ISO8601                                              │
│  changelog: string                                                 │
│  deprecationDate?: ISO8601                                         │
│  eolDate?: ISO8601                                                 │
│  sbomUrl: string                ─────► URL to SBOM in object store│
│  vulnerabilities?: {                                               │
│    critical: number                                                │
│    high: number                                                    │
│    medium: number                                                  │
│    low: number                                                     │
│  }                                                                  │
│  resourceRequirements: {                                           │
│    cpu: "500m"                                                     │
│    memory: "512Mi"                                                 │
│  }                                                                  │
│  dependencies: Dependency[]                                        │
│  signature?: Signature          ─────► GPG/Sigstore signature     │
└────────────────────────────────────────────────────────────────────┘
```

**TrustTier Enum:**
```
┌──────────────────────────────────────────────────────────────┐
│  TrustTier                                                    │
├──────────────────────────────────────────────────────────────┤
│  VERIFIED    → Red Hat verified (GPG signed, scanned)        │
│  PARTNER     → Verified partner (partner signing key)        │
│  COMMUNITY   → Community (self-signed, SBOM provided)        │
│  UNVERIFIED  → No verification (blocked by default)          │
└──────────────────────────────────────────────────────────────┘
```

**LifecycleState Enum:**
```
┌──────────────────────────────────────────────────────────────┐
│  LifecycleState                                               │
├──────────────────────────────────────────────────────────────┤
│  EXPERIMENTAL → Early stage, not production-ready            │
│  STABLE       → Production-ready                             │
│  DEPRECATED   → Discouraged, will reach EOL                  │
│  EOL          → End-of-life, deployment blocked              │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Caching Architecture (4-Layer)

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Browser (Layer 1)                            │
│  Cache-Control: max-age=60 (API responses)                          │
│  Cache-Control: max-age=31536000 (static assets)                    │
│  ETag support for conditional requests                              │
└─────────────────────────────────────────────────────────────────────┘
                                 │ Cache miss
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    BFF In-Memory Cache (Layer 2)                     │
│  Library: go-cache                                                   │
│  - Metadata lists: 5 minutes TTL                                    │
│  - Server details: 5 minutes TTL                                    │
│  - Policy results: 1 minute TTL                                     │
└─────────────────────────────────────────────────────────────────────┘
                                 │ Cache miss
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Redis Distributed Cache (Layer 3)                 │
│  Keys:                                                               │
│  - mcp-servers-list:{filters-hash} → 5 min TTL                     │
│  - mcp-server:{id} → 5 min TTL                                      │
│  - mcp-sbom:{id}:{version} → 1 hour TTL (immutable)                │
│  - mcp-policy:{input-hash} → 1 min TTL                             │
│  - federation-sync:{source-id} → 15 min                             │
└─────────────────────────────────────────────────────────────────────┘
                                 │ Cache miss
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Registry API (Layer 4 - Source)                   │
│  GET /api/v1/mcpservers                                             │
│  GET /api/v1/mcpservers/{id}                                        │
│  GET /api/v1/mcpservers/{id}/versions/{version}/sbom                │
└─────────────────────────────────────────────────────────────────────┘
```

**Cache Invalidation Strategies:**
1. **Time-based:** TTL expiration (primary mechanism)
2. **Event-driven:** Registry publishes change events → BFF invalidates cache
3. **Manual:** Admin can flush cache via API endpoint

---

## 7. Federation Architecture (Hub-and-Spoke)

```
                    ┌─────────────────────┐
                    │   Primary Registry  │
                    │   (Red Hat)         │
                    │   - 100 MCP servers │
                    │   - Authoritative   │
                    └──────────┬──────────┘
                               │
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│ Partner        │    │ Partner        │    │ Internal       │
│ Registry A     │    │ Registry B     │    │ Registry       │
│ - 30 servers   │    │ - 25 servers   │    │ - 10 servers   │
└────────┬───────┘    └────────┬───────┘    └────────┬───────┘
         │                     │                     │
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               │ All sources polled by BFF
                               ▼
              ┌─────────────────────────────────┐
              │      MCP Catalog BFF            │
              │                                 │
              │  Federation Coordinator:        │
              │  - Polls every 15 minutes       │
              │  - Merges results               │
              │  - Applies conflict resolution  │
              │  - Tags source for each server  │
              └─────────────────────────────────┘
                               │
                               │ Merged catalog
                               ▼
              ┌─────────────────────────────────┐
              │      Catalog UI                 │
              │  Displays: 165 MCP servers      │
              │  (100 + 30 + 25 + 10)           │
              │  With source tags               │
              └─────────────────────────────────┘
```

**Conflict Resolution Rules:**
```
If duplicate MCP server ID exists in multiple sources:
  1. Primary registry wins (authoritative)
  2. Other versions tagged with source: "partner-A", "internal"
  3. User can see all versions with source indicator
  4. Policy can restrict to specific sources

Example:
  postgres-mcp-server v2.0.0 in Primary (verified)
  postgres-mcp-server v2.0.0 in Partner-A (partner)
  → Display: postgres-mcp-server v2.0.0 (verified, source: primary)
            postgres-mcp-server v2.0.0 (partner, source: partner-A)
```

---

## 8. Disconnected Environment Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                   CONNECTED ENVIRONMENT                               │
│                                                                        │
│  ┌────────────────────┐                                              │
│  │ Public MCP Registry│                                              │
│  └─────────┬──────────┘                                              │
│            │                                                          │
│            │ 1. Sync metadata, SBOMs, images                         │
│            ▼                                                          │
│  ┌────────────────────┐                                              │
│  │ mcp-catalog-mirror │                                              │
│  │ export tool        │                                              │
│  └─────────┬──────────┘                                              │
│            │                                                          │
│            │ 2. Create bundle                                        │
│            ▼                                                          │
│  ┌────────────────────────────────────────────┐                     │
│  │ Bundle TAR.GZ                              │                     │
│  │ - metadata.json (all MCP servers)          │                     │
│  │ - sboms/ (all SBOM files)                  │                     │
│  │ - images/ (container images)               │                     │
│  │ - signatures/ (GPG signatures)             │                     │
│  │ - checksums.txt (integrity verification)   │                     │
│  └────────────────────────────────────────────┘                     │
│            │                                                          │
└────────────┼──────────────────────────────────────────────────────────┘
             │
             │ 3. Transfer via USB/secure channel
             │
┌────────────┼──────────────────────────────────────────────────────────┐
│            ▼                  AIR-GAPPED ENVIRONMENT                   │
│  ┌────────────────────────────────────────────┐                     │
│  │ Bundle TAR.GZ                              │                     │
│  └────────────────────────────────────────────┘                     │
│            │                                                          │
│            │ 4. Import bundle                                        │
│            ▼                                                          │
│  ┌────────────────────┐                                              │
│  │ mcp-catalog-mirror │                                              │
│  │ import tool        │                                              │
│  │ - Verify signatures│                                              │
│  │ - Validate bundle  │                                              │
│  └─────────┬──────────┘                                              │
│            │                                                          │
│            │ 5. Push to local mirror registry                        │
│            ▼                                                          │
│  ┌────────────────────┐                                              │
│  │ Mirror Registry    │                                              │
│  │ (Disconnected)     │                                              │
│  └─────────┬──────────┘                                              │
│            │                                                          │
│            │ 6. Catalog reads from local mirror                      │
│            ▼                                                          │
│  ┌────────────────────┐                                              │
│  │ MCP Catalog        │                                              │
│  │ - Configured for   │                                              │
│  │   disconnected mode│                                              │
│  │ - No external sync │                                              │
│  └────────────────────┘                                              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**Export Command:**
```bash
mcp-catalog-mirror export \
  --source https://registry.mcp.redhat.com \
  --output mcp-catalog-bundle-2025-10-20.tar.gz \
  --include-sboms \
  --include-images \
  --sign-with gpg-key-id
```

**Import Command:**
```bash
mcp-catalog-mirror import \
  --bundle mcp-catalog-bundle-2025-10-20.tar.gz \
  --destination mirror-registry.internal.corp/mcp \
  --verify-signatures
```

---

## 9. Deprecation Notification Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                        Registry / Admin                             │
│  1. Update MCP server lifecycle: stable → deprecated               │
│     Set deprecationDate: 2025-11-01                                │
│     Set eolDate: 2026-02-01                                        │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                                 │ 2. Publish lifecycle change event
                                 ▼
                    ┌─────────────────────────┐
                    │  Message Queue          │
                    │  (Kafka / NATS)         │
                    └────────┬────────────────┘
                             │
                             │ 3. Consume event
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Notification Service                             │
│                                                                      │
│  4. Query: Which namespaces have this MCP server deployed?          │
│     → BFF API: GET /mcp-catalog/api/v1/deployments?                │
│                     mcpServerId=postgres-mcp-server&version=1.5.0  │
│     ← [{ namespace: "ml-project-1", user: "engineer-1" }, ...]     │
│                                                                      │
│  5. For each namespace, determine owners (RBAC groups)              │
│     → Kubernetes API: GET /api/v1/namespaces/ml-project-1           │
│     ← ownerGroup: "ml-team@company.com"                            │
│                                                                      │
│  6. Send notifications:                                             │
│     a) Email to namespace owners                                    │
│        Subject: "Action Required: MCP Server Deprecated"           │
│        Body: "postgres-mcp-server v1.5.0 will be EOL on 2026-02-01│
│               Please upgrade to v2.0.0"                            │
│                                                                      │
│     b) Dashboard persistent banner                                  │
│        → Write to notifications table                               │
│        → User sees banner: "3 deprecated servers in use"           │
│                                                                      │
│     c) Slack webhook (if configured)                                │
│        POST to Slack webhook URL                                    │
└─────────────────────────────────────────────────────────────────────┘
```

**Notification Timeline:**
```
Version Deprecated: 2025-11-01
EOL Date: 2026-02-01 (90 days later)

T-90 days (2025-11-03): First notice
  "Your MCP server will be EOL in 90 days"

T-30 days (2026-01-02): Second notice
  "Your MCP server will be EOL in 30 days"

T-7 days (2026-01-25): Final warning
  "Your MCP server will be EOL in 7 days - URGENT"

T-0 days (2026-02-01): EOL reached
  "MCP server is EOL - deployment blocked by policy"
  Policy: deny if lifecycle == "eol"
```

---

## 10. Audit Log Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Catalog UI / BFF                             │
│  User actions:                                                       │
│  - View catalog page                                                 │
│  - Search for "postgres"                                             │
│  - View MCP server detail                                            │
│  - View SBOM                                                         │
│  - Deploy MCP server                                                 │
│  - Deployment blocked by policy                                      │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ For each action, write audit event
                             ▼
              ┌──────────────────────────────────┐
              │         Audit Writer             │
              │  (BFF Repository Layer)          │
              └────────────┬─────────────────────┘
                           │
                           │ INSERT INTO audit_events
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    TimescaleDB (PostgreSQL)                          │
│                                                                      │
│  audit_events table (hypertable, partitioned by time):              │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ id            UUID                                             │ │
│  │ timestamp     TIMESTAMPTZ   ← Partition key (1 day chunks)    │ │
│  │ event_type    TEXT           (catalog.view, deployment.create)│ │
│  │ user          TEXT                                             │ │
│  │ user_groups   TEXT[]                                           │ │
│  │ namespace     TEXT                                             │ │
│  │ mcp_server_id TEXT                                             │ │
│  │ version       TEXT                                             │ │
│  │ action        TEXT           (viewed, deployed, policy-blocked)│ │
│  │ result        TEXT           (success, failure)                │ │
│  │ metadata      JSONB          (flexible additional data)        │ │
│  │ ip_address    INET                                             │ │
│  │ user_agent    TEXT                                             │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  Indexes:                                                            │
│  - (timestamp DESC) for time-range queries                          │
│  - (user, timestamp) for user activity queries                      │
│  - (namespace, timestamp) for namespace queries                     │
│  - (mcp_server_id, timestamp) for server deployment tracking        │
│                                                                      │
│  Retention Policy (TimescaleDB):                                    │
│  - Keep last 90 days in hot storage (fast queries)                 │
│  - Compress data older than 30 days                                 │
│  - Archive to S3 after 90 days (for compliance, slower queries)    │
│  - Delete after 7 years (or per customer requirement)               │
└─────────────────────────────────────────────────────────────────────┘
                           ▲
                           │
                           │ Query audit logs
┌─────────────────────────────────────────────────────────────────────┐
│                      Audit Query API                                 │
│  GET /mcp-catalog/api/v1/audit?user=engineer-1&days=90             │
│  GET /mcp-catalog/api/v1/audit?namespace=ml-project-1              │
│  GET /mcp-catalog/api/v1/audit?mcpServerId=postgres-mcp-server     │
│  GET /mcp-catalog/api/v1/audit?eventType=policy.violation          │
└─────────────────────────────────────────────────────────────────────┘
```

**Example Audit Queries:**
```sql
-- Show all deployments by user in last 90 days
SELECT * FROM audit_events
WHERE user = 'platform-engineer-1'
  AND event_type = 'deployment.create'
  AND timestamp > NOW() - INTERVAL '90 days'
ORDER BY timestamp DESC;

-- Show all policy violations by namespace
SELECT * FROM audit_events
WHERE namespace = 'ml-project-1'
  AND event_type = 'policy.violation'
ORDER BY timestamp DESC;

-- Show deployment history for specific MCP server
SELECT user, namespace, version, timestamp, result
FROM audit_events
WHERE mcp_server_id = 'postgres-mcp-server'
  AND event_type = 'deployment.create'
ORDER BY timestamp DESC;
```

---

## 11. Observability Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                     MCP Catalog BFF                                  │
│  Instrumentation:                                                    │
│  - Prometheus metrics export (:8081/metrics)                        │
│  - OpenTelemetry traces                                             │
│  - Structured JSON logs                                             │
└────┬─────────────────────┬──────────────────────┬───────────────────┘
     │                     │                      │
     │ Metrics             │ Traces               │ Logs
     ▼                     ▼                      ▼
┌──────────┐         ┌──────────┐         ┌──────────┐
│Prometheus│         │  Jaeger  │         │   Loki   │
│  Server  │         │          │         │          │
└────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                     │
     │ Visualize          │ Visualize           │ Visualize
     ▼                    ▼                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Grafana Dashboards                           │
│                                                                      │
│  ┌──────────────────────┐  ┌──────────────────────┐               │
│  │ MCP Catalog Usage    │  │ MCP Deployments      │               │
│  │ - Page views         │  │ - Deployments/day    │               │
│  │ - Search queries     │  │ - Success rate       │               │
│  │ - Popular servers    │  │ - Active deployments │               │
│  └──────────────────────┘  └──────────────────────┘               │
│                                                                      │
│  ┌──────────────────────┐  ┌──────────────────────┐               │
│  │ MCP Policy Dashboard │  │ MCP Performance      │               │
│  │ - Policy violations  │  │ - API latency        │               │
│  │ - Top violated       │  │ - Cache hit rate     │               │
│  │ - Exemptions used    │  │ - Error rates        │               │
│  └──────────────────────┘  └──────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Metrics:**
```
# Catalog usage
mcp_catalog_page_views_total{namespace, user}
mcp_catalog_search_queries_total{namespace, search_term}
mcp_catalog_server_views_total{mcp_server_id}

# Deployments
mcp_deployments_total{mcp_server_id, version, namespace, status}
mcp_deployments_active{mcp_server_id, version, namespace}

# Policy
mcp_policy_evaluations_total{policy_id, result}
mcp_policy_violations_total{policy_id, namespace}

# Performance
mcp_catalog_api_request_duration_seconds{endpoint, method}
mcp_catalog_cache_hit_ratio{layer}
mcp_registry_api_request_duration_seconds{registry_id}
```

---

## Summary

These diagrams provide a visual reference for the key architectural patterns in the MCP Catalog Integration:

1. **High-Level Architecture:** BFF pattern with Registry, Gateway, cache, and audit storage
2. **Repository Pattern:** Clean separation of handlers, domain logic, and infrastructure
3. **Policy Enforcement:** Hybrid client-side + server-side validation
4. **Policy Architecture:** CRDs + OPA for flexible, auditable policies
5. **Data Model:** Comprehensive metadata with trust tiers and lifecycle states
6. **Caching:** 4-layer cache for performance (browser → BFF memory → Redis → Registry)
7. **Federation:** Hub-and-spoke with conflict resolution
8. **Disconnected:** Mirror tooling for air-gapped environments
9. **Deprecation:** Event-driven notifications with timeline
10. **Audit Logs:** TimescaleDB with time-series partitioning
11. **Observability:** Prometheus + Jaeger + Loki → Grafana

These patterns align with the north star architecture for OpenShift AI as an enterprise-grade platform, leveraging industry best practices while maintaining consistency with existing ODH Dashboard architecture.

---

**Document Version:** 1.0
**Last Updated:** 2025-10-20
**Related Documents:**
- Full Architecture Review: /workspace/sessions/agentic-session-1760968191/workspace/vTeam-specs/architecture-review.md
- Executive Summary: /workspace/sessions/agentic-session-1760968191/workspace/vTeam-specs/architecture-summary.md
