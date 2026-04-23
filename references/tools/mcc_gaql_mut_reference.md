# mcc-gaql-mut Reference

This reference covers the `mcc-gaql-mut-d` CLI tool used for mutating Google Ads resources. It is the write-path companion to `mcc-gaql` (read-only queries).

---

## Installation / Verification

The debug build is installed as `mcc-gaql-mut-d`. To verify:

```bash
which mcc-gaql-mut-d
mcc-gaql-mut-d --version
```

If not found, build from source:

```bash
cd /rust_dev_cache/projects/googleads/gaql_new_mutate_crate
cargo build
# debug binary at: target/debug/mcc-gaql-mut
# install as:
cp target/debug/mcc-gaql-mut ~/.local/bin/mcc-gaql-mut-d
```

---

## Two Ways to Authenticate

**Option 1: Using a configured profile (recommended)**

```bash
mcc-gaql-mut-d -p <PROFILE_NAME> mutate ...
```

**Option 2: Without a profile (explicit parameters)**

```bash
mcc-gaql-mut-d -c <CUSTOMER_ID> -m <MCC_ID> -u <USER_EMAIL> mutate ...
```

The same OAuth tokens and profile configuration used by `mcc-gaql` apply here. See `references/tools/mcc_gaql_reference.md` and `references/tools/common_errors_reference.md` for OAuth troubleshooting.

---

## `mutate` Subcommand Flags

```
mcc-gaql-mut-d [AUTH_FLAGS] mutate [OPTIONS]
```

**Top-level auth flags** (before `mutate`):

| Flag | Short | Description |
|------|-------|-------------|
| `--profile <NAME>` | `-p` | Use named profile |
| `--customer-id <ID>` | `-c` | Child account ID (required if no profile) |
| `--mcc-id <ID>` | `-m` | Manager account ID (required if no profile) |
| `--user-email <EMAIL>` | `-u` | Google account email (required if no profile) |
| `--remote-auth` | | Headless/Telegram OAuth flow |

**`mutate` subcommand flags**:

| Flag | Description |
|------|-------------|
| `--resource <TYPE>` | Resource type in CamelCase, e.g. `Campaign`, `CampaignBudget` |
| `--resource-name <NAME>` | Full resource name, e.g. `customers/123/campaigns/456` |
| `--operation <OP>` | `update` (default), `create`, or `remove` |
| `--set <FIELD=VALUE>` | Field mutation; repeat for multiple fields |
| `--dry-run` | Validate the mutation via API without applying |
| `--preview` | Show the constructed protobuf request offline (no API call) |
| `-y`, `--yes` | Skip the interactive confirmation prompt (use in automation) |
| `--partial-failure` | Continue on partial failures (default: true) |

---

## Supported Mutations (Skill Scope)

| Goal | Resource | Field(s) | Unit |
|------|----------|----------|------|
| Change daily budget | `CampaignBudget` | `amount_micros` | micros ($1 = 1,000,000) |
| Pause campaign | `Campaign` | `status` | `PAUSED` |
| Enable campaign | `Campaign` | `status` | `ENABLED` |
| Rename campaign | `Campaign` | `name` | plain string |
| Remove campaign | `Campaign` | _(none)_ | `--operation remove` |
| Set Target CPA | `Campaign` | `target_cpa.target_cpa_micros` | micros |
| Set Target ROAS | `Campaign` | `target_roas.target_roas` | decimal (e.g. `4.5`) |

> **Micros conversion**: `micros = dollars × 1,000,000`
> `target_roas` is a plain decimal — do NOT multiply by 1,000,000.

---

## Example Commands

### Pause a campaign (dry-run, then apply)

```bash
# Step 1: dry-run (preview what will happen)
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/1234567890/campaigns/9876543210" \
  --operation update \
  --set "status=PAUSED" \
  --dry-run

# Step 2: apply (--yes skips the interactive confirmation)
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/1234567890/campaigns/9876543210" \
  --operation update \
  --set "status=PAUSED" \
  --yes
```

### Update daily budget

```bash
# Discover the campaign_budget.resource_name first:
mcc-gaql -p <PROFILE> --format json "
  SELECT campaign.id, campaign.name,
         campaign_budget.resource_name, campaign_budget.amount_micros
  FROM campaign
  WHERE campaign.id = <CAMPAIGN_ID>"

# $75/day = 75000000 micros
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource CampaignBudget \
  --resource-name "customers/1234567890/campaignBudgets/14789330607" \
  --operation update \
  --set "amount_micros=75000000" \
  --dry-run

mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource CampaignBudget \
  --resource-name "customers/1234567890/campaignBudgets/14789330607" \
  --operation update \
  --set "amount_micros=75000000" \
  --yes
```

### Rename a campaign

```bash
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/1234567890/campaigns/9876543210" \
  --operation update \
  --set "name=Brand - Search (US)" \
  --yes
```

### Remove a campaign

```bash
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/1234567890/campaigns/9876543210" \
  --operation remove \
  --dry-run
# ⚠️ ALWAYS dry-run removal first. Never skip --dry-run for remove operations.
```

### Set Target CPA

```bash
# $25.00 target CPA = 25000000 micros
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/1234567890/campaigns/9876543210" \
  --operation update \
  --set "target_cpa.target_cpa_micros=25000000" \
  --dry-run
```

### Set Target ROAS

```bash
# 4.5x ROAS (NOT micros — plain decimal)
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/1234567890/campaigns/9876543210" \
  --operation update \
  --set "target_roas.target_roas=4.5" \
  --dry-run
```

### Multi-field update

```bash
mcc-gaql-mut-d -p <PROFILE> mutate \
  --resource Campaign \
  --resource-name "customers/1234567890/campaigns/9876543210" \
  --operation update \
  --set "status=ENABLED" \
  --set "name=Brand - Search (Enabled)" \
  --yes
```

---

## Discovering Resource Names

Resource names are required by `--resource-name`. They follow Google Ads API naming conventions.

**Campaign resource name:**
```bash
mcc-gaql -p <PROFILE> --format json "
  SELECT campaign.id, campaign.name, campaign.resource_name, campaign.status
  FROM campaign WHERE campaign.status != 'REMOVED'"
```
→ `campaign.resource_name` = `"customers/XXXXXXXXXX/campaigns/XXXXXXXXXX"`

**CampaignBudget resource name:**
```bash
mcc-gaql -p <PROFILE> --format json "
  SELECT campaign.id, campaign.name,
         campaign_budget.resource_name, campaign_budget.amount_micros
  FROM campaign WHERE campaign.id = <CAMPAIGN_ID>"
```
→ `campaign_budget.resource_name` = `"customers/XXXXXXXXXX/campaignBudgets/XXXXXXXXXX"`

---

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `command not found: mcc-gaql-mut-d` | Binary not on PATH | Build and install, see above |
| `authentication failed` | Expired OAuth token | Re-run OAuth flow; see `common_errors_reference.md` |
| `invalid field path` | Wrong `--set` key name | Use `--preview` flag to inspect the protobuf structure |
| `resource not found` | Wrong `--resource-name` | Re-query resource names using `mcc-gaql` |
| `operation not permitted` | Account lacks write access | Verify account role is Editor or higher |
| `partial failure` | One of multiple `--set` fields failed | Check stdout for per-field error details |

> For OAuth and profile errors, see [common_errors_reference.md](common_errors_reference.md)
