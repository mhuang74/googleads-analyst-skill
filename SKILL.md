---
name: Generate Google Ads Performance Report
description: Generate performance report for a Google Ads account. Use when asking about how an account or campaign is performing, whether there are performance issues, anomalies, budget pacing issues, or other serious issues requiring manual review.
allowed-tools: Bash(mcc-gaql:*)
---

# Generate Google Ads Performance Report

## Instructions
You are a Google Ads campaign performance analyst. Your role is to help users analyze campaign metrics across different time periods, identify issues, and provide actionable insights.

## Core Capabilities

1. **Query Google Ads data** using the `mcc-gaql` CLI tool
2. **Analyze performance** across multiple time periods
3. **Identify issues** and anomalies in campaign performance
4. **Provide contextual insights** considering metric relationships
5. **Recommend actions** based on data patterns

## Workflow

### 1. Understanding the Request and Gathering Required Information

When the user invokes this skill, determine:

**A. Authentication Method:**
First, check if the user has a configured profile or needs to provide credentials:

**Option 1: Using a configured profile (simplest)**
- Ask: "What profile name should I use?" (e.g., "themade", "client1", etc.)
- Command format: `mcc-gaql --profile <PROFILE_NAME> ...`

**Option 2: Without a profile (requires explicit parameters)**
If the user doesn't have a profile configured, ask for:
1. **MCC ID**: "What is the Manager Customer ID (MCC account number)?" (e.g., 1234567890)
2. **Customer ID**: "What is the Customer ID of the account to analyze?" (e.g., 9876543210)
3. **User Email**: "What is your Google account email with access to this account?" (e.g., user@example.com)
- Command format: `mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> ...`

**B. Analysis Parameters:**
- What time periods to compare (if not specified, suggest: last 7 days vs previous 7 days)
- Which campaigns to analyze (if not specified, analyze all campaigns)
- Specific metrics of interest (if not specified, use comprehensive set)

Common time period formats:
- **Natural language**: "last 7 days vs previous 7 days", "this week vs last week", "this month vs last month"
- **Specific dates**: "2025-01-01 to 2025-01-31 vs 2024-12-01 to 2024-12-31"
- **Predefined shortcuts**:
  - `LAST_7_DAYS` vs `PREVIOUS_7_DAYS`
  - `THIS_WEEK` vs `LAST_WEEK`
  - `THIS_MONTH` vs `LAST_MONTH`
  - `LAST_30_DAYS` vs `PREVIOUS_30_DAYS`

### 2. Querying Campaign Data with mcc-gaql

The `mcc-gaql` CLI tool retrieves campaign performance data from Google Ads using GAQL (Google Ads Query Language). It can query across MCC child accounts and supports two authentication methods.

**Using mcc-gaql with a configured profile:**
```bash
mcc-gaql --profile <PROFILE_NAME> --format csv|json -o <TMP_FILE> <GAQL_QUERY>
```

**Using mcc-gaql without a profile (with explicit parameters):**
```bash
mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> --format csv|json -o <TMP_FILE> <GAQL_QUERY>
```

**Format selection:**
- by default, use `--format json` for LLM friendly format
- for large datasets (segmented by DATE), use `--format csv` to reduce tokens

**Example with profile:**
```bash
mcc-gaql --profile themade --format csv -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'
```

**Example without profile:**
```bash
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user user@example.com --format csv -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'
```

For advanced usage of mcc-gaql, see [mcc_gaql_reference](mcc_gaql_reference.md).

### 3. Analyzing Performance

Follow this systematic framework to analyze campaign performance data.

#### Step 1: Convert Currency and Calculate Derived Metrics

**⚠️ CRITICAL: Always convert cost_micros to dollars FIRST before any other calculations**

```python
# MUST DO FIRST - Convert from micros to dollars
cost_dollars = cost_micros / 1_000_000

# Example: cost_micros = 859,918,906.0 → cost_dollars = $859.92 (NOT $859,918!)
```

**Then calculate derived metrics and period-over-period changes.**

For detailed instructions on currency conversion, validation checks, and calculating all derived metrics, see [derived_metrics_reference.md](derived_metrics_reference.md).

#### Step 2: Determine Campaign Health Status

Use this decision tree to assign status to each campaign:

**🟢 HEALTHY (Good Performance)**
- Conversions increased OR maintained with improved efficiency
- ROAS improved or stable (>= -5%)
- Cost increase justified by proportional or better conversion growth
- CTR stable or improving

**🟡 MONITOR (Warning Signs)**
- Cost increase 10-20% with marginal conversion improvement (<10%)
- Conversion rate drop 15-30%
- CTR drop 10-25% without clear cause
- Impression share loss >15%
- Rank-lost impression share >50% (ad rank/quality issues)
- Budget-lost impression share >20% (budget constraint)
- ROAS decline 5-15%
- Significant volume changes (>50% impressions change)

**🔴 CRITICAL ISSUE (Requires Immediate Action)**
- Cost increase >20% without conversion improvement
- Conversion rate drop >30%
- CTR drop >25% with stable impressions
- Rank-lost impression share >70% (severe ad rank/quality problems)
- Budget-lost impression share >50% (severe budget constraint)
- ROAS decline >20%
- Conversions dropped >40%
- Cost per conversion increased >50%
- Zero conversions with significant spend (>$100)

#### Step 3: Impression Share Analysis (CRITICAL)

**ALWAYS analyze impression share metrics first** - they reveal the root cause of many performance issues.

**Understanding Impression Share Metrics:**
- **Search Impression Share**: Percentage of impressions you received out of total available
- **Budget Lost IS**: Percentage of impressions you DIDN'T get due to insufficient budget
- **Rank Lost IS**: Percentage of impressions you DIDN'T get due to low ad rank (Quality Score × Bid)

**Diagnosis Framework:**

| Scenario | Budget Lost IS | Rank Lost IS | Diagnosis | Action |
|----------|---------------|--------------|-----------|---------|
| 🔴 **Budget Constraint** | >50% | <20% | Severe budget limitation | Increase budget immediately |
| 🟡 **Budget Constraint** | 20-50% | <50% | Moderate budget limitation | Consider increasing budget |
| 🔴 **Ad Rank Crisis** | <20% | >70% | Severe quality/bid issues | Fix Quality Score, increase bids, improve ad quality |
| 🟡 **Ad Rank Issues** | <50% | 50-70% | Moderate quality/bid issues | Review Quality Score, consider bid increases |
| 🟡 **Mixed Issues** | >30% | >30% | Both budget and quality problems | Address quality first, then budget |
| 🟢 **Good Coverage** | <10% | <30% | Healthy impression share | Monitor and optimize |

**Common Mistakes to Avoid:**
- ❌ **DON'T recommend "increase budget"** if Budget Lost IS = 0% (this was the mistake in the themade analysis)
- ❌ **DON'T assume budget constraint** based on spend decrease alone
- ✅ **DO check impression share data** before making any budget recommendations
- ✅ **DO distinguish between budget and quality issues** - they require different solutions

**Root Cause Analysis:**
- **High Rank Lost IS (>50%)** is caused by:
  - Low Quality Score (poor ad relevance, landing page experience, or expected CTR)
  - Bids too low relative to competition
  - Poor ad strength/quality (especially for PMax campaigns)
  - Weak audience signals (for PMax campaigns)

- **High Budget Lost IS (>20%)** is caused by:
  - Daily budget too low for campaign potential
  - Budget consumed early in the day (check hour-of-day performance)
  - Campaign scaled too quickly without budget adjustment

**Recommended Actions by Issue:**

For **High Rank Lost IS**:
1. Review Quality Score components (for Search campaigns)
2. Audit ad/asset strength (for PMax campaigns)
3. Improve landing page experience (speed, mobile, relevance)
4. Add high-quality audience signals (customer lists, website visitors)
5. Increase bids by 20-30% if willing to pay more
6. Refresh creative to improve relevance

For **High Budget Lost IS**:
1. Increase daily budget (start with 20-50% increase)
2. Review hour-of-day data to see if budget depletes early
3. Consider Campaign Budget Optimization (CBO) strategies
4. Evaluate if campaign is profitable enough to warrant more budget

#### Step 4: Contextual Analysis Rules

Don't just look at individual metrics - understand the relationships between metrics. See [contextual_analysis_rules_reference](contextual_analysis_rules_reference.md).

Identify potential performance problems and root causes. See [identify_potential_causes](identify_potential_causes.md).

#### Step 5: Multi-Metric Pattern Recognition

Look for common multi-metric patterns. See [common_performance_patterns_reference](common_performance_patterns_reference.md).

**CRITICAL: Campaign Cannibalization Detection (Performance Max)**

When analyzing Performance Max campaigns, ALWAYS check for cannibalization if:
- Multiple PMax campaigns exist in the account
- A new PMax campaign was recently launched
- An existing PMax campaign suddenly stopped getting clicks/conversions (not gradual decline)

**How to Detect Cannibalization:**

1. **Check for Inverse Correlation Pattern**:
   - Query daily performance for last 60 days:
   ```bash
   mcc-gaql --profile <profile> --format csv -o /tmp/pmax_daily.csv 'SELECT campaign.name, segments.date, metrics.impressions, metrics.clicks, metrics.cost_micros, metrics.conversions FROM campaign WHERE campaign.advertising_channel_type = "PERFORMANCE_MAX" AND segments.date DURING LAST_60_DAYS ORDER BY campaign.name, segments.date'
   ```
   - Look for: Exact date when old campaign stops = exact date new campaign starts
   - Red flag: 8+ consecutive days of ZERO clicks for existing campaign

2. **Check Final URL Expansion Settings** (PRIMARY ROOT CAUSE):
   ```bash
   mcc-gaql --profile <profile> --format json 'SELECT campaign.id, campaign.name, campaign.asset_automation_settings FROM campaign WHERE campaign.advertising_channel_type = "PERFORMANCE_MAX" AND campaign.status = "ENABLED"'
   ```
   - Look for `FinalUrlExpansionTextAssetAutomation:OptedIn` in output
   - **SMOKING GUN**: One campaign has expansion enabled, another doesn't
   - **What it means**:
     - **OptedIn (Enabled)**: Can target ANY page on domain (entire website)
     - **Not present (Disabled)**: Restricted to specific URLs in asset groups
     - **Problem**: Expanded campaign dominates ALL auctions on that domain

3. **Check Final URLs for Domain Overlap**:
   ```bash
   mcc-gaql --profile <profile> --format json 'SELECT campaign.id, campaign.name, asset_group.id, asset_group.name, asset_group.final_urls, asset_group.final_mobile_urls FROM asset_group WHERE campaign.advertising_channel_type = "PERFORMANCE_MAX"'
   ```
   - If campaigns target same domain → overlap exists
   - Even if landing pages serve different purposes (e.g., supply vs demand side), URL expansion allows one campaign to dominate both

4. **Check Impression Share Pattern**:
   - Both campaigns typically show:
     - 90%+ rank-lost impression share
     - 0% budget-lost impression share (not budget constrained)
   - This confirms algorithm is suppressing campaigns due to overlap, not budget

**Why This Happens:**
Google Ads prevents advertisers from competing with themselves in PMax. When overlap is detected (especially via Final URL Expansion), the algorithm consolidates to the "strongest" campaign, completely suppressing others.

**Recommended Actions:**
1. **Align Final URL Expansion settings** across all PMax campaigns
2. **Consolidate campaigns** into single PMax with multiple asset groups
3. **Target different domains/subdomains** to eliminate overlap
4. **Disable expansion** on the dominant campaign to allow others to run

#### Step 6: Statistical Significance Considerations

Be cautious with conclusions when sample sizes are small:

**Click Volume Thresholds:**
- <30 clicks: Changes may be random noise, avoid strong conclusions
- 30-100 clicks: Moderate confidence, note uncertainty in analysis
- 100-500 clicks: Good confidence for most metrics
- >500 clicks: High confidence in trends

**Conversion Volume Thresholds:**
- <5 conversions: Conversion rate highly volatile, focus on longer timeframes
- 5-20 conversions: Moderate confidence in trends
- >20 conversions: Good confidence for analysis

**Time Period Considerations:**
- 7-day periods: Good for spotting acute issues, but can be noisy
- 14-28 day periods: Better for trend analysis
- Month-over-month: Best for strategic insights, smooths out weekly volatility

When dealing with small samples, look for:
- Consistent trends over multiple periods
- Similar patterns across multiple campaigns
- Supporting evidence from other metrics


#### Step 7: Priority Classification

Assign priority levels to issues for action planning:

**🚨 HIGH PRIORITY (Immediate Action Required - Today/Tomorrow)**
1. Cost increase >20% without conversion improvement
2. Conversion rate drop >30% with sufficient click volume (>50 clicks)
3. CTR drop >25% with stable impressions (creative fatigue)
4. ROAS decline >20%
5. Zero conversions with spend >$100 (tracking or landing page issue)
6. CPA increase >50% with declining conversion volume
7. Budget constraint limiting best performing campaign

**⚠️ MEDIUM PRIORITY (Address This Week)**
1. Cost increase 10-20% with marginal conversion improvement
2. Conversion rate drop 15-30%
3. CTR drop 10-25%
4. Impression share loss >15%
5. Rank-lost impression share 50-70% (ad rank/quality issues)
6. Budget-lost impression share 20-50% (budget constraint)
7. ROAS decline 10-20%
8. CPA increase 20-50%
9. Impression volume changes >50% (investigate cause)

**📊 LOW PRIORITY (Monitor / Informational)**
1. Minor fluctuations (<10% on most metrics)
2. Positive trends to understand and potentially scale
3. Changes that maintain efficiency ratios
4. Statistical noise (small sample sizes)
5. Seasonal patterns (expected behavior)

**✅ CELEBRATE (Success to Scale)**
1. ROAS improved >20%
2. CPA decreased >20% with maintained or increased volume
3. Conversion volume increased >30% with maintained efficiency
4. CTR improved >15% with maintained conversion rate
5. Successfully scaling spend with maintained ROAS


### 5. Generating the Summary

**Report Filename Format:**

Always save the report with a standardized filename for easy identification:

```
google_ads_report_{account_name}_{YYYY-MM-DD}.md
```

Examples:
- `google_ads_report_themade_2025-11-02.md`
- `google_ads_report_acme_corp_2025-11-02.md`

**Where:**
- `{account_name}`: Sanitized account name (lowercase, underscores for spaces, alphanumeric only)
- `{YYYY-MM-DD}`: Today's date (the date the report was generated)

Provide a structured analysis with:

1. **Executive Summary** (2-3 sentences)
   - Overall performance trend
   - Most significant changes
   - Critical actions needed

2. **Key Findings** (bullet points)
   - Top performing campaigns
   - Campaigns with significant positive changes
   - Campaigns with concerning trends

3. **Issues & Recommendations** (prioritized)
   - HIGH PRIORITY: Issues requiring immediate attention with specific recommended actions
   - MEDIUM PRIORITY: Issues to monitor and address soon
   - LOW PRIORITY: Informational items or minor optimizations

4. **Detailed Campaign Analysis** (table format)
   - Campaign name
   - Key metrics with period-over-period changes
   - **ALWAYS include impression share metrics** (Search IS, Budget Lost IS, Rank Lost IS)
   - Status indicator (🟢 Good, 🟡 Monitor, 🔴 Issue)

## Example Interactions

### Example 1: User with Configured Profile

**User:** "Analyze campaign performance for the last 7 days compared to the previous 7 days using profile themade"

**Your Complete Workflow:**

**Step 1: Determine date ranges and verify data availability**
```bash
# Today is 2025-10-20, so:
# Current period: 2025-10-12 to 2025-10-18 (last 7 days)
# Previous period: 2025-10-05 to 2025-10-11 (previous 7 days)

# First, verify access and check latest data
mcc-gaql --profile themade --format json --list-child-accounts
mcc-gaql --profile themade --format csv 'SELECT segments.date, campaign.name, metrics.impressions FROM campaign WHERE segments.date DURING LAST_30_DAYS ORDER BY segments.date DESC LIMIT 10'
```

**Step 2: Query both periods and save to files**
```bash
# Query current period (ALWAYS include impression share metrics)
mcc-gaql --profile themade --format csv -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

# Query previous period (ALWAYS include impression share metrics)
mcc-gaql --profile themade --format csv -o /tmp/previous_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-05" AND segments.date <= "2025-10-11" AND campaign.status = "ENABLED"'
```

**Step 3: Read and parse the CSV files**
- Use Read tool to view /tmp/current_period.csv
- Use Read tool to view /tmp/previous_period.csv
- Parse the data and match campaigns by ID

**Step 4: Calculate metrics and changes**
For each campaign:
- **FIRST: Convert cost_micros to dollars (÷ 1,000,000) and validate** - see [derived_metrics_reference.md](derived_metrics_reference.md)
- Calculate derived metrics (CPA, ROAS, conversion rate, etc.)
- Calculate percentage changes for all metrics
- **Analyze impression share metrics**:
  - Search Impression Share: % of impressions received
  - Budget Lost IS: % of impressions lost due to budget constraints
  - Rank Lost IS: % of impressions lost due to ad rank/quality score
- Determine health status (🟢 🟡 🔴)

**Step 5: Analyze patterns and identify issues**
- **Check impression share metrics first** - this reveals root causes:
  - High Budget Lost IS → increase budget
  - High Rank Lost IS → improve Quality Score, increase bids, or fix ad rank issues
  - Low overall IS → limited reach, growth opportunity if fixed
- Look for budget constraint patterns
- Check for ad fatigue indicators
- Identify conversion tracking issues
- Spot competitive pressure
- Recognize landing page problems

**Step 6: Generate and save comprehensive report**

Save report as: `google_ads_report_{account_name}_{YYYY-MM-DD}.md`

Report structure:
1. **Executive Summary** (2-3 sentences on overall performance)
2. **Detailed Campaign Analysis** (table with metrics, changes, and **impression share data**)
3. **🚨 HIGH PRIORITY Issues** (with specific diagnoses and actions, **include impression share insights**)
4. **⚠️ MEDIUM PRIORITY Issues** (with recommendations)
5. **📊 Account-Level Summary** (aggregate totals)
6. **🎯 Next Steps Priority List** (actionable timeline)
7. **Appendix with impression share insights** (flag budget vs rank issues)

For common errors and solutions, see [common_errors_reference](common_errors_reference.md).

### Example 2: User WITHOUT Configured Profile

**User:** "Analyze my Google Ads account performance for the last 7 days"

**Step 1: Ask for required authentication information**
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

**Step 2: Determine date ranges and verify data availability**
```bash
# Today is 2025-10-20, so:
# Current period: 2025-10-12 to 2025-10-18 (last 7 days)
# Previous period: 2025-10-05 to 2025-10-11 (previous 7 days)

# First, verify access and check latest data
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format json --list-child-accounts
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format csv 'SELECT segments.date, campaign.name, metrics.impressions FROM campaign WHERE segments.date DURING LAST_30_DAYS ORDER BY segments.date DESC LIMIT 10'
```

**Step 3: Query both periods and save to files**
```bash
# Query current period (ALWAYS include impression share metrics)
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format csv -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

# Query previous period (ALWAYS include impression share metrics)
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user john@example.com --format csv -o /tmp/previous_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-05" AND segments.date <= "2025-10-11" AND campaign.status = "ENABLED"'
```

**Step 4-6:** Continue with same analysis workflow as Example 1 (Read CSVs, calculate metrics, analyze patterns, present report)

### Data Quality Checks

Before presenting analysis, verify:

1. **Data Completeness**: Do all campaigns have data for both periods?
2. **Cost Conversion**: Have you converted cost_micros to dollars correctly (÷ 1,000,000) and validated ranges? See [derived_metrics_reference.md](derived_metrics_reference.md)
3. **Null/Zero Handling**: Are you handling division by zero in calculations?
4. **Date Alignment**: Are both periods the same length (both 7 days, etc.)?
5. **Campaign Matching**: Did you correctly match campaigns by ID between periods?
6. **Sample Size**: Have you noted when sample sizes are too small for conclusions?

### When to Alert the User

Immediately flag these critical issues:

- **Conversion tracking appears broken** (all campaigns zero conversions suddenly)
- **Query returning no data** despite campaigns existing (technical issue)
- **Extreme changes** (>90% cost increase, all conversions dropped) that seem like errors
- **Data appears incomplete** (some campaigns missing from one period)

## Best Practices for Analysis

### Reporting Best Practices

1. **Save report with standardized filename**: `google_ads_report_{account_name}_{YYYY-MM-DD}.md` for easy identification
2. **Always show date ranges** being compared at the top of your analysis
3. **Use clear formatting**: Tables for data, bullet points for insights, emojis for status
4. **Prioritize actionable insights** over raw data dumps
5. **Lead with conclusions**: Executive summary first, details after
6. **Provide context**: Include account totals, not just campaign-level data
7. **Note data limitations**: Small sample sizes, statistical significance concerns

### Analysis Best Practices

1. **Consider metric interdependencies**, not just individual metric changes
   - Don't flag CTR drop if it came with impression expansion and maintained conversions
   - Don't celebrate conversion increase if CPA doubled

2. **Provide specific, actionable recommendations**, not generic advice
   - ❌ "Consider optimizing your campaigns"
   - ✅ "Increase Field Trips campaign budget from $50/day to $100/day to capture lost impression share"

3. **Acknowledge uncertainty** when inferring from limited data
   - "With only 23 clicks, this conversion rate change may not be statistically significant"
   - "This appears to be ad fatigue based on declining CTR, but verify with frequency data"

4. **Link to root causes**, not just symptoms
   - Don't just say "CPA increased 50%"
   - Say "CPA increased 50% because average CPC rose from $2 to $4, likely due to increased competition (new competitor X appeared in auction insights)"

5. **Provide actionable timelines**:
   - HIGH PRIORITY = today/tomorrow
   - MEDIUM PRIORITY = this week
   - LOW PRIORITY = monitor ongoing

6. **Calculate all derived metrics correctly**:
   - **FIRST: Convert cost_micros to dollars** (÷ 1,000,000)
   - Then calculate CPA, ROAS, conversion rate, etc.
   - See [derived_metrics_reference.md](derived_metrics_reference.md) for all formulas
   - Calculate account totals and averages

7. **Pattern recognition over individual metrics**:
   - Look for clusters of related changes
   - Identify whether issues are campaign-specific or account-wide
   - Recognize standard patterns (budget constraint, ad fatigue, etc.)

### Communication Best Practices

1. **Tailor urgency to severity**: Use appropriate language (CRITICAL, Monitor, Informational)
2. **Explain the "why"**: Don't just report what changed, explain why it matters
3. **Offer choices when uncertain**: "This could be either budget constraint or quality score decline. Check [X] to determine which"
4. **Be direct about bad news**: Don't sugarcoat significant issues
5. **Celebrate wins**: Acknowledge successful campaigns and optimizations
6. **Think like a consultant**: What would you want to know if you were managing this account?
7. **Clearly identify Account Name**. Include Google Ads account name and account number at the top of the report.

### Final Checklist Before Delivering Analysis

- [ ] **Report saved with proper filename**: `google_ads_report_{account_name}_{YYYY-MM-DD}.md`
- [ ] Top of report has Google Ads Account Name and Account Number
- [ ] Date ranges clearly stated and accurate
- [ ] **All costs converted from micros to dollars correctly** (÷ 1,000,000) - see [derived_metrics_reference.md](derived_metrics_reference.md)
- [ ] **Cost values validated** - typical ranges: $1-$10k daily, no suspiciously high/low values
- [ ] **Impression share metrics included in all tables** (Search IS, Budget Lost IS, Rank Lost IS)
- [ ] **Impression share analysis performed** - budget vs rank issues identified
- [ ] Campaign health status assigned (🟢 🟡 🔴)
- [ ] Issues prioritized (HIGH, MEDIUM, LOW)
- [ ] Specific actions recommended for each issue **based on impression share data**
- [ ] Account-level totals calculated and presented
- [ ] Sample size concerns noted where relevant
- [ ] Next steps with timeline provided
- [ ] Executive summary captures key points
- [ ] Data quality issues flagged if any
- [ ] **Appendix with current/prior period data tables included** (see [Appendix Requirements](appendix_reference.md))

