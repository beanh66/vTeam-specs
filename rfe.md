# MCP Catalog Integration for OpenShift AI Dashboard

**Feature Overview:**
*An elevator pitch (value statement) that describes the Feature in a clear, concise way. ie: Executive Summary of the user goal or problem that is being solved, why does this matter to the user? The "What & Why"...*

Enterprise teams deploying AI workloads on OpenShift AI need a governed, vetted catalog of Model Context Protocol (MCP) servers that provides visibility, control, and trust. This feature integrates a comprehensive MCP Catalog directly into the OpenShift AI Dashboard UI, enabling platform engineers and security teams to discover, evaluate, and manage MCP servers with enterprise-grade governance, lifecycle management, and interoperability. By providing a curated catalog with rich metadata, trust tiers, and policy enforcement, this feature transforms MCP server deployment from an ungoverned process into a secure, manageable enterprise capability.

**Goals:**

*Provide high-level goal statement, providing user context and expected user outcome(s) for this Feature. Who benefits from this Feature, and how? What is the difference between today's current state and a world with this Feature?*

* **Empower Platform Engineers** with a self-service catalog to discover, evaluate, and deploy vetted MCP servers that meet organizational governance standards
* **Enable Security Teams** to enforce policies, verify publishers, track dependencies through SBOMs, and maintain audit trails for all MCP server deployments
* **Provide Enterprise Governance** by integrating lifecycle management, versioning, deprecation flows, and policy enforcement directly into the deployment workflow
* **Accelerate Safe Adoption** of MCP technology by reducing the friction between innovation and compliance through pre-vetted, trusted catalog entries
* **Bridge the Gap** between today's manual, ungoverned MCP server selection and a future state where enterprise controls are seamlessly embedded in the developer experience

**Current State vs. Future State:**
- **Today:** Platform engineers manually discover and deploy MCP servers without centralized governance, version tracking, or security verification
- **Future:** A curated, integrated catalog provides trust-tiered MCP servers with full lifecycle management, policy enforcement, and observability built into the OpenShift AI Dashboard

**Out of Scope:**

*High-level list of items or personas that are out of scope.*

* Development or hosting of MCP servers themselves (catalog is for discovery and metadata only)
* Custom MCP server development tooling or SDKs
* End-user data scientists or ML engineers as primary personas (focus is platform/security teams)
* Runtime execution of MCP servers (handled by Gateway dependency)
* General-purpose software catalog features unrelated to MCP entities
* Migration tooling for legacy/pre-MCP integration patterns

**Requirements:**

*A list of specific needs, capabilities, or objectives that a Feature must deliver to satisfy the Feature. Some requirements will be flagged as MVP. If an MVP gets shifted, the Feature shifts. If a non MVP requirement slips, it does not shift the feature.*

* **[MVP] Catalog UI Integration:** Embed MCP Catalog UI within the existing OpenShift AI Dashboard following the established model catalog UX pattern
* **[MVP] Registry Integration:** Connect to the MCP Registry for fetching server metadata, versions, and descriptors
* **[MVP] Gateway Integration:** Integrate with the MCP Gateway for runtime policy enforcement and server lifecycle coordination
* **[MVP] Versioning Support:** Display and manage multiple versions of MCP servers with clear version indicators and upgrade paths
* **[MVP] Lifecycle States:** Support and display lifecycle states (e.g., experimental, stable, deprecated, end-of-life) for all catalog entries
* **[MVP] Governance Policy Enforcement:** Enforce organizational policies (e.g., only allow stable versions, require security scans) at selection/deployment time
* **[MVP] Trust Tiers:** Display trust/verification levels for MCP servers (e.g., verified publisher, community, unverified)
* **[MVP] Filtering & Search:** Provide robust filtering by lifecycle state, trust tier, capabilities, publisher, and version
* **Deprecation Flows:** Automated workflows and notifications for deprecated MCP servers with migration guidance
* **Federation Support:** Ability to federate with external MCP catalogs and registries beyond the primary registry
* **Publisher Verification:** Display publisher identity verification status and chain of custody
* **SBOM Integration:** Display Software Bill of Materials for each MCP server version to support security scanning and vulnerability tracking
* **Change Lineage:** Provide audit logs showing version history, changes, and deployment lineage for compliance tracking
* **Usage Logs:** Track which MCP servers are deployed, by whom, and where within the cluster
* **Observability Integration:** Surface runtime metrics and health status from Gateway integration
* **Rich Metadata Display:** Show comprehensive metadata including capabilities, resource requirements, dependencies, and documentation links
* **Input Simplification:** Streamline the process of selecting and configuring MCP servers for deployment
* **Responsive Design:** Ensure catalog UI works across desktop and tablet form factors consistent with ODH Dashboard standards

**Done - Acceptance Criteria:**

*Acceptance Criteria articulates and defines the value proposition - what is required to meet the goal and intent of this Feature. The Acceptance Criteria provides a detailed definition of scope and the expected outcomes - from a users point of view*

* Platform engineers can access the MCP Catalog from the OpenShift AI Dashboard navigation with a UX consistent with the existing model catalog
* Users can browse, search, and filter MCP servers by version, lifecycle state, trust tier, publisher, and capabilities
* Each MCP server entry displays comprehensive metadata including description, version history, lifecycle state, trust tier, publisher verification status, SBOM, and documentation links
* Users can view multiple versions of the same MCP server and understand differences between versions
* Governance policies configured by administrators are enforced automatically when users attempt to select or deploy MCP servers (e.g., blocking deprecated versions)
* Trust tiers are clearly indicated with visual cues (badges, colors, icons) to guide safe selection
* Users receive clear notifications when MCP servers are deprecated with guidance on migration paths
* Catalog data is synchronized from the integrated Registry with configurable refresh intervals
* Deployment actions integrate with the Gateway for runtime enforcement and lifecycle management
* Audit logs capture all catalog interactions including searches, selections, and deployments
* Security teams can access usage reports showing which MCP servers are deployed across the platform
* The catalog supports federation with at least one external registry source
* Performance: Catalog loads within 2 seconds and supports pagination for large result sets (100+ entries)
* Accessibility: Catalog meets WCAG 2.1 AA standards consistent with ODH Dashboard

**Use Cases - i.e. User Experience & Workflow:**

*Include use case diagrams, main success scenarios, alternative flow scenarios.*

**Use Case 1: Platform Engineer Discovers and Deploys a Vetted MCP Server**

*Main Success Scenario:*
1. Platform engineer navigates to "MCP Catalog" from ODH Dashboard left navigation
2. Engineer searches for "database" to find MCP servers with database capabilities
3. Catalog displays filtered results with trust tier badges, version info, and lifecycle states
4. Engineer selects a "Postgres MCP Server" entry with "Verified Publisher" and "Stable" lifecycle
5. Detail view shows comprehensive metadata: description, capabilities, versions (1.0.0, 1.1.0, 2.0.0), SBOM, documentation links, and resource requirements
6. Engineer selects version 2.0.0 (latest stable) and clicks "Deploy"
7. Gateway integration validates governance policies (passes: stable version, verified publisher)
8. Deployment is initiated with configuration wizard pre-populated from catalog metadata
9. Audit log records deployment with user, timestamp, and selected configuration

*Alternative Flow 1: Policy Violation*
- At step 7, if engineer selected an experimental version and policy requires stable versions only, deployment is blocked with clear error message explaining policy requirement

*Alternative Flow 2: Deprecated Server Warning*
- At step 4, if selected server version is deprecated, a prominent warning displays with recommended migration path to newer version

**Use Case 2: Security Team Audits MCP Server Usage**

*Main Success Scenario:*
1. Security team member accesses MCP Catalog with elevated permissions
2. Navigates to "Usage & Audit" section
3. Views dashboard showing: all deployed MCP servers, versions in use, publishers, deployment timestamps, and deploying users
4. Filters to show only "community" trust tier servers (unverified publishers)
5. Selects a specific server to view SBOM and checks for known vulnerabilities
6. Downloads audit report for compliance documentation
7. Identifies deprecated server still in use and initiates notification to owning team

**Use Case 3: Administrator Configures Governance Policies**

*Main Success Scenario:*
1. Administrator accesses MCP Catalog admin settings
2. Configures policy: "Block deployment of MCP servers with trust tier below 'Verified'"
3. Configures policy: "Require approval for experimental lifecycle servers"
4. Sets up federation with external partner's MCP registry
5. Configures automated deprecation notifications (email platform engineers 30 days before EOL)
6. Saves policy configuration
7. Policies are immediately enforced for all subsequent catalog interactions

**Workflow Diagram:**
```
[Platform Engineer]
    ↓
[ODH Dashboard] → [MCP Catalog UI]
    ↓
[Search/Filter/Browse]
    ↓
[Select MCP Server] → [View Details: Metadata, SBOM, Versions]
    ↓
[Deploy Action] → [Policy Check via Gateway]
    ↓
    ├─ [Pass] → [Deploy to Runtime] → [Audit Log]
    └─ [Fail] → [Error: Policy Violation]

[Registry] ← [Sync Metadata] → [MCP Catalog UI]
[Gateway] ← [Runtime Enforcement] → [Deployed MCP Servers]
```

**Documentation Considerations:**

*Provide information that needs to be considered and planned so that documentation will meet customer needs. If the feature extends existing functionality, provide a link to its current documentation.*

* **User Guide:** Comprehensive documentation for platform engineers on browsing, filtering, and deploying from the MCP Catalog
  - How to interpret trust tiers and lifecycle states
  - How to read and understand SBOMs
  - How to select appropriate MCP server versions
  - Migration guidance for deprecated servers
* **Administrator Guide:** Documentation for configuring governance policies, setting up federation, and managing catalog settings
  - Policy configuration reference
  - Federation setup with external registries
  - RBAC and permission models
  - Audit log access and reporting
* **Security Guide:** Documentation for security teams on audit capabilities, SBOM interpretation, and vulnerability tracking workflows
* **Architecture Documentation:** Integration points with Registry and Gateway, data flow diagrams, API contracts
* **Release Notes:** Version-specific release notes for catalog updates, new trust tier definitions, and policy capabilities
* **Integration with Existing ODH Dashboard Docs:** This feature extends the existing OpenShift AI Dashboard, so documentation should be integrated into current ODH Dashboard documentation structure
  - Current ODH Dashboard docs: [OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/)
  - Model Catalog pattern reference (similar UX): Link to existing model serving catalog documentation
* **API Documentation:** REST API documentation for programmatic catalog access (if applicable for automation scenarios)

**Questions to answer:**

*Include a list of refinement / architectural questions that may need to be answered before coding can begin.*

* **Registry Integration:**
  - What is the API contract between the MCP Catalog UI and the Registry? (REST, gRPC, GraphQL?)
  - What authentication/authorization mechanism is used for Registry communication?
  - What is the expected latency for Registry metadata fetches?
  - How frequently should catalog metadata be synced from Registry? (real-time, cached with TTL?)
  - What is the data schema for MCP server metadata in the Registry?

* **Gateway Integration:**
  - What APIs does the Gateway expose for policy enforcement?
  - How are deployment requests passed from Catalog UI to Gateway?
  - What feedback does Gateway provide on policy pass/fail? (sync or async?)
  - How does Gateway communicate runtime status back to Catalog UI for observability?
  - What authentication/authorization is required for Gateway integration?

* **Governance & Policy Engine:**
  - Where are governance policies defined and stored? (ConfigMaps, CRDs, external policy engine?)
  - What policy language/format is used? (OPA, custom DSL, declarative YAML?)
  - Are policies evaluated client-side (UI) or server-side (Gateway/Registry)?
  - How are policy conflicts resolved when multiple policies apply?
  - What is the RBAC model for policy administration?

* **Trust Tier Definition:**
  - What are the specific trust tier levels? (e.g., Verified, Partner, Community, Unverified?)
  - What criteria determine trust tier assignment? (signing keys, organizational verification, audit?)
  - Who can modify trust tiers? (only Registry admins, or distributed trust model?)
  - How are trust tiers represented in metadata? (enum, numeric score, multi-dimensional?)

* **Federation:**
  - What protocols are supported for external catalog federation? (OCI registries, custom API, catalog standards?)
  - How are conflicts resolved when federated catalogs contain duplicate entries?
  - How is trust tier inherited or translated from external catalogs?
  - What metadata transformations are required for federated entries?

* **SBOM Format & Integration:**
  - What SBOM formats are supported? (SPDX, CycloneDX, both?)
  - Where are SBOMs stored? (in Registry, external vulnerability DB?)
  - Is SBOM parsing done client-side (UI) or server-side (API layer)?
  - How are vulnerability matches from SBOMs surfaced in the UI?

* **Deprecation & Lifecycle:**
  - What triggers lifecycle state transitions? (manual admin action, automated rules, time-based?)
  - How far in advance are deprecation notifications sent?
  - What notification channels are supported? (email, dashboard alerts, webhook?)
  - What happens to running instances when a server reaches EOL state?

* **UI Framework & Integration:**
  - What UI framework does ODH Dashboard use? (React, Angular, PatternFly?)
  - Are there existing shared components for catalog patterns we should reuse?
  - What routing strategy is used for new catalog pages?
  - How is navigation extended to include MCP Catalog entry?

* **Performance & Scale:**
  - What is the expected number of MCP servers in catalog? (10s, 100s, 1000s?)
  - How many versions per server should be supported in UI?
  - What pagination strategy should be used?
  - Are there caching strategies for metadata to reduce Registry load?

* **Audit & Observability:**
  - Where are audit logs stored? (Elasticsearch, object storage, database?)
  - What observability stack is used? (Prometheus, Grafana, custom?)
  - What metrics should be collected? (catalog views, deployments, policy violations?)
  - How long are audit logs retained?

* **Multi-tenancy:**
  - Is catalog visibility scoped per namespace/project, or cluster-wide?
  - Can different teams have different governance policies?
  - How are usage logs filtered by tenant/namespace?

**Background & Strategic Fit:**

*Provide any additional context is needed to frame the feature.*

The Model Context Protocol (MCP) represents a significant shift in how AI applications integrate with data sources, tools, and external systems. As enterprises adopt OpenShift AI for production AI workloads, they require the same governance, security, and lifecycle management for MCP servers that they expect for other enterprise infrastructure components.

**Strategic Context:**
- **Enterprise AI Governance:** As AI moves from experimentation to production, enterprises demand the same rigor for AI infrastructure (including MCP servers) as they apply to traditional application infrastructure
- **Supply Chain Security:** The software supply chain security landscape requires transparency through SBOMs, publisher verification, and audit trails for all components in the AI stack
- **OpenShift AI Platform Maturity:** This feature elevates OpenShift AI from an AI development platform to a comprehensive enterprise AI platform with built-in governance
- **Competitive Differentiation:** Providing a vetted, governed MCP catalog positions OpenShift AI as the enterprise-ready choice compared to open-source alternatives lacking governance

**Strategic Fit with Red Hat Portfolio:**
- Aligns with Red Hat's "Enterprise Open Source" positioning by adding governance to open MCP ecosystem
- Complements Red Hat Trusted Software Supply Chain initiatives
- Leverages existing OpenShift strengths in multi-tenancy, RBAC, and policy enforcement
- Extends the successful model serving catalog pattern to MCP entities

**Market Drivers:**
- Increasing regulatory requirements (EU AI Act, NIST AI RMF) demand traceability and governance
- Enterprise security teams require visibility into all components in the AI stack
- Platform engineering teams need self-service catalogs to accelerate adoption while maintaining control
- MCP ecosystem growth creates urgency for curation and trust mechanisms

**Technical Strategic Fit:**
- Builds on existing ODH Dashboard UI patterns (model catalog, data science pipelines)
- Leverages Kubernetes-native patterns for policy enforcement and lifecycle management
- Positions OpenShift AI to integrate with emerging MCP ecosystem standards (registries, gateways)
- Creates foundation for future MCP entity types beyond servers (tools, prompts, workflows)

**Customer Considerations**

*Provide any additional customer-specific considerations that must be made when designing and delivering the Feature.*

* **Hybrid & Air-Gapped Environments:**
  - Many enterprise customers operate in air-gapped or hybrid environments with limited external connectivity
  - Catalog must support local mirroring of MCP server metadata and disconnected operation
  - Federation must support on-premise registries and local catalogs
  - Consider providing tooling for syncing catalog metadata to disconnected environments

* **Compliance & Regulatory Requirements:**
  - Financial services, healthcare, and government customers have strict compliance requirements
  - Audit logs must be immutable and support tamper-evident storage
  - SBOM and provenance information must meet industry-specific standards (e.g., SLSA, NIST SSDF)
  - Consider supporting compliance report generation (SOC 2, ISO 27001, FedRAMP)

* **Enterprise Change Management:**
  - Large enterprises have formal change control processes for infrastructure changes
  - Deprecation workflows must integrate with customer ITSM systems (ServiceNow, Jira)
  - Approval workflows may be required for certain deployment actions
  - Consider providing webhook integrations for external workflow systems

* **Multi-Cluster & Multi-Region:**
  - Enterprise customers often deploy across multiple OpenShift clusters and regions
  - Catalog state and governance policies should be consistent across clusters
  - Usage and audit logs may need to aggregate across clusters for central visibility
  - Consider multi-cluster catalog federation patterns

* **Performance at Enterprise Scale:**
  - Large enterprises may have hundreds of platform engineers accessing catalog concurrently
  - Catalog must maintain performance with 1000+ MCP server entries
  - Audit logs may grow to millions of entries over time
  - Consider resource limits, rate limiting, and caching strategies

* **Organizational RBAC Complexity:**
  - Enterprise RBAC models are often complex with many roles and fine-grained permissions
  - Different teams may have different catalog visibility (e.g., separate catalogs per business unit)
  - Some customers may require approval chains based on organizational hierarchy
  - Consider flexible RBAC integration with existing OpenShift RBAC and external IdP groups

* **Existing Tool Integration:**
  - Customers have existing security scanning tools (Snyk, Aqua, Twistlock)
  - Customers use existing observability stacks (Splunk, Datadog, Dynatrace)
  - Consider providing integration points or export capabilities for third-party tools

* **Migration from Existing Solutions:**
  - Some customers may have homegrown MCP server management solutions
  - Consider providing migration tooling or bulk import capabilities
  - Document migration best practices and patterns

* **Training & Enablement:**
  - Enterprise customers require training programs for platform teams
  - Consider providing customer enablement materials: workshops, labs, best practices
  - Partner with Red Hat Training & Certification for potential course development

* **Support & SLA Expectations:**
  - Enterprise customers expect clear SLAs and support escalation paths
  - Critical production issues with catalog may block AI workload deployments
  - Consider monitoring and alerting for catalog service health
  - Document troubleshooting playbooks for support teams
