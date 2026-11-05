# Ours: Structured Validation-Driven Synthesis

Results of **our framework** (RQ1) for the CAIN 2027 submission _Validation-Driven Excel-to-Java Synthesis with Cell-to-Code Traceability and Targeted Repair_.

The framework combines deterministic workbook analysis (cell dependency graphs, a loop-aware intermediate representation, the `XlType` Excel-compatible runtime) with LLM-based synthesis, cell-to-code traceability $\phi$, and localized repair with cumulative regression checks. Prompt templates are in [`prompts/Ours/Prompt_Templates.md`](../../../prompts/Ours/Prompt_Templates.md).

## Evaluation settings

For each workbook, the company-supplied scenarios were randomly split once into two disjoint subsets that all approaches share:

| Folder     | Paper notation                            | Meaning                                                                                                                                   |
| ---------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `in_loop`  | $T_{in}$ (~20%, 45 scenarios)             | Scenarios available during synthesis and repair.                                                                                          |
| `out_loop` | $T_{out}$ (~80%, 176 scenarios), held-out | Never seen during synthesis. Once $T_{in}$ was satisfied, the program was frozen and evaluated once on $T_{out}$, with no further repair. |

A scenario (`set_<n>`) **passes** only if the Java program compiles, runs, and every formula cell matches Excel under the type-aware predicate ≈. Numbers and dates are compared within tolerances. Strings, Booleans and Excel error codes are compared exactly.

## Results (Ours columns of Table III)

| WB  | Workbook                                                                                  | In-loop passed | Held-out passed |
| --- | ----------------------------------------------------------------------------------------- | -------------- | --------------- |
| A   | [`sip_cal`](in_loop/sip_cal/)                                                             | 4/4            | 16/16           |
| B   | [`mortgage_calculator`](in_loop/mortgage_calculator/)                                     | 5/5            | 19/19           |
| C   | [`surrendercharge_freeamount`](in_loop/surrendercharge_freeamount/)                       | 5/5            | 20/20           |
| D   | [`pmi_db_life`](in_loop/pmi_db_life/)                                                     | 6/6            | 24/24           |
| E   | [`riderquarterlyfeecalc`](in_loop/riderquarterlyfeecalc/)                                 | 4/4            | 16/16           |
| F   | [`policyvalue_cashvalue_mrcv_premiumtax`](in_loop/policyvalue_cashvalue_mrcv_premiumtax/) | 6/6            | 22/22           |
| G   | [`lifeinsurancecalculator`](in_loop/lifeinsurancecalculator/)                             | 7/7            | 27/27           |
| H   | [`gaulengine`](in_loop/gaulengine/)                                                       | 8/8            | 32/32           |
|     | **Total**                                                                                 | **45/45**      | **176/176**     |

Every compared cell has `status = MATCH`: WBPass 8/8 and **100%** of held-out scenarios passed.

### Synthesis cost (from `logs/summary.txt`)

Synthesis and repair use only in-loop scenarios, so these figures describe the **one-time synthesis**. The frozen program was then run on the held-out scenarios, and `out_loop/<wb>/logs/summary.txt` repeats the same figures.

| WB  | Workbook                                | LLM calls | Tokens  | Repairs | Time (s) |
| --- | --------------------------------------- | --------- | ------- | ------- | -------- |
| A   | `sip_cal`                               | 8         | 29,534  | 1       | 321.3    |
| B   | `mortgage_calculator`                   | 7         | 36,434  | 3       | 350.9    |
| C   | `surrendercharge_freeamount`            | 20        | 101,638 | 3       | 901.1    |
| D   | `pmi_db_life`                           | 7         | 31,616  | 0       | 490.1    |
| E   | `riderquarterlyfeecalc`                 | 21        | 112,846 | 7       | 1,168.2  |
| F   | `policyvalue_cashvalue_mrcv_premiumtax` | 15        | 102,096 | 2       | 1,180.3  |
| G   | `lifeinsurancecalculator`               | 19        | 136,070 | 6       | 906.9    |
| H   | `gaulengine`                            | 37        | 289,605 | 15      | 2,333.9  |

Model: `gpt-5.6-sol` (GPT-5.6 via Azure OpenAI).

## Folder structure

```
results/rq1_results_and_logs/Ours/
├── in_loop/<workbook>/
│   ├── logs/
│   │   ├── summary.txt
│   │   └── execution_log.json
│   └── output/
│       ├── <block>_diff_set_<n>.csv
│       └── <block>_result.json
└── out_loop/<workbook>/        (same structure)
```

| File                              | Content                                                                                                                                |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `logs/summary.txt`                | LLM token usage, per-sheet modules and repairs, per-block scenario validation, stage timings, `Final status` (0 = all scenarios pass). |
| `logs/execution_log.json`         | Ordered pipeline steps: `ongoing_module`, `ongoing_sheet`, `remarks`.                                                                  |
| `output/<block>_diff_set_<n>.csv` | Cell-level comparison of Excel vs. generated Java for scenario `set_<n>`.                                                              |
| `output/<block>_result.json`      | Values computed by the generated Java code for the block.                                                                              |

A **block** is a formula region named `<sheet>_<startCol>_<startRow>_<endCol>_<endRow>` (e.g. `calculator_A_3_F_362`); loop regions are synthesized as a single loop through the loop-aware IR.

### `diff_set` CSV columns

| Column                | Meaning                                                 |
| --------------------- | ------------------------------------------------------- |
| `sheet`, `col`, `row` | Cell address                                            |
| `xltype`              | `XlType` value type (e.g. `XlNumber`, `XlDateSerial`)   |
| `expected`            | Value recalculated by Excel (oracle)                    |
| `actual`              | Value computed by generated Java                        |
| `status`              | `MATCH` / `MISMATCH` under ≈                            |
| `remark`              | `loop_cell` (inside a repeated region) or `normal_cell` |
| `scn_id`              | Scenario ID (`set_<n>`)                                 |

**Reproducing Table III:** a scenario passes if every row with that `scn_id`, across all `diff_set` files of the workbook, has `status = MATCH`.
