# ADR-004: API vs Direct Database Access

## Status
Accepted

## Context

Multiple teams and systems require access to platform data:

- **Internal Analytics Team**: Direct SQL access for ad-hoc analysis
- **Product Team**: REST API for application integration
- **External Partners**: Controlled data sharing (no direct database access)
- **Data Science Team**: Python/R access for modeling
- **Business Users**: Tableau dashboards with live connections

**Current State:**
- No formal data access layer; direct database access widely shared
- 15+ users with read-only warehouse credentials
- Security concerns: no row-level access control, audit logging incomplete
- Performance impact: ad-hoc queries compete with production ETL

**Requirements:**
- Support sub-100ms response for API consumers
- Enable fine-grained access control (row-level security)
- Maintain complete audit trail for compliance
- Minimize data exposure risk while enabling self-service

## Decision

**Selected: Layered Access Architecture**

We will implement a multi-tier access strategy:

1. **Tier 1 — Direct Access (Internal Analytics)**
   - Snowflake native SQL for analytics team
   - Role-based access control via Snowflake roles
   - Query tagging for cost attribution

2. **Tier 2 — API Layer (Application Integration)**
   - GraphQL API via Hasura or custom FastAPI
   - Row-level security enforced at API level
   - Caching layer (Redis) for frequently accessed data

3. **Tier 3 — Partner Data Share (External)**
   - Snowflake Data Sharing (secure views)
   - Read-only, time-bounded access
   - No direct database connection; provider-controlled

**Rationale:**

Different consumers have fundamentally different needs. A one-size-fits-all approach (either all API or all direct access) creates friction:

- **Direct access** is too permissive for application integration and creates security risk
- **API-only** is too restrictive for analytics and adds unnecessary latency

The layered approach allows:
- **Performance**: Analytics queries run directly in warehouse
- **Security**: Application consumers get validated, limited API surface
- **Compliance**: Partner data shared without exposing underlying schema

## Alternatives Considered

### API-First (All Access via REST/GraphQL)

**Why Not Selected:**
- Adds 50-100ms latency for every query
- Overhead for analytics team; unnecessary abstraction
- API development and maintenance burden (estimated 2 FTE)
- Limits ad-hoc exploration flexibility

**When to Reconsider:** If security requirements mandate no direct database access from any consumer.

### Direct Database Access Only

**Why Not Selected:**
- No row-level security; risk of data leakage
- Incomplete audit trail for compliance
- Performance isolation impossible; ad-hoc queries impact production
- External partners cannot access without warehouse credentials

**When to Reconsider:** If all consumers are trusted internal teams with no external access requirements.

### Data Virtualization (Denodo/Dremio)

**Why Not Selected:**
- Additional infrastructure cost ($50K+ annually)
- Adds another system to maintain and monitor
- Query performance unpredictable for complex analytical queries
- Over-engineering for our access patterns

**When to Reconsider:** If we need to unify data across multiple warehouses or data lakes.

## Consequences

### Positive Impacts

- **Security**: Row-level security reduces data exposure risk by 80%
- **Performance Isolation**: Analytics queries cannot impact API response times
- **Auditability**: Complete access log for compliance (SOC 2, GDPR)
- **Flexibility**: Each team uses optimal access method for their use case

### Negative Trade-offs

- **Operational Complexity**: Three tiers to maintain and monitor
- **Consistency Risk**: API schema must stay synchronized with warehouse schema
- **Latency Variance**: API adds latency; must cache strategically
- **Skill Requirements**: Need API development expertise alongside data engineering

### Mitigation Strategies

- Implement Infrastructure-as-Code (Terraform) for API deployment consistency
- Use OpenAPI spec to generate both client SDKs and server stubs
- Create automated schema sync between warehouse and API layer
- Establish SLA: API p99 < 200ms, direct query < 30s for 95th percentile

## Notes

- Monitor access patterns quarterly; adjust tier assignments as needs evolve
- Evaluate managed API solutions (Snowflake Cortex, Hasura Cloud) for reduced ops burden
- Consider GraphQL federation if multiple data sources emerge
- Track API usage to right-size caching infrastructure