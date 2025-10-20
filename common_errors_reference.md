## Error Handling & Troubleshooting

### Common Errors and Solutions

**Error: `mcc-gaql: command not found`**
- **Cause**: Tool not installed or not in PATH
- **Solution**: Inform user to install mcc-gaql or verify installation
- **Check**: Ask user to run `which mcc-gaql` or provide installation instructions

**Error: Authentication/API errors**
- **Cause**: Invalid credentials or API access issues
- **Solution**: Suggest checking Google Ads API credentials and profile configuration
- **Check**: Verify profile exists and credentials are current

**Error: Query returns no output (empty result)**
- **Possible Causes**:
  1. Incompatible metric combinations (conversion_rate, cost_per_conversion)
  2. Date range in the future or before account existed
  3. No campaigns match filters (status = "ENABLED")
  4. No actual data for that period
- **Diagnostic Steps**:
  1. Test with single recent date: `WHERE segments.date = "2025-10-18"`
  2. Remove problematic metrics, use only basic ones
  3. Check without status filter
  4. Verify data exists with: `WHERE segments.date DURING LAST_30_DAYS`
- **Solution**: Simplify query and narrow down which component is causing the issue

**Error: "Customer not found" or access errors**
- **Cause**: Profile configured for wrong account or insufficient permissions
- **Solution**:
  1. Run `--list-child-accounts` to verify access
  2. Check MCC account ID in profile configuration
  3. Verify user has access to the account in Google Ads UI

**Error: Dates showing unexpected results**
- **Cause**: Timezone differences between system and Google Ads account
- **Solution**:
  1. Check account timezone with `--list-child-accounts`
  2. Use explicit dates and be aware data may align to account timezone
  3. Note: Google Ads data typically has 1-2 day lag

**Issue: Very different numbers than expected**
- **Cause**: Cost in micros not converted to dollars
- **Solution**: Always divide cost_micros by 1,000,000 to get actual currency value
- **Check**: If costs are in millions, you forgot to convert

**Issue: Conversion metrics showing zero across all campaigns**
- **Critical Check**: This is likely a tracking issue, not a performance issue
- **Actions**:
  1. Alert user immediately this may be tracking problem
  2. Check if previous periods had conversions
  3. Recommend verifying conversion tags in Google Ads
  4. Suggest checking Google Tag Manager and website for tag issues
