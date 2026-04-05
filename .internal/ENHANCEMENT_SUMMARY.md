# Enhancement Summary: Dynamic GAQL Generation for Performance Reports

**Date:** April 4, 2026  
**Based on:** [specs/dynamic_gaql_performance_reports.md](specs/dynamic_gaql_performance_reports.md)

## Overview

Enhanced the Google Ads Analyst Skill with dynamic GAQL generation capabilities for deeper root cause analysis and more specific, data-driven recommendations.

## What Was Added

### 1. New Section in SKILL.md: Dynamic Investigation Mode

**Location:** Section 4 - Dynamic Investigation Mode (Advanced Root Cause Analysis)

**Purpose:** Enable symptom-driven dynamic investigation to:
- Drill down from campaign → ad group → keyword/ad level based on observed issues
- Correlate performance changes with user-initiated changes via `change_event` analysis
- Generate specific recommendations with calculated impact estimates
- Provide follow-up queries for monitoring

**Key Features:**
- **When to use guidelines** - HIGH/MEDIUM priority issues vs minor fluctuations
- **5-phase investigation workflow**:
  1. Symptom Classification (from baseline analysis)
  2. Generate Investigation Queries (using `mcc-gaql-gen`)
  3. Change Event Correlation (user vs market changes)
  4. Evidence Synthesis (correlation scoring)
  5. Formulate Specific Recommendations (with calculated impacts)
- **Validation and fallback strategies** for generated queries
- **Common investigation patterns** with examples
- **Depth selection guidelines** (quick/standard/deep investigation)

### 2. New Reference Files

#### investigation_patterns_reference.md (20 KB)

Comprehensive symptom-to-query mapping guide with:

**Covered Symptoms:**
- Conversion rate decline
- Cost increase without conversion improvement
- Impression share loss
- CTR decline
- Zero/low conversions with spend

**For Each Symptom:**
- Investigation queries (with mcc-gaql-gen prompts)
- Expected generated GAQL output
- Analysis frameworks
- Expected patterns and root causes
- Recommended actions

**Additional Content:**
- Validation and fallback strategy
- Common generation issues and solutions
- Multi-level drill-down strategy (5 levels deep)
- Query generation best practices
- Iteration guidelines

#### action_templates_reference.md (26 KB)

Standardized templates for specific, data-driven recommendations:

**Template Structure:**
```
Action | Campaign | Current State | Recommended State | 
Expected Impact | Evidence | Confidence | Follow-up Query
```

**Included Templates:**
1. **Budget Increase** - with calculation formulas for new budget, impression gain, conversion estimates
2. **Pause Underperforming Ad Group** - with statistical significance checks
3. **Refresh Ad Creative** - for ad fatigue issues
4. **Add Negative Keywords** - with wasted spend calculations
5. **Exclude Poor Performing Locations** - geographic optimization
6. **Improve Quality Score / Ad Rank** - with component-specific actions
7. **Consolidate PMax Campaigns** - for cannibalization issues

**Each Template Includes:**
- When to recommend criteria
- Calculation formulas (Python examples)
- Impact estimation methodology
- Confidence scoring guidelines
- Follow-up query patterns
- Real-world examples

#### change_correlation_reference.md (22 KB)

Guide for correlating performance anomalies with user changes:

**Core Content:**
- **Change event query patterns** - using `change_event` resource
- **Resource types mapping** - 10+ change types with performance impact
- **Correlation analysis framework**:
  - 4-step process: Identify anomaly → Query changes → Match types → Calculate score
  - Correlation scoring methodology (100-point scale)
  - Change-symptom matching tables
  
**Correlation Factors (weighted):**
1. Temporal proximity (30 points)
2. Change-symptom match (30 points)
3. Magnitude alignment (20 points)
4. Pattern scope (20 points)

**Additional Features:**
- Parsing change event JSON (budget, bid strategy, keyword changes)
- 5+ common correlation patterns with examples
- Cross-campaign pattern detection (account-wide vs campaign-specific)
- Seasonal vs user change differentiation (YoY comparison)
- Root cause analysis output format

### 3. Updates to SKILL.md

**Core Capabilities (updated):**
- Added: Generate dynamic investigation queries
- Added: Drill down dynamically based on symptoms
- Added: Correlate performance changes with user changes
- Updated: Recommend specific actions → with calculated impact estimates

**Allowed Tools (updated):**
```yaml
allowed-tools: Bash(mcc-gaql:*,mcc-gaql-gen:*)
```

**Example Interactions (enhanced):**
- Added Step 5b: Dynamic Investigation for Complex Issues
- Shows when and how to use `mcc-gaql-gen`
- Demonstrates change event correlation workflow

**Final Checklist (updated):**
Added items:
- [ ] Dynamic investigation performed for HIGH/MEDIUM priority issues
- [ ] Root cause analysis with correlation scores
- [ ] Change event correlation checked
- [ ] Specific actions with calculated impact estimates
- [ ] Follow-up queries included

## How It Works

### Before Enhancement (Static Workflow)

```
Query campaigns → Calculate metrics → Pattern match → Generic recommendations
```

**Limitations:**
- Fixed queries, same depth for all issues
- No drill-down capability
- Can't distinguish user changes from market changes
- Generic recommendations ("increase budget")

### After Enhancement (Dynamic Workflow)

```
Query campaigns (baseline scan)
         ↓
Identify symptoms with severity
         ↓
[CRITICAL/HIGH ISSUES] → Dynamic Investigation:
         ├─ Generate targeted drill-down queries
         ├─ Query change_event for correlation
         ├─ Calculate correlation scores
         └─ Formulate specific recommendations
         ↓
Generate PDF report with:
  - Root cause analysis
  - Evidence-backed recommendations
  - Calculated impact estimates
  - Follow-up monitoring queries
```

**Benefits:**
- Adaptive investigation depth based on issue severity
- Drill-down to ad group, keyword, ad, device, location levels
- User vs market change attribution (with confidence scores)
- Specific recommendations with numbers ("$50 → $90/day budget based on 58% budget-lost IS")

## Example Scenarios

### Scenario 1: Budget Constraint

**Baseline Detection:**
- Campaign "Brand Search": Cost -45%, Conversions -50%
- Budget-lost IS: 58% (was 5%)

**Dynamic Investigation:**
```bash
# Check for budget changes
mcc-gaql-gen generate "Show change events for campaign Brand Search in last 30 days"
# Result: Budget reduced from $100 to $50 on 2025-10-14

# Check hourly performance
mcc-gaql-gen generate "Show hourly impressions and cost for campaign Brand Search for last 7 days"
# Result: Budget depletes by 2pm daily
```

**Root Cause:** User Change - Budget Reduction (92% confidence)

**Recommendation:**
```
Action: Increase daily budget for "Brand Search"
Current: $50/day, 58% budget-lost IS
Recommended: $90/day
Expected Impact: +50% impressions, 12-15 additional conversions/week, $750 revenue
Evidence: Budget reduced on 2025-10-14, ROAS 4.2x (profitable), depletes at 2pm
Confidence: 92%
```

### Scenario 2: Unknown Conversion Drop

**Baseline Detection:**
- Campaign "Services": Conversion rate dropped 35%
- No obvious metric patterns

**Dynamic Investigation:**
```bash
# Drill down to ad groups
mcc-gaql-gen generate "Show ad group performance for campaign Services with conversions"
# Result: "New - Consultations" ad group has 0 conversions, $400 spend

# Check keywords in problematic ad group
mcc-gaql-gen generate "Show keywords for ad group New - Consultations with conversions"
# Result: Broad match keywords capturing low-intent searches

# Search term report
mcc-gaql-gen generate "Show search terms for campaign Services ordered by cost"
# Result: 35% of clicks from "free", "jobs", "training" queries
```

**Root Cause:** Targeting Expansion - Poor Keyword Selection (88% confidence)

**Recommendation:**
```
Action: Pause ad group "New - Consultations" + Add negative keywords
Current: $400/week wasted on low-intent traffic
Recommended: Pause ad group, add 12 negative keywords (free, jobs, etc.)
Expected Impact: Save $400/week, improve conversion rate 15-20%
Evidence: 330 clicks with 0 conversions, keywords target job-seekers not buyers
Confidence: 88%
```

## Integration Points

The enhancement integrates seamlessly with existing workflow:

1. **Steps 1-3** remain unchanged (gather info, query data, baseline analysis)
2. **NEW Step 4** (Dynamic Investigation) activates for significant issues
3. **Step 5** (Generate Report) now includes root cause analysis and specific recommendations
4. **Checklist** ensures dynamic investigation is used appropriately

## Files Modified

1. **SKILL.md** - Added Section 4, updated core capabilities, examples, checklist
2. **Created:** investigation_patterns_reference.md
3. **Created:** action_templates_reference.md
4. **Created:** change_correlation_reference.md

## Files Referenced

- `specs/dynamic_gaql_performance_reports.md` - Original specification
- `docs/gaql_tools_overview.md` - Tool documentation for mcc-gaql and mcc-gaql-gen

## Next Steps for Users

When using the enhanced skill:

1. **Run baseline analysis as usual** (Steps 1-3)
2. **For HIGH/MEDIUM priority issues**, the skill will now:
   - Use `mcc-gaql-gen` to generate drill-down queries
   - Query `change_event` to check for user changes
   - Calculate correlation scores for root cause attribution
   - Provide specific recommendations with impact estimates
3. **Review recommendations** with confidence scores and follow-up queries
4. **Implement changes** and use provided follow-up queries to monitor

## Key Improvements

| Aspect | Before | After |
|--------|--------|-------|
| **Investigation depth** | Campaign-level only | Campaign → Ad Group → Keyword/Ad |
| **Root cause attribution** | Pattern matching | Correlation scores with confidence % |
| **Recommendation specificity** | "Increase budget" | "$50 → $90/day (based on 58% budget-lost IS)" |
| **Change detection** | None | User vs market with evidence |
| **Follow-up** | None | Runnable GAQL queries for monitoring |
| **Evidence** | Limited | Data points, calculations, reasoning |

## Technical Notes

**Query Generation:**
- Uses `mcc-gaql-gen generate` with `--use-query-cookbook` flag for best results
- Always validates with `mcc-gaql --validate` before execution
- Fallback to manual GAQL if generation fails

**Correlation Scoring:**
- 100-point scale across 4 factors
- 80-100 = Very likely cause (>90% confidence)
- 60-79 = Probable cause (75-90% confidence)
- <60 = Lower confidence, multiple hypotheses

**Resource Usage:**
- Quick investigation: 1-2 additional queries
- Standard investigation: 3-4 additional queries
- Deep investigation: 5+ queries (for critical issues only)

## References

For detailed usage, see:
- [SKILL.md Section 4](SKILL.md#4-dynamic-investigation-mode-advanced-root-cause-analysis)
- [investigation_patterns_reference.md](investigation_patterns_reference.md)
- [action_templates_reference.md](action_templates_reference.md)
- [change_correlation_reference.md](change_correlation_reference.md)
- [Original Spec](specs/dynamic_gaql_performance_reports.md)
