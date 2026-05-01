# ADR-002: Batch vs Streaming Processing

## Status
Accepted

## Context

The data platform must process incoming data from multiple sources with varying latency requirements:

- **E-commerce transactions**: ~50K events/hour, need near real-time inventory updates
- **User behavior events**: ~500K events/day, used for daily analytics and cohort analysis
- **Financial transactions**: ~10K records/hour, require exactly-once processing with audit trail
- **CRM updates**: ~5K records/day, batch sync sufficient

**Current Challenges:**
- Existing batch pipelines have 6-12 hour data latency
- Real-time dashboards require manual refresh
- No unified processing framework; Python scripts mixed with SQL jobs

**Business Drivers:**
- Product team wants real-time inventory visibility
- Finance requires same-day reporting (currently T+1)
- Operations wants sub-minute dashboards for monitoring

## Decision

**Selected: Hybrid Architecture — Primarily Batch with Selective Streaming**

We will implement a two-tier processing strategy:

1. **Batch Layer (Primary)**: Apache Airflow for ETL, dbt for transformations
   - All business intelligence and analytical workloads
   - Daily/hourly aggregated metrics
   - Financial reporting and compliance

2. **Streaming Layer (Targeted)**: Apache Kafka + Flink for specific use cases
   - Real-time inventory synchronization
   - User session tracking for personalization
   - Anomaly detection for security monitoring

**Rationale:**

The bulk of our data processing (>85%) does not require sub-minute latency. A pure streaming architecture would introduce unnecessary complexity and cost for these workloads. By reserving streaming for specific high-value use cases, we optimize for:

- **Cost Efficiency**: Batch processing is 60-70% cheaper per record for analytical workloads
- **Operational Simplicity**: Single orchestration tool (Airflow) for 85% of pipelines
- **Team Skill Alignment**: Existing SQL/Python skills transfer directly to batch patterns

## Alternatives Considered

### Pure Streaming (Kafka + Flink)

**Why Not Selected:**
- 3x infrastructure cost compared to batch for our volume
- Requires dedicated streaming expertise; current team would need 4-6 month upskill
- Exactly-once semantics add significant complexity for financial data
- Over-engineering for 85% of use cases that tolerate hourly/daily latency

**When to Reconsider:** If real-time requirements grow to >40% of workloads, or if competitive pressure demands sub-minute insights across all data types.

### Event-Driven Microservices with Change Data Capture (CDC)

**Why Not Selected:**
- Requires Kafka expertise and CDC tooling (Debezium, Fivetran)
- Higher operational overhead; more failure points to monitor
- Schema evolution becomes more complex with real-time propagation
- Current team has limited distributed systems experience

**When to Reconsider:** If we adopt a microservices architecture requiring event-based integration between services.

### Serverless Batch (dbt Cloud + Fivetran)

**Why Not Selected:**
- Vendor lock-in concerns
- Less control over data pipeline logic
- Cost unpredictable at our scale
- Limited custom transformation flexibility

**When to Reconsider:** If startup speed is prioritized over long-term customization.

## Consequences

### Positive Impacts

- **Cost Optimization**: Estimated $45K annual savings vs pure streaming
- **Team Productivity**: Single orchestration framework reduces context switching
- **Reliability**: Batch systems easier to debug and recover from failures
- **Compliance**: Easier audit trail with immutable batch logs

### Negative Trade-offs

- **Latency Floor**: 85% of data has minimum 1-hour latency
- **Dual Codebase**: Must maintain both batch and streaming pipelines
- **Integration Complexity**: Need to reconcile batch and streaming outputs
- **Skill Gap**: Streaming expertise still required for targeted use cases

### Mitigation Strategies

- Document streaming patterns in internal wiki; cross-train batch engineers
- Use Kafka Schema Registry to ensure consistency between layers
- Implement dual-write prevention: streaming writes to Redis, batch reconciles to warehouse
- Create unified data model that abstracts batch vs streaming origin

## Notes

- Re-evaluate ratio quarterly; streaming adoption may grow organically
- Consider Apache Iceberg as unifying table format to simplify hybrid reads
- Monitor Databricks/Flink managed service pricing for potential future migration
- Budget for streaming training: 2 engineers × $5K = $10K annually