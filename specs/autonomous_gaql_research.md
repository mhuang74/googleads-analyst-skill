# Plan: Enhanced Google Ads Analyst Skill with GAQL Generation

## Context

The current Google Ads Analyst Skill (`SKILL.md`) generates performance reports using manually-crafted GAQL queries following documented patterns. The user wants to enhance this skill to:

1. **Generate high-quality GAQL dynamically** using `mcc-gaql-gen` (LLM + RAG pipeline)
2. **Validate queries** using `mcc-gaql --validate` before execution
3. **Diagnose performance issues** through iterative multi-dimensional analysis
4. **Correlate with change events** to categorize causes as user-driven vs market-driven
5. **Provide tool feedback** when mcc-gaql-gen fails to generate valid GAQL

The approach is **hybrid**: autonomous for initial exploration, interactive for deep-dive investigation.

---

## Phase 1: Skill Enhancement - Diagnostic Framework

### 1.1 Add Diagnostic Workflow to SKILL.md

Add a new "Diagnostic Analysis Mode" section that defines:

**Autonomous Initial Exploration:**
1. Generate and validate GAQL queries using mcc-gaql-gen + mcc-gaql --validate
2. Pull campaign-level performance data for recent periods
3. Identify anomalies and performance inflection points
4. Query change_event and change_status for timeline correlation
5. Present preliminary findings interactively

**Interactive Deep-Dive:**
- User guides investigation focus (specific campaigns, time periods, dimensions)
- Skill proposes drill-down queries for approval
- Full dimensional analysis: device, geo, audience, hour-of-day segments

**Files to modify:**
- `/rust_dev_cache/projects/googleads/gads_skill_gaql_gen/SKILL.md`

### 1.2 Add GAQL Generation Workflow

New section documenting the iterative GAQL generation process:

```
1. Use `mcc-gaql-gen metadata <resource>` to understand available fields
2. Use `mcc-gaql-gen generate "<request>"` to generate GAQL
3. Use `mcc-gaql --validate "<query>"` to validate before execution
4. If validation fails, refine prompt or fall back to manual patterns
5. Document failures for tool improvement feedback
```

**Files to create:**
- `/rust_dev_cache/projects/googleads/gads_skill_gaql_gen/gaql_generation_reference.md`

### 1.3 Add Change Event Correlation Reference

Document queries and analysis patterns for:
- `change_event` resource: User actions (bid changes, budget changes, targeting changes)
- `change_status` resource: Entity modifications with timestamps
- Timeline correlation methodology
- Multi-factor scoring for cause attribution

**Files to create:**
- `/rust_dev_cache/projects/googleads/gads_skill_gaql_gen/change_correlation_reference.md`

### 1.4 Add Multi-Dimensional Analysis Reference

Document drill-down patterns for full dimensional analysis:
- Hierarchy levels: Campaign > Ad Group > Ad > Keyword
- Segment dimensions: date, device, geo_target_constant, audience, hour
- Resource-specific queries for each level
- Aggregation and comparison patterns

**Files to create:**
- `/rust_dev_cache/projects/googleads/gads_skill_gaql_gen/dimensional_analysis_reference.md`

### 1.5 Add Root Cause Attribution Framework

Multi-factor scoring system for categorizing causes:

| Factor | Weight | User Change Indicator | Market Change Indicator |
|--------|--------|----------------------|------------------------|
| Change proximity | 30% | change_event within 7 days | No changes found |
| Change type impact | 25% | Budget/bid change matches symptom | N/A |
| Seasonal pattern | 20% | No YoY pattern | Matches YoY trend |
| Cross-campaign effect | 15% | Single campaign affected | Account-wide pattern |
| Competitor signals | 10% | Auction insights stable | IS loss to competitors |

**Files to modify:**
- `/rust_dev_cache/projects/googleads/gads_skill_gaql_gen/identify_potential_causes.md` (extend)

---

## Phase 2: mcc-gaql-gen Tool Improvements

### 2.1 Enhance Metadata Command

Add new output modes and RAG search to the `metadata` command:

**New capabilities:**
- `mcc-gaql-gen metadata --search "<query>"` - Semantic search across all fields
- `mcc-gaql-gen metadata --related <resource>` - Show related/joinable resources
- `mcc-gaql-gen metadata --for-diagnosis <symptom>` - Show fields relevant to diagnosing a symptom

**Files to modify:**
- `/rust_dev_cache/projects/googleads/cookbook_gen_test/crates/mcc-gaql-gen/src/main.rs` (CLI args)
- `/rust_dev_cache/projects/googleads/cookbook_gen_test/crates/mcc-gaql-gen/src/formatter.rs` (new output modes)

### 2.2 Build Diagnostic Query Cookbook

Create a TOML cookbook with validated diagnostic query patterns:

```toml
[[query]]
category = "diagnosis"
symptom = "declining_conversions"
description = "Daily conversion trend with change events"
query = """
SELECT segments.date, campaign.name, metrics.conversions, metrics.cost_micros
FROM campaign WHERE segments.date DURING LAST_30_DAYS ORDER BY segments.date
"""

[[query]]
category = "change_history"
description = "Recent campaign changes for attribution"
query = """
SELECT change_event.change_date_time, change_event.change_resource_type,
       change_event.changed_fields, change_event.old_resource, change_event.new_resource
FROM change_event WHERE change_event.change_date_time DURING LAST_30_DAYS
"""
```

**Files to create:**
- `/rust_dev_cache/projects/googleads/cookbook_gen_test/crates/mcc-gaql-gen/resources/diagnostic_queries.toml`

### 2.3 Enrich Metadata for Diagnostic Fields

Prioritize enrichment quality for fields critical to diagnosis:
- `change_event.*` fields - user action attribution
- `change_status.*` fields - entity modification tracking
- Impression share metrics - competitive analysis
- Segment fields for dimensional drill-down

**Files to modify:**
- `/rust_dev_cache/projects/googleads/cookbook_gen_test/crates/mcc-gaql-gen/src/enricher.rs` (priority enrichment)

### 2.4 Add Resource Relationship Discovery

New command to expose resource compatibility and join patterns:

```bash
mcc-gaql-gen resources --show-joins campaign
# Output: campaign -> ad_group (via ad_group.campaign)
#         campaign -> change_event (via change_event.campaign)
#         campaign -> geographic_view (via geographic_view.campaign)
```

**Files to modify:**
- `/rust_dev_cache/projects/googleads/cookbook_gen_test/crates/mcc-gaql-gen/src/main.rs` (new subcommand)

---

## Phase 3: Validation and Feedback Loop

### 3.1 Integrate Validation into Generation

Modify `mcc-gaql-gen generate` to:
1. Auto-validate generated queries when `--validate` flag is set
2. Report validation errors with field-level specifics
3. Suggest corrections based on error patterns

**Files to modify:**
- `/rust_dev_cache/projects/googleads/cookbook_gen_test/crates/mcc-gaql-gen/src/rag.rs` (post-generation validation)

### 3.2 Create Failure Logging for Tool Improvement

When mcc-gaql-gen produces invalid GAQL:
1. Log the prompt, generated query, and validation error
2. Identify the failure pattern (wrong field, incompatible metrics, etc.)
3. Output structured feedback for metadata/cookbook improvement

**Files to create:**
- `/rust_dev_cache/projects/googleads/cookbook_gen_test/crates/mcc-gaql-gen/src/feedback.rs`

### 3.3 Add Skill Instructions for Failure Handling

Document the fallback workflow in SKILL.md:
1. When generation fails, check `mcc-gaql --show-fields <resource>` for correct field names
2. Use existing reference patterns from `mcc_gaql_reference.md`
3. Document the failure for future tool improvement

---

## Phase 4: Output Format Enhancement

### 4.1 Interactive Summary Format

Define the interactive diagnostic output structure:
- Key finding headlines with severity indicators
- Timeline visualization of changes vs performance
- Confidence scores for cause attribution
- Suggested next investigation steps

### 4.2 Diagnostic PDF Report Extension

Extend `pdf_generation_reference.md` with:
- Root cause analysis section
- Change event timeline visualization
- Multi-factor attribution scoring display
- Drill-down evidence tables

**Files to modify:**
- `/rust_dev_cache/projects/googleads/gads_skill_gaql_gen/pdf_generation_reference.md`

---

## Verification Plan

### Test GAQL Generation Flow
```bash
# Test metadata discovery
~/bin/mcc-gaql-gen metadata change_event --format llm

# Test query generation with validation
~/bin/mcc-gaql-gen generate "show campaign budget changes in last 30 days" --validate --explain

# Test fallback to mcc-gaql --show-fields
~/bin/mcc-gaql --show-fields change_event
```

### Test Diagnostic Workflow
1. Invoke skill with a known performance issue scenario
2. Verify autonomous exploration generates appropriate queries
3. Verify change event correlation identifies relevant changes
4. Verify multi-factor scoring produces reasonable attribution
5. Test interactive drill-down prompting

### Test Tool Feedback Loop
1. Intentionally request a query that generates invalid GAQL
2. Verify failure is logged with structured feedback
3. Verify skill falls back to manual patterns gracefully

---

## Implementation Order

1. **Phase 2.2**: Build diagnostic query cookbook (enables better generation immediately)
2. **Phase 1.2**: Add GAQL generation workflow to skill
3. **Phase 1.3**: Add change event correlation reference
4. **Phase 2.1**: Enhance metadata command with search
5. **Phase 1.4**: Add multi-dimensional analysis reference
6. **Phase 1.5**: Add root cause attribution framework
7. **Phase 3.1-3.3**: Validation and feedback loop
8. **Phase 4**: Output format enhancements
9. **Phase 1.1**: Final integration into SKILL.md diagnostic mode

---

## Key Files Summary

**Skill files to modify:**
- `SKILL.md` - Main skill instructions with new diagnostic mode
- `identify_potential_causes.md` - Extend with multi-factor scoring
- `pdf_generation_reference.md` - Add diagnostic report sections

**Skill files to create:**
- `gaql_generation_reference.md` - mcc-gaql-gen usage patterns
- `change_correlation_reference.md` - Change event analysis
- `dimensional_analysis_reference.md` - Multi-level drill-down patterns

**Tool files to modify (cookbook_gen_test):**
- `crates/mcc-gaql-gen/src/main.rs` - New CLI args/commands
- `crates/mcc-gaql-gen/src/rag.rs` - Validation integration
- `crates/mcc-gaql-gen/src/formatter.rs` - New output modes
- `crates/mcc-gaql-gen/src/enricher.rs` - Priority enrichment

**Tool files to create:**
- `crates/mcc-gaql-gen/resources/diagnostic_queries.toml` - Query cookbook
- `crates/mcc-gaql-gen/src/feedback.rs` - Failure logging
