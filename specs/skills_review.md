# Google Ads Analyst Skill Review

## Context

Review of the `gads_skill_gaql_gen` repository against Skills Best Practices and Guidelines. Focus areas: instruction clarity, topic organization, context indexing, and LLM understandability.

---

## Top 5 Concerns

### 1. **Monolithic SKILL.md is Too Large (1,434 lines / ~16K tokens)**

**Issue:** The entire SKILL.md loads into context on every skill invocation, consuming significant context window space regardless of the specific task.

**Impact:**
- ~16,000 tokens loaded upfront for every analysis request
- Many sections (e.g., PDF generation details) are irrelevant for initial analysis
- Reduces available context for actual query results and analysis

**Recommendation:** Split SKILL.md into a lean core file (~300-500 lines) with essential workflow steps, using `include` or reference links for detailed sections. Consider:
```
SKILL.md (core workflow)
├── phase1_data_gathering.md
├── phase2_analysis.md
├── phase3_investigation.md
├── phase4_reporting.md
└── phase5_pdf_generation.md
```

---

### 2. **No Table of Contents or Navigation Index**

**Issue:** Neither SKILL.md nor the repo has a formal index of all content modules. The LLM must parse 1,400+ lines to understand what's available.

**Impact:**
- LLM may miss relevant reference files
- No quick way to understand skill capabilities at a glance
- Cognitive overhead to locate specific guidance

**Recommendation:** Add a structured TOC at the top of SKILL.md:
```markdown
## Quick Reference Index
| Topic | File | When to Load |
|-------|------|--------------|
| CLI Tool Usage | mcc_gaql_reference.md | When constructing queries |
| Metric Calculations | derived_metrics_reference.md | When converting micros/percentages |
| Investigation Patterns | investigation_patterns_reference.md | For HIGH/MEDIUM priority issues |
| ... | ... | ... |
```

---

### 3. **Reference File Loading is Not Signaled Clearly**

**Issue:** Links like `[investigation_patterns_reference.md](investigation_patterns_reference.md)` appear inline, but there's no clear instruction on WHEN the LLM should load these files vs. continue with inline content.

**Impact:**
- LLM may either load too many files (wasting context) or too few (missing critical info)
- No conditional loading triggers (e.g., "Load X when condition Y")
- Reference files are "discovered" mid-workflow rather than "indexed" upfront

**Recommendation:** Add explicit loading triggers:
```markdown
> **LOAD REFERENCE:** When investigating HIGH priority issues, read 
> [investigation_patterns_reference.md] before proceeding.
```

---

### 4. **Orphaned Documentation: docs/gaql_tools_overview.md Not Linked**

**Issue:** The comprehensive tool documentation at `docs/gaql_tools_overview.md` (503 lines) is NOT referenced from SKILL.md. This creates fragmented knowledge.

**Impact:**
- LLM won't discover this detailed tool reference
- Duplicate/inconsistent information may exist between this and `mcc_gaql_reference.md`
- Valuable context about tools is inaccessible during skill execution

**Recommendation:** Either:
1. Link `docs/gaql_tools_overview.md` from SKILL.md with clear loading instructions, OR
2. Consolidate its content into `mcc_gaql_reference.md`

---

### 5. **Instruction Clarity: Mixed Abstraction Levels**

**Issue:** SKILL.md jumps between high-level workflow steps and deep implementation details (specific GAQL queries, CSS styling for PDFs, statistical formulas) within the same sections.

**Examples found:**
- Section 2 mixes "how to query" concepts with raw GAQL syntax examples
- Investigation section includes both decision framework AND specific query templates
- PDF generation section has full CSS embedded (~200 lines)

**Impact:**
- Harder for LLM to extract the core workflow without getting lost in details
- Details that should be in reference files clutter the main instructions

**Recommendation:** Apply consistent abstraction layering:
- **SKILL.md**: High-level workflow, decision points, when to load references
- **Reference files**: Implementation details, templates, examples, CSS

---

## Additional Observations

### What's Working Well
- Clear YAML frontmatter with proper `allowed-tools` declaration
- Reference files are topically organized (investigation, correlation, actions, etc.)
- Workflow has clear numbered steps (1-6 phases)
- Validation-first approach is well documented
- Decision tables (health status, priority classification) are clear

### Minor Issues
- Internal enhancement documents (VALIDATION_ENHANCEMENT_SUMMARY.md, etc.) clutter the root directory
- Some reference files are large (action_templates: 718 lines, change_correlation: 698 lines) - could be split further
- Log and HTML output files in root directory create noise

---

## Recommended File Structure

```
gads_skill_gaql_gen/
├── SKILL.md                          # Lean core (~400 lines max)
├── REFERENCE_INDEX.md                # Master index of all references
├── references/                       # All reference files in subdirectory
│   ├── tools/
│   │   ├── mcc_gaql.md
│   │   └── mcc_gaql_gen.md
│   ├── analysis/
│   │   ├── derived_metrics.md
│   │   ├── investigation_patterns.md
│   │   └── performance_patterns.md
│   ├── correlation/
│   │   ├── change_events.md
│   │   └── potential_causes.md
│   ├── actions/
│   │   └── recommendation_templates.md
│   └── output/
│       ├── pdf_generation.md
│       └── appendix_format.md
├── docs/                             # Development documentation
├── specs/                            # Design specifications
└── .internal/                        # Enhancement summaries, logs
```

---

## Verification

To validate improvements:
1. Count tokens in SKILL.md before/after refactoring (target: <5K tokens core)
2. Test skill invocation and verify LLM correctly identifies which references to load
3. Measure if investigation quality improves with clearer reference loading instructions
