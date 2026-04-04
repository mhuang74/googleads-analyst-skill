# Validation Enhancement Summary

**Date:** April 4, 2026  
**Enhancement:** Added mandatory GAQL query validation instructions throughout the skill

## Overview

Added comprehensive instructions to **always validate GAQL queries** using `mcc-gaql --validate` before execution. This prevents wasted time on queries with hallucinated fields, syntax errors, or field incompatibilities.

---

## Why This Matters

### Problem
LLMs (including mcc-gaql-gen) can hallucinate non-existent field names or generate invalid GAQL:
- ❌ `metrics.video_views` (doesn't exist)
- ✅ `metrics.video_trueview_views` (correct)

Without validation, you only discover errors after execution failure, wasting time and effort.

### Solution
**Mandatory validation** catches errors immediately:
1. Generate/write query
2. **Validate** with `mcc-gaql --validate`
3. Execute only if validation passes

---

## Changes Made to SKILL.md

### 1. Section 2: Querying Campaign Data (New Subsection)

**Added:** "⚠️ CRITICAL: Always Validate GAQL Before Executing"

**Content:**
- 2-step workflow: Validate first, then execute
- Validation command examples
- Why validation is critical
- What to do if validation fails
- Updated examples to include validation

**Before:**
```bash
mcc-gaql --profile themade --format csv -o output.csv 'SELECT ...'
```

**After:**
```bash
# STEP 1: Validate first
QUERY='SELECT ...'
mcc-gaql --profile themade --validate "$QUERY"

# STEP 2: Execute only if validation passes
mcc-gaql --profile themade --format csv -o output.csv "$QUERY"
```

### 2. Section 4: Dynamic Investigation Mode (Enhanced)

**Added:** "⚠️ CRITICAL: 3-Step Process for Every Query"

**Content:**
- Never skip validation directive
- Complete workflow with validation checks
- Bash script examples with exit code checking
- Enhanced fallback strategy with detailed error handling

**Key Addition:**
```bash
# STEP 1: Generate
QUERY=$(mcc-gaql-gen generate "..." --use-query-cookbook)

# STEP 2: VALIDATE (REQUIRED)
mcc-gaql --profile <PROFILE> --validate "$QUERY"

# STEP 3: Execute only if validation passed
if [ $? -eq 0 ]; then
  mcc-gaql --profile <PROFILE> --format json -o output.json "$QUERY"
fi
```

**Added:** "Why Validation is Critical" with 5 key benefits
- Catches hallucinated fields
- Detects field incompatibilities
- Validates syntax
- Saves time
- No quota cost

### 3. Example Interactions (Updated)

**Updated Example 1 & 2:**
- Step 2 now includes validation before execution
- Uses query variables for reuse
- Shows validation commands explicitly
- Added echo statements for clarity

**Before:**
```bash
mcc-gaql --profile themade --format csv -o /tmp/current.csv 'SELECT ...'
```

**After:**
```bash
CURRENT_QUERY='SELECT ...'

echo "Validating current period query..."
mcc-gaql --profile themade --validate "$CURRENT_QUERY"

echo "Executing queries..."
mcc-gaql --profile themade --format csv -o /tmp/current.csv "$CURRENT_QUERY"
```

**Updated Step 5b (Dynamic Investigation):**
- Shows complete 3-step process
- Uses query variables
- Includes exit code checking
- Demonstrates two separate investigations with validation

### 4. Best Practices (New Section Added)

**Added:** "Query Best Practices" section (before Reporting Best Practices)

**Content:**
1. **NEVER skip validation** - mandatory for all queries
2. **Use 3-step workflow** - generate/define, validate, execute
3. **When validation fails** - systematic troubleshooting
4. **Common validation errors** - examples with fixes
5. **Store queries in variables** - for reuse and clarity
6. **Validation benefits** - 6 key advantages

**Example common errors:**
```
Field 'metrics.video_views' does not exist 
→ Use 'metrics.video_trueview_views'

Field incompatibility 
→ Check which fields can be selected together

Invalid WHERE clause 
→ Verify date format, enum values, filter syntax
```

### 5. Final Checklist (New Section Added)

**Added:** "Query Execution Quality" checklist (before Content Quality)

**New items:**
- [ ] **ALL GAQL queries validated** using `mcc-gaql --validate`
- [ ] No queries executed without successful validation (exit code 0)
- [ ] Validation errors fixed before execution
- [ ] For mcc-gaql-gen queries: validated every generated query
- [ ] Used `--show-fields` to verify field names when validation failed
- [ ] No hallucinated fields in executed queries

---

## Validation Workflow

### For Manual Queries

```bash
# Define query
QUERY='SELECT campaign.name, metrics.impressions 
       FROM campaign 
       WHERE segments.date DURING LAST_7_DAYS'

# VALIDATE (required)
mcc-gaql --profile <PROFILE> --validate "$QUERY"

# Execute only if validation passes
if [ $? -eq 0 ]; then
  mcc-gaql --profile <PROFILE> --format csv -o output.csv "$QUERY"
else
  echo "Validation failed - fix query before executing"
fi
```

### For Generated Queries (mcc-gaql-gen)

```bash
# STEP 1: Generate
PROMPT="Show campaign performance with impressions and cost for last 7 days"
QUERY=$(mcc-gaql-gen generate "$PROMPT" --use-query-cookbook)

# STEP 2: VALIDATE (required)
mcc-gaql --profile <PROFILE> --validate "$QUERY"

# Check result
if [ $? -eq 0 ]; then
  echo "✅ Query validated successfully"
  
  # STEP 3: Execute
  mcc-gaql --profile <PROFILE> --format json -o output.json "$QUERY"
else
  echo "❌ Validation failed"
  
  # Fallback: Check fields
  mcc-gaql --show-fields campaign
  
  # Fix and regenerate
fi
```

---

## Error Handling Guide

### When Validation Fails

**Step 1: Read the error message**
```
Error: Field 'metrics.video_views' does not exist on resource 'campaign'
```

**Step 2: Find correct field name**
```bash
# Option A: Search metadata
mcc-gaql-gen metadata "video"

# Option B: Show all fields for resource
mcc-gaql --show-fields campaign | grep video
```

**Step 3: Fix the query**
```bash
# Wrong:
SELECT metrics.video_views FROM campaign

# Correct:
SELECT metrics.video_trueview_views FROM campaign
```

**Step 4: Re-validate**
```bash
mcc-gaql --validate "$CORRECTED_QUERY"
```

**Step 5: Execute only after validation passes**

---

## Common Validation Errors

### 1. Hallucinated Field Names

**Error:**
```
Field 'metrics.video_views' does not exist
```

**Fix:**
```bash
# Check metadata for correct name
mcc-gaql-gen metadata "video_views"
# Returns: No matching fields

mcc-gaql-gen metadata "video_trueview"
# Returns: metrics.video_trueview_views
```

### 2. Field Incompatibility

**Error:**
```
Cannot select 'ad_group_criterion.keyword.text' with 'FROM campaign'
```

**Fix:**
```bash
# Use correct resource
# Wrong: FROM campaign
# Correct: FROM keyword_view
```

### 3. Invalid Enum Values

**Error:**
```
Invalid value for campaign.advertising_channel_type: 'PMAX'
```

**Fix:**
```bash
# Check valid enum values
mcc-gaql-gen metadata "campaign.advertising_channel_type"
# Shows: PERFORMANCE_MAX (not PMAX)
```

### 4. Syntax Errors

**Error:**
```
Expected 'FROM' but found 'WHERE'
```

**Fix:**
```sql
-- Wrong:
SELECT campaign.name 
WHERE campaign.status = 'ENABLED'

-- Correct:
SELECT campaign.name 
FROM campaign
WHERE campaign.status = 'ENABLED'
```

---

## Benefits of Mandatory Validation

| Benefit | Impact |
|---------|--------|
| **Catch errors early** | Fix issues before wasting time on execution |
| **Prevent hallucinated fields** | LLMs can't invent field names that pass validation |
| **Faster debugging** | Validation errors are specific and actionable |
| **No quota waste** | Validation doesn't count against API quota |
| **Better learning** | See correct field names immediately |
| **Confidence** | Know queries will work before running them |

---

## Example: Before vs After

### Before (No Validation)

```bash
# Generate query
QUERY=$(mcc-gaql-gen generate "show campaigns with video views")

# Execute directly (RISKY)
mcc-gaql --profile profile1 --format csv -o output.csv "$QUERY"

# ERROR during execution: Field 'metrics.video_views' does not exist
# Now you have to:
# 1. Figure out what went wrong
# 2. Find the correct field name
# 3. Fix the query
# 4. Re-execute
# Total time wasted: 5-10 minutes
```

### After (With Validation)

```bash
# Generate query
QUERY=$(mcc-gaql-gen generate "show campaigns with video views")

# VALIDATE first (SAFE)
mcc-gaql --profile profile1 --validate "$QUERY"

# ERROR immediately: Field 'metrics.video_views' does not exist
# Find correct field:
mcc-gaql-gen metadata "video" | grep views
# → metrics.video_trueview_views

# Fix and validate again:
QUERY=$(mcc-gaql-gen generate "show campaigns with video_trueview_views")
mcc-gaql --profile profile1 --validate "$QUERY"
# ✅ Validation passed

# Execute with confidence
mcc-gaql --profile profile1 --format csv -o output.csv "$QUERY"

# Total time saved: 3-7 minutes per error
```

---

## Files Updated

- **SKILL.md** - Added validation instructions in 5 sections:
  1. Section 2: Querying Campaign Data
  2. Section 4: Dynamic Investigation Mode
  3. Example Interactions (Example 1 & 2)
  4. Best Practices (new "Query Best Practices" section)
  5. Final Checklist (new "Query Execution Quality" section)

---

## Usage Guidelines

**When to validate:**
- ✅ Always - for every single GAQL query
- ✅ Manual queries - before first execution
- ✅ Generated queries - every mcc-gaql-gen output
- ✅ Modified queries - after any changes
- ✅ Copy-pasted queries - even from documentation

**When NOT to skip validation:**
- ❌ Never - there's no scenario where skipping is acceptable
- ❌ Even for "simple" queries - validation is instant
- ❌ Even if query worked before - API fields can change
- ❌ Even if you're confident - humans make typos

**Command format:**
```bash
# With profile
mcc-gaql --profile <PROFILE_NAME> --validate "<QUERY>"

# Without profile
mcc-gaql --mcc <MCC> --customer-id <ID> --user <EMAIL> --validate "<QUERY>"

# Exit code
# 0 = validation passed (query is valid)
# non-zero = validation failed (query has errors)
```

---

## Integration with Existing Workflow

The validation enhancement integrates seamlessly:

1. **Step 1-3** (existing) - Gather info, query baseline data
   - **Now includes:** Validation before each query execution

2. **Step 4** (existing) - Dynamic Investigation Mode
   - **Now includes:** Validation in 3-step process (generate, validate, execute)

3. **Step 5** (existing) - Generate report
   - **No change** - validation happened earlier

4. **Checklist** (existing) - Quality checks
   - **Now includes:** Query execution quality checks

---

## Key Takeaways

1. **Validation is mandatory** - not optional, for every query
2. **Validate before execute** - always in this order
3. **Use exit codes** - check if validation passed before executing
4. **Store in variables** - reuse query between validation and execution
5. **Fix errors systematically** - use --show-fields, check metadata
6. **No quota cost** - validation is free, execution costs quota

---

## Success Metrics

**Expected improvements:**
- ⬇️ 90% reduction in query execution errors
- ⬇️ 80% reduction in debugging time for field issues
- ⬆️ 100% of queries validated before execution
- ⬆️ Faster time-to-insight (fewer error cycles)
- ⬆️ Higher confidence in generated queries

---

## Last Updated
**Date:** April 4, 2026  
**By:** Enhanced all query execution workflows with mandatory validation  
**Result:** Comprehensive validation instructions integrated throughout skill
