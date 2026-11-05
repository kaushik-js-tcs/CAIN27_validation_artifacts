# RQ3: Ablation Study (Table IV)

Ablation runs for the CAIN 2027 submission _Validation-Driven Excel-to-Java Synthesis with Cell-to-Code Traceability and Targeted Repair_.

Each variant removes **one** component of our pipeline and re-runs synthesis and repair on all 8 workbooks with the same model (GPT-5.6-sol) and settings. The reference **full pipeline** is the run in [`../rq1_results_and_logs/Ours/in_loop/`](../rq1_results_and_logs/Ours/in_loop/).

## Results (Table IV)

WBPass counts the workbooks for which the variant produced a program on which every evaluated scenario matches Excel. Cost columns are ratios of the variant's **summed** time, tokens and repairs over the 8 workbooks to those of the full pipeline.

| Variant                 | Folder                                                                           | WBPass | Time × Full | Tokens × Full | Repairs × Full | Tokens    | Repairs | Time (s) |
| ----------------------- | -------------------------------------------------------------------------------- | ------ | ----------- | ------------- | -------------- | --------- | ------- | -------- |
| Full pipeline           | [`../rq1_results_and_logs/Ours/in_loop/`](../rq1_results_and_logs/Ours/in_loop/) | 8/8    | 1.00        | 1.00          | 1.00           | 839,839   | 37      | 7,652.7  |
| − Fault localization    | [`without_fault_localization/`](without_fault_localization/)                     | 7/8    | 1.71        | 1.12          | 1.22           | 939,983   | 45      | 13,069.5 |
| − Multi-scenario repair | [`without_multi_scenario_repairs/`](without_multi_scenario_repairs/)             | 7/8    | 0.92        | 0.89          | 0.78           | 749,391   | 29      | 7,061.9  |
| − XlType                | [`without_xltype/`](without_xltype/)                                             | 7/8    | 1.26        | 1.78          | 6.97           | 1,492,898 | 258     | 9,664.9  |
| − Loop inference        | [`without_loop_inference/`](without_loop_inference/)                             | 4/8    | 1.73        | 6.54          | 7.68           | 5,489,725 | 284     | 13,263.9 |

All values match Table IV of the paper and can be recomputed from the files in this folder (see _Reproducing Table IV_).

**Key observations (from the data):**

- Removing **loop inference** has the largest effect: 4 workbooks fail (C, F, G, H), tokens grow 6.5× and repairs 7.7×, driven mostly by compile-error repairs of cell-by-cell code (e.g. 119 on H, 63 on F).
- Removing **XlType** fails C (SurrenderCharge) after 227 compile-error repairs, which accounts for most of the 7× increase in repairs.
- Removing **fault localization** fails H (gaulEngine) and increases time by 1.7×.
- Removing **multi-scenario repair** is slightly cheaper (0.9× time and tokens) but fails F (`set_16`, 2 mismatching cells).

## Variants and per-workbook results

### − Fault localization (`without_fault_localization/`)

Repair without the mismatch frontier / sub-formula localization: the repair prompt receives the failing block without the localized responsible computation.

| WB  | Workbook                                | Pass | Scenarios evaluated | Mismatching cells | Failing scenarios                                                                                          | Tokens  | Repairs | Time (s) |
| --- | --------------------------------------- | ---- | ------------------- | ----------------- | ---------------------------------------------------------------------------------------------------------- | ------- | ------- | -------- |
| A   | `sip_cal`                               | ✓    | 11                  | 0                 | –                                                                                                          | 29,349  | 1       | 307.1    |
| B   | `mortgage_calculator`                   | ✓    | 11                  | 0                 | –                                                                                                          | 16,727  | 0       | 209.7    |
| C   | `surrendercharge_freeamount`            | ✓    | 11                  | 0                 | –                                                                                                          | 97,941  | 3       | 865.3    |
| D   | `pmi_db_life`                           | ✓    | 11                  | 0                 | –                                                                                                          | 36,892  | 1       | 353.3    |
| E   | `riderquarterlyfeecalc`                 | ✓    | 11                  | 0                 | –                                                                                                          | 127,259 | 10      | 1,390.9  |
| F   | `policyvalue_cashvalue_mrcv_premiumtax` | ✓    | 11                  | 0                 | –                                                                                                          | 133,816 | 6       | 1,715.1  |
| G   | `lifeinsurancecalculator`               | ✓    | 11                  | 0                 | –                                                                                                          | 124,612 | 5       | 1,497.7  |
| H   | `gaulengine`                            | ✗    | 11                  | 7,857             | `set_0`, `set_1`, `set_10`, `set_11`, `set_12`, `set_13`, `set_14`, `set_15`, `set_16`, `set_17`, `set_18` | 373,387 | 19      | 6,730.5  |

### − Multi-scenario repair (`without_multi_scenario_repairs/`)

Repairs are validated against the current failing scenario only, without cumulative regression checks over previously passing scenarios.

| WB  | Workbook                                | Pass | Scenarios evaluated | Mismatching cells | Failing scenarios | Tokens  | Repairs | Time (s) |
| --- | --------------------------------------- | ---- | ------------------- | ----------------- | ----------------- | ------- | ------- | -------- |
| A   | `sip_cal`                               | ✓    | 11                  | 0                 | –                 | 29,816  | 1       | 337.6    |
| B   | `mortgage_calculator`                   | ✓    | 11                  | 0                 | –                 | 16,242  | 0       | 197.0    |
| C   | `surrendercharge_freeamount`            | ✓    | 11                  | 0                 | –                 | 93,459  | 2       | 888.6    |
| D   | `pmi_db_life`                           | ✓    | 11                  | 0                 | –                 | 30,763  | 0       | 341.0    |
| E   | `riderquarterlyfeecalc`                 | ✓    | 11                  | 0                 | –                 | 133,850 | 10      | 1,200.7  |
| F   | `policyvalue_cashvalue_mrcv_premiumtax` | ✗    | 11                  | 2                 | `set_16`          | 92,324  | 1       | 1,158.0  |
| G   | `lifeinsurancecalculator`               | ✓    | 11                  | 0                 | –                 | 102,886 | 3       | 837.8    |
| H   | `gaulengine`                            | ✓    | 11                  | 0                 | –                 | 250,051 | 12      | 2,101.1  |

### − XlType (`without_xltype/`)

Generated code uses plain Java types instead of the Excel-compatible `XlType` value model and runtime.

| WB  | Workbook                                | Pass | Scenarios evaluated | Mismatching cells | Failing scenarios | Tokens  | Repairs | Time (s) |
| --- | --------------------------------------- | ---- | ------------------- | ----------------- | ----------------- | ------- | ------- | -------- |
| A   | `sip_cal`                               | ✓    | 11                  | 0                 | –                 | 30,449  | 1       | 334.2    |
| B   | `mortgage_calculator`                   | ✓    | 11                  | 0                 | –                 | 16,283  | 0       | 209.2    |
| C   | `surrendercharge_freeamount`            | ✗    | 1                   | 174               | `set_0`           | 801,992 | 230     | 3,489.6  |
| D   | `pmi_db_life`                           | ✓    | 11                  | 0                 | –                 | 37,229  | 1       | 356.2    |
| E   | `riderquarterlyfeecalc`                 | ✓    | 11                  | 0                 | –                 | 106,561 | 7       | 1,092.4  |
| F   | `policyvalue_cashvalue_mrcv_premiumtax` | ✓    | 11                  | 0                 | –                 | 101,359 | 2       | 1,151.0  |
| G   | `lifeinsurancecalculator`               | ✓    | 11                  | 0                 | –                 | 100,329 | 3       | 859.0    |
| H   | `gaulengine`                            | ✓    | 11                  | 0                 | –                 | 298,696 | 14      | 2,173.1  |

### − Loop inference (`without_loop_inference/`)

Repeated regions are not lifted into loops by the loop-aware IR; formulas are synthesized cell by cell.

| WB  | Workbook                                | Pass | Scenarios evaluated | Mismatching cells | Failing scenarios                                                                                          | Tokens    | Repairs | Time (s) |
| --- | --------------------------------------- | ---- | ------------------- | ----------------- | ---------------------------------------------------------------------------------------------------------- | --------- | ------- | -------- |
| A   | `sip_cal`                               | ✓    | 11                  | 0                 | –                                                                                                          | 241,396   | 4       | 460.2    |
| B   | `mortgage_calculator`                   | ✓    | 11                  | 0                 | –                                                                                                          | 117,639   | 0       | 220.5    |
| C   | `surrendercharge_freeamount`            | ✗    | 11                  | 129               | `set_0`                                                                                                    | 165,854   | 7       | 1,124.7  |
| D   | `pmi_db_life`                           | ✓    | 11                  | 0                 | –                                                                                                          | 36,720    | 1       | 357.5    |
| E   | `riderquarterlyfeecalc`                 | ✓    | 11                  | 0                 | –                                                                                                          | 112,057   | 7       | 1,153.4  |
| F   | `policyvalue_cashvalue_mrcv_premiumtax` | ✗    | 11                  | 6,718             | `set_0`                                                                                                    | 692,518   | 65      | 1,555.8  |
| G   | `lifeinsurancecalculator`               | ✗    | 11                  | 2,002             | `set_0`, `set_1`, `set_10`, `set_11`, `set_12`, `set_13`, `set_14`, `set_15`, `set_16`, `set_17`, `set_18` | 599,808   | 16      | 1,773.5  |
| H   | `gaulengine`                            | ✗    | 1                   | 7,217             | `set_0`                                                                                                    | 3,523,733 | 184     | 6,618.3  |

### Full pipeline reference (from `../rq1_results_and_logs/Ours/in_loop/<wb>/logs/summary.txt`)

| WB  | Workbook                                | Tokens  | Repairs | Time (s) |
| --- | --------------------------------------- | ------- | ------- | -------- |
| A   | `sip_cal`                               | 29,534  | 1       | 321.3    |
| B   | `mortgage_calculator`                   | 36,434  | 3       | 350.9    |
| C   | `surrendercharge_freeamount`            | 101,638 | 3       | 901.1    |
| D   | `pmi_db_life`                           | 31,616  | 0       | 490.1    |
| E   | `riderquarterlyfeecalc`                 | 112,846 | 7       | 1,168.2  |
| F   | `policyvalue_cashvalue_mrcv_premiumtax` | 102,096 | 2       | 1,180.3  |
| G   | `lifeinsurancecalculator`               | 136,070 | 6       | 906.9    |
| H   | `gaulengine`                            | 289,605 | 15      | 2,333.9  |

## Folder structure

```
rq3-ablation/
├── without_fault_localization/<workbook>/
│   ├── logs/
│   │   ├── summary.txt          tokens, repair counts by type, per-block validation, timings
│   │   └── execution_log.json   ordered pipeline steps
│   └── output/
│       ├── <block>_diff_set_<n>.csv   per-cell Excel vs. Java comparison for scenario set_<n>
│       └── <block>_result.json        values computed by the generated code
├── without_multi_scenario_repairs/<workbook>/   (same structure)
├── without_xltype/<workbook>/                   (same structure)
└── without_loop_inference/<workbook>/           (same structure)
```

The `diff_set` CSV columns are the same as in `results/rq1_results_and_logs/Ours/`: `sheet, col, row, xltype, expected, actual, status, remark, scn_id`.

## Reproducing Table IV

| Quantity            | Source                                                                                                                  |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Pass for a workbook | `Final status: 0` in `logs/summary.txt`, equivalently every row of every `output/*_diff_set_*.csv` has `status = MATCH` |
| Tokens              | `total_tokens` in `logs/summary.txt`                                                                                    |
| Repairs             | `## Overall Repairs` in `logs/summary.txt` (breakdown by type in `## Detailed Repairs`)                                 |
| Time                | Sum of the `[TIME]` stage lines (preprocessing, seed synthesis, validation on scenarios) in `logs/summary.txt`          |
| Ratios              | Sum over the 8 workbooks for the variant ÷ the same sum for the full pipeline                                           |
