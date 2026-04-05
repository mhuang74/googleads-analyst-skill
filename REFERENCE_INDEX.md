# Reference Index

This index shows all available reference files and when to load them during analysis.

## Quick Reference Table

| Topic | File | When to Load |
|-------|------|--------------|
| **GAQL Tools** | | |
| Tool usage & validation | [tools/gaql_validation.md](references/tools/gaql_validation.md) | Before executing ANY GAQL query |
| mcc-gaql reference | [tools/mcc_gaql_reference.md](references/tools/mcc_gaql_reference.md) | When using mcc-gaql CLI tool |
| mcc-gaql-gen overview | [tools/gaql_tools_overview.md](references/tools/gaql_tools_overview.md) | When generating queries from natural language |
| Common errors | [tools/common_errors_reference.md](references/tools/common_errors_reference.md) | When encountering query errors |
| **Analysis** | | |
| Metric calculations | [analysis/derived_metrics_reference.md](references/analysis/derived_metrics_reference.md) | Before calculating CPA, ROAS, conversion rate |
| Health status & impression share | [analysis/health_status.md](references/analysis/health_status.md) | When categorizing campaigns (🟢🟡🔴) |
| Investigation patterns | [analysis/investigation_patterns_reference.md](references/analysis/investigation_patterns_reference.md) | For HIGH/MEDIUM priority issues requiring drill-down |
| Performance patterns | [analysis/common_performance_patterns_reference.md](references/analysis/common_performance_patterns_reference.md) | When recognizing standard performance scenarios |
| Contextual analysis rules | [analysis/contextual_analysis_rules_reference.md](references/analysis/contextual_analysis_rules_reference.md) | When interpreting metric relationships |
| **Correlation** | | |
| Change event analysis | [correlation/change_correlation_reference.md](references/correlation/change_correlation_reference.md) | When investigating performance anomalies (within 30 days) |
| Potential cause identification | [correlation/identify_potential_causes.md](references/correlation/identify_potential_causes.md) | When determining root causes of issues |
| **Actions** | | |
| Recommendation templates | [actions/action_templates_reference.md](references/actions/action_templates_reference.md) | When formulating specific recommendations |
| **Output** | | |
| PDF generation | [output/pdf_generation_reference.md](references/output/pdf_generation_reference.md) | Only when user requests PDF report |
| Appendix formatting | [output/appendix_reference.md](references/output/appendix_reference.md) | When including detailed data tables in PDF |
| **Workflows** | | |
| Complete examples | [workflow_examples.md](references/workflow_examples.md) | For end-to-end workflow reference |
| Best practices | [best_practices.md](references/best_practices.md) | Before delivering analysis (final checklist) |

---

## Loading Triggers by Phase

### Phase 1: Query Preparation
> **LOAD:** [tools/gaql_validation.md](references/tools/gaql_validation.md)  
> **LOAD:** [tools/mcc_gaql_reference.md](references/tools/mcc_gaql_reference.md)

Every query must be validated before execution.

### Phase 2: Metric Calculation
> **LOAD:** [analysis/derived_metrics_reference.md](references/analysis/derived_metrics_reference.md)

FIRST convert cost_micros to dollars, THEN calculate derived metrics.

### Phase 3: Campaign Analysis
> **LOAD:** [analysis/health_status.md](references/analysis/health_status.md)

Assign health status and analyze impression share metrics FIRST.

### Phase 4: Investigation (HIGH/MEDIUM Issues Only)
> **LOAD:** [analysis/investigation_patterns_reference.md](references/analysis/investigation_patterns_reference.md)  
> **LOAD:** [correlation/change_correlation_reference.md](references/correlation/change_correlation_reference.md) (if issue within 30 days)  
> **LOAD:** [actions/action_templates_reference.md](references/actions/action_templates_reference.md)

Dynamic investigation is MANDATORY for HIGH/MEDIUM priority issues.

### Phase 5: Reporting
> **LOAD:** [best_practices.md](references/best_practices.md)

Review final checklist before presenting analysis.

### Phase 6: PDF Generation (Optional)
> **LOAD:** [output/pdf_generation_reference.md](references/output/pdf_generation_reference.md)

Only when user explicitly requests PDF.

---

## Conditional Loading

### Load When Generating Queries
- [tools/gaql_tools_overview.md](references/tools/gaql_tools_overview.md) - for mcc-gaql-gen usage
- [tools/mcc_gaql_reference.md](references/tools/mcc_gaql_reference.md) - for field discovery

### Load When Query Fails
- [tools/common_errors_reference.md](references/tools/common_errors_reference.md) - error troubleshooting

### Load for Specific Investigation Types
- **Budget issues**: [analysis/health_status.md](references/analysis/health_status.md) (impression share section)
- **Ad fatigue**: [analysis/investigation_patterns_reference.md](references/analysis/investigation_patterns_reference.md)
- **Performance anomalies**: [correlation/change_correlation_reference.md](references/correlation/change_correlation_reference.md)
- **PMax issues**: [analysis/investigation_patterns_reference.md](references/analysis/investigation_patterns_reference.md)

---

## File Organization

```
references/
├── tools/                    # CLI tools usage
│   ├── gaql_validation.md   # Validation workflow (ALWAYS load)
│   ├── mcc_gaql_reference.md          # Tool reference
│   ├── gaql_tools_overview.md  # Generation guide
│   └── common_errors_reference.md     # Error troubleshooting
├── analysis/                 # Analysis frameworks
│   ├── derived_metrics_reference.md   # Calculation formulas
│   ├── health_status.md     # Status & impression share
│   ├── investigation_patterns_reference.md  # Drill-down strategies
│   ├── common_performance_patterns_reference.md  # Pattern recognition
│   └── contextual_analysis_rules_reference.md  # Metric relationships
├── correlation/              # Root cause analysis
│   ├── change_correlation_reference.md  # Change event analysis
│   └── identify_potential_causes.md  # Cause identification
├── actions/                  # Recommendations
│   └── action_templates_reference.md  # Standardized templates
└── output/                   # Report formatting
    ├── pdf_generation_reference.md    # PDF creation
    └── appendix_reference.md          # Detailed tables

workflow_examples.md          # Complete workflow examples
best_practices.md             # Final checklist
```
