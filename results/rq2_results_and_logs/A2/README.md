# A2: Role-Specialized Agentic Workflow (Baseline)

Results of baseline **A2** for the _Validation-Driven Excel-to-Java Synthesis with Cell-to-Code Traceability and Targeted Repair_.

A2 follows role-based multi-agent frameworks (MetaGPT, ChatDev, Self-Collaboration). An orchestrator coordinates extraction, semantic analysis, formula grouping, Java generation, testing and repair agents. It used the same GPT-5.6-sol deployment, workbooks, scenario partition, Excel-derived oracle, matching predicate, Java environment and budget as the other approaches.

Prompt templates (synthesis S1–S6, repair R0–R4) are in [`prompts/A2/Prompt_Templates.md`](../../../prompts/A2/Prompt_Templates.md). The defect labels for A2's held-out failures (RQ2, Table II) are in [`analysis/defect_labels.xlsx`](../../../analysis/defect_labels.xlsx).

## Evaluation settings

For each workbook, the company-supplied scenarios were randomly split once into two disjoint subsets that all approaches share:

| Folder     | Paper notation                            | Meaning                                                                                                                                   |
| ---------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `in_loop`  | $T_{in}$ (~20%, 45 scenarios)             | Scenarios available during synthesis and repair.                                                                                          |
| `out_loop` | $T_{out}$ (~80%, 176 scenarios), held-out | Never seen during synthesis. Once $T_{in}$ was satisfied, the program was frozen and evaluated once on $T_{out}$, with no further repair. |

A scenario (`set_<n>`) **passes** only if the Java program compiles, runs, and every formula cell matches Excel under the type-aware predicate ≈. Numbers and dates are compared within tolerances. Strings, Booleans and Excel error codes are compared exactly.

## Results (A2 columns of Table III)

| WB  | Workbook                                                                                       | In-loop passed | Held-out passed | Held-out failed |
| --- | ---------------------------------------------------------------------------------------------- | -------------- | --------------- | --------------- |
| A   | [`sip_cal`](in_loop/runs/sip_cal/)                                                             | 4/4            | 13/16           | 3               |
| B   | [`mortgage_calculator`](in_loop/runs/mortgage_calculator/)                                     | 5/5            | 14/19           | 5               |
| C   | [`SurrenderCharge_FreeAmount`](in_loop/runs/SurrenderCharge_FreeAmount/)                       | 5/5            | 20/20           | 0               |
| D   | [`PMI_DB_Life`](in_loop/runs/PMI_DB_Life/)                                                     | 6/6            | 17/24           | 7               |
| E   | [`RiderQuarterlyFeeCalc`](in_loop/runs/RiderQuarterlyFeeCalc/)                                 | 4/4            | 12/16           | 4               |
| F   | [`PolicyValue_CashValue_MRCV_PremiumTax`](in_loop/runs/PolicyValue_CashValue_MRCV_PremiumTax/) | 6/6            | 22/22           | 0               |
| G   | [`LifeInsuranceCalculator`](in_loop/runs/LifeInsuranceCalculator/)                             | 7/7            | 27/27           | 0               |
| H   | [`gaulEngine`](in_loop/runs/gaulEngine/)                                                       | 8/8            | 32/32           | 0               |
|     | **Total**                                                                                      | **45/45**      | **157/176**     | **19**          |

A2 passed every in-loop scenario (WBPass 8/8), and **157/176 (89.2%)** held-out scenarios.

## Folder structure

```
results/rq2_results_and_logs/A2/
├── in_loop/runs/<workbook>/
│   ├── summary.json
│   ├── reports/
│   │   ├── passed_cells.json
│   │   └── failed_cells.json
│   └── logs/<workbook>_<timestamp>.jsonl
└── out_loop/runs/<workbook>/        (same structure)
```

| File                        | Content                                                                                                                                                                     |
| --------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `summary.json`              | `test_sets_passed` / `test_sets_total` (the numbers in Table III), `pass_rate`, `total_cells`, `validated_cells`, `elapsed_time_seconds`.                                   |
| `reports/passed_cells.json` | Every passing cell per scenario: `cell` (`Sheet!A1`), `expected` (Excel), `actual` (Java). `run_log` names the source log.                                                  |
| `reports/failed_cells.json` | Failing cells per scenario with `expected`, `actual` and `failure_reason` (e.g. `numeric_tolerance_exceeded`, `date_mismatch`, `missing_output`).                           |
| `logs/*.jsonl`              | Event log, one JSON object per line: `run_start`, one `test_set_result` per scenario (`passed`, `failing_cells_count`, `per_cell_pass_rate`, `per_cell_detail`), `run_end`. |

### Note

For gaulEngine (held-out), reports/passed_cells.json and the .jsonl event log are zipped (\*.zip) because they exceed GitHub's 100 MB file limit. Unzip them before use.
