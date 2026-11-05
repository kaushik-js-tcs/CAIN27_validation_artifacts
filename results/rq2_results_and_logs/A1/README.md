### Final Results – A1 Implementation

**These are the results of the A1 implementation** (global agentic repair). They are used for RQ1 (Table III) and for the failure analysis in RQ2 (Table II).

Prompts: [`prompts/A1/`](../../../prompts/A1/README.md). Defect labels: [`analysis/defect_labels.xlsx`](../../../analysis/defect_labels.xlsx).

This artifact contains the evaluation results of the Java programs generated from Excel workbooks
by the **A1** implementation. Each A1-generated Java program was run on a set of test scenarios,
and its output was compared cell-by-cell against the values computed by the original Excel workbook.

#### 1. Folder Structure

```
results/rq2_results_and_logs/A1/   (A1 implementation results)
├── InLoopResults/
│   └── <Workbook>/
│       ├── final_logs.json
│       └── scenario_comparisons/
│           └── set_<n>/
│               └── set_<n>_comparison_report.xlsx
└── OutLoopResults/
    └── <Workbook>/
        ├── final_logs.json
        └── scenario_comparisons/
            └── set_<n>/
                └── set_<n>_comparison_report.xlsx
```

| Level                           | Description                                                                            |
| ------------------------------- | -------------------------------------------------------------------------------------- |
| `A1/`                           | Root folder containing all **A1 implementation** results.                              |
| `InLoopResults/`                | A1 results for the test scenarios used **inside** the iterative refinement loop.       |
| `OutLoopResults/`               | A1 results for **held-out** test scenarios that were **not** used during refinement.   |
| `<Workbook>/`                   | One folder per Excel workbook.                                                         |
| `<Workbook>/final_logs.json`    | Workbook-level summary of the A1 run for all scenarios in that folder (see Section 3). |
| `scenario_comparisons/set_<n>/` | Excel-vs-Java comparison report for test scenario `n`.                                 |
| `set_<n>`                       | One test scenario (one set of input values). Set numbers are scenario IDs.             |

**Path examples (`sip_cal`):**

- `InLoopResults/sip_cal/final_logs.json`
- `InLoopResults/sip_cal/scenario_comparisons/set_0/set_0_comparison_report.xlsx`
- `OutLoopResults/sip_cal/final_logs.json`
- `OutLoopResults/sip_cal/scenario_comparisons/set_4/set_4_comparison_report.xlsx`

#### 2. Workbooks and Scenario Counts

| Workbook                              | InLoopResults sets  | OutLoopResults sets                                                                      | Results available             |
| ------------------------------------- | ------------------- | ---------------------------------------------------------------------------------------- | ----------------------------- |
| LifeInsuranceCalculator               | 7 (`set_0`–`set_6`) | 27 (`set_7`–`set_33`)                                                                    | Yes                           |
| mortgage_Calculator                   | 5 (`set_0`–`set_4`) | 19 (`set_5`–`set_9`, `set_20`, `set_21`, `set_25`, `set_27`–`set_32`, `set_34`–`set_38`) | Yes                           |
| PMI_DB_Life                           | 6 (`set_0`–`set_5`) | 24 (`set_6`–`set_9`, `set_20`–`set_39`)                                                  | Yes                           |
| PolicyValue_CashValue_MRCV_PremiumTax | 6 (`set_0`–`set_5`) | 22 (`set_6`–`set_25`, `set_27`, `set_28`)                                                | Yes                           |
| RiderQuarterlyFeeCalc                 | 4 (`set_0`–`set_3`) | 16 (`set_4`–`set_15`, `set_20`–`set_23`)                                                 | Yes                           |
| sip_cal                               | 4 (`set_0`–`set_3`) | 16 (`set_4`–`set_16`, `set_20`–`set_22`)                                                 | Yes                           |
| SurrenderCharge_FreeAmount            | 5 (`set_0`–`set_4`) | 20 (`set_5`–`set_9`, `set_22`–`set_36`)                                                  | Yes                           |
| GaulEngine                            | 8                   | 32                                                                                       | **No** (not executable on A1) |
| **Total (available results)**         | **37**              | **144**                                                                                  | 7 of 8 workbooks              |
| **Total (including GaulEngine)**      | **45**              | **176**                                                                                  | —                             |

**Note – GaulEngine:** The evaluation also includes the `GaulEngine` Excel workbook, with
8 InLoopResults scenario IDs and 32 OutLoopResults scenario IDs. Its results are **not available**
in this artifact because the workbook could not be executed with the A1 implementation.
For this reason, there is no `GaulEngine` folder (and no `final_logs.json`) under `InLoopResults/` or `OutLoopResults/`.

#### 3. Final Logs Format

**File:** `<Workbook>/final_logs.json`

Each workbook folder in `InLoopResults/` and `OutLoopResults/` has one `final_logs.json` file.
It gives the overall result of the A1-generated Java program for **all scenarios in that folder**:
whether it compiled and ran, how many scenarios passed or failed, and how many cells matched.

##### Fields

| Field                  | Type             | Description                                                 |
| ---------------------- | ---------------- | ----------------------------------------------------------- |
| `final_status`         | string           | Overall result of the run, e.g. `SUCCESS`.                  |
| `final_message`        | string           | Short human-readable summary of the result.                 |
| `compile_success`      | boolean          | `true` if the A1-generated Java program compiled.           |
| `run_success`          | boolean          | `true` if the Java program ran on the test scenarios.       |
| `compare_success`      | boolean          | `true` if the Excel-vs-Java comparison completed.           |
| `scenario_total_count` | integer          | Number of scenarios evaluated.                              |
| `scenario_pass_count`  | integer          | Number of scenarios where all compared cells matched.       |
| `scenario_fail_count`  | integer          | Number of scenarios with at least one unmatched cell.       |
| `scenario_all_passed`  | boolean          | `true` if every scenario passed.                            |
| `scenario_some_passed` | boolean          | `true` if some, but not all, scenarios passed.              |
| `scenario_none_passed` | boolean          | `true` if no scenario passed.                               |
| `passed_scenario_ids`  | list of strings  | IDs of the passed scenarios (e.g. `set_0`).                 |
| `failed_scenario_ids`  | list of strings  | IDs of the failed scenarios.                                |
| `matched_count`        | integer          | Total matched cells across all scenarios.                   |
| `unmatched_count`      | integer          | Total unmatched cells across all scenarios.                 |
| `total_compared_count` | integer          | Total cells compared (`matched_count` + `unmatched_count`). |
| `time_in_sec`          | integer / `null` | Execution time in seconds; `null` if not applicable.        |
| `total_tokens`         | integer / `null` | Total number of tokens; `null` if not applicable.           |
| `repair_attempts`      | integer / `null` | Number of repair attempts; `null` if not applicable.        |

##### Example – `InLoopResults/sip_cal/final_logs.json`

```json
{
  "final_status": "SUCCESS",
  "final_message": "All scenarios passed Excel-vs-Java comparison.",
  "compile_success": true,
  "run_success": true,
  "compare_success": true,
  "scenario_total_count": 4,
  "scenario_pass_count": 4,
  "scenario_fail_count": 0,
  "scenario_all_passed": true,
  "scenario_some_passed": false,
  "scenario_none_passed": false,
  "passed_scenario_ids": ["set_0", "set_1", "set_2", "set_3"],
  "failed_scenario_ids": [],
  "matched_count": 1524,
  "unmatched_count": 0,
  "total_compared_count": 1524,
  "time_in_sec": 79,
  "total_tokens": 33724,
  "repair_attempts": 0
}
```

For `sip_cal`, the `OutLoopResults/sip_cal/final_logs.json` file shows all 16 scenarios passed
(`set_4`–`set_16`, `set_20`–`set_22`) with 14,608 of 14,608 cells matched.

#### 4. Comparison Report Format

**File:** `scenario_comparisons/set_<n>/set_<n>_comparison_report.xlsx`

Each report has three sheets:

| Sheet            | Contents                                                                                                                   |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `Summary`        | Summary of Matched and Unmatched Expected Cells.                                                                           |
| `CellComparison` | Every compared formula cell for the scenario (matched and unmatched).                                                      |
| `UnmatchedOnly`  | Only the cells where the A1 Java value does **not** match the Excel value. An empty sheet means the scenario passed fully. |

##### Columns (`CellComparison` and `UnmatchedOnly`)

| Column       | Description                                                        | Example    |
| ------------ | ------------------------------------------------------------------ | ---------- |
| `Sheet`      | Name of the worksheet in the original Excel workbook.              | `MRCV`     |
| `Cell`       | Cell address in that worksheet.                                    | `B3`       |
| `ExcelValue` | Expected value calculated by Excel (ground truth).                 | `85382.59` |
| `JavaValue`  | Value produced by the A1-generated Java program for the same cell. | `85382.58` |
| `Status`     | Comparison result: whether `JavaValue` matches `ExcelValue`.       | `Matched`  |

#### 5. How to Read the Results

1. Pick the setting: `InLoopResults` (scenarios seen during refinement) or `OutLoopResults` (unseen scenarios).
2. Open the workbook folder and check `final_logs.json` for the overall result:
   - `scenario_all_passed: true` means every scenario passed.
   - Otherwise, `failed_scenario_ids` lists the scenarios to inspect.
3. For a scenario, open `scenario_comparisons/set_<n>/set_<n>_comparison_report.xlsx`:
   - Check `UnmatchedOnly` first. If it is empty, the A1-generated Java program reproduced Excel exactly for that scenario.
   - Otherwise, use `Sheet` + `Cell` to locate the formula in the original workbook and compare `ExcelValue` with `JavaValue`.
4. A scenario **passes** when all its cells match, and **fails** when at least one cell does not match.
