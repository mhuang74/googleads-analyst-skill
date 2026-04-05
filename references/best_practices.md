# Analysis Best Practices

## Data Quality Checks

Before presenting analysis, verify:

1. **Data Completeness**: Do all campaigns have data for both periods?
2. **Cost Conversion**: Have you converted cost_micros to dollars correctly (÷ 1,000,000) and validated ranges? See [analysis/derived_metrics_reference.md](analysis/derived_metrics_reference.md)
3. **Null/Zero Handling**: Are you handling division by zero in calculations?
4. **Date Alignment**: Are both periods the same length (both 7 days, etc.)?
5. **Campaign Matching**: Did you correctly match campaigns by ID between periods?
6. **Sample Size**: Have you noted when sample sizes are too small for conclusions?

## When to Alert the User

Immediately flag these critical issues:

- **Conversion tracking appears broken** (all campaigns zero conversions suddenly)
- **Query returning no data** despite campaigns existing (technical issue)
- **Extreme changes** (>90% cost increase, all conversions dropped) that seem like errors
- **Data appears incomplete** (some campaigns missing from one period)

---

## Query Best Practices

**⚠️ CRITICAL: Always Validate Before Executing** - See [tools/gaql_validation.md](tools/gaql_validation.md)

### Standard Query Pattern

1. **NEVER skip validation** - Always run `mcc-gaql --validate` before executing
2. Use the 3-step workflow for every query:
   - Step 1: Define/Generate query
   - Step 2: VALIDATE (required)
   - Step 3: Execute only if validation passes
3. Store queries in variables for reuse between validation and execution

### mcc-gaql-gen Usage

- Use `--use-query-cookbook` flag for improved generation quality
- Always validate generated queries before execution
- Include explicit date ranges in prompts
- Specify exact metrics and dimensions needed

---

## Reporting Best Practices

1. **Present text report first**: Use markdown tables directly in the terminal before offering PDF
2. **Adapt structure to complexity**: More detail for critical issues, simpler for healthy accounts
3. **Always show date ranges** being compared at the top of your analysis
4. **Prioritize actionable insights** over raw data dumps
5. **Lead with conclusions**: Executive summary first, details after
6. **Provide context**: Include account totals, not just campaign-level data
7. **Note data limitations**: Small sample sizes, statistical significance concerns
8. **Engage in Q&A**: Pause after presenting report, invite questions, continue until user is satisfied
9. **Mention PDF at the end**: "A PDF version is available if you need it for sharing"
10. **Generate PDF only when requested**: Use standardized filename `google_ads_report_{account_name}_{YYYY-MM-DD}.pdf`

---

## Analysis Best Practices

### 1. Consider metric interdependencies

Not just individual metric changes:
- Don't flag CTR drop if it came with impression expansion and maintained conversions
- Don't celebrate conversion increase if CPA doubled

### 2. Provide specific, actionable recommendations

Not generic advice:
- ❌ "Consider optimizing your campaigns"
- ✅ "Increase Field Trips campaign budget from $50/day to $100/day to capture lost impression share"

### 3. Acknowledge uncertainty

When inferring from limited data:
- "With only 23 clicks, this conversion rate change may not be statistically significant"
- "This appears to be ad fatigue based on declining CTR, but verify with frequency data"

### 4. Link to root causes

Not just symptoms:
- Don't just say "CPA increased 50%"
- Say "CPA increased 50% because average CPC rose from $2 to $4, likely due to increased competition (new competitor X appeared in auction insights)"

### 5. Provide actionable timelines

- HIGH PRIORITY = today/tomorrow
- MEDIUM PRIORITY = this week
- LOW PRIORITY = monitor ongoing

### 6. Calculate all derived metrics correctly

- **FIRST: Convert cost_micros to dollars** (÷ 1,000,000)
- Then calculate CPA, ROAS, conversion rate, etc.
- See [analysis/derived_metrics_reference.md](analysis/derived_metrics_reference.md) for all formulas
- Calculate account totals and averages

### 7. Pattern recognition over individual metrics

- Look for clusters of related changes
- Identify whether issues are campaign-specific or account-wide
- Recognize standard patterns (budget constraint, ad fatigue, etc.)

---

## Communication Best Practices

1. **Tailor urgency to severity**: Use appropriate language (CRITICAL, Monitor, Informational)
2. **Explain the "why"**: Don't just report what changed, explain why it matters
3. **Offer choices when uncertain**: "This could be either budget constraint or quality score decline. Check [X] to determine which"
4. **Be direct about bad news**: Don't sugarcoat significant issues
5. **Celebrate wins**: Acknowledge successful campaigns and optimizations
6. **Think like a consultant**: What would you want to know if you were managing this account?
7. **Clearly identify Account Name**: Include Google Ads account name and account number at the top of the report

---

## Final Checklist Before Delivering Analysis

### 🎯 PRIMARY FOCUS (MANDATORY - Check These FIRST):

#### Dynamic Investigation
- [ ] **Dynamic investigation performed to gather evidence for specific recommendations**
  - [ ] Drilled down from campaign → ad group → keyword/ad level for each HIGH/MEDIUM issue
  - [ ] Generated targeted queries using mcc-gaql-gen for root cause analysis
  - [ ] Collected quantifiable data supporting each recommendation
  - [ ] Calculated projected impact using actual account metrics
  - [ ] Provided concrete examples (specific keywords, ad groups, ads with issues)

#### Correlation Analysis
- [ ] **Correlation scores calculated between user changes and performance anomalies**
  - [ ] Queried change_event resource for EVERY performance anomaly detected (within last 30 days)
  - [ ] Matched change timestamps with metric change timestamps
  - [ ] Scored correlations on 0-100 scale (temporal proximity + impact + change type)
  - [ ] Distinguished user-caused changes from market/external factors
  - [ ] Provided evidence-based attribution for all performance shifts

### Query Execution Quality:

- [ ] **ALL GAQL queries validated** using `mcc-gaql --validate` before execution
- [ ] No queries executed without successful validation (exit code 0)
- [ ] Validation errors fixed before execution (if any occurred)
- [ ] For mcc-gaql-gen queries: validated every generated query
- [ ] Used `--show-fields` to verify field names when validation failed
- [ ] No hallucinated fields in executed queries

### Content Quality:

- [ ] **Converted cost_micros to dollars FIRST** (÷ 1,000,000) before any other calculations
- [ ] All derived metrics calculated correctly (CPA, ROAS, conversion rate, etc.)
- [ ] Validated cost ranges are reasonable (not in micros)
- [ ] Calculated account totals and campaign-level metrics
- [ ] Impression share metrics analyzed BEFORE making recommendations
- [ ] Metrics interpreted in context (interdependencies considered)
- [ ] Root causes identified, not just symptoms described
- [ ] Campaign health status assigned correctly (🟢 🟡 🔴)
- [ ] Recommendations are specific and actionable with quantified impact

### Reporting Quality:

- [ ] Date ranges clearly stated at the top
- [ ] Text report with markdown tables presented first
- [ ] Report structure adapted to complexity (simpler for healthy accounts)
- [ ] Executive summary leads with key conclusions
- [ ] Issues prioritized appropriately (HIGH/MEDIUM/LOW)
- [ ] Statistical significance noted when relevant
- [ ] Interactive Q&A conducted until user satisfied
- [ ] PDF mentioned only after text report and Q&A
- [ ] PDF generated only if explicitly requested

---

## Investigation Best Practices

For detailed investigation patterns and strategies, see:
- [analysis/investigation_patterns_reference.md](analysis/investigation_patterns_reference.md)
- [correlation/change_correlation_reference.md](correlation/change_correlation_reference.md)
- [actions/action_templates_reference.md](actions/action_templates_reference.md)
