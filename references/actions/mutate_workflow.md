# Mutate Workflow Reference

This reference documents the mandatory four-step verification loop for all `mcc-gaql-mut` mutations. Every change to a live Google Ads account must follow these steps in order.

> **Tool reference:** [tools/mcc_gaql_mut_reference.md](../tools/mcc_gaql_mut_reference.md)  
> **Recommendation templates:** [action_templates_reference.md](action_templates_reference.md)

---

## Entry Conditions

Only enter the mutate workflow when the user **explicitly requests a change** — e.g.:
- "Pause campaign X"
- "Increase the budget for Brand Search to $75/day"
- "Change the target CPA to $25"
- "Rename the campaign to Y"

**Never** infer intent and execute. If the user says "should I pause it?" — that is a question, not a request to act. Wait for a direct "yes, pause it" or "go ahead".

---

## Five-Step Verification Loop

### Step A — Query BEFORE

Run `mcc-gaql` to capture the current value(s) and the resource name needed by `mcc-gaql-mut`. Display the result clearly before proceeding.

```bash
mcc-gaql -p <PROFILE> --format json "<BEFORE_QUERY>"
```

Display: "Current state of [field]: [value]"

### Step B — Dry-run (internal self-check)

Run `--dry-run` to validate that the resource name, field path, and value are accepted by the API. If it fails, diagnose the error (wrong resource name, bad field path, unit mismatch), correct the arguments, and retry internally. Do not prompt the user during this correction loop.

```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource <RESOURCE_TYPE> \
  --resource-name "<RESOURCE_NAME>" \
  --operation <update|remove> \
  --set "<FIELD>=<VALUE>" \
  --dry-run
```

**⚠️ Do NOT skip dry-run — it is the argument-validation gate before apply.**

### Step C — Confirm with user

Once dry-run succeeds, show the user the exact validated command and ask for explicit confirmation before applying. This step exists because the dry-run correction loop may have adjusted arguments (e.g. resolved resource names, converted units) from what the user originally stated.

Example prompt:
```
Ready to apply:
  mcc-gaql-mut -p themade mutate \
    --resource CampaignBudget \
    --resource-name "customers/3902228771/campaignBudgets/14789330607" \
    --operation update \
    --set "amount_micros=75000000"

This will change the daily budget from $50.00 to $75.00. Proceed?
```

### Step D — Apply

After user confirms, apply using `--yes` to bypass the interactive prompt.

```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource <RESOURCE_TYPE> \
  --resource-name "<RESOURCE_NAME>" \
  --operation <update|remove> \
  --set "<FIELD>=<VALUE>" \
  --yes
```

Record the full stdout output.

### Step E — Query AFTER

Re-run the same query from Step A to confirm the change is reflected in the API.

```bash
mcc-gaql -p <PROFILE> --format json "<BEFORE_QUERY>"
```

### Step F — Summarize

Present a markdown diff table:

```markdown
## Change Applied

| Field | Before | After | Status |
|-------|--------|-------|--------|
| campaign.status | ENABLED | PAUSED | ✅ |
```

If the after-value does not match the intended change: surface both the dry-run and apply stdout, and stop. Do not retry automatically.

---

## Unit Conversions

```python
# Dollars to micros (for budget and target_cpa)
micros = dollars * 1_000_000
# Examples:
# $75/day  → 75000000
# $25 CPA  → 25000000

# Micros to dollars (for display in Step A / Step E)
dollars = micros / 1_000_000

# target_roas is a plain decimal — NOT micros
# 4.5x ROAS → target_roas=4.5 (do NOT multiply by 1_000_000)
```

---

## Per-Mutation Recipes

### 1. Daily Budget Update

**Goal:** Change a campaign's daily budget (modifies `CampaignBudget`, not `Campaign`).

**Step A — Before query:**
```bash
QUERY='SELECT campaign.id, campaign.name,
              campaign_budget.resource_name, campaign_budget.amount_micros
       FROM campaign
       WHERE campaign.id = <CAMPAIGN_ID>'
mcc-gaql -p <PROFILE> --format json "$QUERY"
```
Note: `campaign_budget.resource_name` (e.g. `customers/123/campaignBudgets/456`) is required for the mutate command, NOT the campaign resource name.

**Step B — Dry-run:**
```bash
# $75/day = 75000000 micros
mcc-gaql-mut -p <PROFILE> mutate \
  --resource CampaignBudget \
  --resource-name "customers/XXXXXXXXXX/campaignBudgets/XXXXXXXXXX" \
  --operation update \
  --set "amount_micros=75000000" \
  --dry-run
```

**Step C — Apply:**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource CampaignBudget \
  --resource-name "customers/XXXXXXXXXX/campaignBudgets/XXXXXXXXXX" \
  --operation update \
  --set "amount_micros=75000000" \
  --yes
```

**Step D — After query:** Same GAQL as Step A.

**Step E — Summary row:**
```
| campaign_budget.amount_micros | 50000000 ($50.00) | 75000000 ($75.00) | ✅ |
```

---

### 2. Campaign Status (Pause / Enable)

**Goal:** Pause or re-enable a campaign.

**Step A — Before query:**
```bash
QUERY='SELECT campaign.id, campaign.name, campaign.resource_name, campaign.status
       FROM campaign
       WHERE campaign.id = <CAMPAIGN_ID>'
mcc-gaql -p <PROFILE> --format json "$QUERY"
```

**Step B — Dry-run (PAUSED example):**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "status=PAUSED" \
  --dry-run
```
Valid status values: `PAUSED`, `ENABLED`

**Step C — Apply:**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "status=PAUSED" \
  --yes
```

**Step D — After query:** Same GAQL as Step A.

**Step E — Summary row:**
```
| campaign.status | ENABLED | PAUSED | ✅ |
```

---

### 3. Campaign Rename

**Goal:** Change a campaign's name.

**Step A — Before query:**
```bash
QUERY='SELECT campaign.id, campaign.name, campaign.resource_name
       FROM campaign
       WHERE campaign.id = <CAMPAIGN_ID>'
mcc-gaql -p <PROFILE> --format json "$QUERY"
```

**Step B — Dry-run:**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "name=Brand - Search (US)" \
  --dry-run
```

**Step C — Apply:**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "name=Brand - Search (US)" \
  --yes
```

**Step D — After query:** Same GAQL as Step A.

---

### 4. Campaign Remove

**Goal:** Permanently remove a campaign. **This is irreversible.**

**⚠️ CRITICAL SAFETY RULES for remove:**
- Always run the before-query (Step A) and show the result to the user — confirm you have the right campaign before proceeding.
- Always dry-run first as internal validation before applying.
- At Step C (user confirmation), clearly restate the campaign name, ID, and that removal is permanent and irreversible.

**Step A — Before query:**
```bash
QUERY='SELECT campaign.id, campaign.name, campaign.resource_name,
              campaign.status, metrics.cost_micros
       FROM campaign
       WHERE campaign.id = <CAMPAIGN_ID>
         AND segments.date DURING LAST_30_DAYS'
mcc-gaql -p <PROFILE> --format json "$QUERY"
```

**Step B — Dry-run:**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation remove \
  --dry-run
```

**Step C — Apply (only after explicit user confirmation):**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation remove \
  --yes
```

**Step D — After query:** Re-run Step A; a removed campaign returns no results or `status=REMOVED`.

---

### 5. Target CPA

**Goal:** Update a Target CPA bidding strategy's CPA target. The field lives on the campaign's bidding configuration.

**Step A — Before query:**
```bash
QUERY='SELECT campaign.id, campaign.name, campaign.resource_name,
              campaign.target_cpa.target_cpa_micros
       FROM campaign
       WHERE campaign.id = <CAMPAIGN_ID>'
mcc-gaql -p <PROFILE> --format json "$QUERY"
```

**Step B — Dry-run ($25.00 CPA = 25000000 micros):**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "target_cpa.target_cpa_micros=25000000" \
  --dry-run
```

**Step C — Apply:**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "target_cpa.target_cpa_micros=25000000" \
  --yes
```

**Step D — After query:** Same GAQL as Step A.

**Step E — Summary row:**
```
| campaign.target_cpa.target_cpa_micros | 30000000 ($30.00) | 25000000 ($25.00) | ✅ |
```

---

### 6. Target ROAS

**Goal:** Update a Target ROAS bidding strategy's ROAS target. Unlike CPA, this is a plain decimal (not micros).

**Step A — Before query:**
```bash
QUERY='SELECT campaign.id, campaign.name, campaign.resource_name,
              campaign.target_roas.target_roas
       FROM campaign
       WHERE campaign.id = <CAMPAIGN_ID>'
mcc-gaql -p <PROFILE> --format json "$QUERY"
```

**Step B — Dry-run (4.5x ROAS):**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "target_roas.target_roas=4.5" \
  --dry-run
```
**⚠️ `target_roas` is a decimal (e.g. `4.5`), NOT micros. Do not multiply by 1,000,000.**

**Step C — Apply:**
```bash
mcc-gaql-mut -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/XXXXXXXXXX/campaigns/XXXXXXXXXX" \
  --operation update \
  --set "target_roas.target_roas=4.5" \
  --yes
```

**Step D — After query:** Same GAQL as Step A.

---

## Connecting Recommendations to Mutations

When a recommendation from `action_templates_reference.md` is accepted, use its `Follow-up Query` field as both the Step-A before-query and the Step-D after-query. This ensures the same metric that motivated the recommendation is used to confirm the change took effect.

Example flow:
1. Analysis identifies Budget Lost IS > 50% → Recommendation: increase budget from $50 to $90
2. User agrees in Q&A → enter mutate workflow
3. Step A: query `campaign_budget.amount_micros` (shows 50000000)
4. Step B: dry-run `amount_micros=90000000`
5. User confirms → Step C: apply
6. Step D: same query shows 90000000
7. Step E: diff table + "Budget increased from $50.00 to $90.00"
