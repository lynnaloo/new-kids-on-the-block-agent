# Salesforce Data Cloud: Architectural Patterns

## Core Architecture

### Data Flow
```
Source Systems
    |
    v
Data Streams (connectors, Ingestion API, file, Web SDK)
    |
    v
Data Lake Objects (DLOs) — raw, schema preserved
    |
    v
Data Mapping / Harmonization → Data Model Objects (DMOs)
    |
    v
Identity Resolution → Unified Individual / Unified Account
    |
    +--→ Calculated Insights (metrics, aggregations)
    |
    +--→ Segments → Activation Targets (MC, CRM, ad platforms, S3)
    |
    +--→ Data Actions (real-time triggers to Flow, Webhook, Platform Events)
    |
    +--→ Agentforce / Einstein (grounded customer context)
```

### Key Components

| Component | Purpose |
|-----------|---------|
| Data Streams | Ingestion layer — connects source data into Data Cloud |
| Data Lake Objects (DLOs) | Raw landing zone; schema-on-read |
| Data Model Objects (DMOs) | Harmonized, canonical data layer |
| Identity Resolution | Ruleset engine creating Unified Individual / Account profiles |
| Calculated Insights | SQL-based aggregations stored as queryable objects |
| Segments | Audience filters built on Unified Profiles |
| Activation Targets | Destinations for segment data |
| Data Actions | Event-driven triggers (not segment-based) |
| Data Spaces | Logical partitioning within one org (GA Winter '25) |

---

## Ingestion Patterns

| Pattern | Latency | Best For |
|---------|---------|----------|
| CRM Connector (near-real-time) | 5–15 min | Salesforce CRM object sync |
| Ingestion API (streaming) | Seconds | Custom events, IoT, web |
| Ingestion API (bulk) | Minutes–hours | Historical loads, large files |
| Cloud Storage (S3, GCS, Azure) | Scheduled (≥15 min) | Nightly batch ETL |
| Zero-copy (Snowflake share) | Near-real-time query | External lake integration |
| Web / Mobile SDK | Seconds | Behavioral events |

**Key rules:**
- Data Stream schema changes: adding columns is safe; removing/renaming requires recreating the stream.
- Primary key must be truly unique — duplicates cause silent data quality issues.
- Standardize all datetime fields to UTC at ingestion to avoid reconciliation errors.

---

## Identity Resolution Patterns

### Match Rule Strategy
- **Always start deterministic:** Exact match on email or phone as the primary rule.
- **Add secondary rules:** Normalized phone + name, loyalty ID, SSO ID.
- **Use fuzzy matching cautiously:** Only as a tertiary rule combined with a deterministic key.
- A single unified profile linked to >100 source records is a signal of an overly broad match rule.

### Reconciliation Strategies

| Strategy | When to Use |
|----------|-------------|
| Most Recent | Operational data (addresses, preferences) |
| Most Frequent | Stable attributes (name, gender) |
| Source Priority | When one source is the authoritative system of record |

### B2C vs B2B
- **B2C:** Unified Individual as the target entity.
- **B2B:** Unified Account + Unified Individual, with explicit Account-Contact relationship mapping.

---

## Segmentation & Activation Patterns

### Segment Design
- Filter on Unified Individual fields, related DMOs, and Calculated Insights.
- Add consent-based suppression filters to every segment — Data Cloud does not apply consent automatically.
- Use Waterfall Segments (Einstein Segmentation) for mutually exclusive priority audiences.

### Activation Target Reference

| Target | Type |
|--------|------|
| Marketing Cloud | Salesforce Native |
| Salesforce CRM | Salesforce Native |
| Amazon S3 / GCS / Azure | File-based |
| Meta, Google Ads, LinkedIn | Ad Platforms |
| Snowflake | Zero-copy |
| LiveRamp | Partner/Clean Room |

**Critical:** Changing activation attributes after a Marketing Cloud activation is live alters the Data Extension schema and can break journeys. Treat attribute lists as versioned — create a new activation rather than modifying a live one.

---

## Common Pitfalls

| Area | Pitfall | Fix |
|------|---------|-----|
| Data Modeling | Shared email (e.g., corporate domain) as match key collapses thousands of profiles | Cleanse upstream or use Source Priority |
| Data Modeling | DMO relationships not defined → segment filters can't traverse data | Explicitly create DMO relationships |
| Identity Resolution | Fuzzy name matching without a deterministic key | Always combine fuzzy with email or phone |
| Ingestion | Historical loads via streaming API hit rate limits | Use Bulk Ingestion API or S3 for backfill |
| Calculated Insights | CI failures are silent unless monitoring is configured | Set up Data Cloud alerts |
| Activation | No consent suppression in segment | Add consent filters to every segment |

---

## Platform Limits (Early 2026)

| Limit | Value |
|-------|-------|
| Max DLOs per org | ~1,000 |
| Max DMOs per org | ~1,000 |
| Max Calculated Insights | 100 |
| Max Segments | 1,000 |
| Max Activation Targets | 50 |
| Max match rules per ruleset | 25 |
| Max rulesets | 5 |
| Streaming Ingestion API rate | ~50,000 records/min |

*Always validate current limits against Salesforce release notes — these change each release.*

---

## Architecture Decision Guide

| Scenario | Recommendation |
|----------|---------------|
| Unified profile across 3+ systems | Data Cloud |
| Simple list import to MC | MC Import Wizard (no Data Cloud needed) |
| Real-time web personalization | Data Cloud Web SDK + MC Personalization |
| B2B account-based marketing | Data Cloud Unified Account + ABM activation |
| AI grounding for Agentforce | Data Cloud (required for unified customer context) |
| Analytics/reporting only | CRM Analytics first; Data Cloud if cross-source unification needed |
| HIPAA/regulated data | Validate compliance posture with Salesforce account team |
