# Correct GAQL Validation Usage

**Date:** April 4, 2026  
**Issue:** Incorrect `--validate` flag usage with `mcc-gaql-gen generate`

## The Problem

`mcc-gaql-gen generate` does **NOT** accept a `--validate` flag.

### ❌ INCORRECT Usage

```bash
# This will FAIL - mcc-gaql-gen doesn't have --validate flag
mcc-gaql-gen generate "show campaign performance" --validate
```

### ✅ CORRECT Usage

There are two correct methods to validate generated queries:

#### Method 1: Pipe to mcc-gaql --validate

```bash
mcc-gaql-gen generate "show campaign performance" | mcc-gaql --profile <PROFILE> --validate
```

**Pros:**
- Single command pipeline
- Compact syntax

**Cons:**
- Can't reuse the query easily for execution
- Harder to debug if validation fails

#### Method 2: Store in Variable (RECOMMENDED)

```bash
# Step 1: Generate and store
QUERY=$(mcc-gaql-gen generate "show campaign performance" --use-query-cookbook)

# Step 2: Validate
mcc-gaql --profile <PROFILE> --validate "$QUERY"

# Step 3: Execute only if validation passed
if [ $? -eq 0 ]; then
  mcc-gaql --profile <PROFILE> --format csv -o output.csv "$QUERY"
fi
```

**Pros:**
- Same query used for validation and execution (no duplication)
- Easy to debug - can inspect `$QUERY` variable
- Clear separation of steps
- Can add logging/echo statements

**Cons:**
- Slightly more verbose

---

## Complete Example Workflows

### Basic Workflow (Manual Query)

```bash
# Define the query
QUERY='SELECT campaign.name, metrics.impressions, metrics.clicks
       FROM campaign
       WHERE segments.date DURING LAST_7_DAYS'

# Validate
mcc-gaql --profile myprofile --validate "$QUERY"

# Execute if validation passes
if [ $? -eq 0 ]; then
  mcc-gaql --profile myprofile --format csv -o results.csv "$QUERY"
fi
```

### Generated Query Workflow (Recommended)

```bash
# Step 1: Generate query
PROMPT="Show campaign performance with impressions, clicks, cost for last 7 days"
QUERY=$(mcc-gaql-gen generate "$PROMPT" --use-query-cookbook)

# Optional: Display generated query for review
echo "Generated query:"
echo "$QUERY"
echo ""

# Step 2: Validate
echo "Validating query..."
mcc-gaql --profile myprofile --validate "$QUERY"

# Step 3: Check validation result and execute
if [ $? -eq 0 ]; then
  echo "✅ Validation passed, executing query..."
  mcc-gaql --profile myprofile --format json -o results.json "$QUERY"
  echo "✅ Query executed successfully"
else
  echo "❌ Validation failed - fix query before executing"
  exit 1
fi
```

### Piped Workflow (Compact)

```bash
# Generate and validate in one line
if mcc-gaql-gen generate "show campaign performance" | mcc-gaql --profile myprofile --validate; then
  # Regenerate for execution (query is not preserved from pipe)
  QUERY=$(mcc-gaql-gen generate "show campaign performance")
  mcc-gaql --profile myprofile --format csv -o output.csv "$QUERY"
fi
```

**Note:** This approach requires generating the query twice - once for validation, once for execution.

---

## Common Patterns

### Pattern 1: Dynamic Investigation Query

```bash
# Generate investigation query
INVESTIGATION_QUERY=$(mcc-gaql-gen generate \
  "Show ad group performance for campaign 'Brand Search' with clicks, conversions, cost for last 7 days" \
  --use-query-cookbook)

# Validate
mcc-gaql --profile myprofile --validate "$INVESTIGATION_QUERY"

# Execute if valid
if [ $? -eq 0 ]; then
  mcc-gaql --profile myprofile --format json -o /tmp/ad_groups.json "$INVESTIGATION_QUERY"
fi
```

### Pattern 2: Change Event Correlation

```bash
# Generate change event query
CHANGE_QUERY=$(mcc-gaql-gen generate \
  "Show change events for campaign 'Brand Search' in last 30 days with change type and user email" \
  --use-query-cookbook)

# Validate
mcc-gaql --profile myprofile --validate "$CHANGE_QUERY"

# Execute if valid
if [ $? -eq 0 ]; then
  mcc-gaql --profile myprofile --format json -o /tmp/changes.json "$CHANGE_QUERY"
fi
```

### Pattern 3: Multiple Queries with Error Handling

```bash
# Function to generate, validate, and execute
run_query() {
  local prompt="$1"
  local output_file="$2"
  
  echo "Generating query for: $prompt"
  local query=$(mcc-gaql-gen generate "$prompt" --use-query-cookbook)
  
  echo "Validating..."
  if mcc-gaql --profile myprofile --validate "$query"; then
    echo "✅ Valid, executing..."
    mcc-gaql --profile myprofile --format json -o "$output_file" "$query"
    echo "✅ Saved to $output_file"
    return 0
  else
    echo "❌ Validation failed for: $prompt"
    return 1
  fi
}

# Use the function
run_query "Show campaign performance for last 7 days" "/tmp/campaigns.json"
run_query "Show ad group breakdown for campaign X" "/tmp/ad_groups.json"
run_query "Show change events for last 30 days" "/tmp/changes.json"
```

---

## Why Variable Storage is Recommended

### Problem with Piping

```bash
# Generate and validate - query is lost after validation
mcc-gaql-gen generate "show campaigns" | mcc-gaql --validate

# Now you need to regenerate to execute (inefficient)
mcc-gaql-gen generate "show campaigns" | mcc-gaql --format csv -o out.csv
```

**Issues:**
- Query generated twice (wastes time, may generate differently)
- Can't inspect what was validated
- Harder to debug validation errors

### Solution with Variables

```bash
# Generate once, store in variable
QUERY=$(mcc-gaql-gen generate "show campaigns")

# Validate the stored query
mcc-gaql --validate "$QUERY"

# Execute the SAME query that was validated
mcc-gaql --format csv -o out.csv "$QUERY"

# Can also inspect the query for debugging
echo "Query that was validated: $QUERY"
```

**Benefits:**
- Query generated only once
- Same query validated and executed (guaranteed consistency)
- Easy to debug - can inspect the variable
- Can log the query for auditing

---

## mcc-gaql-gen Flags

**Available flags for `mcc-gaql-gen generate`:**

```bash
mcc-gaql-gen generate "<prompt>" [OPTIONS]

Options:
  --use-query-cookbook    Enable RAG search for query examples (recommended)
  --explain              Show LLM reasoning for field selection
  --max-candidates <N>   Maximum field candidates to consider
```

**NOT available:**
- ❌ `--validate` (doesn't exist - use `mcc-gaql --validate` instead)
- ❌ `--execute` (doesn't exist - use `mcc-gaql` for execution)
- ❌ `--profile` (doesn't exist - profiles are for mcc-gaql only)

---

## Quick Reference Card

| Task | Command |
|------|---------|
| Generate query | `QUERY=$(mcc-gaql-gen generate "...")` |
| Validate query | `mcc-gaql --validate "$QUERY"` |
| Execute query | `mcc-gaql --format csv -o out.csv "$QUERY"` |
| Generate + validate (pipe) | `mcc-gaql-gen generate "..." \| mcc-gaql --validate` |
| Generate + validate + execute | See "Complete Example Workflows" above |

---

## Fixes Applied

### Files Updated

1. **SKILL.md**
   - Section "Phase 3: Change Event Correlation" - Fixed incorrect `--validate` usage
   - Added "⚠️ IMPORTANT: How to Validate mcc-gaql-gen Output" section
   - Clarified that `mcc-gaql-gen generate` does NOT have `--validate` flag

2. **change_correlation_reference.md**
   - Fixed example showing incorrect `--validate` usage
   - Added correct piping and variable-based examples
   - Added warning about the non-existent `--validate` flag

### Incorrect Pattern Removed

```bash
# ❌ REMOVED - This was incorrect
mcc-gaql-gen generate "..." --validate
```

### Correct Pattern Added

```bash
# ✅ ADDED - Correct approach
QUERY=$(mcc-gaql-gen generate "..." --use-query-cookbook)
mcc-gaql --profile <PROFILE> --validate "$QUERY"
```

---

## Summary

**Remember:**
1. `mcc-gaql-gen generate` **does NOT** have a `--validate` flag
2. Use `mcc-gaql --validate` to validate queries
3. Store generated queries in variables for validation and execution
4. Always validate before executing (separate step)

**Pattern to memorize:**
```bash
QUERY=$(mcc-gaql-gen generate "...")  # Generate
mcc-gaql --validate "$QUERY"          # Validate
mcc-gaql --format csv "$QUERY"        # Execute
```
