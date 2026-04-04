# Action Templates Reference

This reference provides standardized templates for formulating specific, data-driven recommendations with calculated impact estimates.

## Template Structure

Each recommendation should follow this structure:

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

---

## Action Template: Budget Increase

### When to Recommend

- Budget-lost impression share > 20%
- Campaign ROAS > target (profitable enough to scale)
- Rank-lost impression share < 30% (quality is acceptable)
- Historical performance shows campaign can convert additional traffic

### Template

```
Action: Increase daily budget for [CAMPAIGN_NAME]
Campaign: [EXACT_CAMPAIGN_NAME]
Current State: $[CURRENT_BUDGET]/day budget, [BUDGET_LOST_IS]% budget-lost impression share
Recommended State: $[NEW_BUDGET]/day
Expected Impact: 
  - Capture ~[IMPRESSION_GAIN]% more impressions
  - Estimated [CONVERSION_ESTIMATE] additional conversions per week
  - Projected additional revenue: $[REVENUE_ESTIMATE]
Evidence:
  - Budget-lost IS: [VALUE]% (missing [VALUE]% of available impressions)
  - Rank-lost IS: [VALUE]% (ad quality is [acceptable/good])
  - Historical ROAS: [VALUE]x (campaign is profitable)
  - Current conversion rate: [VALUE]%
  - Budget exhaustion time: [TIME] daily (missing [prime hours/evening traffic/etc])
Confidence: [High (>85%) / Medium (70-85%) / Low (<70%)]%
Follow-up Query: |
  SELECT segments.date, 
         metrics.search_budget_lost_impression_share, 
         metrics.impressions, 
         metrics.conversions,
         metrics.cost_micros
  FROM campaign 
  WHERE campaign.name = "[CAMPAIGN_NAME]" 
    AND segments.date DURING LAST_7_DAYS
  ORDER BY segments.date DESC
```

### Calculation Formulas

```python
# New budget calculation
new_budget = current_budget / (1 - (budget_lost_is / 100))
# Example: $50 / (1 - 0.45) = $90.91, round to $90 or $95

# Impression gain estimate
impression_gain_pct = budget_lost_is  # percentage
impression_gain_absolute = current_impressions * (budget_lost_is / 100)

# Conversion estimate (weekly)
additional_weekly_clicks = impression_gain_absolute * (current_ctr / 100) * 7
conversion_estimate = additional_weekly_clicks * (current_conv_rate / 100)

# Revenue estimate (if ROAS/conversion value available)
revenue_estimate = conversion_estimate * avg_conversion_value
# OR
revenue_estimate = (new_budget - current_budget) * 7 * current_roas
```

### Example

```
Action: Increase daily budget for Brand - Search
Campaign: Brand - Search
Current State: $50/day budget, 45% budget-lost impression share
Recommended State: $90/day budget
Expected Impact: 
  - Capture ~45% more impressions (estimated 4,500 additional impressions/week)
  - Estimated 12-15 additional conversions per week
  - Projected additional revenue: $720/week (based on historical $50 avg conversion value)
Evidence:
  - Budget-lost IS: 45% (missing nearly half of available impressions)
  - Rank-lost IS: only 8% (ad quality is not the issue)
  - Historical ROAS: 4.2x (campaign is highly profitable)
  - Current conversion rate: 3.2%
  - Budget exhaustion time: 2pm daily (missing evening traffic prime hours)
Confidence: High (92%)
```

---

## Action Template: Pause Underperforming Ad Group

### When to Recommend

- Ad group has >50 clicks with 0 conversions (or conversion rate <0.5%)
- Ad group cost per conversion >3x campaign average
- Ad group has negative or very low ROAS (<1.5x when target is >3x)
- Clear statistical significance (not just small sample noise)

### Template

```
Action: Pause underperforming ad group "[AD_GROUP_NAME]"
Campaign: [CAMPAIGN_NAME]
Current State: 
  - $[COST] spend, [CONVERSIONS] conversions
  - [CONVERSION_RATE]% conversion rate (vs [CAMPAIGN_AVG]% campaign average)
  - [CPA] CPA (vs $[CAMPAIGN_AVG_CPA] campaign average)
Recommended State: Paused
Expected Impact: 
  - Save ~$[WEEKLY_SAVINGS]/week in wasted spend
  - Improve campaign-level ROAS from [CURRENT]x to [PROJECTED]x
  - Reallocate budget to [BETTER_PERFORMING_AD_GROUP]
Evidence:
  - [CLICKS] clicks with [CONVERSIONS] conversions ([sufficient/insufficient] for statistical significance)
  - Keywords targeting [keyword_theme] with low purchase intent
  - [ADDITIONAL_SUPPORTING_DATA]
Confidence: [High/Medium]%
Follow-up Query: |
  SELECT ad_group.name,
         ad_group.status,
         metrics.cost_micros, 
         metrics.conversions
  FROM ad_group
  WHERE campaign.name = "[CAMPAIGN_NAME]"
    AND segments.date DURING LAST_7_DAYS
```

### Calculation Formulas

```python
# Weekly savings estimate
weekly_savings = (cost_in_period / days_in_period) * 7

# Projected ROAS improvement
total_cost_without_ad_group = campaign_total_cost - ad_group_cost
total_revenue_without_ad_group = campaign_total_revenue - ad_group_revenue
projected_roas = total_revenue_without_ad_group / total_cost_without_ad_group

# Statistical significance check
from scipy import stats
# Use chi-square or binomial test if conversions < 30
```

### Example

```
Action: Pause underperforming ad group "New - Tablets Deals"
Campaign: Product Display
Current State: 
  - $400 spend, 0 conversions
  - 0% conversion rate (vs 2.4% campaign average)
  - Infinite CPA (vs $35 campaign average)
Recommended State: Paused
Expected Impact: 
  - Save ~$400/week in wasted spend
  - Improve campaign-level ROAS from 2.1x to 3.8x
  - Reallocate budget to "Laptops Premium" ad group
Evidence:
  - 330 clicks with 0 conversions (statistically significant sample)
  - Keywords targeting deal-seekers ("cheap tablets", "tablet deals") with low purchase intent
  - Ad group launched 7 days ago, no conversions yet
  - Other ad groups in same campaign converting at healthy 2-3% rate
Confidence: High (92%)
```

---

## Action Template: Refresh Ad Creative

### When to Recommend

- CTR declined >20% with stable impressions
- Same ads running >45-60 days
- Frequency data shows >3-4 impressions per user (if available)
- No other explanation for CTR decline (position, targeting stable)

### Template

```
Action: Refresh ad creative for [CAMPAIGN/AD_GROUP]
Campaign: [CAMPAIGN_NAME]
Current State: 
  - Same [NUMBER] ads running for [DAYS] days
  - CTR declined [PERCENTAGE]% from [PREVIOUS_CTR]% to [CURRENT_CTR]%
  - [AD_TYPE] with [AD_STRENGTH] strength
Recommended State: 
  - Create [NUMBER] new ad variations
  - Test messaging: [SUGGESTED_THEMES]
  - Pause [oldest/worst performing] ads
Expected Impact: 
  - Restore CTR to [BASELINE]% (based on historical performance)
  - Improve conversion volume by ~[PERCENTAGE]%
  - Estimated [NUMBER] additional conversions per week
Evidence:
  - CTR trend: -[PERCENTAGE]% over [TIME_PERIOD] (gradual decline indicates fatigue)
  - Impressions [stable/increased] (audience not exhausted)
  - Ad creation dates: all >[DAYS] days ago
  - [FREQUENCY_DATA if available]
  - Position and impression share stable (not a visibility issue)
Confidence: [Medium/High]%
Follow-up Query: |
  SELECT ad_group_ad.ad.id, 
         ad_group_ad.ad.name,
         ad_group_ad.ad.type,
         metrics.ctr, 
         metrics.impressions,
         metrics.conversions
  FROM ad_group_ad 
  WHERE campaign.name = "[CAMPAIGN_NAME]"
    AND segments.date DURING LAST_7_DAYS
  ORDER BY metrics.impressions DESC
```

### Calculation Formulas

```python
# CTR restoration estimate
baseline_ctr = ctr_before_decline  # from historical data
expected_ctr = baseline_ctr * 0.9  # conservative estimate (90% of historical)

# Conversion impact
current_clicks = impressions * (current_ctr / 100)
projected_clicks = impressions * (expected_ctr / 100)
additional_clicks = projected_clicks - current_clicks
additional_conversions_weekly = additional_clicks * (conversion_rate / 100) * 7
```

### Example

```
Action: Refresh ad creative for Product - Display
Campaign: Product - Display
Current State: 
  - Same 3 ads running for 45 days
  - CTR declined 28% from 1.8% to 1.3%
  - Responsive Display Ads with "Average" strength
Recommended State: 
  - Create 3 new ad variations
  - Test messaging: seasonal offers, user testimonials, product benefits focus
  - Pause 2 oldest ads after new ads gather data
Expected Impact: 
  - Restore CTR to ~1.6% (based on historical performance)
  - Improve conversion volume by ~20%
  - Estimated 8-10 additional conversions per week
Evidence:
  - CTR trend: -28% over 6 weeks (gradual decline indicates fatigue pattern)
  - Impressions increased 12% (audience not exhausted, expanding reach)
  - Ad creation dates: all >40 days ago
  - Frequency: 4.2 impressions per user (high, indicates saturation)
  - Position (1.8 avg) and impression share (68%) stable (not a visibility issue)
Confidence: Medium (75%)
```

---

## Action Template: Add Negative Keywords

### When to Recommend

- Search term report shows >20% irrelevant clicks
- CTR declining with impression expansion
- Cost increasing without conversion improvement (after reviewing search terms)
- Broad match keywords capturing unintended queries

### Template

```
Action: Add negative keywords to [CAMPAIGN/AD_GROUP]
Campaign: [CAMPAIGN_NAME]
Current State: 
  - [PERCENTAGE]% of clicks from irrelevant search terms
  - Top wasted spend terms: [LIST_TOP_3_5]
  - Total wasted spend: $[AMOUNT] in [TIME_PERIOD]
Recommended State: 
  - Add [NUMBER] negative keywords:
    [LIST_OF_NEGATIVES]
  - [Campaign-level / Ad-group-level / Account-level] application
Expected Impact: 
  - Reduce wasted spend by ~$[AMOUNT]/[period]
  - Improve CTR from [CURRENT]% to ~[PROJECTED]%
  - Improve conversion rate by ~[PERCENTAGE]% (focusing on relevant traffic)
  - Estimated ROI: $[AMOUNT] saved per month
Evidence:
  - Search term analysis: [PERCENTAGE]% of clicks from irrelevant terms
  - Top irrelevant terms: "[TERM1]" ([CLICKS] clicks, [CONVERSIONS] conversions, $[COST])
  - Broad match capturing [description of unwanted traffic]
  - Conversion rate by match type: Exact [RATE]%, Phrase [RATE]%, Broad [RATE]%
Confidence: [High/Medium]%
Follow-up Query: |
  SELECT segments.search_term, 
         metrics.clicks,
         metrics.conversions, 
         metrics.cost_micros
  FROM search_term_view 
  WHERE campaign.name = "[CAMPAIGN_NAME]"
    AND segments.date DURING LAST_14_DAYS
  ORDER BY metrics.cost_micros DESC
  LIMIT 50
```

### Calculation Formulas

```python
# Wasted spend calculation
irrelevant_terms_cost = sum(cost for terms with 0 conversions and clearly irrelevant)
wasted_spend_pct = (irrelevant_terms_cost / total_cost) * 100

# Projected CTR improvement
# Removing low-CTR terms improves average
current_weighted_ctr = total_clicks / total_impressions * 100
# Estimate that irrelevant terms have CTR ~50% of average
projected_ctr = (current_weighted_ctr * total_impressions - irrelevant_impressions * irrelevant_ctr) / (total_impressions - irrelevant_impressions) * 100

# Monthly savings
weekly_wasted = irrelevant_terms_cost / period_days * 7
monthly_savings = weekly_wasted * 4.33
```

### Example

```
Action: Add negative keywords to Services - Search
Campaign: Services - Search
Current State: 
  - 35% of clicks from irrelevant search terms
  - Top wasted spend terms:
    • "free consultations" (45 clicks, 0 conversions, $89)
    • "consulting jobs" (38 clicks, 0 conversions, $76)
    • "what is business consulting" (29 clicks, 0 conversions, $55)
  - Total wasted spend: $340 in last 14 days
Recommended State: 
  - Add 12 negative keywords (campaign-level):
    free, jobs, careers, salary, how to, what is, courses, training, 
    certification, schools, degree, internship
Expected Impact: 
  - Reduce wasted spend by ~$170/week ($735/month)
  - Improve CTR from 2.1% to ~2.8%
  - Improve conversion rate by ~15-20% (focusing on relevant traffic)
  - Estimated ROI: $735 saved per month
Evidence:
  - Search term analysis: 35% of clicks from irrelevant terms
  - Top irrelevant terms show job-seeking, free service, or educational intent (not purchase intent)
  - Broad match keywords capturing unintended informational queries
  - Conversion rate by match type: Exact 8%, Phrase 4%, Broad 2% (broad is problem)
Confidence: High (88%)
```

---

## Action Template: Exclude Poor Performing Locations

### When to Recommend

- Geographic analysis shows locations with >50 clicks and 0 conversions
- Specific locations have CPA >3x account average
- ROI/ROAS significantly negative in certain geos
- Budget is limited and needs to focus on best performers

### Template

```
Action: Exclude poor performing locations from [CAMPAIGN]
Campaign: [CAMPAIGN_NAME]
Current State: 
  - [NUMBER] locations with significant spend and poor performance
  - Total wasted spend: $[AMOUNT] in [TIME_PERIOD]
  - Locations to exclude: [LIST]
Recommended State: 
  - Add location exclusions for: [LIST_WITH_DETAILS]
  - Reallocate budget to top performing locations: [LIST]
Expected Impact: 
  - Reduce wasted spend by ~$[AMOUNT]/[period]
  - Improve campaign conversion rate from [CURRENT]% to ~[PROJECTED]%
  - Estimated [NUMBER] additional conversions from budget reallocation
Evidence:
  - [LOCATION1]: [CLICKS] clicks, [CONVERSIONS] conversions, $[COST] (CPA: $[VALUE] vs $[AVG] avg)
  - [LOCATION2]: [CLICKS] clicks, [CONVERSIONS] conversions, $[COST]
  - Top performing locations: [LIST] with [CONV_RATE]% conversion rate
Confidence: [High/Medium]%
Follow-up Query: |
  SELECT geographic_view.country_criterion_id,
         geographic_view.location_type,
         metrics.clicks,
         metrics.conversions,
         metrics.cost_micros
  FROM geographic_view
  WHERE campaign.name = "[CAMPAIGN_NAME]"
    AND segments.date DURING LAST_30_DAYS
  ORDER BY metrics.cost_micros DESC
```

### Example

```
Action: Exclude poor performing locations from Brand - National
Campaign: Brand - National
Current State: 
  - 3 states with significant spend and zero conversions
  - Total wasted spend: $285 in last 30 days
  - Locations: Montana, Wyoming, Vermont
Recommended State: 
  - Add location exclusions for:
    • Montana (45 clicks, 0 conversions, $98)
    • Wyoming (38 clicks, 0 conversions, $82)
    • Vermont (49 clicks, 0 conversions, $105)
  - Reallocate budget to top performers: California, Texas, Florida
Expected Impact: 
  - Reduce wasted spend by ~$285/month
  - Improve campaign conversion rate from 2.8% to 3.2%
  - Estimated 3-4 additional conversions per month from budget reallocation
Evidence:
  - Combined 132 clicks with 0 conversions across these 3 states
  - States represent <5% of total impressions but 12% of costs
  - Top performing locations (CA, TX, FL) have 4.5% conversion rate
  - No business presence or service availability in excluded states
Confidence: High (85%)
```

---

## Action Template: Improve Quality Score / Ad Rank

### When to Recommend

- Rank-lost impression share >50%
- Budget-lost impression share <20% (not a budget issue)
- Quality Score <6 on major keywords
- Ad strength "Poor" or "Average" for RSAs

### Template

```
Action: Improve Quality Score for [CAMPAIGN/AD_GROUP]
Campaign: [CAMPAIGN_NAME]
Current State: 
  - Rank-lost impression share: [PERCENTAGE]%
  - Average Quality Score: [SCORE]/10
  - Ad strength: [POOR/AVERAGE]
  - Missing [PERCENTAGE]% of impressions due to ad rank
Recommended State: 
  - Improve Quality Score components:
    1. [COMPONENT1]: [SPECIFIC_ACTION]
    2. [COMPONENT2]: [SPECIFIC_ACTION]
    3. [COMPONENT3]: [SPECIFIC_ACTION]
Expected Impact: 
  - Reduce rank-lost IS from [CURRENT]% to <30%
  - Capture [NUMBER] additional impressions per day
  - Estimated [NUMBER] additional conversions per week
  - Potentially reduce CPC by [PERCENTAGE]% with improved quality
Evidence:
  - Rank-lost IS: [VALUE]% (severe quality/bid issue)
  - Budget-lost IS: [VALUE]% (budget is not the constraint)
  - Quality Score breakdown:
    • Expected CTR: [Below Avg/Avg/Above Avg]
    • Ad Relevance: [Below Avg/Avg/Above Avg]
    • Landing Page Experience: [Below Avg/Avg/Above Avg]
  - [SPECIFIC_QUALITY_ISSUES]
Confidence: [Medium/High]%
Follow-up Query: |
  SELECT ad_group_criterion.keyword.text,
         ad_group_criterion.quality_info.quality_score,
         ad_group_criterion.quality_info.creative_quality_score,
         ad_group_criterion.quality_info.post_click_quality_score,
         ad_group_criterion.quality_info.search_predicted_ctr,
         metrics.impressions
  FROM keyword_view
  WHERE campaign.name = "[CAMPAIGN_NAME]"
    AND segments.date DURING LAST_7_DAYS
  ORDER BY metrics.impressions DESC
```

### Specific Actions by Quality Score Component

**Expected CTR (Below Average):**
- Improve ad copy relevance to keywords
- Add more keyword-specific headlines
- Test stronger calls-to-action
- Add ad extensions (sitelinks, callouts, structured snippets)

**Ad Relevance (Below Average):**
- Tighten ad group keyword themes
- Split broad ad groups into tighter groups
- Ensure ad copy directly mentions target keywords
- Remove irrelevant keywords

**Landing Page Experience (Below Average):**
- Improve page load speed (<3 seconds)
- Enhance mobile responsiveness
- Add relevant content matching ad message
- Improve navigation and UX
- Add trust signals (reviews, certifications)

### Example

```
Action: Improve Quality Score for Services - General
Campaign: Services - General
Current State: 
  - Rank-lost impression share: 68%
  - Average Quality Score: 4.2/10
  - Ad strength: Average (need 2 more headlines for "Good")
  - Missing 68% of impressions due to ad rank
Recommended State: 
  - Improve Quality Score components:
    1. Expected CTR: Add 5 new keyword-specific headlines to RSAs, test stronger CTAs
    2. Ad Relevance: Split "Services - General" into 3 tighter ad groups by service type
    3. Landing Page: Improve mobile load speed (currently 6.2s → target <3s), add client testimonials
Expected Impact: 
  - Reduce rank-lost IS from 68% to <30%
  - Capture estimated 8,000 additional impressions per day
  - Estimated 15-20 additional conversions per week
  - Potentially reduce average CPC by 15-25% with improved quality
Evidence:
  - Rank-lost IS: 68% (severe quality/bid issue)
  - Budget-lost IS: 8% (budget is not the constraint)
  - Quality Score breakdown:
    • Expected CTR: Below Average (ads not compelling enough)
    • Ad Relevance: Below Average (ad group too broad - 47 keywords)
    • Landing Page Experience: Below Average (slow load time, high bounce rate)
  - Ad strength "Average" - only using 8 of 15 headline slots
  - Mobile landing page speed: 6.2 seconds (Google target: <3s)
Confidence: Medium (72%)
```

---

## Action Template: Consolidate PMax Campaigns (Cannibalization)

### When to Recommend

- Multiple PMax campaigns targeting same domain
- One campaign has Final URL Expansion enabled, others don't
- Inverse correlation: one campaign starts exactly when another stops
- High rank-lost IS (>80%) on cannibalized campaign with 0% budget-lost IS

### Template

```
Action: Consolidate Performance Max campaigns to eliminate cannibalization
Campaigns Affected: [CAMPAIGN1], [CAMPAIGN2]
Current State: 
  - [CAMPAIGN1]: [STATUS] - Final URL Expansion: [Enabled/Disabled]
  - [CAMPAIGN2]: [STATUS] - Final URL Expansion: [Enabled/Disabled]
  - Overlap detected: Both targeting [DOMAIN]
  - [CANNIBALIZED_CAMPAIGN] stopped getting clicks on [DATE]
Recommended State: 
  Option 1 (Recommended): Consolidate into single PMax campaign with multiple asset groups
  Option 2: Align Final URL Expansion settings (both enabled or both disabled)
  Option 3: Separate by targeting different domains/subdomains
Expected Impact: 
  - Restore traffic to [CANNIBALIZED_CAMPAIGN]: ~[CLICKS] clicks/day, ~[CONVERSIONS] conversions/week
  - OR improve [DOMINANT_CAMPAIGN] by consolidating assets and signals
  - Eliminate wasted auction competition with self
  - Improve overall account efficiency
Evidence:
  - Inverse correlation: [CAMPAIGN1] clicks stopped on [DATE], exact day [CAMPAIGN2] launched
  - [CANNIBALIZED_CAMPAIGN]: [DAYS] consecutive days of zero or near-zero clicks
  - Final URL Expansion mismatch: [CAMPAIGN1] = [State], [CAMPAIGN2] = [State]
  - Both campaigns target [DOMAIN] (confirmed via asset_group.final_urls)
  - Impression share pattern:
    • [CANNIBALIZED]: 92% rank-lost IS, 0% budget-lost IS
    • [DOMINANT]: Normal IS pattern
  - Google Ads suppresses overlapping PMax to prevent self-competition
Confidence: High (>90%)
Follow-up Query: |
  SELECT campaign.name,
         campaign.advertising_channel_type,
         campaign.asset_automation_settings,
         metrics.impressions,
         metrics.clicks
  FROM campaign
  WHERE campaign.advertising_channel_type = 'PERFORMANCE_MAX'
    AND segments.date DURING LAST_7_DAYS
```

### Example

```
Action: Consolidate Performance Max campaigns to eliminate cannibalization
Campaigns Affected: "PMax - Supply Side", "PMax - Demand Side"
Current State: 
  - PMax - Supply Side: ENABLED - Final URL Expansion: Enabled (OptedIn)
  - PMax - Demand Side: ENABLED - Final URL Expansion: Disabled (not present)
  - Overlap detected: Both targeting marketplace.com domain
  - "PMax - Demand Side" stopped getting clicks on 2025-10-01 (day Supply Side launched)
Recommended State: 
  Option 1 (Recommended): Consolidate into "PMax - Marketplace" with two asset groups:
    • Asset Group 1: Supply-side audience signals and creative
    • Asset Group 2: Demand-side audience signals and creative
  Option 2: Disable Final URL Expansion on "PMax - Supply Side" to level playing field
Expected Impact: 
  - Restore traffic to Demand Side: ~150 clicks/day, ~8-10 conversions/week
  - OR improve consolidated campaign by combining audience signals and creative assets
  - Eliminate wasted auction competition (Google suppressing one campaign)
  - Improve overall account efficiency and learning speed
Evidence:
  - Inverse correlation: Demand Side went from 150 clicks/day → <5 clicks/day on 2025-10-01
  - "PMax - Demand Side": 12 consecutive days of near-zero clicks (was healthy before)
  - Final URL Expansion mismatch: Supply Side OptedIn (can target entire domain), Demand Side disabled (restricted to specific URLs)
  - Both campaigns target marketplace.com (even though serving different user types)
  - Impression share pattern:
    • Demand Side: 92% rank-lost IS, 0% budget-lost IS (suppressed by algorithm)
    • Supply Side: 68% rank-lost IS, 12% budget-lost IS (normal pattern)
  - Google Ads prevents self-competition in PMax; detected domain overlap triggers suppression
Confidence: High (95%)
```

---

## Confidence Scoring Guidelines

**High Confidence (>85%):**
- Clear statistical significance (>100 clicks, >10 conversions)
- Single obvious root cause
- Historical precedent for similar issue/action
- Minimal uncertainty in impact estimate

**Medium Confidence (70-85%):**
- Moderate statistical significance (50-100 clicks, 5-10 conversions)
- Multiple potential contributing factors
- Impact estimate has wider range
- Some assumptions required

**Low Confidence (<70%):**
- Limited statistical significance (<50 clicks, <5 conversions)
- Multiple competing hypotheses
- Impact estimate highly uncertain
- Significant assumptions or unknowns

---

## Impact Estimation Best Practices

1. **Be conservative**: Better to under-promise and over-deliver
2. **Use ranges**: "8-12 conversions" better than "10 conversions"
3. **Show math**: Explain how you calculated the estimate
4. **Note assumptions**: "Assumes conversion rate remains at 3.2%"
5. **Include timeframe**: Per day/week/month
6. **Acknowledge uncertainty**: Higher for smaller sample sizes
7. **Reference historical data**: "Based on historical performance when budget was increased in Q2..."

---

## Follow-up Query Best Practices

Each recommendation should include a GAQL query to monitor the impact of changes:

**Requirements:**
- Executable as-is (valid GAQL syntax)
- Includes date filter (usually LAST_7_DAYS or LAST_14_DAYS)
- Focuses on metrics relevant to the recommendation
- Includes necessary dimensions (campaign name, ad group, etc.)
- Ordered logically (usually by date or primary metric)

**Common Follow-up Query Patterns:**

```sql
-- Monitor budget increase impact
SELECT segments.date, 
       metrics.search_budget_lost_impression_share,
       metrics.impressions, 
       metrics.conversions, 
       metrics.cost_micros
FROM campaign 
WHERE campaign.name = "[CAMPAIGN]" 
  AND segments.date DURING LAST_14_DAYS
ORDER BY segments.date DESC

-- Monitor ad group pause impact
SELECT campaign.name,
       ad_group.name,
       ad_group.status,
       metrics.cost_micros, 
       metrics.conversions
FROM ad_group
WHERE campaign.name = "[CAMPAIGN]"
  AND segments.date DURING LAST_7_DAYS

-- Monitor negative keyword impact
SELECT segments.search_term,
       metrics.clicks, 
       metrics.conversions,
       metrics.cost_micros
FROM search_term_view
WHERE campaign.name = "[CAMPAIGN]"
  AND segments.date DURING LAST_14_DAYS
ORDER BY metrics.cost_micros DESC
LIMIT 50

-- Monitor creative refresh impact
SELECT ad_group_ad.ad.id,
       ad_group_ad.ad.name,
       metrics.impressions,
       metrics.ctr,
       metrics.conversions
FROM ad_group_ad
WHERE campaign.name = "[CAMPAIGN]"
  AND segments.date DURING LAST_14_DAYS
ORDER BY metrics.impressions DESC
```
