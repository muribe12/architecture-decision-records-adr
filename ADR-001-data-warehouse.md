# ADR-001: Data Warehouse Selection

## Status
Accepted

## Context

The organization requires a modern cloud data warehouse to support:
- Enterprise analytics and reporting
- Real-time and batch data processing
- Multi-source data integration (SaaS applications, operational databases, third-party APIs)
- Self-service analytics for business users

**Current State:**
- Legacy on-premises SQL Server data warehouse approaching end-of-life
- Siloed data across multiple systems (Salesforce, PostgreSQL, MongoDB, external APIs)
- Growing data volume (~5TB current, projected 20TB in 24 months)
- Team has SQL expertise but limited Spark/distributed computing experience

**Constraints:**
- Budget: $150K-250K annual cloud spend
- Must support <5 second query latency for dashboard workloads
- Requires SOC 2 Type II compliance
- Integration with existing tooling (dbt, Airflow, Tableau)

## Decision

**Selected: Snowflake**

After evaluating Snowflake, Google BigQuery, and Databricks, we have selected Snowflake as our primary cloud data warehouse for the following reasons:

1. **Performance for Mixed Workloads**: Snowflake's multi-cluster architecture separates compute from storage, allowing independent scaling of concurrent query workloads. This is critical for our mixed workload pattern (heavy ETL at night, ad-hoc analytics during business hours).

2. **Mature SQL Interface**: The team has strong SQL expertise. Snowflake provides the most intuitive SQL dialect with native support for window functions, semi-structured data (JSON, Avro), and time-travel queries without requiring additional abstraction layers.

3. **Zero-Management Architecture**: Unlike Databricks which requires ongoing cluster tuning and Spark expertise, Snowflake handles infrastructure automatically. This reduces operational overhead and allows the team to focus on data modeling rather than cluster management.

4. **Ecosystem Integration**: Native connectors for dbt, Airflow, Tableau, and Python. The Partner Connect feature simplifies integration with Fivetran/Hightouch for ELT.

5. **Time-Travel and Clone Capabilities**: Built-in time-travel (up to 90 days) and zero-cost cloning for dev/test environments significantly reduces data recovery time and enables rapid experimentation.

## Alternatives Considered

### Google BigQuery

**Why Not Selected:**
- Less predictable pricing model; on-demand slots can cause cost spikes
- Weaker semi-structured data handling compared to Snowflake
- Limited native time-travel; requires partitioning and clustering manual configuration
- Our team lacks Google Cloud-specific expertise

**When to Reconsider:** If GCP becomes primary cloud provider, or if we need BigQuery ML for embedded ML workflows.

### Databricks

**Why Not Selected:**
- Requires Spark expertise; current team skill gap
- More complex operational model (cluster sizing, auto-scaling configuration)
- SQL endpoint costs higher than pure warehouse solutions for our use case
- Better suited for data engineering/ML workloads than BI/reporting focus

**When to Reconsider:** If we develop advanced ML pipelines requiring feature store or real-time streaming at scale.

## Consequences

### Positive Impacts

- **Rapid Onboarding**: Team productive within 2 weeks (vs 6-8 weeks for Databricks)
- **Cost Predictability**: Standard edition with predictable per-credit pricing
- **Data Sharing**: Can securely share data with external partners without copying
- **Time-to-Value**: Pre-built connectors reduce integration effort by ~40%

### Negative Trade-offs

- **Vendor Lock-in**: Proprietary format; migration cost estimated at 3-6 months
- **Limited Custom Extensions**: No user-defined functions in Python (only Java/Scala)
- **Concurrency Limits**: Standard edition caps at 10 concurrent queries; need Enterprise for unlimited
- **Storage Costs**: Egress charges apply when moving data outside Snowflake

### Mitigation Strategies

- Implement data lake integration (Iceberg tables) to reduce lock-in risk
- Use Terraform for infrastructure-as-code to maintain reproducibility
- Budget for Enterprise edition if concurrency becomes a bottleneck
- Document all custom SQL patterns for future migration reference

## Notes

- Review decision annually against workload evolution and competitive landscape
- Monitor emergence of open formats (Iceberg, Delta) that may reduce vendor dependency
- Track Snowflake's Cortex AI features for potential future ML integration