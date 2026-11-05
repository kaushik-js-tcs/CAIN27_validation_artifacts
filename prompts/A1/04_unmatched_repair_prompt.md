# P4 – Mismatch Repair Prompt

[← Back to Prompt Index](README.md)

**Used when:** The Java code compiles and runs, but **no** scenario matches Excel.

**Purpose:** Fixes logic errors found by the cell-by-cell comparison, using diagnostics from one failed scenario. Expected Excel values and formulas are treated as ground truth, and structural errors (missing cells, row shifts) are fixed first.

## Placeholders

| Placeholder | Content |
|---|---|
| `{test_cases_text}` | Test scenarios as input-cell overrides (`Sheet!Cell=Value`) |
| `{compare_error}` | Comparison warnings or errors, or `(None)` |
| `{mismatch_analysis_text}` | Summary of mismatches: counts, problem columns, missing/extra cells, and row-shift patterns |
| `{unmatched_expected_text}` | Expected Excel value and formula for each unmatched cell |
| `{unmatched_details_text}` | Excel value vs Java value for each unmatched cell |
| `{current_java_code}` | Java code from the previous attempt |
| `{workbook_text}` | Workbook metadata: sheets, cell values, formulas, and named ranges |

## Prompt Template

```text
The previous Java code compiled and ran, but the Excel-vs-Java comparison still has mismatches.

Important repair strategy:
A. Fix structural omissions first.
B. If diagnostics show a repeated row/column shift, fix the underlying schedule alignment instead of changing decimal formatting only.
C. If many cells in one column are wrong in a repeated pattern, infer the underlying logic defect and repair the logic once.
D. Use the EXPECTED EXCEL CELLS section as the source of truth for those coordinates.
E. Preserve cells that already match.
F. When ExcelFormula is provided for an unmatched cell, you MUST consider that formula while repairing the Java logic and emitted CSV output.
G. If ExcelFormula is not present, treat that cell as a direct input/static/non-formula value that still must be emitted exactly.
H. The same compiled Java program is executed once per scenario with one scenario id and one set of runtime overrides at a time.
I. Repair the scenario-aware logic, not just a single hardcoded case.

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
7. The Java program must parse all runtime overrides and apply them to the workbook's direct/non-formula input cells for the current scenario before computing dependent values.
8. If a direct input cell is not overridden at runtime, use the workbook's default input value from the workbook metadata.
9. The Java program must create/write both files.
10. CSV header must be exactly:
   Sheet,Cell,Value
11. Preserve logic and cells that already match. Make targeted fixes for the unmatched cells.
12. Correct the underlying cause, not just formatting, when a repeated mismatch pattern is present.
13. Every unmatched item listed below must be corrected using the exact same sheet name and cell coordinate.
14. Do NOT emit blank cells.
15. Do NOT emit obvious trailing artifact rows beyond the meaningful schedule.
16. Dates in CSV must use ISO format:
   - yyyy-MM-dd if no time
   - yyyy-MM-dd HH:mm:ss if time exists
17. Numbers in CSV must use plain decimal strings:
   - no commas
   - no scientific notation
18. For CSV text/header cells, emit single-line text only:
   - replace CR/LF/tab with a single space
   - collapse repeated spaces
   - trim leading/trailing spaces
19. Do NOT rename or paraphrase header text.
20. The repaired Java code must compile with javac on Windows.
21. Prefer BigDecimal for financial calculations where appropriate.
22. Do NOT only increase numeric precision if the mismatches indicate a row-offset or logic defect.
23. If diagnostics indicate that Java cell at row r matches Excel row r+1 or r-1, fix the schedule row placement or opening/ending balance logic.
24. For amortization schedules, verify whether Excel includes an opening balance row before payment 1.
25. When an unmatched cell includes ExcelFormula, use that formula as an important signal for repairing the corresponding Java logic.
26. The repaired code must work across all listed scenarios, not just one scenario.
27. Runtime override values may represent decimals, integers, negative numbers, dates, datetimes, booleans, or strings.

TEST CASE SCENARIOS THE JAVA PROGRAM MUST SUPPORT:
{test_cases_text}

COMPARISON WARNINGS / ERRORS:
{compare_error}

DIAGNOSTIC SUMMARY:
{mismatch_analysis_text}

EXPECTED EXCEL CELLS FOR THE CURRENT UNMATCHED SET:
{unmatched_expected_text}

UNMATCHED CELLS TO FIX:
{unmatched_details_text}

Current Java code:
{current_java_code}

Original workbook metadata:
{workbook_text}
```
