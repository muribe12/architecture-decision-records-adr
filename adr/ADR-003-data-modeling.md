# ADR-003: Data Modeling Strategy

## Status
Accepted

## Context

The data platform must support diverse analytical workloads:

- **Executive Dashboards**: High-level KPIs, trend analysis
- **Operational Reports**: Daily/weekly operational metrics
- **Ad-hoc Analysis**: Deep-dive investigations by analysts
- **ML Feature Engineering**: Feature extraction for predictive models
- **Data Science**: Cohort analysis, experimentation

**Current State:**
- No enforced modeling standards; each dataset has ad-hoc structure
- 40+ tables with inconsistent naming conventions
- Data quality issues: duplicate records, missing foreign keys
- Query performance degrades significantly with data growth

**Requirements:**
- Support self-service analytics without engineering intervention
- Enable clear data lineage from source to dashboard
- Ensure consistent metrics across reports (single source of truth)
- Scale to 20TB without performance degradation

## Decision

**Selected: Kimball Star Schema with Data Vault Extensions**

We will implement a hybrid approach:

1. **Core Analytical Tables**: Star schema for frequently accessed reporting tables
   - Fact tables for transactions, events, and measurements
   - Dimension tables with SCD Type 2 for historical tracking
   - Aggregates for common report queries

2. **Integration Layer**: Data Vault for raw staging and auditability
   - Hub tables for business keys (customer, product, order)
   - Satellite tables for attributes with change tracking
   - Link tables for many-to-many relationships

3. **Presentation Layer**: Denormalized views for specific consumer needs
   - Tableau-optimized extracts
   - dbt-exposed datasets for self-service tools

**Rationale:**

Star schema provides the best query performance for analytical workloads (denormalized fact tables with direct joins to dimensions). Data Vault adds auditability and flexibility for changing source systems without disrupting downstream consumers.

This hybrid approach balances:
- **Query Performance**: Star schema optimizes for BI tool joins
- **Auditability**: Data Vault tracks all source changes with full lineage
- **Adaptability**: New sources can be added without modifying existing structures
- **Governance**: Centralized metric definitions via dbt macros

## Alternatives Considered

### Pure Normalized (3NF)

**Why Not Selected:**
- Excessive joins for analytical queries; 5-10x slower than star schema
- Requires specialized analytical modeling expertise to design well
- Less intuitive for business users exploring data
- Over-normalized for write-once analytical workloads

**When to Reconsider:** If operational reporting (not analytical) becomes primary use case.

### Pure Data Vault

**Why Not Selected:**
- Query complexity too high for business analysts
- Performance requires additional marts layer (adding another transformation step)
- Steeper learning curve; requires specialized Data Vault training
- Over-engineering for our dataset size and team expertise

**When to Reconsider:** If we have 100+ source systems with complex integration requirements.

### Wide Denormalized Tables (One Big Table)

**Why Not Selected:**
- Schema changes require table rebuilds
- Data redundancy increases storage costs 3-4x
- No historical tracking; difficult to support SCD Type 2
- Columnar warehouse (Snowflake) handles joins efficiently; denormalization provides minimal benefit

**When to Reconsider:** If query patterns are highly predictable and schema is stable.

## Consequences

### Positive Impacts

- **Query Performance**: 3-5x faster for common analytical queries vs normalized
- **Self-Service Enablement**: Business users can navigate star schema intuitively
- **Data Governance**: Clear metric definitions in dbt; one source of truth
- **Scalability**: Partitioned fact tables handle 20TB+ without degradation

### Negative Trade-offs

- **Initial Investment**: 4-6 weeks to model core business processes
- **Maintenance Overhead**: Must manage SCD Type 2 for slowly changing dimensions
- **Flexibility Trade-off**: Star schema less flexible than raw tables for ad-hoc exploration
- **Learning Curve**: Team requires Kimball methodology training

### Mitigation Strategies

- Prioritize modeling for top 5 business processes (revenue, orders, customers, inventory, support)
- Use dbt to automate SCD Type 2 generation and maintainability
- Create canonical dimension tables shared across fact tables
- Document modeling decisions in data dictionary with business definitions

## Notes

- Review model quarterly; add new dimensions as business processes evolve
- Consider implementing bridge tables for hierarchical dimensions (org charts, product categories)
- Evaluate incremental fact table loading to reduce refresh times
- Track query patterns via Snowflake's query history to identify optimization opportunities