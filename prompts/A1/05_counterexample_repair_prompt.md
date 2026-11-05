# P5 – Counterexample-Guided Repair Prompt

[← Back to Prompt Index](README.md)

**Used when:** **Some** scenarios match Excel and some do not.

**Purpose:** Fixes the failing scenarios without breaking the passing ones. Up to three passing scenarios are given as regression constraints and up to three failing scenarios as counterexamples.

## Placeholders

| Placeholder | Content |
|---|---|
| `{test_cases_text}` | Input overrides for the selected passing and failing scenarios |
| `{passed_scenario_ids_text}` | IDs of the selected passing scenarios |
| `{failed_scenario_ids_text}` | IDs of the selected failing scenarios |
| `{scenario_summary_text}` | Run and comparison result for each selected scenario |
| `{passed_scenarios_text}` | Results of the passing scenarios |
| `{failed_scenarios_text}` | Results of the failing scenarios |
| `{compare_error}` | Comparison warnings or errors, or `(None)` |
| `{mismatch_analysis_text}` | Summary of mismatches: counts, problem columns, missing/extra cells, and row-shift patterns |
| `{unmatched_expected_text}` | Expected Excel value and formula for each unmatched cell |
| `{unmatched_details_text}` | Excel value vs Java value for each unmatched cell |
| `{current_java_code}` | Java code from the previous attempt |
| `{workbook_text}` | Workbook metadata: sheets, cell values, formulas, and named ranges |

## Prompt Template

```text
The previous Java code is partially correct.

Observed result:
- One or more scenarios already pass.
- One or more scenarios still fail.

Your task:
Repair the Java code so the failing scenarios pass WITHOUT breaking the scenarios that already pass.

Important repair strategy:
A. Treat the passing scenarios as regression constraints. Preserve their behavior.
B. Treat the failing scenarios as counterexamples that expose where the current logic is too narrow, shifted, omitted, or hardcoded.
C. Fix the underlying scenario-aware logic, not a single hardcoded scenario.
D. Use the EXPECTED EXCEL CELLS section as the source of truth for failing coordinates.
E. Preserve cells that already match in passing scenarios unless a broader structural fix requires consistent handling across all scenarios.
F. When ExcelFormula is provided for an unmatched cell, you MUST use that formula as a strong signal for the intended workbook logic.
G. If repeated mismatch patterns appear across failed scenarios, repair the common cause once.
H. Do NOT regress the scenarios that already pass.
I. Do NOT hardcode output rows for specific scenario ids.

Hard requirements:
1. Return ONLY Java code. No markdown fences. No explanation.
2. The Java code MUST contain exactly one public class named WorkbookProgram.
3. No package declaration.
4. Use only standard Java libraries.
5. Do NOT read the Excel file at runtime.
6. main(String[] args) must accept:
   - args[0] = output TXT file path
   - args[1] = output CSV file path
   - args[2] = scenario id
   - args[3...] = zero or more runtime overrides in the exact format Sheet!Cell=Value
7. The same compiled Java program is executed once per scenario.
8. The Java program must parse all runtime overrides and apply them to the workbook's direct/non-formula input cells for the current scenario before computing dependent values.
9. If a direct input cell is not overridden at runtime, use the workbook's default input value from the workbook metadata.
10. The Java program must create/write both files.
11. CSV header must be exactly:
   Sheet,Cell,Value
12. CSV rows must represent the meaningful workbook output at cell level using the original sheet names and original cell coordinates.
13. Do NOT emit blank cells.
14. Do NOT emit obvious trailing artifact rows beyond the meaningful schedule.
15. Dates in CSV must use ISO format:
   - yyyy-MM-dd if no time
   - yyyy-MM-dd HH:mm:ss if time exists
16. Numbers in CSV must use plain decimal strings:
   - no commas
   - no scientific notation
17. For CSV text/header cells, emit single-line text only:
   - replace CR/LF/tab with a single space
   - collapse repeated spaces
   - trim leading/trailing spaces
18. Do NOT rename or paraphrase header text.
19. Preserve exact coordinate meaning, not just visual table shape.
20. Prefer BigDecimal for financial calculations where appropriate.
21. If diagnostics indicate row shifts, opening-balance issues, or schedule alignment defects, fix the underlying placement logic.
22. The repaired code must work across ALL listed scenarios, especially preserving already-passing scenarios.
23. Runtime override values may represent decimals, integers, negative numbers, dates, datetimes, booleans, or strings.
24. Do NOT hardcode only one scenario's values.
25. Make the smallest reliable logic correction that fixes failing scenarios while preserving passing ones.

TEST CASE SCENARIOS THE JAVA PROGRAM MUST SUPPORT:
{test_cases_text}

PASSING SCENARIO IDS (must remain correct):
{passed_scenario_ids_text}

FAILING SCENARIO IDS (must be fixed):
{failed_scenario_ids_text}

SCENARIO SUMMARY:
{scenario_summary_text}

PASSING SCENARIO RESULTS:
{passed_scenarios_text}

FAILING SCENARIO RESULTS:
{failed_scenarios_text}

COMPARISON WARNINGS / ERRORS:
{compare_error}

DIAGNOSTIC SUMMARY FOR FAILED SCENARIOS:
{mismatch_analysis_text}

EXPECTED EXCEL CELLS FOR FAILED SCENARIOS:
{unmatched_expected_text}

UNMATCHED CELLS TO FIX FOR FAILED SCENARIOS:
{unmatched_details_text}

Current Java code:
{current_java_code}

Original workbook metadata:
{workbook_text}
```
