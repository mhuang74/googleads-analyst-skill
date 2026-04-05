# Complete Workflow Examples

This document provides end-to-end examples of the Google Ads analyst workflow.

## Example 1: User with Configured Profile

**User:** "Analyze campaign performance for the last 7 days compared to the previous 7 days using profile themade"

**Account Type Check:**
"Is this a Google Ads Grants account (for non-profits)? Grants accounts have special rules - $10K/month cap and $2 max CPC."

**User:** "No, this is a regular account."

### Step 1: Determine date ranges and verify data availability

```bash
# Today is 2025-10-20, so:
# Current period: 2025-10-12 to 2025-10-18 (last 7 days)
# Previous period: 2025-10-05 to 2025-10-11 (previous 7 days)

# First, verify access and check latest data
mcc-gaql --profile themade --format json --list-child-accounts
mcc-gaql --profile themade --format csv 'SELECT segments.date, campaign.name, metrics.impressions FROM campaign WHERE segments.date DURING LAST_30_DAYS ORDER BY segments.date DESC LIMIT 10'
```

### Step 2: Query both periods and save to files

```bash
# Define the query (ALWAYS include impression share metrics)
CURRENT_QUERY='SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

PREVIOUS_QUERY='SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-05" AND segments.date <= "2025-10-11" AND campaign.status = "ENABLED"'

# VALIDATE queries before execution (REQUIRED)
echo "Validating current period query..."
mcc-gaql --profile themade --validate "$CURRENT_QUERY"

echo "Validating previous period query..."
mcc-gaql --profile themade --validate "$PREVIOUS_QUERY"

# Only execute if validation passed
echo "Executing queries..."
mcc-gaql --profile themade --format csv -o /tmp/current_period.csv "$CURRENT_QUERY"
mcc-gaql --profile themade --format csv -o /tmp/previous_period.csv "$PREVIOUS_QUERY"
```

### Step 3: Read and parse the CSV files

- Use Read tool to view /tmp/current_period.csv
- Use Read tool to view /tmp/previous_period.csv
- Parse the data and match campaigns by ID

### Step 4: Calculate metrics and changes

For each campaign:
- **FIRST: Convert cost_micros to dollars (÷ 1,000,000) and validate** - see [derived_metrics_reference.md](../analysis/derived_metrics_reference.md)
- Calculate derived metrics (CPA, ROAS, conversion rate, etc.)
- Calculate percentage changes for all metrics
- **Analyze impression share metrics**:
  - Search Impression Share: % of impressions received
  - Budget Lost IS: % of impressions lost due to budget constraints
  - Rank Lost IS: % of impressions lost due to ad rank/quality score
- Determine health status (🟢 🟡 🔴) - see [health_status.md](../analysis/health_status.md)

### Step 5: Analyze patterns and identify issues

- **Check impression share metrics first** - this reveals root causes:
  - High Budget Lost IS → increase budget
  - High Rank Lost IS → improve Quality Score, increase bids, or fix ad rank issues
  - Low overall IS → limited reach, growth opportunity if fixed
- Look for budget constraint patterns
- Check for ad fatigue indicators
- Identify conversion tracking issues
- Spot competitive pressure
- Recognize landing page problems

### Step 5b: Dynamic Investigation (REQUIRED when issues detected)

**This step is MANDATORY when HIGH or MEDIUM priority issues are found.** Use Dynamic Investigation Mode to drill down and gather evidence:

```bash
# Example: Campaign with conversion rate drop, unclear cause

# INVESTIGATION 1: Generate ad group breakdown to drill down
# Step 1: Generate query
AD_GROUP_QUERY=$(mcc-gaql-gen generate "Show ad group performance for campaign 'Brand Search' with clicks, conversions, cost for 2025-10-12 to 2025-10-18" --use-query-cookbook)

# Step 2: VALIDATE before executing (REQUIRED)
mcc-gaql --profile themade --validate "$AD_GROUP_QUERY"

# Step 3: Execute only if validation passed
if [ $? -eq 0 ]; then
  mcc-gaql --profile themade --format json -o /tmp/ad_group_drill.json "$AD_GROUP_QUERY"
fi

# INVESTIGATION 2: Check for user-initiated changes that might explain the issue
# NOTE: change_event has strict requirements - LIMIT required, dates must be within 30 days
# Step 1: Generate change event query (mcc-gaql-gen should generate with LIMIT)
CHANGE_QUERY=$(mcc-gaql-gen generate "Show change events for campaign 'Brand Search' in last 30 days with change type and user")

# Step 2: VALIDATE before executing (REQUIRED)
mcc-gaql --profile themade --validate "$CHANGE_QUERY"

# Step 3: Execute only if validation passed
if [ $? -eq 0 ]; then
  mcc-gaql --profile themade --format json -o /tmp/changes.json "$CHANGE_QUERY"
fi

# ANALYSIS: Analyze correlation between changes and performance anomaly
# Calculate correlation score (see change_correlation.md)
```

**Skip Dynamic Investigation ONLY for:**
- Minor fluctuations (<10% on metrics) with clear explanations
- Small sample sizes (<30 clicks) where deeper analysis won't be conclusive
- Low-priority issues with minimal spend impact (<$50/week)
- Obvious causes already identified in baseline analysis

### Step 5c: Present Text Report

Present your analysis as a **text report with markdown tables** directly in the terminal:

```markdown
## Executive Summary
[2-3 sentences on overall performance]

## Campaign Performance
| Campaign | Spend | Conv | CPA | ROAS | Change | Status |
|----------|-------|------|-----|------|--------|--------|
| ... | ... | ... | ... | ... | ... | ... |

## 🔴 Critical Issues
[If any - with root causes and recommendations]

## 🟡 Issues to Monitor  
[If any]

## Top Recommendations
1. [Action item]
2. [Action item]
```

Adapt structure based on complexity - simpler report for accounts with no major issues.

### Step 5d: Interactive Q&A

After presenting the text report, **pause and invite questions:**

```
"Do you have questions about any of these findings?"
"Would you like me to investigate any specific campaign or metric further?"
```

**Continue Q&A** until the user is satisfied.

### Step 6: Wrap Up

Once the user is happy with the analysis:

```
"A PDF version is available if you need it for sharing."
```

**Only generate PDF if user explicitly requests it.**

---

## Example 2: User WITHOUT Configured Profile

**User:** "Analyze my Google Ads account performance for the last 7 days"

### Step 1: Ask for required authentication information

```
I'll help you analyze your Google Ads performance. Since you don't have a profile configured, I need a few details:

1. What is your MCC (Manager Customer ID)? (e.g., 1234567890)
2. What is the Customer ID of the account you want to analyze? (e.g., 9876543210)
3. What is your Google account email with access to this account? (e.g., user@example.com)
```

**User provides:**
- MCC ID: 1234567890
- Customer ID: 9876543210
- User: john@example.com

**Account Type Check:**
"Is this a Google Ads Grants account (for non-profits)? Grants accounts have special rules - $10K/month cap and $2 max CPC."

**User:** "No, this is a regular account."

### Step 2: Determine date ranges and verify data availability

```bash
# Today is 2025-10-20, so:
# Current period: 2025-10-12 to 2025-10-18 (last 7 days)
# Previous period: 2025-10-05 to 2025-10-11 (previous 7 days)

# First, verify access and check latest data
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format json --list-child-accounts
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format csv 'SELECT segments.date, campaign.name, metrics.impressions FROM campaign WHERE segments.date DURING LAST_30_DAYS ORDER BY segments.date DESC LIMIT 10'
```

### Step 3: Query both periods and save to files

```bash
# Query current period (ALWAYS include impression share metrics)
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format csv -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

# Query previous period (ALWAYS include impression share metrics)
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format csv -o /tmp/previous_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-05" AND segments.date <= "2025-10-11" AND campaign.status = "ENABLED"'
```

### Step 4-6: Continue with same analysis workflow as Example 1

- Read CSVs and calculate metrics
- Analyze patterns and identify issues
- Perform dynamic investigation (when needed)
- Present text report
- Interactive Q&A
- Mention PDF availability

---

## Note: Grants Account Workflow Variation

If the user answers **"Yes"** to the Grants account question, respond with:

```
**Note:** Since this is a Grants account, I'll focus on CTR for Search campaigns (must stay above 5% for eligibility; Performance Max campaigns are exempt from this requirement), conversion metrics, and Quality Score. High lost impression share is expected for Grants accounts due to bid caps and won't be flagged as an issue.
```

Then proceed with analysis but:
- **Skip all impression share-based issue detection**
- **Do NOT recommend budget increases** (capped at $10K/month)
- **Do NOT recommend bid increases above $2** (Grants max CPC)
- **Focus on:**
  - CTR for Search campaigns (critical - must maintain >5% for Grants eligibility; Performance Max campaigns are exempt from this requirement)
  - Conversion metrics (CPA, conversion rate, ROAS)
  - Quality Score improvements (improve ad rank within bid constraints)
  - Ad relevance and landing page experience

---

For common errors and solutions, see [common_errors_reference.md](../tools/common_errors_reference.md).
