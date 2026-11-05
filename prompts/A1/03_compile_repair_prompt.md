# P3 – Compile-Error Repair Prompt

[← Back to Prompt Index](README.md)

**Used when:** The generated Java code fails to compile with `javac`.

**Purpose:** Fixes compilation errors while keeping the workbook logic and the same program contract (arguments, overrides, and TXT/CSV output).

## Placeholders

| Placeholder | Content |
|---|---|
| `{test_cases_text}` | Test scenarios as input-cell overrides (`Sheet!Cell=Value`) |
| `{compile_stderr}` | Compiler errors from `javac` |
| `{current_java_code}` | Java code from the previous attempt |
| `{workbook_text}` | Workbook metadata: sheets, cell values, formulas, and named ranges |

## Prompt Template

```text
The previous Java code failed to compile. Fix it.

Hard requirements:
1. Return ONLY Java code. No markdown fences. No explanation.
2. The Java code MUST contain exactly one public class named WorkbookProgram.
3. No package declaration.
4. Use only standard Java libraries.
5. Keep the code equivalent to the workbook logic.
6. Do NOT read the Excel file at runtime.
7. main(String[] args) must accept:
   - args[0] = output TXT file path
   - args[1] = output CSV file path
   - args[2] = scenario id
   - args[3...] = zero or more runtime overrides in the exact format Sheet!Cell=Value
8. The same compiled Java program will be executed many times, once per scenario.
9. The Java program must parse all runtime overrides and apply them to the workbook's direct/non-formula input cells for the current scenario before computing dependent values.
10. If a direct input cell is not overridden at runtime, use the workbook's default input value from the workbook metadata.
11. The Java program must create/write both files.
12. TXT output must include the scenario id and current scenario inputs.
13. CSV header must be exactly:
   Sheet,Cell,Value
14. CSV rows must represent the meaningful workbook output at cell level using the original sheet names and original cell coordinates.
16. Do NOT emit blank cells.
17. Do NOT emit obvious trailing artifact rows beyond the meaningful schedule.
18. Dates in CSV must use ISO format:
   - yyyy-MM-dd if no time
   - yyyy-MM-dd HH:mm:ss if time exists
19. Numbers in CSV must use plain decimal strings:
   - no commas
   - no scientific notation
20. For CSV text/header cells, emit single-line text only:
   - replace CR/LF/tab with a single space
   - collapse repeated spaces
   - trim leading/trailing spaces
21. Do NOT rename or paraphrase header text.
22. Preserve exact coordinate meaning, not just visual table shape.
23. Prefer BigDecimal for financial calculations where appropriate.
24. The Java code must compile with javac on Windows.
25. Runtime override values may represent decimals, integers, negative numbers, dates, datetimes, booleans, or strings.
26. Do NOT hardcode only one scenario's values.

TEST CASE SCENARIOS THE JAVA PROGRAM MUST SUPPORT:
{test_cases_text}

Compilation errors:
{compile_stderr}

Current Java code:
{current_java_code}

Original workbook metadata:
{workbook_text}
```
