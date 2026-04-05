# Change Event Correlation Reference

This reference provides guidance on correlating performance anomalies with user-initiated changes in Google Ads accounts using the `change_event` and `change_status` resources.

## Overview

**Purpose:** Distinguish between performance changes caused by:
- **User actions** (budget changes, bid adjustments, targeting modifications)
- **Market dynamics** (competition, seasonality, external events)
- **Algorithm changes** (Google Ads system updates)

**Why It Matters:** Recommendations differ significantly based on cause:
- User change → Evaluate if change should be reversed or adjusted
- Market change → Adapt strategy to new conditions
- Algorithm change → Monitor and optimize within new constraints

---

## Querying Change Events

**⚠️ API Limitation:** Change events are only available for the last 30 days via Google Ads API. Only query change events if the performance issues you're investigating occurred within this 30-day window.

### ⚠️ CRITICAL: change_event Query Requirements

The `change_event` resource has **strict API requirements** that differ from other resources:

| Requirement | Rule | Example |
|-------------|------|---------|
| **LIMIT** | MUST include `LIMIT` clause, max 10,000 | `LIMIT 5000` |
| **Start Date** | MUST be within last 30 days | `>= "2026-03-06"` (if today is 2026-04-05) |
| **End Date** | MUST be specified (no open-ended ranges) | `<= "2026-04-05"` |
| **Both Dates** | MUST specify both start AND end | Cannot omit either |

**Correct Template:**
```sql
SELECT ...
FROM change_event
WHERE change_event.change_date_time >= "[START_DATE within 30 days]"
  AND change_event.change_date_time <= "[END_DATE up to today]"
ORDER BY change_event.change_date_time DESC
LIMIT 5000
```

**Common Mistakes:**
- ❌ Missing LIMIT → "Change event requests must specify a LIMIT"
- ❌ Start date >30 days ago → "The requested start date is too old"
- ❌ Missing end date → "filtering with an infinite range"

### Basic Change Event Query

```sql
SELECT 
  change_event.change_date_time,
  change_event.change_resource_type,
  change_event.change_resource_name,
  change_event.user_email,
  change_event.old_resource,
  change_event.new_resource,
  campaign.id,
  campaign.name
FROM change_event
WHERE campaign.id = [CAMPAIGN_ID]
  AND change_event.change_date_time >= '[START_DATE]'
  AND change_event.change_date_time <= '[END_DATE]'
ORDER BY change_event.change_date_time DESC
LIMIT 5000
```

### Using mcc-gaql-gen to Generate Change Queries

**Prompt Pattern:**
```
Show change events for campaign [CAMPAIGN_NAME] in last [N] days including change type, user, and changed fields
```

**Example:**
```bash
# Generate the query
QUERY=$(mcc-gaql-gen generate "Show change events for campaign Brand Search in last 30 days with change type and user email" --use-query-cookbook)

# Validate using pipe (mcc-gaql-gen does NOT have --validate flag)
echo "$QUERY" | mcc-gaql --profile <PROFILE> --validate

# Alternative: Validate directly
mcc-gaql --profile <PROFILE> --validate "$QUERY"

# Execute if validation passes
if [ $? -eq 0 ]; then
  mcc-gaql --profile <PROFILE> --format json -o output.json "$QUERY"
fi
```

**IMPORTANT:** `mcc-gaql-gen generate` does NOT accept `--validate` flag. Always validate the generated query separately using `mcc-gaql --validate`.

### Change Event Query by Resource Type

To focus on specific change types:

```sql
-- Budget changes only
SELECT 
  change_event.change_date_time,
  change_event.user_email,
  change_event.old_resource,
  change_event.new_resource,
  campaign.name
FROM change_event
WHERE campaign.id = [CAMPAIGN_ID]
  AND change_event.change_resource_type = 'CAMPAIGN_BUDGET'
  AND change_event.change_date_time >= '[START_DATE]'
  AND change_event.change_date_time <= '[END_DATE]'
ORDER BY change_event.change_date_time DESC
LIMIT 5000

-- Bid strategy changes
WHERE change_event.change_resource_type = 'BIDDING_STRATEGY'
  AND change_event.change_date_time >= '[START_DATE]'
  AND change_event.change_date_time <= '[END_DATE]'
LIMIT 5000

-- Ad changes
WHERE change_event.change_resource_type = 'AD'
  AND change_event.change_date_time >= '[START_DATE]'
  AND change_event.change_date_time <= '[END_DATE]'
LIMIT 5000

-- Keyword changes
WHERE change_event.change_resource_type = 'AD_GROUP_CRITERION'
  AND change_event.change_date_time >= '[START_DATE]'
  AND change_event.change_date_time <= '[END_DATE]'
LIMIT 5000
```

---

## Change Event Resource Types

| Resource Type | What It Captures | Common Performance Impact |
|---------------|------------------|--------------------------|
| `CAMPAIGN` | Campaign settings (status, name, etc.) | Status change → traffic stops/starts |
| `CAMPAIGN_BUDGET` | Budget changes | Budget increase → more impressions; decrease → lost IS |
| `BIDDING_STRATEGY` | Bid strategy modifications | CPC changes, conversion volume shifts |
| `AD_GROUP` | Ad group settings | Status, targeting changes |
| `AD_GROUP_CRITERION` | Keyword/audience changes | Targeting expansion/contraction |
| `AD` | Ad creative changes | CTR impact, messaging changes |
| `CAMPAIGN_CRITERION` | Campaign-level targeting (geo, device) | Impression volume, audience quality shifts |
| `ASSET` | Asset changes (for PMax, RSA) | Creative performance impact |
| `FEED` | Feed modifications | Product/location data changes |

---

## Correlation Analysis Framework

### Step 1: Identify Performance Anomaly Start Date

For each detected performance issue, determine when it started:

```python
# Example: Conversion rate dropped
# Method 1: Daily trend analysis
daily_data = query_daily_performance(campaign, last_30_days)
anomaly_start_date = identify_change_point(daily_data, metric='conversion_rate')

# Method 2: Simple comparison
if current_period_conv_rate < previous_period_conv_rate * 0.85:  # 15% drop
    anomaly_start_date = current_period_start_date
```

### Step 2: Query Changes Within Detection Window

Query change_event for 7 days **before** anomaly start (changes may take time to impact performance):

```sql
SELECT 
  change_event.change_date_time,
  change_event.change_resource_type,
  change_event.user_email,
  change_event.old_resource,
  change_event.new_resource,
  campaign.name
FROM change_event
WHERE campaign.id = [CAMPAIGN_ID]
  AND change_event.change_date_time >= '[ANOMALY_START - 7 days]'
  AND change_event.change_date_time <= '[ANOMALY_START + 2 days]'
ORDER BY change_event.change_date_time DESC
LIMIT 5000
```

**Detection Window Rationale:**
- **-7 to -1 days**: Changes often need 1-7 days to fully impact performance (learning period, gradual rollout)
- **0 to +2 days**: Account for reporting lag and same-day changes

### Step 3: Match Change Types to Symptoms

Use this mapping to determine relevance:

| Symptom | Likely Related Change Types | What to Check |
|---------|----------------------------|---------------|
| **Cost spike** | CAMPAIGN_BUDGET (increase), BIDDING_STRATEGY (more aggressive), AD_GROUP_CRITERION (new keywords) | Budget amount, bid adjustments, keyword additions |
| **Conversion drop** | AD (creative change), LANDING_PAGE, AD_GROUP_CRITERION (targeting change) | Ad changes, audience exclusions, keyword removals |
| **Impression drop** | CAMPAIGN_BUDGET (decrease), CAMPAIGN_CRITERION (geo exclusion), CAMPAIGN (status) | Budget cuts, location exclusions, pauses |
| **CTR decline** | AD (creative change), AD_GROUP_CRITERION (impression expansion) | New ads with poor CTR, broad match expansion |
| **CPC increase** | BIDDING_STRATEGY (bid increase), competitive pressure (no change) | Bid strategy modifications, check if change_event empty |
| **Impression Share loss** | CAMPAIGN_BUDGET (decrease), BIDDING_STRATEGY (less aggressive) | Budget reductions, target CPA/ROAS made more restrictive |

### Step 4: Calculate Correlation Score

For each change event found within the detection window:

```python
def calculate_correlation_score(change_event, anomaly):
    score = 0
    
    # Factor 1: Temporal proximity (30 points max)
    days_apart = abs((anomaly.start_date - change_event.date).days)
    if days_apart == 0:
        score += 30
    elif days_apart <= 2:
        score += 25
    elif days_apart <= 5:
        score += 15
    elif days_apart <= 7:
        score += 10
    
    # Factor 2: Change-symptom match (30 points max)
    if is_matching_change_type(change_event.type, anomaly.symptom):
        score += 30
    elif is_related_change_type(change_event.type, anomaly.symptom):
        score += 15
    
    # Factor 3: Magnitude alignment (20 points max)
    # E.g., 50% budget increase aligns with 45% cost increase
    change_magnitude = calculate_change_magnitude(change_event)
    anomaly_magnitude = abs(anomaly.pct_change)
    if abs(change_magnitude - anomaly_magnitude) < 10:
        score += 20
    elif abs(change_magnitude - anomaly_magnitude) < 25:
        score += 10
    
    # Factor 4: Exclusivity (20 points max)
    # Single campaign affected → likely user change
    # All campaigns affected → likely market/algorithm change
    if anomaly.affected_campaigns == 1:
        score += 20
    elif anomaly.affected_campaigns <= 3:
        score += 10
    
    return score

# Interpretation:
# 80-100: Very likely cause (>90% confidence)
# 60-79: Probable cause (75-90% confidence)
# 40-59: Possible cause (50-75% confidence)
# <40: Unlikely or coincidental
```

---

## Parsing Change Event Data

### Extracting Budget Changes

```python
import json

def parse_budget_change(change_event):
    """Extract old and new budget values from change_event"""
    old_resource = json.loads(change_event.old_resource) if change_event.old_resource else {}
    new_resource = json.loads(change_event.new_resource) if change_event.new_resource else {}
    
    old_budget = old_resource.get('campaignBudget', {}).get('amountMicros', 0) / 1_000_000
    new_budget = new_resource.get('campaignBudget', {}).get('amountMicros', 0) / 1_000_000
    
    return {
        'old_budget': old_budget,
        'new_budget': new_budget,
        'change_pct': ((new_budget - old_budget) / old_budget * 100) if old_budget > 0 else 0,
        'change_amount': new_budget - old_budget
    }

# Example output:
# {'old_budget': 50.0, 'new_budget': 100.0, 'change_pct': 100.0, 'change_amount': 50.0}
```

### Extracting Bid Strategy Changes

```python
def parse_bid_strategy_change(change_event):
    """Extract bid strategy modifications"""
    old_resource = json.loads(change_event.old_resource) if change_event.old_resource else {}
    new_resource = json.loads(change_event.new_resource) if change_event.new_resource else {}
    
    old_type = old_resource.get('biddingStrategyType', 'UNKNOWN')
    new_type = new_resource.get('biddingStrategyType', 'UNKNOWN')
    
    # For Target CPA
    old_target_cpa = old_resource.get('targetCpa', {}).get('targetCpaMicros', 0) / 1_000_000
    new_target_cpa = new_resource.get('targetCpa', {}).get('targetCpaMicros', 0) / 1_000_000
    
    # For Target ROAS
    old_target_roas = old_resource.get('targetRoas', {}).get('targetRoas', 0)
    new_target_roas = new_resource.get('targetRoas', {}).get('targetRoas', 0)
    
    return {
        'old_type': old_type,
        'new_type': new_type,
        'old_target_cpa': old_target_cpa,
        'new_target_cpa': new_target_cpa,
        'old_target_roas': old_target_roas,
        'new_target_roas': new_target_roas,
        'strategy_changed': old_type != new_type
    }
```

### Extracting Keyword Changes

```python
def parse_keyword_change(change_event):
    """Extract keyword additions/removals/modifications"""
    old_resource = json.loads(change_event.old_resource) if change_event.old_resource else {}
    new_resource = json.loads(change_event.new_resource) if change_event.new_resource else {}
    
    # Check if keyword was added, removed, or modified
    if not old_resource and new_resource:
        change_type = 'ADDED'
        keyword_text = new_resource.get('keyword', {}).get('text', '')
        match_type = new_resource.get('keyword', {}).get('matchType', '')
    elif old_resource and not new_resource:
        change_type = 'REMOVED'
        keyword_text = old_resource.get('keyword', {}).get('text', '')
        match_type = old_resource.get('keyword', {}).get('matchType', '')
    else:
        change_type = 'MODIFIED'
        keyword_text = new_resource.get('keyword', {}).get('text', '')
        match_type = new_resource.get('keyword', {}).get('matchType', '')
        old_status = old_resource.get('status', '')
        new_status = new_resource.get('status', '')
    
    return {
        'change_type': change_type,
        'keyword_text': keyword_text,
        'match_type': match_type
    }
```

---

## Common Change-Symptom Correlation Patterns

### Pattern 1: Budget Increase → Cost Increase

**Change Event:**
```json
{
  "change_date_time": "2025-10-01T10:23:15Z",
  "change_resource_type": "CAMPAIGN_BUDGET",
  "old_budget": "$50/day",
  "new_budget": "$100/day"
}
```

**Symptom:**
- Cost increased 95% (expected: ~100%)
- Impressions increased 80%
- Conversions increased 85%

**Correlation Score:** 95/100 (Very High)

**Analysis:**
```
Root Cause: User Change (95% confidence)
- Budget doubled on 2025-10-01
- Cost increase (95%) aligns perfectly with budget increase (100%)
- Performance metrics maintained efficiency (conversion rate stable)
- This is expected behavior, not an issue

Recommendation: 
- No action needed - budget increase working as intended
- Continue monitoring to ensure ROAS remains above target
```

### Pattern 2: Budget Decrease → Impression Share Loss

**Change Event:**
```json
{
  "change_date_time": "2025-09-28T14:10:05Z",
  "change_resource_type": "CAMPAIGN_BUDGET",
  "old_budget": "$200/day",
  "new_budget": "$75/day"
}
```

**Symptom:**
- Impressions dropped 60%
- Budget-lost impression share increased from 5% to 58%
- Cost decreased 62%

**Correlation Score:** 92/100 (Very High)

**Analysis:**
```
Root Cause: User Change (92% confidence)
- Budget reduced by 62.5% on 2025-09-28
- Impression drop (60%) and cost decrease (62%) align with budget cut
- Budget-lost IS jumped to 58% (was 5%) - clear budget constraint

Recommendation:
- Reverse budget reduction if campaign was profitable
- If budget cut was intentional, accept the reduced impression volume
- Alternatively, pause less important campaigns to allocate budget here
```

### Pattern 3: New Keywords Added → Cost Spike with Poor Performance

**Change Event:**
```json
{
  "change_date_time": "2025-10-05T09:15:30Z",
  "change_resource_type": "AD_GROUP_CRITERION",
  "change_type": "ADDED",
  "keywords_added": 15,
  "match_type": "BROAD"
}
```

**Symptom:**
- Cost increased 45%
- Conversions flat (+2%)
- CTR declined 18%
- CPA increased 42%

**Correlation Score:** 85/100 (High)

**Analysis:**
```
Root Cause: User Change - Targeting Expansion (85% confidence)
- 15 broad match keywords added on 2025-10-05
- Cost spike started same day
- Drill-down shows new keywords driving 40% of cost with 0.3% conversion rate
- Existing keywords maintained 2.8% conversion rate

Recommendation:
- Review search term report for new keywords
- Pause or bid down poor performers
- Add negative keywords for irrelevant traffic
- Consider using phrase/exact match instead of broad
```

### Pattern 4: No Change Event → Competitive Pressure

**Change Event:**
```
No change events found in detection window
```

**Symptom:**
- CPC increased 35%
- Average position worsened from 1.8 to 2.4
- Impression share stable
- Cost increased 30% with impressions flat

**Correlation Score:** N/A (No user change)

**Analysis:**
```
Root Cause: Market Change - Competitive Pressure (78% confidence)
- No change events detected in account
- CPC inflation across multiple campaigns (not isolated)
- Position decline indicates competitors bidding more aggressively
- Alternative hypothesis: Seasonal demand increase (25% confidence)

Recommendation:
- Review auction insights to identify competitive changes
- Evaluate if Quality Score improvements can offset CPC increase
- Consider if higher CPCs are acceptable given conversion value
- Check year-over-year data for seasonal patterns
```

### Pattern 5: Bid Strategy Change → Conversion Volume Drop

**Change Event:**
```json
{
  "change_date_time": "2025-10-03T16:42:10Z",
  "change_resource_type": "BIDDING_STRATEGY",
  "old_type": "TARGET_CPA",
  "new_type": "TARGET_CPA",
  "old_target_cpa": "$45",
  "new_target_cpa": "$30"
}
```

**Symptom:**
- Conversions dropped 55%
- CPC decreased 32%
- Impressions dropped 45%
- Actual CPA: $28 (achieved target, but lost volume)

**Correlation Score:** 90/100 (Very High)

**Analysis:**
```
Root Cause: User Change - Bid Strategy Constraint (90% confidence)
- Target CPA reduced from $45 to $30 on 2025-10-03
- Google algorithm reduced bids to achieve new CPA target
- Achieved $28 CPA but sacrificed 55% conversion volume
- Trade-off: Efficiency improved, but volume decreased significantly

Recommendation:
- Evaluate if $30 CPA target is too aggressive
- Consider relaxing target to $35-$40 to recapture volume
- Calculate total profit: (conversions × value) - (conversions × CPA)
- Compare: High volume/moderate efficiency vs Low volume/high efficiency
```

---

## Cross-Campaign Pattern Detection

When the same symptom appears across multiple campaigns:

### All Campaigns Affected → Likely Market or Algorithm Change

```python
def detect_account_wide_pattern(campaigns_with_issues):
    """Determine if issue is campaign-specific or account-wide"""
    
    # Count campaigns with similar symptom
    affected_count = len(campaigns_with_issues)
    total_campaigns = get_total_active_campaigns()
    affected_pct = (affected_count / total_campaigns) * 100
    
    # Query change events for all affected campaigns
    all_change_events = []
    for campaign in campaigns_with_issues:
        events = query_change_events(campaign, detection_window)
        all_change_events.extend(events)
    
    # Interpretation
    if affected_pct > 70 and len(all_change_events) == 0:
        return {
            'pattern': 'ACCOUNT_WIDE',
            'likely_cause': 'MARKET_OR_ALGORITHM_CHANGE',
            'confidence': 'HIGH',
            'reasoning': f'{affected_pct}% of campaigns affected with no user changes detected'
        }
    elif affected_pct > 70 and len(all_change_events) > 0:
        # Check if single change event (e.g., account-level setting)
        unique_change_times = set([e.change_date_time for e in all_change_events])
        if len(unique_change_times) == 1:
            return {
                'pattern': 'ACCOUNT_WIDE',
                'likely_cause': 'USER_CHANGE',
                'confidence': 'HIGH',
                'reasoning': 'Account-level change made at single point in time'
            }
    else:
        return {
            'pattern': 'CAMPAIGN_SPECIFIC',
            'likely_cause': 'USER_CHANGE_OR_ISOLATED_ISSUE',
            'confidence': 'MEDIUM',
            'reasoning': f'Only {affected_pct}% of campaigns affected'
        }
```

### Examples

**Account-Wide Market Change:**
- 8 out of 10 campaigns have CPC increase 20-30%
- No change events found for any campaign
- Started on same date across all campaigns
- → **Competitive pressure or seasonal demand spike**

**Account-Level Setting Change:**
- All campaigns lost conversions simultaneously
- Change event shows conversion action was paused
- → **Conversion tracking disabled**

**Campaign-Specific User Change:**
- 1 campaign has cost spike, others stable
- Change event shows budget increase for that campaign only
- → **Intentional budget reallocation**

---

## Seasonal vs User Change Differentiation

To distinguish seasonal patterns from user changes:

### Year-Over-Year Comparison

```sql
-- Current period
SELECT 
  segments.week,
  SUM(metrics.cost_micros) as cost,
  SUM(metrics.conversions) as conversions
FROM campaign
WHERE segments.year = 2025
  AND segments.month IN (9, 10)
GROUP BY segments.week

-- Same period last year
WHERE segments.year = 2024
  AND segments.month IN (9, 10)
```

**Analysis:**
```python
if current_year_pattern_similar_to_last_year(correlation > 0.8):
    likely_cause = "SEASONAL_PATTERN"
    confidence = "HIGH"
    evidence = "Performance change aligns with year-over-year seasonal trend"
else:
    likely_cause = "USER_CHANGE_OR_MARKET_SHIFT"
    confidence = "MEDIUM"
    evidence = "Pattern differs from historical seasonal behavior"
```

---

## Output Format: Root Cause Analysis

When presenting change correlation analysis, use this format:

```markdown
## Root Cause Analysis: [Campaign Name]

**Symptom:** [Description of performance issue]
- [Metric 1]: [Changed from X to Y] ([pct change]%)
- [Metric 2]: [Changed from X to Y] ([pct change]%)
- Started: [Date or date range]

**Change Events Found:** [Number] events in detection window

### Most Likely Cause: [Cause Type] ([Confidence]%)

**Change Event Details:**
- Date/Time: [timestamp]
- Change Type: [RESOURCE_TYPE]
- User: [email]
- Change: [Specific change description]
  - Before: [old value]
  - After: [new value]

**Correlation Evidence:**
- Temporal proximity: [X days between change and symptom]
- Change magnitude: [Y]% aligns with [Z]% performance impact
- Pattern scope: [Single campaign / Multiple campaigns]
- Change-symptom match: [Strong/Moderate/Weak]

**Correlation Score:** [Score]/100

### Alternative Hypotheses:

**[Alternative Cause 1]:** ([Confidence]%)
- [Evidence supporting this hypothesis]
- [Why it's less likely than primary cause]

**[Alternative Cause 2]:** ([Confidence]%)
- [Evidence supporting this hypothesis]

---

## Recommendation:

[Specific action based on root cause, with reference to action_templates_reference.md]
```

### Example Output

```markdown
## Root Cause Analysis: Brand - Search

**Symptom:** Cost increased significantly without conversion improvement
- Cost: Increased from $1,250 to $2,430 (+94%)
- Conversions: Increased from 42 to 45 (+7%)
- CPA: Increased from $30 to $54 (+80%)
- Started: 2025-10-01

**Change Events Found:** 1 event in detection window

### Most Likely Cause: User Change - Budget Increase (95% confidence)

**Change Event Details:**
- Date/Time: 2025-10-01 10:23:15 UTC
- Change Type: CAMPAIGN_BUDGET
- User: john@example.com
- Change: Daily budget increased
  - Before: $50/day
  - After: $100/day (+100%)

**Correlation Evidence:**
- Temporal proximity: 0 days (change and symptom on same date)
- Change magnitude: 100% budget increase aligns with 94% cost increase
- Pattern scope: Single campaign affected (others stable)
- Change-symptom match: Strong (budget → cost direct relationship)

**Correlation Score:** 95/100

### Alternative Hypotheses:

**Competitive Pressure:** (5% confidence)
- No evidence of CPC inflation (CPC stable at ~$2.10)
- Other campaigns not affected
- Unlikely given strong correlation with budget change

---

## Recommendation:

The cost increase is a direct result of the budget increase made on 2025-10-01. However, the efficiency declined significantly:
- ROAS dropped from 5.0x to 2.8x
- CPA increased 80% ($30 → $54)

**Action:** Reduce budget back to $50-60/day range OR investigate why conversion efficiency declined
- If budget increase was intentional: Investigate why conversion rate dropped from 3.2% to 1.8%
- If budget increase was accidental: Reverse to previous $50/day budget
- Monitor search term report and quality score for quality dilution with expanded spend

See action_templates_reference.md → "Budget Increase" for monitoring query.
```

---

## Best Practices

1. **Always query change_event when investigating anomalies** - Don't assume cause without checking
2. **Use 7-day detection window before anomaly** - Changes may take time to impact performance
3. **Calculate correlation scores** - Quantify confidence in cause attribution
4. **Consider alternative hypotheses** - Avoid confirmation bias
5. **Parse change event JSON carefully** - Extract specific old/new values for context
6. **Cross-reference multiple campaigns** - Pattern scope helps determine cause type
7. **Combine with YoY data** - Distinguish seasonal vs user-driven changes
8. **Present evidence clearly** - Show your reasoning for cause attribution
9. **Adjust recommendations based on cause** - User change vs market change require different actions
10. **Include correlation score in confidence rating** - High correlation = high confidence in recommendations
