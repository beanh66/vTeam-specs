# Feature Specification: MCP Catalog Integration for OpenShift AI Dashboard

**Feature Branch**: `001-this-is-a`
**Created**: 2025-10-20
**Status**: Draft
**Input**: User description: "this is a test for the pilot for the mcp catalog rfe"

## Execution Flow (main)
```
1. Parse user description from Input
   → Feature identified: MCP Catalog Integration based on rfe.md
2. Extract key concepts from description
   → Actors: Platform Engineers, Security Teams, Administrators
   → Actions: Discover, evaluate, deploy, govern, audit MCP servers
   → Data: MCP server metadata, versions, SBOMs, policies, audit logs
   → Constraints: Enterprise governance, policy enforcement, trust verification
3. For each unclear aspect:
   → Marked with [NEEDS CLARIFICATION: specific question]
4. Fill User Scenarios & Testing section
   → User flows clearly defined in RFE
5. Generate Functional Requirements
   → Each requirement is testable
   → Ambiguous requirements marked
6. Identify Key Entities (if data involved)
   → MCP Servers, Versions, Policies, Trust Tiers
7. Run Review Checklist
   → Multiple [NEEDS CLARIFICATION] items identified
   → No implementation details included
8. Return: SUCCESS (spec ready for clarification phase)
```

---

## ⚡ Quick Guidelines
- ✅ Focus on WHAT users need and WHY
- ❌ Avoid HOW to implement (no tech stack, APIs, code structure)
- 👥 Written for business stakeholders, not developers

### Section Requirements
- **Mandatory sections**: Must be completed for every feature
- **Optional sections**: Include only when relevant to the feature
- When a section doesn't apply, remove it entirely (don't leave as "N/A")

---

## User Scenarios & Testing *(mandatory)*

### Primary User Story
Platform engineers need a governed, enterprise-grade catalog of Model Context Protocol (MCP) servers integrated directly into the OpenShift AI Dashboard. They need to discover, evaluate, and deploy MCP servers with confidence that organizational security policies, trust requirements, and compliance standards are automatically enforced. Security teams need visibility into what MCP servers are deployed, who deployed them, and whether they meet security standards, while administrators need to configure and maintain governance policies that balance innovation with risk management.

### Acceptance Scenarios

1. **Given** a platform engineer is logged into the OpenShift AI Dashboard, **When** they navigate to the MCP Catalog section, **Then** they see a catalog interface consistent with the existing model catalog UX displaying available MCP servers with trust tier badges, lifecycle states, and version information

2. **Given** the catalog displays MCP servers, **When** the engineer searches for "database" and filters by "Verified Publisher" trust tier and "Stable" lifecycle, **Then** the catalog displays only MCP servers matching those criteria with clear visual indicators

3. **Given** an engineer selects a specific MCP server entry, **When** they view the details, **Then** they see comprehensive metadata including description, all available versions, SBOM, publisher verification status, capabilities, resource requirements, and documentation links

4. **Given** an engineer selects a stable, verified MCP server version and initiates deployment, **When** governance policies permit stable versions from verified publishers, **Then** the deployment proceeds successfully and an audit log entry is created

5. **Given** an engineer attempts to deploy an experimental version MCP server, **When** organizational policy requires only stable versions, **Then** the deployment is blocked with a clear error message explaining the policy requirement

6. **Given** an MCP server version is marked as deprecated in the Registry, **When** an engineer views that server, **Then** a prominent warning displays with recommended migration path to a newer version

7. **Given** a security team member accesses the catalog with elevated permissions, **When** they navigate to the Usage & Audit section, **Then** they see all deployed MCP servers with deployment details, versions, publishers, trust tiers, and can access SBOMs for vulnerability analysis

8. **Given** an administrator configures a governance policy, **When** they save the policy (e.g., "Block deployment of MCP servers with trust tier below 'Verified'"), **Then** the policy is immediately enforced for all subsequent catalog interactions

9. **Given** the catalog is synchronized with the Registry, **When** new MCP servers or versions are added to the Registry, **Then** they appear in the catalog UI within the configured sync interval

10. **Given** a user with appropriate permissions, **When** they attempt to federate with an external MCP registry, **Then** they can configure the external registry and entries from that registry appear in the catalog with appropriate trust tier inheritance

### Edge Cases

- **What happens when the Registry is unavailable?** System should display cached catalog data with a notification that data may be stale, and retry connection with exponential backoff
- **How does the system handle MCP servers with no SBOM data?** Server should be marked with missing SBOM indicator, and policy may optionally block deployment if SBOM is required
- **What if multiple federated registries contain the same MCP server with conflicting metadata?** [NEEDS CLARIFICATION: conflict resolution strategy - first wins, priority order, admin resolution?]
- **How does the system handle a policy change that makes currently deployed MCP servers non-compliant?** [NEEDS CLARIFICATION: retroactive enforcement - alert only, mark for review, force removal?]
- **What happens if Gateway is unavailable during deployment attempt?** Deployment should fail with clear error, not bypass policy enforcement
- **How are very large catalog sizes (1000+ entries) handled?** Pagination with configurable page size and efficient filtering/search with performance target of <2s load time
- **What if an MCP server version transitions to EOL while actively deployed?** [NEEDS CLARIFICATION: notification timing, grace period, forced migration strategy?]
- **How are permission denied scenarios presented to users?** Clear error messages indicating which permission or policy is blocking the action
- **What happens when SBOM contains known critical vulnerabilities?** [NEEDS CLARIFICATION: automatic blocking, warning only, configurable severity threshold?]
- **How does the system handle network partitions in multi-cluster deployments?** [NEEDS CLARIFICATION: catalog state consistency model across clusters]

## Requirements *(mandatory)*

### Functional Requirements

**Discovery & Browsing**
- **FR-001**: System MUST display an MCP Catalog accessible from the OpenShift AI Dashboard navigation
- **FR-002**: System MUST display MCP servers with trust tier badges (e.g., Verified Publisher, Partner, Community, Unverified)
- **FR-003**: System MUST display lifecycle states for each MCP server (e.g., experimental, stable, deprecated, end-of-life)
- **FR-004**: System MUST provide search capability across MCP server names, descriptions, and capabilities
- **FR-005**: System MUST provide filtering by trust tier, lifecycle state, publisher, capabilities, and version
- **FR-006**: System MUST display results in paginated format with configurable page size for catalogs exceeding 100 entries
- **FR-007**: System MUST load catalog initial view within 2 seconds under normal network conditions

**Metadata & Detail View**
- **FR-008**: System MUST display comprehensive metadata for each MCP server including description, capabilities, publisher information, and documentation links
- **FR-009**: System MUST display all available versions for each MCP server with version numbers and release dates
- **FR-010**: System MUST display Software Bill of Materials (SBOM) for each MCP server version
- **FR-011**: System MUST display publisher verification status with visual indicators
- **FR-012**: System MUST display resource requirements (CPU, memory, dependencies) for each MCP server
- **FR-013**: System MUST provide visual comparison capability between different versions of the same MCP server

**Governance & Policy Enforcement**
- **FR-014**: System MUST enforce governance policies at deployment time before initiating deployment
- **FR-015**: System MUST block deployments that violate configured policies with clear error messages explaining the violation
- **FR-016**: System MUST support policy rules based on trust tier (e.g., minimum trust tier requirement)
- **FR-017**: System MUST support policy rules based on lifecycle state (e.g., block experimental, require stable)
- **FR-018**: System MUST support policy rules based on SBOM requirements (e.g., require SBOM presence, vulnerability thresholds)
- **FR-019**: System MUST apply policies consistently across all users subject to their role-based permissions
- **FR-020**: Administrators MUST be able to create, update, and delete governance policies through admin interface
- **FR-021**: System MUST validate policy syntax and configuration before activation

**Lifecycle Management**
- **FR-022**: System MUST display prominent warnings when users view or attempt to deploy deprecated MCP server versions
- **FR-023**: System MUST provide recommended migration paths for deprecated MCP servers
- **FR-024**: System MUST support automated notifications for deprecated MCP servers with configurable advance notice period
- **FR-025**: System MUST prevent deployment of end-of-life MCP servers unless explicitly overridden by admin policy
- **FR-026**: System MUST track lifecycle state transitions in audit logs

**Registry Integration**
- **FR-027**: System MUST synchronize MCP server metadata from connected Registry with configurable refresh interval
- **FR-028**: System MUST handle Registry unavailability gracefully by serving cached data with staleness indicator
- **FR-029**: System MUST authenticate with Registry using [NEEDS CLARIFICATION: authentication method - OAuth, API key, mutual TLS, service account?]
- **FR-030**: System MUST validate metadata schema received from Registry before displaying
- **FR-031**: System MUST log Registry synchronization status and errors

**Gateway Integration**
- **FR-032**: System MUST invoke Gateway for policy enforcement checks during deployment actions
- **FR-033**: System MUST pass deployment requests to Gateway with complete MCP server metadata and user context
- **FR-034**: System MUST display policy enforcement results returned from Gateway (pass/fail with reason)
- **FR-035**: System MUST retrieve runtime status from Gateway for deployed MCP servers (if observability integration is enabled)
- **FR-036**: System MUST authenticate with Gateway using [NEEDS CLARIFICATION: authentication method - OAuth, API key, mutual TLS, service account?]

**Federation**
- **FR-037**: System MUST support configuration of external federated MCP registries
- **FR-038**: System MUST display MCP servers from federated registries in unified catalog view with source indication
- **FR-039**: System MUST inherit or translate trust tiers from federated registries according to [NEEDS CLARIFICATION: trust tier mapping rules]
- **FR-040**: System MUST handle conflicts when federated registries contain duplicate MCP servers according to [NEEDS CLARIFICATION: conflict resolution policy]
- **FR-041**: Administrators MUST be able to add, remove, and configure federation settings

**Audit & Observability**
- **FR-042**: System MUST create audit log entries for all catalog actions including searches, views, deployments, and policy changes
- **FR-043**: Audit logs MUST capture user identity, timestamp, action type, target MCP server, and outcome
- **FR-044**: Security teams MUST be able to access audit logs through dedicated interface
- **FR-045**: System MUST provide usage reports showing all deployed MCP servers across the platform with filtering by namespace, user, trust tier, and lifecycle state
- **FR-046**: System MUST retain audit logs for [NEEDS CLARIFICATION: retention period - 90 days, 1 year, configurable?]
- **FR-047**: System MUST support export of audit logs in [NEEDS CLARIFICATION: format - JSON, CSV, SIEM-compatible?]
- **FR-048**: System MUST track and display metrics including catalog views, deployments, policy violations, and search queries

**Security & Access Control**
- **FR-049**: System MUST integrate with OpenShift RBAC for role-based access control
- **FR-050**: System MUST enforce permission requirements for catalog viewing, deployment actions, policy administration, and audit access
- **FR-051**: System MUST support [NEEDS CLARIFICATION: multi-tenancy model - cluster-wide catalog, namespace-scoped visibility, or both?]
- **FR-052**: System MUST protect sensitive metadata (e.g., internal registry credentials, policy details) from unauthorized access
- **FR-053**: System MUST use encrypted communication channels for Registry and Gateway integration

**User Experience**
- **FR-054**: System MUST provide consistent UX with existing OpenShift AI Dashboard model catalog pattern
- **FR-055**: System MUST provide responsive design supporting desktop and tablet form factors
- **FR-056**: System MUST meet WCAG 2.1 AA accessibility standards
- **FR-057**: System MUST provide contextual help and tooltips for trust tiers, lifecycle states, and policy concepts
- **FR-058**: System MUST streamline MCP server selection and configuration for deployment with pre-populated defaults from metadata
- **FR-059**: System MUST provide clear visual differentiation between trust tiers using badges, colors, and icons
- **FR-060**: System MUST display loading states and progress indicators during async operations

**Data & Performance**
- **FR-061**: System MUST support catalog sizes of at least 1000 MCP server entries
- **FR-062**: System MUST support at least 50 versions per MCP server
- **FR-063**: System MUST cache Registry metadata to reduce load on Registry and improve response times
- **FR-064**: System MUST implement pagination, lazy loading, or virtualization for large result sets
- **FR-065**: System MUST optimize search and filter operations to maintain <2s response time

### Key Entities *(include if feature involves data)*

- **MCP Server**: Represents a Model Context Protocol server catalog entry
  - Attributes: Unique identifier, name, description, publisher, capabilities, current lifecycle state, trust tier, source registry
  - Relationships: Has multiple Versions, belongs to Publisher, associated with Policies, tracked in Audit Logs

- **Version**: Represents a specific version of an MCP Server
  - Attributes: Version number, release date, lifecycle state, SBOM reference, resource requirements, documentation URL, deprecation status
  - Relationships: Belongs to MCP Server, may have SBOM, subject to Policies

- **Trust Tier**: Classification level indicating verification and trustworthiness
  - Attributes: Tier name (e.g., Verified, Partner, Community, Unverified), visual indicator (badge/icon), description
  - Relationships: Assigned to MCP Server, used in Policy rules
  - Note: [NEEDS CLARIFICATION: exact tier definitions, criteria for assignment, who can modify]

- **Lifecycle State**: Current stage in the MCP server's lifecycle
  - Attributes: State name (experimental, stable, deprecated, end-of-life), visual indicator, transition rules
  - Relationships: Assigned to Version, used in Policy rules, triggers Notifications

- **Governance Policy**: Rule set that enforces organizational requirements
  - Attributes: Policy identifier, rule conditions (trust tier, lifecycle, SBOM requirements), enforcement action (block, warn, require approval), scope (global, namespace)
  - Relationships: Applied to MCP Server deployments, configured by Administrator, recorded in Audit Logs
  - Note: [NEEDS CLARIFICATION: policy storage mechanism, evaluation engine, conflict resolution]

- **Publisher**: Entity that publishes MCP servers
  - Attributes: Publisher identifier, name, verification status, signing key reference
  - Relationships: Publishes MCP Servers, has verification status affecting Trust Tier

- **SBOM (Software Bill of Materials)**: Document listing components and dependencies
  - Attributes: SBOM format (SPDX, CycloneDX), component list, vulnerability references, generation timestamp
  - Relationships: Associated with specific Version, used in Policy evaluation, accessed in Audit view
  - Note: [NEEDS CLARIFICATION: storage location, parsing responsibility, vulnerability matching process]

- **Audit Log Entry**: Record of a catalog action
  - Attributes: Timestamp, user identity, action type, target MCP Server/Version, outcome, policy evaluation results
  - Relationships: References MCP Server, Version, User, Policy
  - Note: [NEEDS CLARIFICATION: storage backend, retention policy, immutability requirements]

- **Registry**: External system providing MCP server metadata
  - Attributes: Registry URL, authentication credentials, sync interval, federation flag (primary vs federated)
  - Relationships: Source of MCP Servers, synchronized by Catalog
  - Note: [NEEDS CLARIFICATION: API contract, authentication method, metadata schema]

- **Gateway**: Runtime enforcement system for MCP servers
  - Attributes: Gateway endpoint URL, authentication credentials
  - Relationships: Enforces Policies during deployment, provides runtime status
  - Note: [NEEDS CLARIFICATION: API contract, deployment request format, status reporting]

- **Deployment**: Instance of an MCP server deployed to the platform
  - Attributes: Deployment identifier, target namespace, deployed version, deploying user, deployment timestamp, current status
  - Relationships: References MCP Server and Version, tracked in Audit Logs, visible in Usage Reports

---

## Review & Acceptance Checklist
*GATE: Automated checks run during main() execution*

### Content Quality
- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

### Requirement Completeness
- [ ] No [NEEDS CLARIFICATION] markers remain *(18 clarifications identified - see below)*
- [x] Requirements are testable and unambiguous (where clarified)
- [x] Success criteria are measurable
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

**Outstanding Clarifications:**
1. Conflict resolution strategy for duplicate MCP servers in federated registries
2. Retroactive enforcement strategy when policies change
3. EOL transition notification timing and migration strategy
4. Vulnerability blocking behavior and severity thresholds
5. Multi-cluster catalog state consistency model
6. Registry authentication method
7. Gateway authentication method
8. Trust tier mapping rules for federated registries
9. Conflict resolution policy for federated duplicates
10. Audit log retention period
11. Audit log export format
12. Multi-tenancy model (cluster-wide vs namespace-scoped)
13. Trust tier definitions and assignment criteria
14. Policy storage mechanism and evaluation engine
15. SBOM storage location and parsing responsibility
16. SBOM vulnerability matching process
17. Registry API contract and metadata schema
18. Gateway API contract and deployment request format

---

## Execution Status
*Updated by main() during processing*

- [x] User description parsed
- [x] Key concepts extracted
- [x] Ambiguities marked
- [x] User scenarios defined
- [x] Requirements generated
- [x] Entities identified
- [ ] Review checklist passed (pending clarifications)

---
