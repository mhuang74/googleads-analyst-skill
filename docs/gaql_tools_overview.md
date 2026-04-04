# GAQL Generation and Validation Tools

This document provides a comprehensive overview of the `mcc-gaql-gen` and `mcc-gaql` tools used for generating and validating Google Ads Query Language (GAQL) queries.

## Tool Overview

| Tool | Purpose | Key Features |
|------|---------|--------------|
| `mcc-gaql-gen` | Generate GAQL from natural language | LLM + RAG pipeline, metadata enrichment, query cookbook |
| `mcc-gaql` | Execute and validate GAQL queries | Query execution, validation, field discovery |

## Quick Reference

```bash
# Generate GAQL from natural language
mcc-gaql-gen generate "show top 10 campaigns by cost" --explain

# Validate generated GAQL without executing
mcc-gaql --validate "SELECT campaign.name, metrics.cost_micros FROM campaign"

# Explore available fields for a resource
mcc-gaql-gen metadata campaign --format llm

# Alternative: Show fields via mcc-gaql
mcc-gaql --show-fields campaign
```

---

## mcc-gaql-gen: GAQL Generation

### Architecture Overview

The `mcc-gaql-gen` tool uses a multi-step Retrieval-Augmented Generation (RAG) pipeline to convert natural language queries into valid GAQL. The pipeline consists of:

1. **Resource Selection** - Identify which Google Ads resources to query
2. **Field Candidate Retrieval** - Semantic search for relevant fields
3. **Field Selection** - LLM selects specific fields from candidates
4. **Criteria Assembly** - Build WHERE, ORDER BY, LIMIT clauses
5. **Query Generation** - Produce final GAQL syntax

### Commands

#### `generate` - Generate GAQL from Natural Language

```bash
mcc-gaql-gen generate "<prompt>" [OPTIONS]

Options:
  --explain           Print explanation of LLM selection process
  --validate          Validate query against Google Ads API (requires credentials)
  --profile <NAME>    Profile to use for validation
  --use-query-cookbook  Enable RAG search for query examples
  --no-defaults       Skip implicit filters (e.g., status = ENABLED)
```

**Example:**
```bash
mcc-gaql-gen generate "show campaign performance with impressions and cost for last 7 days" --explain
```

**Output:**
```sql
SELECT
  campaign.id,
  campaign.name,
  segments.date,
  metrics.impressions,
  metrics.cost_micros
FROM campaign
WHERE campaign.status = 'ENABLED'
  AND segments.date DURING LAST_7_DAYS
ORDER BY metrics.cost_micros DESC
```

#### `metadata` - Explore Field Metadata

```bash
mcc-gaql-gen metadata <QUERY> [OPTIONS]

Arguments:
  <QUERY>  Resource name, field name, or pattern (e.g., "campaign", "metrics.cost*")

Options:
  --format <FORMAT>    Output format: llm, full, json [default: llm]
  --category <CAT>     Filter: resource, attribute, metric, segment
  --subset             Use subset resources (campaign, ad_group, ad_group_ad, keyword_view)
  --show-all           Show all fields (default: 15 per category)
  --diff               Show enrichment comparison
```

**Example:**
```bash
mcc-gaql-gen metadata campaign --subset
```

**Output:**
```
=== RESOURCE: campaign ===
Description: A campaign resource represents an advertising campaign...
Key attributes: campaign.id, campaign.name, campaign.status, campaign.start_date_time
Key metrics: metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros
Identity fields: customer.id, customer.descriptive_name, campaign.id, campaign.name

### ATTRIBUTE (15/131 showing)
- campaign.name [filterable] [sortable]: The display name of the campaign...
- campaign.status [filterable]: The serving status of the campaign...
...

### METRIC (15/89 showing)
- metrics.impressions: Count of how often your ad is shown...
- metrics.clicks: The number of clicks on your ad...
...
```

#### `index` - Pre-build Vector Cache

```bash
mcc-gaql-gen index [OPTIONS]

Options:
  --queries <PATH>    Path to query cookbook TOML
  --metadata <PATH>   Path to enriched metadata JSON
```

Pre-builds the LanceDB vector cache for faster generation.

#### `bootstrap` - Download Pre-built Resources

```bash
mcc-gaql-gen bootstrap [OPTIONS]

Options:
  --version <VER>     Specific version to download
  --force             Force re-download
  --verify-only       Check bundle integrity without downloading
```

Downloads pre-built RAG resources (enriched metadata, embeddings) for instant use.

---

## mcc-gaql: Query Execution and Validation

### Key Commands

#### `--validate` - Validate Query Syntax

```bash
mcc-gaql --validate "<GAQL_QUERY>"

# With profile for authentication
mcc-gaql --profile myprofile --validate "SELECT campaign.name FROM campaign"
```

Validates the query against the Google Ads API without executing it. Returns validation errors with field-level specifics if the query is invalid.

#### `--show-fields` - Discover Valid Fields

```bash
mcc-gaql --show-fields <RESOURCE>

# Examples
mcc-gaql --show-fields campaign
mcc-gaql --show-fields ad_group
mcc-gaql --show-fields change_event
```

Shows all valid fields for a resource, organized by category (attributes, metrics, segments).

#### `--show-resources` - Show Resource Hierarchy

```bash
mcc-gaql --show-resources
```

Displays all available resources with field counts, key attributes, and compatibility information.

---

## How Field Metadata Works

### Data Sources

The tools obtain field metadata from multiple sources:

1. **Google Ads API Field Service** - Authoritative source for field names, types, and compatibility
2. **Proto Documentation** - Extracted from googleads-rs proto files for detailed descriptions
3. **LLM Enrichment** - AI-generated descriptions optimized for semantic search

### Field Metadata Cache

Field metadata is cached locally at:
```
~/.config/mcc-gaql/field_metadata_cache.json     # Raw field metadata
~/.config/mcc-gaql/enriched_cache.json           # LLM-enriched metadata
~/.config/mcc-gaql/proto_docs_cache.json         # Proto documentation
```

### Metadata Structure

Each field has the following metadata:

```json
{
  "name": "metrics.cost_micros",
  "category": "metric",
  "data_type": "INT64",
  "description": "The sum of costs during this period, in micros...",
  "is_filterable": true,
  "is_sortable": true,
  "is_selectable": true,
  "selectable_with": ["campaign", "ad_group", "ad_group_ad", ...],
  "enum_values": null,
  "usage_notes": "Divide by 1,000,000 to convert to dollars..."
}
```

---

## How Metadata Enrichment Works

### Enrichment Pipeline

The `mcc-gaql-gen enrich` command enhances raw field metadata with LLM-generated descriptions:

```bash
mcc-gaql-gen enrich [OPTIONS]

Options:
  --use-proto          Use proto documentation as additional context
  --batch-size <N>     Fields per batch (default: 15)
  --concurrency <N>    Parallel LLM calls
```

### Enrichment Process

1. **Batch Grouping** - Fields grouped by resource (15 per batch) to stay within token limits

2. **Context Assembly** - Each batch includes:
   - Field name and type
   - Raw description (if available)
   - Proto documentation
   - Enum values and their meanings
   - Field behavior annotations (OUTPUT_ONLY, REQUIRED, etc.)

3. **LLM Enhancement** - System prompt instructs the LLM:
   ```
   You are a Google Ads API documentation expert. Your task is to write
   concise, technically accurate field descriptions optimized for use in
   a semantic search (RAG) system that helps generate GAQL queries.
   ```

4. **Key Field Selection** - After enrichment, LLM selects 3-5 "key" fields per resource for quick reference

5. **Resource-Level Enrichment** - Each resource gets a summary description and identity field list

### Enriched Output

After enrichment, fields include:
- **Enhanced description** - Optimized for semantic search
- **Usage notes** - Practical guidance (e.g., "divide by 1,000,000 for dollars")
- **Key field flag** - Whether this is a commonly-used field for the resource

---

## How Indexing and RAG Works

### Vector Store Architecture

The RAG system uses **LanceDB** with three vector tables:

| Table | Contents | Purpose |
|-------|----------|---------|
| `field_metadata` | Field embeddings | Semantic field search |
| `resource_entries` | Resource embeddings | Resource selection |
| `query_cookbook` | Example query embeddings | Query pattern retrieval |

### Embedding Model

- **Model**: BGE-Small-EN-v1.5 (via fastembed library)
- **Dimensions**: 384
- **Distance Metric**: Cosine similarity

### Embedding Text Generation

For each field, a rich synthetic description is generated for embedding:

```rust
fn build_embedding_text(&self) -> String {
    // Combines:
    // - Field name (e.g., "metrics.cost_micros")
    // - Category (metric, attribute, segment)
    // - Description
    // - Inferred purpose based on name patterns
    //   (e.g., "clicks" -> performance measurement)
}
```

### RAG Pipeline Phases

#### Phase 1: Resource Selection
```
User query -> Embed -> Search resource_entries -> Top-k candidates
           -> LLM selects primary + related resources
```

The LLM receives:
- User query
- Top resource candidates with descriptions
- Domain knowledge guidance

#### Phase 2: Field Candidate Retrieval
```
User query + selected resources -> Embed -> Search field_metadata
           -> Filter by resource compatibility (selectable_with)
           -> Return compatible candidates
```

#### Phase 2.5: Keyword Pre-scan
Scans user query for filter keywords (date ranges, status values, etc.) to inform Phase 3.

#### Phase 3: Field Selection
```
User query + field candidates -> LLM selects specific fields
           -> Determines ORDER BY, LIMIT requirements
```

The LLM receives:
- User query
- Field candidates with descriptions
- Examples from query cookbook (if enabled)

#### Phase 4: Criteria Assembly
```
Selected fields + keywords -> Build WHERE clauses
           -> Add implicit filters (status = ENABLED)
           -> Apply ORDER BY and LIMIT
```

#### Phase 5: Query Generation
```
Assembled components -> Format as valid GAQL syntax
           -> Optionally validate against API
```

### Query Cookbook

The query cookbook provides example GAQL queries for RAG retrieval:

```toml
# resources/queries.toml
[[query]]
description = "Campaign performance with cost and conversions"
query = """
SELECT campaign.id, campaign.name, metrics.cost_micros, metrics.conversions
FROM campaign
WHERE campaign.status = 'ENABLED'
ORDER BY metrics.cost_micros DESC
"""

[[query]]
description = "Daily keyword performance trend"
query = """
SELECT segments.date, ad_group_criterion.keyword.text, metrics.impressions, metrics.clicks
FROM keyword_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY segments.date DESC
"""
```

When `--use-query-cookbook` is enabled, similar examples are retrieved and included in LLM prompts to improve generation quality.

### Domain Knowledge

A `domain_knowledge.md` file provides static guidance to the LLM:

```markdown
## Resource Selection Guidance
- For campaign-level metrics, use `campaign` resource
- For keyword performance, use `keyword_view` resource
- For search terms, use `search_term_view` resource

## Common Patterns
- Always include segments.date when analyzing trends
- Use metrics.cost_micros / 1000000 for dollar amounts
```

Location: `~/.config/mcc-gaql/domain_knowledge.md` (or embedded default)

---

## Caching and Performance

### Cache Locations

```
~/.config/mcc-gaql/
├── field_metadata_cache.json    # Raw API field metadata
├── enriched_cache.json          # LLM-enriched metadata
├── proto_docs_cache.json        # Proto documentation
├── domain_knowledge.md          # LLM guidance document
└── lancedb/                     # Vector database
    ├── field_metadata/          # Field embeddings
    ├── resource_entries/        # Resource embeddings
    └── query_cookbook/          # Example query embeddings
```

### Cache Refresh Commands

```bash
# Refresh field metadata from Google Ads API
mcc-gaql --refresh-field-cache

# Re-enrich metadata with LLM
mcc-gaql-gen enrich --use-proto

# Rebuild vector indices
mcc-gaql-gen index

# Clear vector cache (forces rebuild on next generate)
mcc-gaql-gen clear-cache
```

---

## Generation Flow Diagram

```
User Query: "Show campaign performance for last 7 days"
                    │
                    ▼
    ┌───────────────────────────────┐
    │  Phase 1: Resource Selection  │
    │  - Embed query                │
    │  - Search resource_entries    │
    │  - LLM selects: campaign      │
    └───────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  Phase 2: Field Retrieval     │
    │  - Embed query                │
    │  - Search field_metadata      │
    │  - Filter by compatibility    │
    │  - Return ~140 candidates     │
    └───────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  Phase 3: Field Selection     │
    │  - LLM reviews candidates     │
    │  - Selects relevant fields    │
    │  - Determines ORDER BY        │
    └───────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  Phase 4: Criteria Assembly   │
    │  - Build WHERE clauses        │
    │  - Add date filter            │
    │  - Add status = ENABLED       │
    └───────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────┐
    │  Phase 5: Query Generation    │
    │  - Format as GAQL             │
    │  - Validate (if --validate)   │
    └───────────────────────────────┘
                    │
                    ▼
    SELECT campaign.id, campaign.name,
           segments.date, metrics.impressions,
           metrics.clicks, metrics.cost_micros
    FROM campaign
    WHERE campaign.status = 'ENABLED'
      AND segments.date DURING LAST_7_DAYS
    ORDER BY metrics.cost_micros DESC
```

---

## Best Practices

### For GAQL Generation

1. **Be specific in prompts** - Include the metrics, dimensions, and filters you need
2. **Use `--explain`** - Understand why the LLM made its selections
3. **Validate before execution** - Use `--validate` to catch errors early
4. **Check field compatibility** - Not all fields can be queried together

### For Validation

1. **Validate generated queries** - Always validate before running expensive queries
2. **Use `--show-fields`** - Discover correct field names when generation fails
3. **Check `selectable_with`** - Ensure fields are compatible with your FROM resource

### For Metadata Discovery

1. **Start with `metadata --subset`** - Focus on common resources first
2. **Use `--format llm`** - Get concise descriptions optimized for understanding
3. **Check key fields** - These are the most commonly-used fields per resource
