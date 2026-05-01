# ADR-005: Schema Evolution Strategy

## Status
Accepted

## Context

The data platform integrates data from diverse sources with evolving schemas:

- **SaaS Applications**: Salesforce, HubSpot (schema controlled by vendors)
- **Operational Databases**: PostgreSQL, MongoDB (schema managed by engineering teams)
- **External APIs**: Payment processors, logistics providers (schema changes with notice)
- **Event Streams**: User behavior, IoT devices (schema evolves with product changes)

**Current Challenges:**
- Schema changes in source systems break downstream pipelines
- No standardized approach for handling new fields
- Data quality issues: 15% of pipelines fail monthly due to schema drift
- Developers must manually track schema changes across 20+ sources

**Requirements:**
- Minimize pipeline failures from source schema changes
- Enable rapid integration of new data fields
- Maintain data quality with automatic validation
- Support both rigid (financial) and flexible (event) data types

## Decision

**Selected: Schema-on-Write with Flexible Extension Zones**

We will implement a hybrid approach:

1. **Core Tables (Schema-on-Write)**
   - Enforced schema for critical business entities (transactions, customers, orders)
   - Explicit column definitions with data type validation
   - DBT tests enforce NOT NULL, uniqueness, referential integrity

2. **Extension Zones (Schema-on-Read)**
   - Semi-structured columns (JSON, VARIANT) for optional/extended attributes
   - Raw event data stored as JSON before normalization
   - New fields from SaaS APIs captured in extension columns

3. **Schema Registry**
   - Confluent Schema Registry for event streams
   - Evolvable schemas with backward/forward compatibility rules
   - Automated schema detection for new fields

**Rationale:**

Different data types require different approaches:

- **Schema-on-Write** for core business entities where data quality is paramount and changes are controlled
- **Schema-on-Read** for exploratory data and rapidly evolving sources where schema enforcement would slow integration

This hybrid approach provides:
- **Governance**: Critical data has enforced quality gates
- **Flexibility**: New data can be captured without pipeline modifications
- **Performance**: Core tables optimized for analytical queries
- **Extensibility**: Extension zones capture unexpected data for future use

## Alternatives Considered

### Pure Schema-on-Write

**Why Not Selected:**
- SaaS API changes require pipeline updates (2-4 week lead time)
- New product features require engineering involvement to capture data
- Over-rigid for event-driven data with unpredictable payloads
- 30% increase in pipeline development time

**When to Reconsider:** If all data sources have stable, well-documented schemas.

### Pure Schema-on-Read

**Why Not Selected:**
- Data quality issues discovered too late (at query time)
- No early validation; errors propagate to dashboards
- Query performance unpredictable with heavy JSON parsing
- Difficult to enforce business rules and referential integrity

**When to Reconsider:** If data quality is less critical than integration speed.

### Schema Registry Only (Event-Driven)

**Why Not Selected:**
- Only covers streaming data; doesn't address batch sources
- Additional infrastructure (Kafka, Schema Registry)
- Overhead for sources that don't need schema evolution
- Team has limited Kafka experience

**When to Reconsider:** If streaming becomes primary data integration pattern.

## Consequences

### Positive Impacts

- **Pipeline Stability**: Schema changes in extension zones don't break core tables
- **Developer Velocity**: New fields captured automatically without pipeline changes
- **Data Quality**: Core entity validation catches issues at load time
- **Future-Proofing**: Extension zones capture data for unknown future use cases

### Negative Trade-offs

- **Query Complexity**: Analysts must understand semi-structured data handling
- **Performance Overhead**: JSON parsing 10-20% slower than native types
- **Governance Gap**: Extension zone data has less validation
- **Documentation Burden**: Must document which fields are core vs extension

### Mitigation Strategies

- Create naming convention: core columns snake_case, extension columns JSON field names
- Implement dbt assertions for core table data quality
- Build self-service tooling: schema browser shows core vs extension fields
- Establish SLA: schema changes in extension zones documented within 48 hours

## Notes

- Quarterly review: promote extension fields to core when stable
- Monitor JSON column storage costs; archive old extension data to cold storage
- Evaluate Iceberg/Hudi for managed schema evolution if available in Snowflake
- Create field usage tracking to identify unused core fields for deprecation