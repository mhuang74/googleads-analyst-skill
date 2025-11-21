#### Understanding mcc-gaql Tool Capabilities

**Key Features:**
- Query multiple child accounts under an MCC
- Profile-based authentication (`--profile` flag) OR explicit parameters
- LLM friendly output format (`--format csv` flag)
- Export results to CSV/files (`-o` flag)
- List available child accounts (`--list-child-accounts`)
- Natural language query conversion (`-n` flag)
- Grouped and sorted results (`--groupby`, `--sortby`)

**Two Ways to Use mcc-gaql:**

1. **With a configured profile** (recommended for repeated use):
   ```bash
   mcc-gaql --profile <PROFILE_NAME> 'SELECT ...'
   ```

2. **Without a profile** (requires explicit parameters):
   ```bash
   mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> 'SELECT ...'
   ```

**Common Options:**
```bash
--profile <PROFILE>        # Use named profile for authentication
--mcc-id <MCC_ID>          # MCC (Manager) Customer ID (required if no profile)
-c, --customer-id <ID>     # Query specific customer account (required if no profile)
--user-email <EMAIL>       # User email for OAuth2 authentication (required if no profile)
--format <FORMAT>          # Output format: use `csv` for daily data, otherwise, `json`
-o <OUTPUT>                # Save output to file (useful for large datasets)
--list-child-accounts      # List all accessible child accounts
--keep-going               # Continue on errors across multiple accounts
--show-fields <RESOURCE>   # Show valid fields for a resource (e.g., campaign, ad_group)
```

**Required Parameters When NOT Using --profile:**
- `--mcc-id <MCC_ID>`: The Manager Customer ID (MCC account number, e.g., 1234567890)
- `--customer-id <CUSTOMER_ID>`: The child account to query (e.g., 9876543210)
- `--user-email <EMAIL>`: The Google account email with access to the account (e.g., user@example.com)

#### Discovering Valid Fields for GAQL Queries

Before writing GAQL queries, use `--show-fields <resource>` to discover valid field names for any resource type. This ensures you use correct field names and understand available attributes, metrics, and segments.

**Usage:**
```bash
# Show all fields available for the campaign resource
mcc-gaql --profile <PROFILE_NAME> --show-fields campaign

# Show fields for ad_group resource
mcc-gaql --profile <PROFILE_NAME> --show-fields ad_group

# Show fields for ad_group_ad resource
mcc-gaql --profile <PROFILE_NAME> --show-fields ad_group_ad

# Show fields for keyword_view resource
mcc-gaql --profile <PROFILE_NAME> --show-fields keyword_view
```

**Without a profile:**
```bash
mcc-gaql --mcc-id <MCC_ID> --customer-id <CUSTOMER_ID> --user-email <EMAIL> --show-fields campaign
```

**Common Resources to Query:**
- `campaign` - Campaign-level data and settings
- `ad_group` - Ad group settings and targeting
- `ad_group_ad` - Individual ads and their performance
- `ad_group_criterion` - Targeting criteria (keywords, audiences, etc.)
- `keyword_view` - Keyword performance data
- `search_term_view` - Search terms that triggered ads
- `geographic_view` - Geographic performance data
- `customer` - Account-level information

**Field Categories Returned:**
- **Attributes** - Properties of the resource (e.g., `campaign.name`, `campaign.status`)
- **Metrics** - Performance data (e.g., `metrics.impressions`, `metrics.clicks`)
- **Segments** - Dimensions for data breakdown (e.g., `segments.date`, `segments.device`)

**Pro Tip:** When building a new query, first run `--show-fields` to verify exact field names, as Google Ads API field names must match exactly (case-sensitive, with proper prefixes like `campaign.`, `metrics.`, `segments.`).

#### Step-by-Step Query Process

**Step 1: Verify Access and List Accounts**

Before querying data, verify you have access to the account.

**With a configured profile:**
```bash
mcc-gaql --profile <PROFILE_NAME> --list-child-accounts --format json
```

**Without a profile (requires MCC, customer ID, and user email):**
```bash
mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> --list-child-accounts --format json
```

This returns a table with:
- `customer_client.id` - Account ID
- `customer_client.descriptive_name` - Account name
- `customer_client.currency_code` - Currency
- `customer_client.time_zone` - Timezone

**Step 2: Test Data Availability**

Check what data exists before running full queries.

**With a configured profile:**
```bash
# Check for recent campaign data
mcc-gaql --profile <PROFILE_NAME> --format json 'SELECT campaign.id, campaign.name, campaign.status FROM campaign WHERE segments.date DURING LAST_30_DAYS LIMIT 10'

# Check for active campaigns with impressions
mcc-gaql --profile <PROFILE_NAME> --format json 'SELECT campaign.name, metrics.impressions, metrics.clicks FROM campaign WHERE campaign.status = "ENABLED" AND segments.date DURING LAST_30_DAYS AND metrics.impressions > 0'

# View daily breakdown to find latest data date
mcc-gaql --profile <PROFILE_NAME> --format csv 'SELECT segments.date, campaign.name, metrics.impressions FROM campaign WHERE campaign.status = "ENABLED" AND segments.date DURING LAST_30_DAYS ORDER BY segments.date DESC LIMIT 20'
```

**Without a profile:**
```bash
# Check for recent campaign data
mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> --format json 'SELECT campaign.id, campaign.name, campaign.status FROM campaign WHERE segments.date DURING LAST_30_DAYS LIMIT 10'

# Check for active campaigns with impressions
mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> --format json 'SELECT campaign.name, metrics.impressions, metrics.clicks FROM campaign WHERE campaign.status = "ENABLED" AND segments.date DURING LAST_30_DAYS AND metrics.impressions > 0'
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

**With a configured profile:**
```bash
# Save to file for easier parsing (RECOMMENDED approach)
mcc-gaql --profile <PROFILE_NAME> --format csv -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

# For previous period
mcc-gaql --profile <PROFILE_NAME> --format csv -o /tmp/previous_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value FROM campaign WHERE segments.date >= "2025-10-05" AND segments.date <= "2025-10-11" AND campaign.status = "ENABLED"'
```

**Without a profile:**
```bash
# Current period
mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> --format csv -o /tmp/current_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18" AND campaign.status = "ENABLED"'

# Previous period
mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <USER_EMAIL> --format csv -o /tmp/previous_period.csv 'SELECT campaign.id, campaign.name, campaign.status, metrics.impressions, metrics.clicks, metrics.ctr, metrics.cost_micros, metrics.average_cpc, metrics.conversions, metrics.conversions_value FROM campaign WHERE segments.date >= "2025-10-05" AND segments.date <= "2025-10-11" AND campaign.status = "ENABLED"'
```

Then read the CSV files to parse and analyze the data.

#### Detailed Query Examples

**Example 1: Basic Campaign Performance Query**
```bash
mcc-gaql --profile themade --format json 'SELECT campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros FROM campaign WHERE segments.date = "2025-10-18"'
```

**Example 2: Active Campaigns with Activity Filter**
```bash
mcc-gaql --profile themade --format json 'SELECT campaign.id, campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros FROM campaign WHERE campaign.status = "ENABLED" AND segments.date DURING LAST_30_DAYS AND metrics.impressions > 0 ORDER BY metrics.cost_micros DESC'
```

**Example 3: Daily Trend Analysis**
```bash
mcc-gaql --profile themade --format csv -o /tmp/daily_trends.csv 'SELECT segments.date, campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros, metrics.conversions FROM campaign WHERE campaign.status = "ENABLED" AND segments.date BETWEEN "2025-10-01" AND "2025-10-18" ORDER BY segments.date DESC, metrics.cost_micros DESC'
```

**Example 4: Period Aggregated Data (No Date Segment)**
```bash
# When you want totals for a period without daily breakdown
mcc-gaql --profile themade --format json 'SELECT campaign.id, campaign.name, metrics.impressions, metrics.clicks, metrics.cost_micros FROM campaign WHERE segments.date >= "2025-10-12" AND segments.date <= "2025-10-18"'
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
mcc-gaql 0.13.0
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
            Apply query to a single account. If no --mcc-id is specified, this will be used as the
            MCC (for solo accounts). To query across many accounts, specify a customerids_filename
            in config file, or query across all child accounts via --all-linked-child-accounts

        --export-field-metadata
            Export field metadata summary to stdout

    -f, --field-service
            Query GoogleAdsFieldService to retrieve available fields

        --format <FORMAT>
            Output format: table, csv, json (defaults to table, or config profile default if set)

        --groupby <GROUPBY>
            Group by columns

    -h, --help
            Print help information

        --keep-going
            Keep going on errors

    -l, --list-child-accounts
            List all child accounts under MCC

    -m, --mcc-id <MCC_ID>
            MCC (Manager) Customer ID for login-customer-id header. Required unless specified in
            config profile. For solo accounts, can be omitted if --customer-id is provided

    -n, --natural-language
            Use natural language prompt instead of GAQL; prompt is converted to GAQL via LLM

    -o, --output <OUTPUT>
            GAQL output filename

    -p, --profile <PROFILE>
            Query using default MCC and Child CustomerIDs file specified for this profile

    -q, --stored-query <STORED_QUERY>
            Load named query from file

        --refresh-field-cache
            Refresh field metadata cache from Google Ads API

        --setup
            Set up configuration with interactive wizard

        --show-config
            Display current configuration and exit

        --show-fields <SHOW_FIELDS>
            Show available fields for a specific resource (e.g., campaign, ad_group)

        --sortby <SORTBY>
            Sort by columns

    -u, --user-email <USER_EMAIL>
            User email for OAuth2 authentication (auto-generates token cache)

    -V, --version
            Print version information

```