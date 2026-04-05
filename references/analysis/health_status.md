# Campaign Health Status and Impression Share Analysis

## Determining Campaign Health Status

Use this decision tree to assign status to each campaign:

### 🟢 HEALTHY (Good Performance)

- Conversions increased OR maintained with improved efficiency
- ROAS improved or stable (>= -5%)
- Cost increase justified by proportional or better conversion growth
- CTR stable or improving

### 🟡 MONITOR (Warning Signs)

- Cost increase 10-20% with marginal conversion improvement (<10%)
- Conversion rate drop 15-30%
- CTR drop 10-25% without clear cause
- Impression share loss >15%
- Rank-lost impression share >50% (ad rank/quality issues)
- Budget-lost impression share >20% (budget constraint)
- ROAS decline 5-15%
- Significant volume changes (>50% impressions change)

### 🔴 CRITICAL ISSUE (Requires Immediate Action)

- Cost increase >20% without conversion improvement
- Conversion rate drop >30%
- CTR drop >25% with stable impressions
- Rank-lost impression share >70% (severe ad rank/quality problems)
- Budget-lost impression share >50% (severe budget constraint)
- ROAS decline >20%
- Conversions dropped >40%
- Cost per conversion increased >50%
- Zero conversions with significant spend (>$100)

---

## Impression Share Analysis (CRITICAL)

**ALWAYS analyze impression share metrics first** - they reveal the root cause of many performance issues.

### Understanding Impression Share Metrics

- **Search Impression Share**: Percentage of impressions you received out of total available
- **Budget Lost IS**: Percentage of impressions you DIDN'T get due to insufficient budget
- **Rank Lost IS**: Percentage of impressions you DIDN'T get due to low ad rank (Quality Score × Bid)

### Diagnosis Framework

| Scenario | Budget Lost IS | Rank Lost IS | Diagnosis | Action |
|----------|---------------|--------------|-----------|---------|
| 🔴 **Budget Constraint** | >50% | <20% | Severe budget limitation | Increase budget immediately |
| 🟡 **Budget Constraint** | 20-50% | <50% | Moderate budget limitation | Consider increasing budget |
| 🔴 **Ad Rank Crisis** | <20% | >70% | Severe quality/bid issues | Fix Quality Score, increase bids, improve ad quality |
| 🟡 **Ad Rank Issues** | <50% | 50-70% | Moderate quality/bid issues | Review Quality Score, consider bid increases |
| 🟡 **Mixed Issues** | >30% | >30% | Both budget and quality problems | Address quality first, then budget |
| 🟢 **Good Coverage** | <10% | <30% | Healthy impression share | Monitor and optimize |

### Common Mistakes to Avoid

- ❌ **DON'T recommend "increase budget"** if Budget Lost IS = 0% (this was the mistake in the themade analysis)
- ❌ **DON'T assume budget constraint** based on spend decrease alone
- ✅ **DO check impression share data** before making any budget recommendations
- ✅ **DO distinguish between budget and quality issues** - they require different solutions

### Root Cause Analysis

**High Rank Lost IS (>50%)** is caused by:
- Low Quality Score (poor ad relevance, landing page experience, or expected CTR)
- Bids too low relative to competition
- Poor ad strength/quality (especially for PMax campaigns)
- Weak audience signals (for PMax campaigns)

**High Budget Lost IS (>20%)** is caused by:
- Daily budget too low for campaign potential
- Budget consumed early in the day (check hour-of-day performance)
- Campaign scaled too quickly without budget adjustment

### Recommended Actions by Issue

#### For High Rank Lost IS:

1. Review Quality Score components (for Search campaigns)
2. Audit ad/asset strength (for PMax campaigns)
3. Improve landing page experience (speed, mobile, relevance)
4. Add high-quality audience signals (customer lists, website visitors)
5. Increase bids by 20-30% if willing to pay more
6. Improve ad copy relevance to match search intent
7. Test new ad variations with stronger CTAs

#### For High Budget Lost IS:

1. Increase daily budget proportional to lost impression share
   - Formula: `new_budget = current_budget / (1 - budget_lost_is)`
   - Example: $50 budget, 40% lost → $50 / 0.6 = $83 new budget
2. Check budget scheduling (is budget running out early in the day?)
3. Consider campaign priorities (shift budget from low-performers)
4. Review whether budget caps align with business goals

#### For Mixed Issues (Both High):

1. **Address quality first** - quality improvements increase efficiency
2. Fix Quality Score/ad strength issues
3. **Then** increase budget once quality is improved
4. Increasing budget with poor quality wastes money

### Priority Classification

Use impression share data to classify issue priority:

- **HIGH PRIORITY**: Budget Lost IS >50% OR Rank Lost IS >70%
- **MEDIUM PRIORITY**: Budget Lost IS 20-50% OR Rank Lost IS 50-70%
- **LOW PRIORITY**: Budget Lost IS <20% AND Rank Lost IS <50%

Higher severity = more investigation depth required in Dynamic Investigation Mode.
