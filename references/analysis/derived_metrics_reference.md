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

---

## Complete Python Workflow Example

This section provides a complete, production-ready Python script for processing Google Ads data from mcc-gaql CSV output, calculating derived metrics, and performing period-over-period comparisons.

### Prerequisites

```bash
pip install pandas
```

### Complete Script

```python
#!/usr/bin/env python3
"""
Google Ads Performance Analysis Script

Processes mcc-gaql CSV output to calculate derived metrics and
perform period-over-period (MoM, WoW, etc.) comparisons.

Usage:
    python analyze_campaigns.py /tmp/current_period.csv /tmp/previous_period.csv
"""

import pandas as pd
import sys
from typing import Dict, Any

def load_campaign_data(csv_path: str) -> pd.DataFrame:
    """
    Load campaign data from mcc-gaql CSV output.
    
    Args:
        csv_path: Path to CSV file from mcc-gaql
        
    Returns:
        DataFrame with campaign data
    """
    df = pd.read_csv(csv_path)
    
    # Convert campaign.id to string for reliable matching
    if 'campaign.id' in df.columns:
        df['campaign.id'] = df['campaign.id'].astype(str)
    
    return df

def convert_currency_fields(df: pd.DataFrame) -> pd.DataFrame:
    """
    Convert all micros fields to dollars.
    
    CRITICAL: This must be done FIRST before any other calculations.
    """
    df = df.copy()
    
    # Convert cost_micros to dollars
    if 'metrics.cost_micros' in df.columns:
        df['cost_dollars'] = df['metrics.cost_micros'] / 1_000_000
        print(f"✓ Converted cost_micros (range: ${df['cost_dollars'].min():.2f} - ${df['cost_dollars'].max():.2f})")
    
    # Convert average_cpc if present
    if 'metrics.average_cpc' in df.columns:
        df['avg_cpc_dollars'] = df['metrics.average_cpc'] / 1_000_000
    
    # Validate conversion
    if 'cost_dollars' in df.columns:
        max_cost = df['cost_dollars'].max()
        if max_cost > 100_000:
            print(f"⚠️  WARNING: Max cost is ${max_cost:,.2f} - verify this is correct")
    
    return df

def calculate_derived_metrics(df: pd.DataFrame) -> pd.DataFrame:
    """
    Calculate all derived metrics from raw API data.
    
    Must be called AFTER convert_currency_fields().
    """
    df = df.copy()
    
    # CTR (if not already provided)
    if 'metrics.ctr' not in df.columns and 'metrics.impressions' in df.columns:
        df['ctr'] = (df['metrics.clicks'] / df['metrics.impressions'] * 100).fillna(0)
    else:
        df['ctr'] = df['metrics.ctr']
    
    # Conversion Rate
    if 'metrics.clicks' in df.columns and 'metrics.conversions' in df.columns:
        df['conversion_rate'] = (df['metrics.conversions'] / df['metrics.clicks'] * 100).fillna(0)
    
    # Cost Per Conversion (CPA)
    if 'cost_dollars' in df.columns and 'metrics.conversions' in df.columns:
        df['cpa'] = (df['cost_dollars'] / df['metrics.conversions']).replace([float('inf')], 0).fillna(0)
    
    # Return on Ad Spend (ROAS)
    if 'cost_dollars' in df.columns and 'metrics.conversions_value' in df.columns:
        df['roas'] = (df['metrics.conversions_value'] / df['cost_dollars']).replace([float('inf')], 0).fillna(0)
    
    # Average CPC (if not already calculated)
    if 'avg_cpc_dollars' not in df.columns and 'cost_dollars' in df.columns:
        df['avg_cpc_dollars'] = (df['cost_dollars'] / df['metrics.clicks']).replace([float('inf')], 0).fillna(0)
    
    return df

def calculate_percent_change(current: float, previous: float) -> float:
    """
    Calculate percentage change between two values.
    
    Returns:
        Percentage change, or 0 if previous is 0
    """
    if previous == 0:
        return 0 if current == 0 else float('inf')
    return ((current - previous) / previous) * 100

def compare_periods(current_df: pd.DataFrame, previous_df: pd.DataFrame) -> pd.DataFrame:
    """
    Compare two periods and calculate changes for all metrics.
    
    Args:
        current_df: Current period data (with derived metrics)
        previous_df: Previous period data (with derived metrics)
        
    Returns:
        DataFrame with current, previous, and change columns
    """
    # Merge on campaign.id
    comparison = current_df.merge(
        previous_df,
        on='campaign.id',
        how='outer',
        suffixes=('_current', '_previous')
    )
    
    # Get campaign name (prefer current, fallback to previous)
    comparison['campaign_name'] = comparison['campaign.name_current'].fillna(
        comparison['campaign.name_previous']
    )
    
    # List of metrics to compare
    metrics = [
        'metrics.impressions',
        'metrics.clicks',
        'cost_dollars',
        'ctr',
        'metrics.conversions',
        'conversion_rate',
        'cpa',
        'roas',
        'metrics.search_impression_share',
        'metrics.search_budget_lost_impression_share',
        'metrics.search_rank_lost_impression_share'
    ]
    
    # Calculate changes for each metric
    for metric in metrics:
        current_col = f"{metric}_current"
        previous_col = f"{metric}_previous"
        change_col = f"{metric}_change_pct"
        
        if current_col in comparison.columns and previous_col in comparison.columns:
            comparison[change_col] = comparison.apply(
                lambda row: calculate_percent_change(
                    row[current_col] if pd.notna(row[current_col]) else 0,
                    row[previous_col] if pd.notna(row[previous_col]) else 0
                ),
                axis=1
            )
    
    return comparison

def format_metric(value: float, metric_type: str) -> str:
    """
    Format a metric value for display.
    
    Args:
        value: Numeric value
        metric_type: Type of metric ('currency', 'percentage', 'number', 'multiplier')
    """
    if pd.isna(value):
        return "N/A"
    
    if metric_type == 'currency':
        return f"${value:,.2f}"
    elif metric_type == 'percentage':
        return f"{value:.2f}%"
    elif metric_type == 'multiplier':
        return f"{value:.2f}x"
    elif metric_type == 'number':
        return f"{int(value):,}"
    else:
        return f"{value:.2f}"

def generate_summary_report(comparison_df: pd.DataFrame) -> None:
    """
    Generate a summary report comparing periods.
    """
    print("\n" + "="*80)
    print("CAMPAIGN PERFORMANCE COMPARISON REPORT")
    print("="*80)
    
    # Filter to campaigns with activity in current period
    active = comparison_df[comparison_df['metrics.impressions_current'] > 0].copy()
    
    if len(active) == 0:
        print("No active campaigns in current period.")
        return
    
    print(f"\nAnalyzing {len(active)} active campaigns\n")
    
    # Summary table
    print(f"{'Campaign':<40} {'Spend':>12} {'Conv':>8} {'CPA':>10} {'Change':>10} Status")
    print("-" * 95)
    
    for _, row in active.iterrows():
        campaign = row['campaign_name'][:38]
        spend_curr = row.get('cost_dollars_current', 0)
        conv_curr = row.get('metrics.conversions_current', 0)
        cpa_curr = row.get('cpa_current', 0)
        conv_change = row.get('metrics.conversions_change_pct', 0)
        
        # Determine status
        if conv_change > 20:
            status = "🟢"
        elif conv_change < -20:
            status = "🔴"
        else:
            status = "🟡"
        
        print(f"{campaign:<40} {format_metric(spend_curr, 'currency'):>12} "
              f"{format_metric(conv_curr, 'number'):>8} "
              f"{format_metric(cpa_curr, 'currency'):>10} "
              f"{conv_change:>9.1f}% {status}")
    
    # Account totals
    print("\n" + "-" * 95)
    total_spend_curr = active['cost_dollars_current'].sum()
    total_spend_prev = active['cost_dollars_previous'].sum()
    total_conv_curr = active['metrics.conversions_current'].sum()
    total_conv_prev = active['metrics.conversions_previous'].sum()
    
    spend_change = calculate_percent_change(total_spend_curr, total_spend_prev)
    conv_change = calculate_percent_change(total_conv_curr, total_conv_prev)
    
    print(f"{'TOTAL':<40} {format_metric(total_spend_curr, 'currency'):>12} "
          f"{format_metric(total_conv_curr, 'number'):>8} "
          f"{'':>10} {conv_change:>9.1f}%")
    
    print(f"\nAccount-level change: Spend {spend_change:+.1f}%, Conversions {conv_change:+.1f}%")

def main():
    """Main execution function."""
    if len(sys.argv) < 3:
        print("Usage: python analyze_campaigns.py <current_period.csv> <previous_period.csv>")
        sys.exit(1)
    
    current_csv = sys.argv[1]
    previous_csv = sys.argv[2]
    
    print(f"Loading data...")
    print(f"  Current period: {current_csv}")
    print(f"  Previous period: {previous_csv}")
    
    # Load data
    current_df = load_campaign_data(current_csv)
    previous_df = load_campaign_data(previous_csv)
    
    print(f"\n✓ Loaded {len(current_df)} campaigns (current), {len(previous_df)} campaigns (previous)")
    
    # Convert currency fields FIRST
    print("\nConverting currency fields...")
    current_df = convert_currency_fields(current_df)
    previous_df = convert_currency_fields(previous_df)
    
    # Calculate derived metrics
    print("\nCalculating derived metrics...")
    current_df = calculate_derived_metrics(current_df)
    previous_df = calculate_derived_metrics(previous_df)
    
    print("✓ Calculated: CTR, Conversion Rate, CPA, ROAS")
    
    # Compare periods
    print("\nComparing periods...")
    comparison = compare_periods(current_df, previous_df)
    
    print(f"✓ Compared {len(comparison)} campaigns")
    
    # Generate report
    generate_summary_report(comparison)
    
    # Optionally save to CSV
    output_file = '/tmp/campaign_comparison.csv'
    comparison.to_csv(output_file, index=False)
    print(f"\n✓ Full comparison saved to: {output_file}")

if __name__ == "__main__":
    main()
```

### Example Usage

```bash
# Query current and previous periods with mcc-gaql
CURRENT_QUERY='SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2026-03-01" AND segments.date <= "2026-03-31" AND campaign.status = "ENABLED"'

PREVIOUS_QUERY='SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value, metrics.search_impression_share, metrics.search_budget_lost_impression_share, metrics.search_rank_lost_impression_share FROM campaign WHERE segments.date >= "2026-02-01" AND segments.date <= "2026-02-28" AND campaign.status = "ENABLED"'

# Execute queries
mcc-gaql --profile myprofile --format csv -o /tmp/current.csv "$CURRENT_QUERY"
mcc-gaql --profile myprofile --format csv -o /tmp/previous.csv "$PREVIOUS_QUERY"

# Run analysis
python analyze_campaigns.py /tmp/current.csv /tmp/previous.csv
```

### Expected Output

```
Loading data...
  Current period: /tmp/current.csv
  Previous period: /tmp/previous.csv

✓ Loaded 12 campaigns (current), 12 campaigns (previous)

Converting currency fields...
✓ Converted cost_micros (range: $45.23 - $8,234.56)

Calculating derived metrics...
✓ Calculated: CTR, Conversion Rate, CPA, ROAS

Comparing periods...
✓ Compared 12 campaigns

================================================================================
CAMPAIGN PERFORMANCE COMPARISON REPORT
================================================================================

Analyzing 12 active campaigns

Campaign                                        Spend      Conv        CPA     Change Status
-----------------------------------------------------------------------------------------------
Brand Search                                 $2,456.78       67    $36.67      +15.2% 🟢
Generic Keywords                             $8,234.56      134    $61.45      -12.3% 🔴
Product Campaign - Electronics               $1,987.34       45    $44.16      +8.7% 🟡
...

-----------------------------------------------------------------------------------------------
TOTAL                                       $24,567.89      567             +12.3%

Account-level change: Spend +18.5%, Conversions +12.3%

✓ Full comparison saved to: /tmp/campaign_comparison.csv
```

### Customization Tips

1. **Add more metrics**: Extend the `metrics` list in `compare_periods()`
2. **Custom status logic**: Modify `generate_summary_report()` status rules
3. **Export formats**: Add JSON, Excel, or HTML export options
4. **Filtering**: Add command-line arguments for campaign filters
5. **Visualizations**: Add matplotlib/seaborn charts for trend analysis

### Integration with PDF Reports

To use this in PDF report generation:

```python
# After running comparison
comparison_df = compare_periods(current_df, previous_df)

# Convert to HTML table for PDF
html_table = comparison_df.to_html(
    columns=['campaign_name', 'cost_dollars_current', 'metrics.conversions_current', 
             'cpa_current', 'metrics.conversions_change_pct'],
    float_format=lambda x: f'{x:.2f}',
    classes='table'
)

# Include in PDF HTML template
# (See references/output/pdf_generation_reference.md)
```
