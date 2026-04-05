# GAQL Validation Best Practices

## ⚠️ CRITICAL: Always Validate GAQL Before Executing

**ALWAYS validate queries before execution** to catch field errors, typos, and incompatibilities early.

### Standard Validation Workflow

```bash
# Step 1: VALIDATE first (required)
mcc-gaql --profile <PROFILE_NAME> --validate '<GAQL_QUERY>'

# Step 2: If validation passes, THEN execute
mcc-gaql --profile <PROFILE_NAME> --format csv -o <OUTPUT_FILE> '<GAQL_QUERY>'
```

### Why Validation is Critical

- Catches hallucinated or non-existent fields immediately
- Identifies field compatibility issues before execution
- Prevents wasted time on queries that will fail
- Validates GAQL syntax and structure

### If Validation Fails

1. Check the error message for specific field issues
2. Use `mcc-gaql --show-fields <resource>` to find correct field names
3. Fix the query and re-validate
4. Only execute after successful validation

---

## Complete Example with Profile

```bash
# STEP 1: Validate the query first
QUERY='SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

mcc-gaql --profile themade --validate "$QUERY"

# STEP 2: Only if validation succeeds, execute the query
mcc-gaql --profile themade --format csv -o /tmp/current_period.csv "$QUERY"
```

## Complete Example without Profile

```bash
# STEP 1: Validate the query first
QUERY='SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user user@example.com --validate "$QUERY"

# STEP 2: Only if validation succeeds, execute the query
mcc-gaql --mcc 1234567890 --customer-id 9876543210 --user user@example.com --format csv -o /tmp/current_period.csv "$QUERY"
```

---

## Validation Best Practices

### 1. NEVER Skip Validation

- **NEVER skip validation** - Always run `mcc-gaql --validate` before executing any GAQL query
  - Manual queries: Validate before first execution
  - Generated queries: Validate every mcc-gaql-gen output
  - Modified queries: Re-validate after any changes

### 2. Use the 3-Step Workflow

```bash
# Step 1: Define/Generate query
QUERY='SELECT ...'

# Step 2: VALIDATE (required)
mcc-gaql --profile <PROFILE> --validate "$QUERY"

# Step 3: Execute only if validation passes
if [ $? -eq 0 ]; then
  mcc-gaql --profile <PROFILE> --format csv -o output.csv "$QUERY"
fi
```

### 3. When Validation Fails

- Read the error message carefully - it tells you exactly what's wrong
- Use `mcc-gaql --show-fields <resource>` to find correct field names
- Don't guess or try random variations - verify field names
- Check `.internal/GAQL_FIELD_VALIDATION_REPORT.md` for validated queries

### 4. Common Validation Errors and Fixes

| Error | Fix |
|-------|-----|
| `Field 'metrics.video_views' does not exist` | Use `metrics.video_trueview_views` |
| `Field incompatibility` | Check which fields can be selected together |
| `Invalid WHERE clause` | Verify date format, enum values, filter syntax |

### 5. Store Queries in Variables

- Makes code cleaner and ensures you validate the same query you execute
- Easier to debug and modify

```bash
# Good practice
QUERY='SELECT campaign.name, metrics.cost_micros FROM campaign'
mcc-gaql --profile themade --validate "$QUERY"
mcc-gaql --profile themade --format csv -o /tmp/output.csv "$QUERY"

# Bad practice (easy to make typos between validate and execute)
mcc-gaql --profile themade --validate 'SELECT campaign.name, metrics.cost_micros FROM campaign'
mcc-gaql --profile themade --format csv -o /tmp/output.csv 'SELECT campaign.name, metrics.cost_micros FROM campagin'  # typo!
```

---

## Validation Benefits

✅ Catches hallucinated/non-existent fields immediately  
✅ Detects field compatibility issues before execution  
✅ Validates GAQL syntax and structure  
✅ Saves time debugging failed queries  
✅ No API quota cost for validation

---

## Related References

- For advanced mcc-gaql usage: [mcc_gaql_reference.md](mcc_gaql_reference.md)
- For generating GAQL from natural language: [gaql_tools_overview.md](gaql_tools_overview.md)
- For common errors and solutions: [common_errors_reference.md](common_errors_reference.md)
