# P2 – Initial Java Generation Prompt

[← Back to Prompt Index](README.md)

**Used when:** First generation attempt for each workbook.

**Purpose:** Converts the workbook into one Java program that runs once per test scenario, applies that scenario's input overrides, and writes a TXT report and a cell-level CSV (`Sheet,Cell,Value`) for comparison with Excel.

## Placeholders

| Placeholder | Content |
|---|---|
| `{test_cases_text}` | Test scenarios as input-cell overrides (`Sheet!Cell=Value`) |
| `{workbook_text}` | Workbook metadata: sheets, cell values, formulas, and named ranges |

## Prompt Template

```text
Convert the following extracted Excel workbook metadata into one equivalent Java program.

Hard requirements:
1. Return ONLY Java code. No markdown fences. No explanation.
2. The Java code MUST contain exactly one public class named WorkbookProgram.
3. No package declaration.
4. Use only standard Java libraries. Do not use Apache POI or any external library.
5. Do NOT read the Excel file at runtime.
6. Implement the workbook logic directly in code using the workbook metadata below.
7. The program must compile with javac on Windows.
8. main(String[] args) must accept:
   - args[0] = output TXT file path
   - args[1] = output CSV file path
   - args[2] = scenario id
   - args[3...] = zero or more runtime overrides in the exact format Sheet!Cell=Value
9. The same compiled Java program will be executed many times, once per scenario.
10. The Java program must parse all runtime overrides and apply them to the workbook's direct/non-formula input cells for the current scenario before computing dependent values.
11. If a direct input cell is not overridden at runtime, use the workbook's default input value from the workbook metadata.
12. Different scenarios must produce different outputs when the overridden inputs differ.
13. The CSV output must reflect the current scenario's actual input values and computed results.
15. The Java program must create/write both files.
16. TXT file must be human-readable and include:
   - scenario id
   - summary of important inputs actually used for the current scenario
   - computed schedule / output tables
   - final totals / final balance where applicable
17. CSV file must be machine-readable and must have this exact header:
   Sheet,Cell,Value
18. CSV rows must represent the meaningful workbook output at cell level using the original sheet names and original cell coordinates.
20. Do NOT emit blank cells.
21. Do NOT emit obvious trailing artifact rows beyond the meaningful schedule such as rows that only become blank/error/zero artifacts after the schedule ends.
22. Dates in CSV must use ISO format:
   - yyyy-MM-dd if no time
   - yyyy-MM-dd HH:mm:ss if time exists
23. Numbers in CSV must use plain decimal strings:
   - no commas
   - no scientific notation
24. Escape CSV values correctly by quoting fields when needed.
25. Preserve Excel logic as closely as possible, including formulas, named ranges, and date logic.
26. Prefer BigDecimal for financial calculations where appropriate.

VERY IMPORTANT FOR RUNTIME OVERRIDES:
27. Runtime override values may represent decimals, integers, negative numbers, dates in ISO yyyy-MM-dd, datetimes, booleans TRUE/FALSE, or raw strings.
28. Parse each override according to the target cell's workbook meaning.
29. Do NOT hardcode logic for only one scenario.
30. Build the code so the scenario id and runtime overrides control the current calculation.
31. The workflow will pass one scenario at a time. The program must correctly compute that scenario only.

TEST CASE SCENARIOS THE JAVA PROGRAM MUST SUPPORT:
{test_cases_text}

Workbook metadata:
{workbook_text}
```
