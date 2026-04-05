### Appendix Requirements

**ALWAYS include an Appendix section** with raw data tables at the end of every report.

The Appendix must contain:

1. **Current Period Raw Data Table**
   - All campaigns with complete metrics from the current analysis period
   - Include both micros and dollar amounts for cost/CPC
   - Account totals row at the bottom

2. **Previous Period Raw Data Table**
   - Same format as current period for easy comparison
   - All campaigns with baseline data
   - Account totals row at the bottom

**Required columns in raw data tables:**
- Campaign ID
- Campaign Name
- Status
- Impressions, Clicks, CTR
- Cost (in micros AND dollars)
- Average CPC (in micros AND dollars)
- Conversions
- Conversion Value
- Search Impression Share (if available)
- Budget Lost Impression Share (if available)
- Rank Lost Impression Share (if available)

**Example Appendix Format:**

```markdown
## Appendix: Raw Data & Detailed Metrics

### Current Period Raw Data (Sept 20 - Oct 19, 2025)

| Campaign ID | Campaign Name | Status | Impressions | Clicks | CTR | Cost (Micros) | Cost ($) | Avg CPC (Micros) | Avg CPC ($) | Conversions | Conv Value |
|-------------|---------------|--------|-------------|--------|-----|---------------|----------|------------------|-------------|-------------|------------|
| 12345678 | Campaign A | Enabled | 1,000 | 50 | 5.00% | 100,000,000 | $100.00 | 2,000,000 | $2.00 | 10 | 500.0 |
| 23456789 | Campaign B | Enabled | 2,000 | 100 | 5.00% | 200,000,000 | $200.00 | 2,000,000 | $2.00 | 20 | 1000.0 |
| **TOTALS** | | | **3,000** | **150** | **5.00%** | **300,000,000** | **$300.00** | **2,000,000** | **$2.00** | **30** | **1,500.0** |

### Previous Period Raw Data (Aug 21 - Sept 19, 2025)

| Campaign ID | Campaign Name | Status | Impressions | Clicks | CTR | Cost (Micros) | Cost ($) | Avg CPC (Micros) | Avg CPC ($) | Conversions | Conv Value |
|-------------|---------------|--------|-------------|--------|-----|---------------|----------|------------------|-------------|-------------|------------|
| 12345678 | Campaign A | Enabled | 800 | 40 | 5.00% | 80,000,000 | $80.00 | 2,000,000 | $2.00 | 8 | 400.0 |
| 23456789 | Campaign B | Enabled | 1,500 | 75 | 5.00% | 150,000,000 | $150.00 | 2,000,000 | $2.00 | 15 | 750.0 |
| **TOTALS** | | | **2,300** | **115** | **5.00%** | **230,000,000** | **$230.00** | **2,000,000** | **$2.00** | **23** | **1,150.0** |
```

This provides complete transparency into the raw data used for all analysis and calculations in the report.
