# SKILL.md Comparison: NanoBot vs Claude — Backport Notes

## Context

Compare the two parallel implementations of the Google Ads analyst skill and catalog which instructions appear in one but not the other, so either side can be backported without re-deriving the diff.

- **NanoBot**: `~/.nanobot/workspace/skills/google-ads-nano-analyst/SKILL.md` (11.5K)
- **Claude**: `/rust_dev_cache/projects/googleads/gads_skills_feature_branch/SKILL.md` (17.8K, symlinked from `~/.claude/skills/googleads-analyst`)

Both share an identical reference-file directory structure (tools/, analysis/, correlation/, actions/, output/ + `best_practices.md`, `workflow_examples.md`). NanoBot's reference files are condensed (~3-6 KB each vs Claude's 5-25 KB). This document covers SKILL.md drift only, plus one extra reference file that exists only in NanoBot.

---

## Frontmatter & Identity Differences

| Field | NanoBot | Claude |
|-------|---------|--------|
| `name` | `Google Ads Nano Analyst` | `Generate Google Ads Performance Report` |
| `description` | Adds explicit keyword triggers ("analyze", "performance", "report", "metrics", "Google Ads", "campaign") | No keyword trigger list |
| `allowed-tools` | absent | `Bash(mcc-gaql:*,mcc-gaql-gen:*)` |

---

## Instructions ONLY in NanoBot (candidates to backport into Claude)

1. **Remote OAuth Authentication flow** (§1.A, full 4-step section) — `--remote-auth` URL capture, browser handoff on any device, paste-back of `4/...` authorization code, post-auth verification. Designed for headless/Telegram environments.
2. **Google Ads Grants escalation rule** (§3 Step 2) — the 🎗️ note that Grant "waste" isn't cash loss; allow 7-14 days learning phase after any bidding/ROAS change; only escalate if Rank Lost IS >70% or zero conversions persist beyond 14 days.
3. **Mandatory Account Information section in every report** (§5) — explicitly requires a Customer ID + Account Name table with a worked example (`4731266055 / Dogs on Deployment`), plus a repeat of the `SELECT customer.id, customer.descriptive_name FROM customer` query.
4. **Tool Usage section** (end of file) — concrete `exec: {"command": "..."}` and `read_file: {"path": "..."}` invocation patterns for the NanoBot harness, including an absolute path to a reference file.
5. **Extra reference file** — `references/resources/geographic_reporting_views.md` (6.3K). Indexed in REFERENCE_INDEX.md as "When choosing between geographic_view vs user_location_view." Not present in Claude.

---

## Instructions ONLY in Claude (candidates to backport into NanoBot)

1. **Authentication framed as Option 1 / Option 2** with explicit command-line templates (`mcc-gaql --profile ...` vs `mcc-gaql --mcc ... --customer-id ... --user ...`).
2. **Grants account hint** — "Grants accounts have a $10K/month cap and $2 max CPC" plus the note that Grants normally show 80-95% lost impression share due to bid caps.
3. **Time-period format catalog** — Natural language / specific dates / predefined shortcuts (`LAST_7_DAYS`, `THIS_WEEK`, etc.).
4. **Multiple `LOAD REFERENCE` pointers per section** — Claude links `mcc_gaql_reference.md`, `gaql_tools_overview.md`, `common_errors_reference.md`, `contextual_analysis_rules_reference.md`, `common_performance_patterns_reference.md`, `appendix_reference.md`. NanoBot keeps only the primary one per section.
5. **`--format json` vs `--format csv` guidance** — "json for LLM-friendly, csv for large datasets to reduce tokens." NanoBot only shows csv.
6. **Health status criteria are richer** — adds "ROAS stable/improved" (🟢), "impression share loss >15%" (🟡), "CTR drop >25%" (🔴).
7. **Priority Classification expanded** — HIGH adds "OR Rank Lost IS >70%"; MEDIUM adds "Rank Lost IS 50-70%"; LOW adds "minimal spend impact" qualifier.
8. **"When to Use Dynamic Investigation" subsection** — explicit ALWAYS criteria (4 bullets) and SKIP criteria (4 bullets: `<10%` fluctuation, `<30 clicks`, `<$50/week`, obvious causes).
9. **"Investigation Best Practices"** — 7-point numbered list (start broad, validate, use cookbook flag, calculate correlation, iterate, stop when confident, document evidence).
10. **Simple-vs-complex report templates** — Claude provides two output structures based on severity; NanoBot only shows the complex one.
11. **PDF filename sanitization rule** — "lowercase, underscores for spaces, alphanumeric only" plus "Do NOT use the profile name." NanoBot only mentions lowercase/underscores.
12. **Workflow examples preview** — Claude calls out the two examples (with-profile / without-profile) inline; NanoBot just links the file.
13. **Final Checklist subsections** — Claude breaks the checklist into Dynamic Investigation, Correlation Analysis, Query Execution Quality, Content Quality, Reporting Quality. NanoBot collapses into 3 groups.

---

## Substantive vs Cosmetic Drift

**Substantive (behavior differs):**
- NanoBot-only: remote-OAuth flow, Grants learning-phase escalation rule.
- Claude-only: json/csv format guidance, dynamic-investigation skip criteria, richer Health/Priority thresholds, PDF filename sanitization.

**Cosmetic / structural:**
- Tool Usage block, skill name, frontmatter `allowed-tools`, two-template reporting structure, multi-LOAD-REFERENCE headers.
- NanoBot reference files are summarized — same filenames and topics, smaller bodies.

---

## Files Compared

- `/home/mhuang/.nanobot/workspace/skills/google-ads-nano-analyst/SKILL.md`
- `/rust_dev_cache/projects/googleads/gads_skills_feature_branch/SKILL.md`
- both `REFERENCE_INDEX.md` files
- reference subdirectories listed and matched name-for-name (only divergence: `references/resources/geographic_reporting_views.md`, NanoBot-only)

No implementation performed; analysis only.
