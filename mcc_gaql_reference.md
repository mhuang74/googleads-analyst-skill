#### Understanding mcc-gaql Tool Capabilities

**Key Features:**
- Query multiple child accounts under an MCC
- Profile-based authentication (`--profile` flag)
- Export results to CSV/files (`-o` flag)
- List available child accounts (`--list-child-accounts`)
- Natural language query conversion (`-n` flag)
- Grouped and sorted results (`--groupby`, `--sortby`)

**Common Options:**
```bash
--profile <PROFILE>        # Use named profile for authentication
-o <OUTPUT>                # Save output to file (useful for large datasets)
--list-child-accounts      # List all accessible child accounts
-c <CUSTOMER_ID>           # Query specific customer account
--keep-going               # Continue on errors across multiple accounts
```

#### Step-by-Step Query Process

**Step 1: Verify Access and List Accounts**

Before querying data, verify you have access to the account:

```bash
mcc-gaql --profile <PROFILE_NAME> --list-child-accounts
```

This returns a table with:
- `customer_client.id` - Account ID
- `customer_client.descriptive_name` - Account name
- `customer_client.currency_code` - Currency
- `customer_client.time_zone` - Timezone

**Step 2: Test Data Availability**

Check what data exists before running full queries:

```bash
# Check for recent campaign data
mcc-gaql --profile <PROFILE_NAME> 'SELECT campaign.id, campaign.name, campaign.status FROM campaign WHERE segments.date DURING LAST_30_DAYS LIMIT 10'

# Check for active campaigns with impressions
mcc-gaql --profile <PROFILE_NAME> 'SELECT campaign.name, metrics.impressions, metrics.clicks FROM campaign WHERE campaign.status = "ENABLED" AND segments.date DURING LAST_30_DAYS AND metrics.impressions > 0'

# View daily breakdown to find latest data date
mcc-gaql --profile <PROFILE_NAME> 'SELECT segments.date, campaign.name, metrics.impressions FROM campaign WHERE campaign.status = "ENABLED" AND segments.date DURING LAST_30_DAYS ORDER BY segments.date DESC LIMIT 20'
```

**Step 3: Determine Appropriate Date Ranges**

IMPORTANT: Google Ads data typically has a 1-2 day lag. Always check what the latest available date is before querying.

**Date Range Methods:**

1. **Predefined Date Ranges** (use these for exploratory queries):
   ```sql
   WHERE segments.date DURING LAST_7_DAYS
   WHERE segments.date DURING LAST_30_DAYS
   WHERE segments.date DURING THIS_MONTH
   WHERE segments.date DURING LAST_MONTH
   ```

2. **Explicit Date Ranges** (RECOMMENDED for period comparison):
   ```sql
   WHERE segments.date BETWEEN "2025-10-12" AND "2025-10-18"
   WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18"
   ```

3. **Single Date Queries** (useful for testing):
   ```sql
   WHERE segments.date = "2025-10-18"
   ```

**Step 4: Query with Full Metrics**

**Standard Metrics to Retrieve:**
- **Campaign identification**: `campaign.id`, `campaign.name`, `campaign.status`
- **Impression metrics**: `metrics.impressions`, `metrics.impression_share`
- **Click metrics**: `metrics.clicks`, `metrics.ctr`
- **Cost metrics**: `metrics.cost_micros`, `metrics.average_cpc`
- **Conversion metrics**: `metrics.conversions`, `metrics.conversions_value`
- **Date segmentation**: `segments.date` (when analyzing trends)

**⚠️ CRITICAL: Some metrics aren't compatible in the same query**

The following metrics may cause queries to return no results when combined:
- `metrics.conversion_rate` - Often unavailable, calculate manually instead
- `metrics.cost_per_conversion` - Often unavailable, calculate manually instead

**Recommended Query Structure:**

```bash
# Save to file for easier parsing (RECOMMENDED approach)
mcc-gaql --profile <PROFILE_NAME> -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

# For previous period
mcc-gaql --profile <PROFILE_NAME> -o /tmp/previous_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value FROM campaign WHERE segments.date >= "2025-10-05" AND segments.date <= "2025-10-11" AND campaign.status = "ENABLED"'
```

Then read the CSV files to parse and analyze the data.

#### Detailed Query Examples

**Example 1: Basic Campaign Performance Query**
```bash
mcc-gaql --profile themade 'SELECT campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros FROM campaign WHERE segments.date = "2025-10-18"'
```

**Example 2: Active Campaigns with Activity Filter**
```bash
mcc-gaql --profile themade 'SELECT campaign.id, campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros FROM campaign WHERE campaign.status = "ENABLED" AND segments.date DURING LAST_30_DAYS AND metrics.impressions > 0 ORDER BY metrics.cost_micros DESC'
```

**Example 3: Daily Trend Analysis**
```bash
mcc-gaql --profile themade -o /tmp/daily_trends.csv 'SELECT segments.date, campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros, metrics.conversions FROM campaign WHERE campaign.status = "ENABLED" AND segments.date BETWEEN "2025-10-01" AND "2025-10-18" ORDER BY segments.date DESC, metrics.cost_micros DESC'
```

**Example 4: Period Aggregated Data (No Date Segment)**
```bash
# When you want totals for a period without daily breakdown
mcc-gaql --profile themade 'SELECT campaign.id, campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18"'
```

#### Common Query Issues and Solutions

**Issue 1: Query returns no output**
- **Cause**: Incompatible metric combinations or date range in the future
- **Solution**:
  1. First test with single date: `WHERE segments.date = "2025-10-18"`
  2. Remove problematic metrics like `metrics.conversion_rate`, `metrics.cost_per_conversion`
  3. Verify date range has data using: `WHERE segments.date DURING LAST_30_DAYS`

**Issue 2: Missing data for recent dates**
- **Cause**: Google Ads data has 1-2 day lag
- **Solution**: Always query a test date first to see latest available data

**Issue 3: Different results between queries**
- **Cause**: Date ranges not aligned, or timezone differences
- **Solution**: Use explicit BETWEEN syntax and verify account timezone

**Issue 4: Costs showing as huge numbers**
- **Cause**: Costs are in micros (1,000,000 = $1.00 USD)
- **Solution**: Divide by 1,000,000 to get actual currency value

#### Calculating Derived Metrics

Since some metrics aren't available directly in queries, calculate them from the data:

```python
# Cost conversion (from micros)
cost_dollars = cost_micros / 1_000_000

# CTR (if needed)
ctr = (clicks / impressions) * 100 if impressions > 0 else 0

# Conversion Rate
conversion_rate = (conversions / clicks) * 100 if clicks > 0 else 0

# Cost Per Conversion
cost_per_conversion = cost_dollars / conversions if conversions > 0 else 0

# ROAS (Return on Ad Spend)
roas = conversions_value / cost_dollars if cost_dollars > 0 else 0

# Average CPC (if needed)
avg_cpc = cost_dollars / clicks if clicks > 0 else 0
```

#### Best Practices for Querying

1. **Always use profile flag**: `--profile <PROFILE_NAME>` for authentication
2. **Filter to enabled campaigns**: `campaign.status = "ENABLED"` unless analyzing removed campaigns
3. **Use explicit date ranges**: `BETWEEN "YYYY-MM-DD" AND "YYYY-MM-DD"` for clarity
4. **Save to files for large datasets**: `-o /tmp/filename.csv` makes parsing easier
5. **Test with LIMIT first**: Add `LIMIT 10` when testing queries
6. **Order by spend**: `ORDER BY metrics.cost_micros DESC` to see highest-spend campaigns first
7. **Check for zero impressions**: Add `metrics.impressions > 0` to filter out inactive periods
8. **Query periods separately**: Don't try to get both periods in one query; run two queries for clearer comparison


```
mcc-gaql 0.12.0
Michael S. Huang <mhuang74@gmail.com>
Efficiently run Google Ads GAQL query across one or more child accounts linked to MCC.

Supports profile-based configuration and ENV VAR override.

USAGE:
    mcc-gaql [OPTIONS] [GAQL_QUERY]

ARGS:
    <GAQL_QUERY>
            Google Ads GAQL query to run

OPTIONS:
    -a, --all-linked-child-accounts
            Force query to run across all linked child accounts (some may not be accessible)

    -c, --customer-id <CUSTOMER_ID>
            Apply query to a single CustomerID. Or use with `--all-linked-child-accounts` to query
            all child accounts

    -f, --field-service
            Query GoogleAdsFieldService to retrieve available fields

        --groupby <GROUPBY>
            Group by columns

    -h, --help
            Print help information

        --keep-going
            Keep going on errors

    -l, --list-child-accounts
            List all child accounts under MCC

    -n, --natural-language
            Use natural language prompt instead of GAQL; prompt is converted to GAQL via LLM

    -o, --output <OUTPUT>
            GAQL output filename

    -p, --profile <PROFILE>
            Query using default MCC and Child CustomerIDs file specified for this profile

    -q, --stored-query <STORED_QUERY>
            Load named query from file

        --sortby <SORTBY>
            Sort by columns

    -V, --version
            Print version information
```