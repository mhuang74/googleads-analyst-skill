# Geographic Reporting Views: geographic_view vs user_location_view

## Overview

Google Ads API provides two views for geographic performance reporting. They answer different questions and should be used for different analytical purposes.

| | **geographic_view** | **user_location_view** |
|---|---|---|
| **Question it answers** | *Where was the user when they saw the ad?* | *Was the user's location in my campaign targeting?* |
| **Focus** | User behavior (physical vs. interest) | Campaign targeting alignment |
| **Resource pattern** | `customers/{id}/geographicViews/{country}~{type}` | `customers/{id}/userLocationViews/{country}~{bool}` |
| **Type values** | `LOCATION_OF_PRESENCE` — physically there<br>`AREA_OF_INTEREST` — interested in area | `true` — matches campaign targeting<br>`false` — outside targeting |

---

## When to Use Each View

### Use geographic_view when:

**→ You want to understand user intent vs. physical presence**

| Scenario | Insight |
|----------|---------|
| "Are locals or tourists searching for my business?" | Compare `LOCATION_OF_PRESENCE` vs `AREA_OF_INTEREST` for the same city |
| "Is my museum attracting Bay Area residents or just researchers?" | High `LOCATION_OF_PRESENCE` = locals; High `AREA_OF_INTEREST` = outside interest |
| "Where are my customers physically located?" | Filter to `LOCATION_OF_PRESENCE` only |

**Best for:** Local businesses, tourism/hospitality, real estate — any business where physical presence matters.

**Example:** A museum in Oakland wants to know if their ads reach actual Oakland residents or just people researching Oakland from elsewhere.

---

### Use user_location_view when:

**→ You want to analyze campaign targeting effectiveness**

| Scenario | Insight |
|----------|---------|
| "Am I getting traffic from places I didn't target?" | Check `false` values for targeting leakage |
| "How much budget is going to untargeted locations?" | Sum cost for `false` rows |
| "Is my geo-targeting working correctly?" | Compare `true` vs `false` distribution |

**Best for:** Budget efficiency analysis, troubleshooting unexpected traffic, validating targeting setup.

**Example:** A national campaign targeting only the US finds 15% of impressions have `false` — indicating international traffic leakage.

---

## Key Differences in Data

### Resource Name Structure

| View | Resource Name Example | Meaning |
|------|----------------------|---------|
| **geographic_view** | `customers/123/geographicViews/2840~LOCATION_OF_PRESENCE` | Country 2840 (US), user was physically there |
| **geographic_view** | `customers/123/geographicViews/2840~AREA_OF_INTEREST` | Country 2840 (US), user showed interest from elsewhere |
| **user_location_view** | `customers/123/userLocationViews/2840~true` | Country 2840, location matches campaign targeting |
| **user_location_view** | `customers/123/userLocationViews/2840~false` | Country 2840, location outside campaign targeting |

### Why Totals May Differ

For the same account and date range, city totals may differ between views:

| View | Oakland Impressions | Why |
|------|---------------------|-----|
| **geographic_view** | 1,374 | Aggregates: 954 (`LOCATION_OF_PRESENCE`) + 420 (`AREA_OF_INTEREST`) |
| **user_location_view** | 420 | Shows only `true` (targeted) traffic; excludes `false` |

**geographic_view** can have higher totals because it combines both presence types for each city.
**user_location_view** segments by targeting match, potentially splitting a city's traffic across `true`/`false`.

---

## GAQL Query Examples

### geographic_view — City Performance by Presence Type

```sql
SELECT 
  geographic_view.resource_name,
  geographic_view.location_type,
  segments.geo_target_city,
  segments.geo_target_state,
  metrics.clicks,
  metrics.impressions,
  metrics.cost_micros
FROM geographic_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.impressions DESC
```

### user_location_view — City Performance by Targeting Match

```sql
SELECT 
  user_location_view.resource_name,
  segments.geo_target_city,
  segments.geo_target_state,
  metrics.clicks,
  metrics.impressions,
  metrics.cost_micros
FROM user_location_view
WHERE segments.date DURING LAST_30_DAYS
ORDER BY metrics.impressions DESC
```

---

## Decision Tree

```
Need to segment by user's PHYSICAL vs INTEREST location?
└── YES → geographic_view
    ├── LOCATION_OF_PRESENCE → User was physically there
    └── AREA_OF_INTEREST → User searched about the area

Need to compare TRAFFIC INSIDE vs OUTSIDE targeting?
└── YES → user_location_view
    ├── true → Location was in campaign targeting
    └── false → Location outside campaign targeting

Just need city-level performance rankings?
└── Both work → Results will be similar but totals may differ
```

---

## Quick Reference Table

| Your Question | Recommended View | Key Field |
|-------------|------------------|-----------|
| "Which cities drive the most traffic?" | Either | `segments.geo_target_city` |
| "Are locals or outsiders seeing my ads?" | **geographic_view** | `geographic_view.location_type` |
| "Is my targeting leaking to untargeted areas?" | **user_location_view** | Resource name suffix (`~true`/`~false`) |
| "How much spend goes to non-targeted locations?" | **user_location_view** | Filter resource name for `~false` |
| "Do people visit my city or just research it?" | **geographic_view** | Compare presence vs interest for same city |

---

## Compatibility Notes

Both views support:
- ✅ `segments.geo_target_city`
- ✅ `segments.geo_target_state`
- ✅ `segments.geo_target_region`
- ✅ Standard metrics (clicks, impressions, cost, conversions)

Neither view supports:
- ❌ `segments.geo_target_country` (use `country_criterion_id` resource field instead)

---

## Real-World Example: The MADE (themuseumoffun)

| Metric | geographic_view | user_location_view |
|--------|-----------------|-------------------|
| **Top city** | Oakland (1,374 imp) | San Francisco (953 imp) |
| **Why different?** | Combines presence + interest | Shows only targeted traffic |
| **Use case** | "Are Bay Area locals finding us?" | "Is my targeting reaching the right cities?" |

For this hyper-local Oakland museum, **geographic_view** revealed more total reach including people interested in Oakland from elsewhere, while **user_location_view** showed the core targeted audience distribution.
