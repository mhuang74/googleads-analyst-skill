# Spec: Dynamic GAQL Generation for Enhanced Performance Reports

## Overview

This specification describes how to leverage `mcc-gaql-gen` for dynamic GAQL generation to improve the Google Ads Performance Report skill's ability to:

1. **Identify underlying issues** beyond surface-level metric changes
2. **Drill down dynamically** based on observed symptoms
3. **Suggest concrete actions** with supporting evidence
4. **Provide specific follow-up queries** for continued investigation

## Current State Analysis

### What the Skill Does Today

The current `SKILL.md` workflow:

1. **Fixed Query Templates** - Uses predefined GAQL patterns for campaign/period comparison
2. **Static Metric Set** - Always queries the same fields (impressions, clicks, cost, conversions, impression share)
3. **Pattern Matching** - Applies rules from reference files to identify known patterns
4. **Generic Recommendations** - Suggests actions based on pattern matching

### Limitations

| Limitation | Impact |
|------------|--------|
| Fixed queries can't adapt to specific symptoms | May miss relevant data for unusual issues |
| No drill-down into ad groups/keywords/ads | Root cause often hidden below campaign level |
| No change event correlation | Can't distinguish user changes from market shifts |
| Same depth for all issues | Wastes effort on minor issues, insufficient for complex ones |
| Recommendations lack specificity | "Increase budget" vs "Increase Campaign X budget from $50 to $85 based on lost IS" |

---

## Proposed Enhancement

### Core Principle: Symptom-Driven Dynamic Investigation

Instead of running the same queries for every report, the enhanced workflow:

1. **Baseline scan** - Quick overview to identify symptoms
2. **Dynamic drill-down** - Generate targeted queries based on what symptoms are found
3. **Evidence gathering** - Pull specific data to support or refute hypotheses
4. **Action formulation** - Construct specific recommendations with data backing

### New Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: Baseline Scan (Same as Current)                      │
│  - Campaign performance comparison (current vs prior period)   │
│  - Identify campaigns with significant changes                 │
│  - Flag symptoms: cost spike, conversion drop, CTR decline     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: Symptom Classification                                │
│  - Categorize each flagged campaign by symptom type            │
│  - Prioritize by severity and spend impact                     │
│  - Select investigation depth per symptom                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: Dynamic Investigation (NEW - uses mcc-gaql-gen)      │
│  For each prioritized symptom:                                  │
│  1. Generate targeted GAQL based on symptom type               │
│  2. Validate query before execution                            │
│  3. Execute and analyze results                                │
│  4. Generate follow-up queries if needed                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4: Evidence Synthesis                                    │
│  - Correlate findings across drill-down levels                 │
│  - Check change_event for user-initiated changes               │
│  - Calculate confidence scores for root causes                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 5: Actionable Recommendations                            │
│  - Formulate specific actions with data support                │
│  - Provide follow-up queries user can run                      │
│  - Estimate impact of recommended changes                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Phase 3 Detail: Dynamic GAQL Generation

### Symptom-to-Query Mapping

For each symptom type, define the investigation strategy:

#### Symptom: Conversion Rate Decline

```
Hypothesis Tree:
├── Traffic quality degraded
│   └── Query: Ad group/keyword performance breakdown
├── Landing page issues
│   └── Query: Device breakdown (mobile vs desktop conversion rate)
├── Audience fatigue
│   └── Query: Daily trend to see if gradual or sudden
└── Targeting expansion
    └── Query: Geographic performance, audience segment performance
```

**Dynamic Query Generation:**

```bash
# Step 1: Ad group breakdown
mcc-gaql-gen generate "Show ad group performance for campaign [NAME] with clicks, conversions, conversion rate for [DATE_RANGE]" --validate

# Step 2: If ad group variance is high, drill into keywords
mcc-gaql-gen generate "Show keyword performance for ad group [NAME] in campaign [NAME] with impressions, clicks, conversions for [DATE_RANGE]" --validate

# Step 3: Check device breakdown
mcc-gaql-gen generate "Show campaign [NAME] performance by device with clicks, conversions, cost for [DATE_RANGE]" --validate
```

#### Symptom: Cost Increase Without Conversion Improvement

```
Hypothesis Tree:
├── CPC inflation (competition)
│   └── Query: Daily CPC trend, auction insights if available
├── Budget reallocation to poor performers
│   └── Query: Ad group cost distribution vs conversion distribution
├── Bid strategy aggressive
│   └── Query: Bidding strategy status, target CPA/ROAS settings
└── New expensive keywords/audiences added
    └── Query: change_event for targeting changes
```

**Dynamic Query Generation:**

```bash
# Step 1: Check for targeting changes
mcc-gaql-gen generate "Show change events for campaign [NAME] in last 30 days including change type and changed fields" --validate

# Step 2: Ad group cost efficiency
mcc-gaql-gen generate "Show ad groups in campaign [NAME] with cost, conversions, average CPC ordered by cost descending" --validate

# Step 3: Daily CPC trend
mcc-gaql-gen generate "Show daily average CPC and cost for campaign [NAME] for last 30 days" --validate
```

#### Symptom: Impression Share Loss

```
Hypothesis Tree:
├── Budget constraint (budget_lost_impression_share > 20%)
│   └── Query: Hourly performance to see budget exhaustion timing
├── Ad rank issues (rank_lost_impression_share > 50%)
│   └── Query: Quality score data via ad_group_criterion
└── Increased competition
    └── Query: Daily impression share trend
```

#### Symptom: CTR Decline

```
Hypothesis Tree:
├── Ad fatigue
│   └── Query: Ad performance breakdown, ad age
├── Impression expansion to low-intent audiences
│   └── Query: Search term report, audience breakdown
├── Competitive ads improved
│   └── Query: Impression share metrics trend
└── Seasonality
    └── Query: Year-over-year comparison
```

### Query Generation Patterns

The skill should use these prompt patterns with `mcc-gaql-gen generate`:

```
# Pattern 1: Entity breakdown
"Show [child_entity] performance for [parent_entity] [NAME] with [metrics] for [date_range]"

# Pattern 2: Segment analysis  
"Show [entity] [NAME] performance by [segment] with [metrics] for [date_range]"

# Pattern 3: Time series
"Show daily [metrics] for [entity] [NAME] for [date_range] ordered by date"

# Pattern 4: Change history
"Show change events for [entity] [NAME] in [date_range] with change type and fields"

# Pattern 5: Top/Bottom performers
"Show top 10 [entities] in [parent_entity] by [metric] for [date_range]"
```

### Validation and Fallback

```
For each generated query:
1. Run: mcc-gaql --validate "<query>"
2. If valid: Execute query
3. If invalid:
   a. Check error message for field issues
   b. Use mcc-gaql --show-fields <resource> to find correct fields
   c. Regenerate with refined prompt OR
   d. Fall back to manual GAQL pattern from mcc_gaql_reference.md
4. Log generation failures for tool improvement
```

---

## Phase 4 Detail: Evidence Synthesis

### Change Event Correlation

Query `change_event` and `change_status` resources to identify user-initiated changes that correlate with performance shifts:

```bash
# Generate change history query
mcc-gaql-gen generate "Show all change events for customer in last 30 days with campaign name, change type, old and new values" --validate
```

**Correlation Logic:**

```
For each performance anomaly:
1. Identify anomaly start date (when metrics shifted)
2. Query changes within 7 days before anomaly start
3. Match change types to symptoms:
   - Budget change → cost/impression changes
   - Bid change → CPC/position changes
   - Targeting change → impression/CTR changes
   - Ad change → CTR changes
4. Calculate correlation score
```

### Multi-Factor Root Cause Scoring

| Factor | Weight | How to Assess |
|--------|--------|---------------|
| Change proximity | 30% | change_event within 7 days of anomaly |
| Change-symptom match | 25% | Budget change + cost symptom = match |
| Cross-campaign pattern | 15% | Single campaign = likely user change |
| Seasonal alignment | 15% | YoY comparison shows same pattern |
| Competitive signals | 15% | Impression share loss to competition |

**Output:**
```
Root Cause Analysis for Campaign "Brand - Search":
- Conversion rate dropped 35% starting 2024-03-15

Likely Cause: User Change (78% confidence)
  - Budget reduced from $100 to $50 on 2024-03-14 (change_event)
  - Correlates with impression drop of 45%
  - Best-performing hours now budget-constrained

Alternative: Market Change (22% confidence)
  - No significant competitor impression share gain
  - No seasonal pattern in YoY data
```

---

## Phase 5 Detail: Actionable Recommendations

### Recommendation Structure

Each recommendation should include:

```
┌────────────────────────────────────────────────────────────────┐
│ RECOMMENDATION                                                  │
├────────────────────────────────────────────────────────────────┤
│ Action: [Specific action to take]                              │
│ Campaign: [Exact campaign name]                                │
│ Current State: [Current value/setting]                         │
│ Recommended State: [New value/setting]                         │
│ Expected Impact: [Quantified improvement estimate]             │
│ Evidence: [Data points supporting this recommendation]         │
│ Confidence: [High/Medium/Low with percentage]                  │
│ Follow-up Query: [GAQL to monitor after change]               │
└────────────────────────────────────────────────────────────────┘
```

### Example Recommendations

**Budget Constraint Issue:**
```
Action: Increase daily budget
Campaign: "Brand - Search"
Current State: $50/day budget, 45% budget-lost impression share
Recommended State: $85/day budget
Expected Impact: Capture ~35% more impressions, estimated 12-15 additional conversions/week
Evidence:
  - Budget-lost IS: 45% (missing nearly half of available impressions)
  - Rank-lost IS: only 8% (ad quality is not the issue)
  - Historical ROAS: 4.2x (campaign is profitable)
  - Budget exhaustion time: 2pm daily (missing evening traffic)
Confidence: High (92%)
Follow-up Query: 
  SELECT segments.date, metrics.search_budget_lost_impression_share, 
         metrics.impressions, metrics.conversions
  FROM campaign 
  WHERE campaign.name = "Brand - Search" 
    AND segments.date DURING LAST_7_DAYS
```

**Ad Fatigue Issue:**
```
Action: Refresh ad creative
Campaign: "Product - Display"
Current State: Same 3 ads running for 45 days, CTR declined 28%
Recommended State: Create 2-3 new ad variations
Expected Impact: Restore CTR to baseline, improve conversion volume by ~20%
Evidence:
  - CTR trend: -28% over 6 weeks (gradual decline = fatigue pattern)
  - Impressions stable (audience not exhausted)
  - Ad creation dates all > 40 days ago
  - Frequency: 4.2 impressions per user (high)
Confidence: Medium (75%)
Follow-up Query:
  SELECT ad_group_ad.ad.id, ad_group_ad.ad.name, 
         metrics.ctr, metrics.impressions
  FROM ad_group_ad 
  WHERE campaign.name = "Product - Display"
    AND segments.date DURING LAST_7_DAYS
```

**Targeting Expansion Gone Wrong:**
```
Action: Add negative keywords and exclude poor locations
Campaign: "Services - Search"
Current State: Broad match keywords capturing irrelevant searches
Recommended State: 
  - Add negative keywords: [list based on search term analysis]
  - Exclude locations: [list based on geo performance]
Expected Impact: Reduce wasted spend by ~$X/day, improve conversion rate by ~15%
Evidence:
  - Search term analysis: 35% of clicks from irrelevant terms
  - Geographic analysis: 3 locations with 0 conversions, $X spend
  - Conversion rate by match type: Exact 8%, Broad 2%
Confidence: High (88%)
Follow-up Query:
  SELECT search_term_view.search_term, metrics.clicks, 
         metrics.conversions, metrics.cost_micros
  FROM search_term_view 
  WHERE campaign.name = "Services - Search"
    AND segments.date DURING LAST_14_DAYS
  ORDER BY metrics.cost_micros DESC
  LIMIT 50
```

---

## Implementation Guide

### SKILL.md Modifications

Add new section: "Dynamic Investigation Mode"

```markdown
### Dynamic Investigation Mode

When significant performance issues are detected, use dynamic GAQL generation 
to investigate root causes:

#### Step 1: Generate Investigation Query
Use mcc-gaql-gen to create targeted queries based on the symptom:

\`\`\`bash
mcc-gaql-gen generate "<investigation prompt>" --validate
\`\`\`

#### Step 2: Validate Before Execution
Always validate generated queries:

\`\`\`bash
mcc-gaql --validate "<generated query>"
\`\`\`

#### Step 3: Execute and Analyze
Run validated query and incorporate findings into analysis.

#### Step 4: Iterate if Needed
Generate follow-up queries based on initial findings.
```

### New Reference File: investigation_patterns_reference.md

Document symptom-to-query mappings:

```markdown
## Symptom: Conversion Rate Decline

### Investigation Queries

1. **Ad Group Breakdown**
   Prompt: "Show ad group performance for campaign [NAME] with conversions, 
            conversion rate, cost for [DATE_RANGE]"
   Purpose: Identify which ad groups are underperforming

2. **Device Analysis**
   Prompt: "Show campaign [NAME] performance by device with clicks, 
            conversions, conversion rate for [DATE_RANGE]"
   Purpose: Check if mobile/desktop has different patterns

3. **Daily Trend**
   Prompt: "Show daily conversions and conversion rate for campaign [NAME] 
            for last 30 days ordered by date"
   Purpose: Identify when decline started (sudden vs gradual)

### Expected Patterns
- Gradual decline across all ad groups → Ad fatigue or market shift
- Sudden decline in specific ad groups → Targeting or landing page issue
- Mobile-only decline → Mobile experience problem
- Decline starting on specific date → Correlate with change_event
```

### New Reference File: action_templates_reference.md

Provide templates for specific recommendations:

```markdown
## Action Template: Budget Increase

### When to Recommend
- Budget-lost impression share > 20%
- Campaign ROAS > target (profitable)
- Rank-lost impression share < 30% (quality is acceptable)

### Template
Action: Increase daily budget for [CAMPAIGN]
Current: $[CURRENT]/day with [BUDGET_LOST_IS]% budget-lost IS
Recommended: $[NEW]/day (calculated as current / (1 - budget_lost_is))
Expected Impact: +[IMPRESSION_GAIN]% impressions, +[CONV_ESTIMATE] conversions
Evidence: [DATA_POINTS]
Confidence: [CONFIDENCE]%

### Calculation
new_budget = current_budget / (1 - budget_lost_impression_share)
# Example: $50 / (1 - 0.45) = $90.90, round to $90

impression_gain = budget_lost_impression_share * current_impressions
conversion_estimate = impression_gain * current_ctr * current_conv_rate
```

---

## Success Metrics

| Metric | Current State | Target |
|--------|--------------|--------|
| Issue identification accuracy | Manual pattern matching | 90%+ with evidence |
| Recommendation specificity | Generic ("increase budget") | Specific with numbers |
| Root cause attribution | None | User vs market with confidence % |
| Follow-up capability | None | Runnable GAQL for monitoring |
| Investigation depth | Campaign-level only | Campaign → Ad Group → Keyword |

---

## Dependencies

### Required Tools
- `mcc-gaql-gen generate` - Query generation
- `mcc-gaql --validate` - Query validation
- `mcc-gaql` - Query execution

### Required Metadata
- Enriched field metadata for accurate generation
- Query cookbook with diagnostic patterns
- Resource compatibility information

### Reference Files to Create/Modify
1. `investigation_patterns_reference.md` - Symptom-to-query mappings
2. `action_templates_reference.md` - Recommendation templates
3. `change_correlation_reference.md` - Change event analysis patterns
4. `SKILL.md` - Add Dynamic Investigation Mode section

---

## Appendix: Sample Investigation Session

```
USER: Generate performance report for profile "acme" for last 7 days

SKILL (Phase 1 - Baseline):
Querying campaign performance...
[Executes standard campaign comparison query]

Results:
- Campaign "Brand Search": Cost +5%, Conversions +8% 🟢
- Campaign "Product Display": Cost +45%, Conversions -12% 🔴
- Campaign "Retargeting": Cost -20%, Conversions -25% 🟡

SKILL (Phase 2 - Classification):
Priority issue detected: "Product Display" - Cost increase without 
conversion improvement (CRITICAL)

SKILL (Phase 3 - Dynamic Investigation):
Investigating root cause...

Step 1: Generating ad group breakdown query
> mcc-gaql-gen generate "Show ad groups in campaign Product Display with 
  cost, clicks, conversions for 2024-03-01 to 2024-03-07" --validate
> [Query validated, executing...]

Findings: 
- Ad Group "Laptops": Cost $500 → $850, Conversions 12 → 8
- Ad Group "Accessories": Cost $200 → $250, Conversions 15 → 18 🟢
- Ad Group "New - Tablets": Cost $0 → $400, Conversions 0 🔴 (NEW)

Step 2: New ad group detected, checking change events
> mcc-gaql-gen generate "Show change events for campaign Product Display 
  in last 14 days" --validate
> [Query validated, executing...]

Findings:
- 2024-02-28: Ad group "New - Tablets" created
- 2024-02-28: Keywords added to "New - Tablets"
- 2024-03-01: Budget increased from $700 to $1500

Step 3: Investigating new ad group performance
> mcc-gaql-gen generate "Show keyword performance for ad group New - Tablets 
  with impressions, clicks, cost for last 7 days" --validate
> [Query validated, executing...]

Findings:
- Keyword "tablet deals": 2000 impressions, 150 clicks, 0 conversions
- Keyword "cheap tablets": 1500 impressions, 120 clicks, 0 conversions
- Keyword "buy tablet": 800 impressions, 60 clicks, 0 conversions

SKILL (Phase 4 - Evidence Synthesis):
Root Cause: User Change (95% confidence)
- New ad group "New - Tablets" launched 2024-02-28
- Received $400 spend with 0 conversions
- Keywords are high-volume but low-intent (deals/cheap seekers)
- Budget increase allowed this spend to occur

SKILL (Phase 5 - Recommendations):
┌────────────────────────────────────────────────────────────────┐
│ RECOMMENDATION #1: Pause underperforming ad group              │
├────────────────────────────────────────────────────────────────┤
│ Action: Pause ad group "New - Tablets"                         │
│ Campaign: Product Display                                      │
│ Current State: $400 spend, 0 conversions, 0% conversion rate   │
│ Expected Impact: Save ~$400/week, restore campaign ROAS        │
│ Evidence:                                                      │
│   - 330 clicks with 0 conversions (statistical significance)   │
│   - Keywords targeting deal-seekers (low purchase intent)      │
│   - No conversions in 7 days of running                        │
│ Confidence: High (92%)                                         │
│ Follow-up Query:                                               │
│   SELECT segments.date, metrics.cost_micros, metrics.conversions│
│   FROM ad_group                                                │
│   WHERE ad_group.name = "New - Tablets"                        │
│     AND segments.date DURING LAST_7_DAYS                       │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ RECOMMENDATION #2: Reduce budget to previous level             │
├────────────────────────────────────────────────────────────────┤
│ Action: Reduce daily budget                                    │
│ Campaign: Product Display                                      │
│ Current State: $1500/day                                       │
│ Recommended State: $700/day (previous level)                   │
│ Expected Impact: Concentrate spend on proven performers        │
│ Evidence:                                                      │
│   - Budget was increased to support new ad group               │
│   - New ad group is not converting                             │
│   - Original ad groups were performing at acceptable ROAS      │
│ Confidence: Medium (78%)                                       │
└────────────────────────────────────────────────────────────────┘

[Report continues with full analysis...]
```
