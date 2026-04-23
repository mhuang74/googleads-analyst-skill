# googleads-analyst-skill

Claude Skill for doing Google Ads Performance Analysis

---

## 📊 What It Does

The **Google Ads Analyst** skill is a Claude-powered tool that analyzes Google Ads account performance, identifies issues, provides actionable insights, and can apply approved setting changes directly to your account.

### Key Capabilities

- **Comprehensive Performance Analysis**: Compare campaign metrics across different time periods
- **Issue Detection**: Identify anomalies, budget pacing problems, impression share issues, quality score trends, and conversion rate changes
- **Root Cause Analysis**: Drill down from campaign → ad group → keyword/ad level to find the source of problems
- **Change Correlation**: Match performance changes with user-initiated account changes using change event analysis (last 30 days)
- **Evidence-Based Recommendations**: Provide specific, actionable next steps backed by real account data
- **Interactive Q&A**: Answer follow-up questions and investigate specific concerns
- **Apply Approved Changes**: Update campaign settings (status, name, daily budget, bidding targets) via a mandatory before/after verification loop
- **Multiple Output Formats**: Text reports with markdown tables (immediate) + optional professional PDF reports

---

## 🚀 How to Use

### Basic Invocation

Invoke the skill in Claude Code using the `/googleads-analyst` command:

```
/googleads-analyst
```

Then respond to prompts for:
1. **Authentication** - Profile name OR account credentials
2. **Account Type** - Is this a Google Ads Grants account?
3. **Analysis Parameters** - Date range, campaigns, metrics of interest

### Example Prompts

**Simple analysis:**
```
/googleads-analyst

What's the performance trend for my accounts last 7 days?
```

**Specific issue investigation:**
```
/googleads-analyst

My Search campaigns had a 30% drop in conversions this week. What happened?
```

**Comparing periods:**
```
/googleads-analyst

Compare March performance vs February - same campaigns, month-over-month.
```

**Campaign focus:**
```
/googleads-analyst

Analyze our Brand campaigns for quality issues and budget efficiency.
```

---

## 📋 Required Information

The skill will ask for the following during setup:

### 1. Authentication

**Option A: Using a configured profile (recommended)**
- Profile name (e.g., "themade", "client1")
- Command: `mcc-gaql --profile <PROFILE_NAME>`

**Option B: Direct credentials**
- Manager Customer ID (MCC ID)
- Customer ID of the account to analyze
- Google account email with access
- Command: `mcc-gaql --mcc <MCC_ID> --customer-id <CUSTOMER_ID> --user <EMAIL>`

### 2. Account Type

The skill needs to know if this is a **Google Ads Grants** account (non-profit):
- Grants accounts have different limitations ($10K/month budget, $2 max CPC)
- Impression share metrics are interpreted differently for Grants
- Performance standards are adjusted accordingly

### 3. Analysis Parameters

Specify or accept defaults for:
- **Time Periods**: Which dates to compare (e.g., "last 7 days vs previous 7 days")
- **Campaigns**: Which campaigns to include (defaults to all)
- **Metrics**: Which metrics matter most to you

### 4. Date Range Confirmation

The skill will confirm the exact date range before querying:
```
I'll analyze March 1-7, 2026 compared to February 22-28, 2026. Does this look correct?
```

---

## 📖 What Happens During Analysis

### Phase 1: Baseline Analysis
The skill queries your account's campaign-level performance:
- Impressions, clicks, cost, conversions, CTR, conversion rate
- Comparison of current period vs previous period
- Metric changes (% difference)
- Priority ranking of issues

### Phase 2: Interactive Text Report
Findings are presented immediately in the terminal as a **markdown table**:
- What changed (metrics with % differences)
- Where issues exist (which campaigns are affected)
- Impact estimation (cost, conversion impact)
- Priority levels (Critical, High, Medium, Low)

### Phase 3: Dynamic Investigation (when needed)
For HIGH or MEDIUM priority issues, the skill automatically:
- Drills down to ad group level to find the source
- Correlates performance changes with recent account activity (change events)
- Calculates correlation scores (0-100) between user actions and metric changes
- Distinguishes between user-caused vs external/market factors

### Phase 4: Interactive Q&A
Answer your follow-up questions:
- "Tell me more about that campaign"
- "What's causing the conversion rate drop?"
- "Which ad group should I focus on?"
- Continue until you're satisfied with the analysis

### Phase 5: Apply Approved Changes (optional)
When you explicitly request a setting change, the skill executes a four-step verification loop:
1. **Query before** — shows the current value and resource name
2. **Dry-run** — previews the mutation without applying; iterates until you confirm the command is correct
3. **Apply** — runs with `--yes` flag; records the result
4. **Query after** — confirms the change took effect; presents a Before/After summary table

Supported mutations: campaign status (pause/enable), campaign name, campaign remove, daily budget, Target CPA, Target ROAS.

### Phase 6: PDF Report (optional)
Only when you request it:
- Professional formatted report with your account name
- Includes Ad Group Analysis and Root Cause Analysis sections
- Chrome-rendered for exceptional visual quality
- Ready for sharing with stakeholders

---

## ✅ Best Practices for Prompting

### ✅ DO:

1. **Be specific about what you want analyzed**
   - ✅ "What's causing the 25% drop in Search campaign conversions this week?"
   - ✅ "Compare March vs February for our Brand campaigns"
   - ✅ "Are we overspending on underperforming ad groups?"

2. **Provide context if you know what changed**
   - ✅ "We changed our bid strategy last Tuesday - is that causing the decline?"
   - ✅ "We paused some keywords on the 15th"
   - ✅ "We increased budget on Display campaigns"

3. **Confirm authentication details match**
   - ✅ Verify your profile name is correct before starting
   - ✅ Confirm the Customer ID matches the account you want to analyze
   - ✅ Confirm date ranges are what you expect

4. **Ask follow-up questions during Q&A**
   - ✅ "Which ad group in that campaign has the worst performance?"
   - ✅ "Should I increase or decrease budget for this campaign?"
   - ✅ "What's the next step to fix this issue?"

5. **Request PDF only when ready to share**
   - ✅ After Q&A is complete and findings are confirmed
   - ✅ When you need a polished, shareable report
   - ✅ The skill will offer this as an option

### ❌ DON'T:

1. **Don't assume the skill knows account names or profile names**
   - ❌ "Analyze my account" (which one? use profile name or IDs)
   - ❌ "Check the 'marketing' profile" (spell it exactly as configured)

2. **Don't skip authentication confirmation**
   - ❌ Provide MCC ID but forget Customer ID
   - ❌ Use the wrong profile name
   - The skill will ask, make sure you answer correctly

3. **Don't request PDF immediately**
   - ❌ "Generate a PDF report right away"
   - ✅ Wait for text analysis first, then request PDF if needed
   - Text reports are faster and let you confirm findings first

4. **Don't ask for analysis beyond 30 days with root cause**
   - ❌ "What caused the decline 60 days ago?"
   - ⚠️ Change events only available for last 30 days (API limitation)
   - The skill will note this limitation

5. **Don't assume all accounts follow same rules**
   - ❌ Grants accounts have different performance standards
   - Make sure to confirm if it's a Grants account

---

## 📚 More Information

- **Full Workflow Documentation**: See [SKILL.md](SKILL.md) for detailed instructions on every phase
- **Reference Index**: See [REFERENCE_INDEX.md](REFERENCE_INDEX.md) for complete guide to analysis frameworks
- **Workflow Examples**: See [references/workflow_examples.md](references/workflow_examples.md) for end-to-end examples
- **Best Practices**: See [references/best_practices.md](references/best_practices.md) for final checklist

---

## 🔧 Technical Requirements

- Access to Google Ads API via `mcc-gaql` tool (read) and `mcc-gaql-mut-d` tool (write)
- Valid Google Ads account credentials (Profile, MCC ID + Customer ID, or email)
- At least 7 days of historical data for meaningful comparison
- Account owner or manager role for access to all campaigns
- **Editor role or higher** to apply campaign setting changes (read-only access is sufficient for analysis only)

### Safety Model for Mutations

The skill never mutates without explicit in-session user approval. Every mutation follows a four-step loop: query current value → dry-run preview (iterated until correct) → apply with `--yes` → confirm via after-query. The skill will not proceed past Step B without the user confirming the dry-run output looks correct.

---

## 🎯 Common Use Cases

| Use Case | Example Prompt |
|----------|---|
| **Identify problems** | "Why did conversions drop this week?" |
| **Compare periods** | "Month-over-month performance for Search campaigns" |
| **Find bottlenecks** | "Which ad group is underperforming in each campaign?" |
| **Validate changes** | "Did our bid strategy change help?" |
| **Deep investigation** | "Let me understand that campaign's performance in detail" |
| **Optimization advice** | "Where should I focus my budget?" |
| **Apply a change** | "Pause the Brand - Search campaign" or "Set budget to $75/day" |
| **Performance report** | "Create a professional report for my stakeholders" |

---

## 💡 Tips

- **Start simple**: Ask for basic performance first, then drill down
- **Use natural language**: You don't need GAQL syntax - just describe what you want to know
- **Confirm dates**: Always verify the date range before the skill proceeds
- **Follow up interactively**: Don't ask for everything at once - the Q&A phase is interactive
- **Request PDF last**: Use text analysis first to confirm findings, then get PDF for sharing
