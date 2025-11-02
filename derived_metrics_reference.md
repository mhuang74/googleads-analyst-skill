# Derived Metrics Reference

This document contains detailed instructions for calculating derived metrics from Google Ads API data.

## CRITICAL: Currency Conversion (MUST DO FIRST)

**⚠️ Google Ads API returns costs in MICROS (micro-currency units)**

Before ANY calculations, you MUST convert `cost_micros` to dollars:

```python
# CORRECT conversion formula
cost_dollars = cost_micros / 1_000_000

# Example with actual data:
# If cost_micros = 859,918,906.0
# Then cost_dollars = 859,918,906.0 / 1,000,000 = $859.92 (NOT $859,918!)

# ❌ WRONG conversions that will cause errors:
# cost_dollars = cost_micros / 1000        # OFF BY 1000X - TOO HIGH
# cost_dollars = cost_micros / 100000      # OFF BY 10X - TOO HIGH
# cost_dollars = cost_micros / 10000000    # OFF BY 10X - TOO LOW
```

### Validation Check - ALWAYS Verify Conversions

After converting costs, check that values make sense:

```python
# After conversion, check range
if cost_dollars > 100_000:
    print("⚠️ WARNING: Cost exceeds $100k - verify conversion is correct")
    print(f"   Raw cost_micros: {cost_micros}")
    print(f"   Converted cost: ${cost_dollars:,.2f}")
    print(f"   Formula used: {cost_micros} / 1,000,000 = {cost_dollars}")

# Also verify against other metrics
if cost_dollars > 0 and clicks > 0:
    cpc = cost_dollars / clicks
    if cpc > 500:
        print("⚠️ WARNING: CPC > $500 - this is extremely high, verify cost conversion")
    if cpc < 0.01:
        print("⚠️ WARNING: CPC < $0.01 - this is extremely low, verify cost conversion")
```

**Reasonable Value Ranges:**
- ✅ Daily campaign spend: typically $1 - $10,000 (most accounts)
- ✅ Weekly campaign spend: typically $7 - $70,000 (most accounts)
- ⚠️ If you see costs >$100,000 for a single campaign → VERIFY YOUR CONVERSION
- ⚠️ If you see costs <$0.01 for active campaigns → VERIFY YOUR CONVERSION

**Common Mistake Example:**
```python
# ❌ WRONG (the actual mistake that was made):
cost_dollars = 859_918_906.0 / 1000 = $859,918.91
# This makes a $860 spend look like $860,000!

# ✅ CORRECT:
cost_dollars = 859_918_906.0 / 1_000_000 = $859.92
```

**Similarly, convert CPC from micros:**
```python
# If the API returns average_cpc in micros:
average_cpc_dollars = average_cpc_micros / 1_000_000
```

## Standard Derived Metrics

After converting currency units, calculate these standard metrics:

### Click-Through Rate (CTR)
```python
# CTR (if needed - often provided by API)
ctr = (clicks / impressions) * 100 if impressions > 0 else 0
```

### Conversion Rate
```python
# Conversion Rate (API often doesn't provide this)
conversion_rate = (conversions / clicks) * 100 if clicks > 0 else 0
```

### Cost Per Conversion (CPA)
```python
# Cost Per Conversion (use converted dollar amounts!)
cost_per_conversion = cost_dollars / conversions if conversions > 0 else 0
```

### Return on Ad Spend (ROAS)
```python
# ROAS (Return on Ad Spend)
roas = conversions_value / cost_dollars if cost_dollars > 0 else 0
```

### Average Cost Per Click (CPC)
```python
# Average CPC (if needed - often provided by API)
avg_cpc = cost_dollars / clicks if clicks > 0 else 0
```

## Period-over-Period Changes

### Absolute Changes
```python
impressions_change = current_impressions - previous_impressions
clicks_change = current_clicks - previous_clicks
cost_change = current_cost_dollars - previous_cost_dollars  # Use converted dollars!
conversions_change = current_conversions - previous_conversions
```

### Percentage Changes
```python
def calculate_percent_change(current, previous):
    if previous == 0:
        return "N/A" if current == 0 else "∞"
    return ((current - previous) / previous) * 100

impressions_pct = calculate_percent_change(current_impressions, previous_impressions)
clicks_pct = calculate_percent_change(current_clicks, previous_clicks)
cost_pct = calculate_percent_change(current_cost, previous_cost)
conversions_pct = calculate_percent_change(current_conversions, previous_conversions)
ctr_pct = calculate_percent_change(current_ctr, previous_ctr)
```

### Efficiency Metric Changes
```python
# CPA (Cost Per Acquisition) change
current_cpa = current_cost / current_conversions if current_conversions > 0 else 0
previous_cpa = previous_cost / previous_conversions if previous_conversions > 0 else 0
cpa_change = calculate_percent_change(current_cpa, previous_cpa)

# ROAS (Return on Ad Spend) change
current_roas = current_conv_value / current_cost if current_cost > 0 else 0
previous_roas = previous_conv_value / previous_cost if previous_cost > 0 else 0
roas_change = calculate_percent_change(current_roas, previous_roas)
```

## Best Practices

1. **Always convert currency FIRST** before any other calculations
2. **Validate conversions** - check that values are in reasonable ranges
3. **Handle division by zero** - always check denominators before dividing
4. **Use converted values** - never mix micros and dollars in calculations
5. **Document units** - clearly label whether values are in dollars or micros in variable names
