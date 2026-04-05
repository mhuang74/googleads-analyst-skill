# Investigation Patterns Reference

This reference provides symptom-to-query mappings for dynamic investigation of Google Ads performance issues.

## How to Use This Reference

When significant performance issues are detected in the baseline campaign analysis, use this reference to:

1. **Identify the symptom type** from the pattern tables below
2. **Generate targeted investigation queries** using `mcc-gaql-gen`
3. **Validate generated queries** before execution
4. **Analyze drill-down results** to identify root causes
5. **Generate follow-up queries** if needed for deeper investigation

---

## Symptom: Conversion Rate Decline

**When to Investigate:**
- Conversion rate dropped >15%
- Conversions dropped >20% with stable or increased traffic
- Cost per conversion increased >30%

### Investigation Queries

#### 1. Ad Group Breakdown
**Purpose:** Identify which ad groups are underperforming

**Prompt for mcc-gaql-gen:**
```
Show ad group performance for campaign [CAMPAIGN_NAME] with clicks, conversions, conversion rate, cost for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  ad_group.id,
  ad_group.name,
  metrics.clicks,
  metrics.conversions,
  metrics.cost_micros
FROM ad_group
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
  AND ad_group.status = 'ENABLED'
ORDER BY metrics.cost_micros DESC
```

**Analysis:** Calculate conversion rate per ad group. If variance is high (some >10%, others <2%), issue is isolated to specific ad groups.

#### 2. Device Analysis
**Purpose:** Check if mobile/desktop has different conversion patterns

**Prompt for mcc-gaql-gen:**
```
Show campaign [CAMPAIGN_NAME] performance by device with clicks, conversions, cost for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  segments.device,
  metrics.impressions,
  metrics.clicks,
  metrics.conversions,
  metrics.cost_micros
FROM campaign
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
ORDER BY segments.device
```

**Analysis:** Compare conversion rates across MOBILE, DESKTOP, TABLET. Mobile-only decline often indicates mobile landing page issues.

#### 3. Daily Trend Analysis
**Purpose:** Identify when decline started (sudden vs gradual)

**Prompt for mcc-gaql-gen:**
```
Show daily conversions and conversion rate for campaign [CAMPAIGN_NAME] for last 30 days ordered by date
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  segments.date,
  metrics.clicks,
  metrics.conversions,
  metrics.cost_micros
FROM campaign
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING LAST_30_DAYS
ORDER BY segments.date ASC
```

**Analysis:** 
- Gradual decline over weeks → ad fatigue or market shift
- Sudden drop on specific date → check change_event for that date
- Weekend vs weekday pattern → seasonal/timing issue

#### 4. Geographic Performance
**Purpose:** Check if specific locations underperform

**Prompt for mcc-gaql-gen:**
```
Show geographic performance for campaign [CAMPAIGN_NAME] with clicks, conversions, cost by location for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  geographic_view.country_criterion_id,
  metrics.clicks,
  metrics.conversions,
  metrics.cost_micros
FROM geographic_view
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
ORDER BY metrics.cost_micros DESC
```

**Analysis:** Identify locations with high spend but zero or low conversions.

### Expected Patterns

| Pattern | Root Cause | Recommended Action |
|---------|------------|-------------------|
| Gradual decline across all ad groups | Ad fatigue or market shift | Refresh creative, test new messaging |
| Sudden decline in specific ad groups | Targeting or landing page issue | Review ad group settings, check landing pages |
| Mobile-only decline | Mobile experience problem | Audit mobile site speed, UX |
| Decline starting on specific date | User change or external event | Correlate with change_event data |
| Geographic concentration | Location targeting too broad | Exclude poor locations, focus budget |

---

## Symptom: Cost Increase Without Conversion Improvement

**When to Investigate:**
- Cost increased >20% with conversions flat or declining
- CPC increased >30%
- ROAS declined >15%

### Investigation Queries

#### 1. Check for Targeting Changes
**Purpose:** Identify if user changes caused the cost increase

**Prompt for mcc-gaql-gen:**
```
Show change events for campaign [CAMPAIGN_NAME] in last 30 days including change type and changed fields
```

**Expected Generated Query:**
```sql
SELECT 
  change_event.change_date_time,
  change_event.change_resource_type,
  change_event.change_resource_name,
  change_event.user_email,
  change_event.old_resource,
  change_event.new_resource
FROM change_event
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND change_event.change_date_time >= '[30_DAYS_AGO]'
ORDER BY change_event.change_date_time DESC
```

**Analysis:** Look for budget increases, bid changes, targeting expansions, new keywords added.

#### 2. Ad Group Cost Efficiency
**Purpose:** Identify which ad groups are driving cost increase

**Prompt for mcc-gaql-gen:**
```
Show ad groups in campaign [CAMPAIGN_NAME] with cost, conversions, average CPC ordered by cost descending for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  ad_group.name,
  metrics.cost_micros,
  metrics.clicks,
  metrics.conversions,
  metrics.average_cpc
FROM ad_group
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
  AND ad_group.status = 'ENABLED'
ORDER BY metrics.cost_micros DESC
```

**Analysis:** Calculate CPA per ad group. High spend + low conversions = inefficient ad group.

#### 3. Daily CPC Trend
**Purpose:** Understand if CPC inflation is gradual or sudden

**Prompt for mcc-gaql-gen:**
```
Show daily average CPC and cost for campaign [CAMPAIGN_NAME] for last 30 days
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  segments.date,
  metrics.average_cpc,
  metrics.cost_micros,
  metrics.clicks
FROM campaign
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING LAST_30_DAYS
ORDER BY segments.date ASC
```

**Analysis:**
- Gradual CPC increase → competitive pressure
- Sudden CPC spike → bid strategy change or auction dynamics shift

#### 4. Keyword Performance (for Search campaigns)
**Purpose:** Identify expensive keywords with poor conversion rates

**Prompt for mcc-gaql-gen:**
```
Show top 20 keywords by cost for campaign [CAMPAIGN_NAME] with clicks, conversions, average CPC for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  ad_group.name,
  ad_group_criterion.keyword.text,
  ad_group_criterion.keyword.match_type,
  metrics.cost_micros,
  metrics.clicks,
  metrics.conversions,
  metrics.average_cpc
FROM keyword_view
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
  AND ad_group_criterion.status = 'ENABLED'
ORDER BY metrics.cost_micros DESC
LIMIT 20
```

**Analysis:** High cost + zero conversions = negative keyword candidate.

### Expected Patterns

| Pattern | Root Cause | Recommended Action |
|---------|------------|-------------------|
| New ad group created with high spend, no conversions | User launched untested targeting | Pause or adjust new ad group |
| Budget increase in change_event + cost increase | Expected result of budget change | Verify if conversion volume justifies spend |
| CPC inflation across all keywords | Increased competition | Evaluate bid strategy, consider Quality Score improvements |
| Specific keywords driving cost with no conversions | Poor keyword selection | Add as negative keywords |
| Broad match expansion | Targeting too wide | Review search term report, add negatives |

---

## Symptom: Impression Share Loss

**When to Investigate:**
- Search impression share dropped >15%
- Budget-lost IS >20% (budget constraint)
- Rank-lost IS >50% (ad rank issues)

### Investigation Queries

#### 1. Hourly Performance (Budget Constraint)
**Purpose:** See when budget depletes during the day

**Prompt for mcc-gaql-gen:**
```
Show hourly performance for campaign [CAMPAIGN_NAME] with impressions, cost, budget lost impression share for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  segments.hour,
  metrics.impressions,
  metrics.cost_micros,
  metrics.search_budget_lost_impression_share
FROM campaign
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
ORDER BY segments.hour
```

**Analysis:** If budget-lost IS is high and cost drops to zero before 5pm, budget depletes early.

#### 2. Daily Impression Share Trend
**Purpose:** Understand if loss is recent or ongoing

**Prompt for mcc-gaql-gen:**
```
Show daily impression share metrics for campaign [CAMPAIGN_NAME] for last 30 days
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  segments.date,
  metrics.search_impression_share,
  metrics.search_budget_lost_impression_share,
  metrics.search_rank_lost_impression_share,
  metrics.impressions
FROM campaign
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING LAST_30_DAYS
ORDER BY segments.date ASC
```

**Analysis:** 
- Sudden drop on specific date → check change_event
- Gradual decline → competitive pressure or quality degradation

#### 3. Quality Score Analysis (for Ad Rank Issues)
**Purpose:** Identify low quality score keywords driving rank loss

**Prompt for mcc-gaql-gen:**
```
Show keywords with quality score for campaign [CAMPAIGN_NAME] ordered by impressions
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  ad_group.name,
  ad_group_criterion.keyword.text,
  ad_group_criterion.quality_info.quality_score,
  ad_group_criterion.quality_info.creative_quality_score,
  ad_group_criterion.quality_info.post_click_quality_score,
  ad_group_criterion.quality_info.search_predicted_ctr,
  metrics.impressions
FROM keyword_view
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING LAST_7_DAYS
  AND ad_group_criterion.status = 'ENABLED'
ORDER BY metrics.impressions DESC
LIMIT 50
```

**Analysis:** Quality score <5 on high-impression keywords = ad rank problem.

### Expected Patterns

| Pattern | Root Cause | Recommended Action |
|---------|------------|-------------------|
| High budget-lost IS + budget depletes before evening | Insufficient budget | Increase daily budget by 30-50% |
| High rank-lost IS + low quality scores | Quality Score issues | Improve ad relevance, landing page experience |
| High rank-lost IS + quality scores >7 | Bids too low | Increase bids by 20-30% |
| Recent impression share drop + budget change in change_event | Budget was reduced | Restore or increase budget |
| Gradual IS decline + stable quality scores | Competitive pressure | Evaluate bid competitiveness |

---

## Symptom: CTR Decline

**When to Investigate:**
- CTR dropped >15% with stable impressions
- Position worsened significantly
- Ad strength declined (PMax/RSA)

### Investigation Queries

#### 1. Ad Performance Breakdown
**Purpose:** Identify which ads have declining CTR

**Prompt for mcc-gaql-gen:**
```
Show ad performance for campaign [CAMPAIGN_NAME] with impressions, clicks, CTR for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  ad_group.name,
  ad_group_ad.ad.id,
  ad_group_ad.ad.type,
  ad_group_ad.ad.responsive_search_ad.headlines,
  metrics.impressions,
  metrics.clicks,
  metrics.ctr
FROM ad_group_ad
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
  AND ad_group_ad.status = 'ENABLED'
ORDER BY metrics.impressions DESC
```

**Analysis:** Check ad age, CTR trends. Ads running >60 days with declining CTR = ad fatigue.

#### 2. Search Term Report
**Purpose:** See if impressions expanded to low-intent queries

**Prompt for mcc-gaql-gen:**
```
Show search terms for campaign [CAMPAIGN_NAME] with impressions, clicks, CTR for [DATE_RANGE] ordered by impressions
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  segments.search_term,
  metrics.impressions,
  metrics.clicks,
  metrics.ctr,
  metrics.conversions
FROM search_term_view
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
ORDER BY metrics.impressions DESC
LIMIT 100
```

**Analysis:** High impressions on irrelevant terms = add negative keywords.

#### 3. Position and Impression Share Metrics
**Purpose:** Check if lower position is causing CTR decline

**Prompt for mcc-gaql-gen:**
```
Show average position and top impression metrics for campaign [CAMPAIGN_NAME] for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  segments.date,
  metrics.average_page_location,
  metrics.search_top_impression_share,
  metrics.search_absolute_top_impression_share,
  metrics.ctr
FROM campaign
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
ORDER BY segments.date ASC
```

**Analysis:** Position dropped from top to side = lower CTR expected.

### Expected Patterns

| Pattern | Root Cause | Recommended Action |
|---------|------------|-------------------|
| Same ads running >60 days + declining CTR | Ad fatigue | Refresh creative, test new messaging |
| Search term expansion to irrelevant queries | Broad match too aggressive | Add negative keywords, tighten targeting |
| Position decline + stable quality scores | Bid competitiveness decreased | Increase bids |
| Sudden CTR drop across all ads | Competitive landscape change | Audit competitor ads, refresh creative |

---

## Symptom: Zero or Low Conversions with Spend

**When to Investigate:**
- Zero conversions with >$100 spend
- Conversion tracking appears broken
- Conversion rate <0.5% for typically converting campaigns

### Investigation Queries

#### 1. Conversion Action Status
**Purpose:** Verify conversion tracking is active

**Prompt for mcc-gaql-gen:**
```
Show conversion actions with status and recent conversions for customer
```

**Expected Generated Query:**
```sql
SELECT 
  conversion_action.id,
  conversion_action.name,
  conversion_action.status,
  conversion_action.type,
  conversion_action.category,
  metrics.all_conversions,
  metrics.conversions
FROM conversion_action
WHERE segments.date DURING LAST_7_DAYS
ORDER BY conversion_action.name
```

**Analysis:** Status should be "ENABLED". Zero conversions across all actions = tracking issue.

#### 2. Landing Page Performance
**Purpose:** Identify if specific landing pages fail to convert

**Prompt for mcc-gaql-gen:**
```
Show landing page performance for campaign [CAMPAIGN_NAME] with clicks, conversions for [DATE_RANGE]
```

**Expected Generated Query:**
```sql
SELECT 
  campaign.name,
  landing_page_view.unexpanded_final_url,
  metrics.clicks,
  metrics.conversions,
  metrics.cost_micros
FROM landing_page_view
WHERE campaign.name = '[CAMPAIGN_NAME]'
  AND segments.date DURING [DATE_RANGE]
ORDER BY metrics.clicks DESC
```

**Analysis:** High clicks, zero conversions on specific URL = landing page issue.

#### 3. Click-to-Conversion Time Lag
**Purpose:** Check if conversions are delayed

**Prompt for mcc-gaql-gen:**
```
Show click to conversion time lag data for campaign [CAMPAIGN_NAME] for [DATE_RANGE]
```

**Note:** This query requires the `click_view` resource with conversion lag segments. May not be available via simple generation.

### Expected Patterns

| Pattern | Root Cause | Recommended Action |
|---------|------------|-------------------|
| All campaigns zero conversions suddenly | Conversion tracking broken | Check tag implementation, Google Tag Manager |
| Specific landing page zero conversions | Landing page issue | Audit page for errors, load time, mobile issues |
| Conversions appear 7-14 days later | Long consideration cycle | Adjust attribution window, be patient |
| New campaign zero conversions, old campaigns converting | Poor targeting/offer | Review audience targeting, ad messaging |

---

## Validation and Fallback Strategy

For each generated query:

```bash
# Step 1: Generate query
QUERY=$(mcc-gaql-gen generate "<investigation prompt>" --use-query-cookbook)

# Step 2: Validate before execution
mcc-gaql --profile <PROFILE> --validate "$QUERY"

# Step 3a: If valid, execute
if [ $? -eq 0 ]; then
  mcc-gaql --profile <PROFILE> --format csv -o /tmp/investigation_result.csv "$QUERY"
else
  # Step 3b: If invalid, diagnose and retry
  echo "Query validation failed. Checking available fields..."
  
  # Extract resource name from error
  RESOURCE=$(echo "$QUERY" | grep -oP "FROM \K\w+")
  
  # Show available fields
  mcc-gaql --show-fields "$RESOURCE"
  
  # Refine prompt or use manual GAQL pattern from mcc_gaql_reference.md
fi
```

### Common Generation Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Field incompatibility | Requested metric not selectable with resource | Use `mcc-gaql --show-fields <resource>` to find compatible fields |
| Wrong resource selected | LLM chose incorrect resource | Explicitly specify resource in prompt: "from keyword_view" |
| Missing date filter | Prompt didn't emphasize time period | Always include "for [specific dates]" in prompt |
| Invalid segment combination | Segments not compatible | Check GoogleAdsFieldService for segment compatibility |

---

## Multi-Level Drill-Down Strategy

For complex issues, use this drill-down sequence:

```
Campaign Issue Detected
        ↓
1. Ad Group Breakdown → Identify underperforming ad groups
        ↓
2a. Keyword Analysis (if Search) → Find expensive/poor keywords
2b. Asset Analysis (if PMax) → Check asset performance
2c. Ad Analysis (if Display/Video) → Review creative performance
        ↓
3. Segment Analysis → Device, location, time, audience
        ↓
4. Change Event Correlation → Link to user changes
        ↓
5. Competitive Analysis → Auction insights if available
```

**Depth Selection:**
- **Quick investigation** (High budget-lost IS): 1-2 levels
- **Complex issue** (conversion rate drop, no clear cause): 3-4 levels
- **Critical issue** (>$500 wasted spend): All 5 levels

---

## Query Generation Best Practices

1. **Be specific with campaign names**: Use exact campaign name in quotes
2. **Always include date ranges**: Specify exact dates or use LAST_N_DAYS
3. **Request metrics explicitly**: Don't assume, list the metrics you need
4. **Use --use-query-cookbook flag**: Improves generation quality with examples
5. **Validate before executing**: Always run --validate first
6. **Start broad, then narrow**: Campaign → Ad Group → Keyword/Ad
7. **Include identity fields**: Always include campaign.name, ad_group.name for context

**Good Prompt Example:**
```
Show ad group performance for campaign "Brand Search - Desktop" with impressions, clicks, 
conversions, cost, average CPC for dates 2025-10-12 to 2025-10-18 ordered by cost descending
```

**Poor Prompt Example:**
```
Get ad group data
```

---

## Iteration Guidelines

After each drill-down query:

1. **Analyze results** - What does this tell us?
2. **Form hypothesis** - What might be the root cause?
3. **Generate follow-up** - What additional data would confirm/refute?
4. **Decide depth** - Is this enough, or drill deeper?
5. **Correlate findings** - How does this fit with baseline analysis?

**Stop drilling when:**
- Root cause is clearly identified with >80% confidence
- No additional actionable insights available at deeper levels
- Sample size becomes too small for reliable conclusions (<30 clicks)
- Sufficient evidence gathered to make specific recommendations

**Continue drilling when:**
- Multiple possible causes exist
- Issue is critical (>$500 spend impact)
- Initial findings are contradictory or unclear
- User specifically requested deep investigation
