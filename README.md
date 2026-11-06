# Validation-Driven Excel-to-Java Synthesis with Cell-to-Code Traceability and Targeted Repair

---

## Overview

Insurance products are often specified as Excel workbooks. Moving that logic into production software means re-implementing it, for example in Java, while preserving Excel's behaviour. The paper presents a **validation-driven, LLM-assisted framework** that translates industrial Excel workbooks into Java. It combines:

- **Deterministic workbook analysis:** cell-level dependency graphs, detection of repeated regions, and a _loop-aware intermediate representation (IR)_.
- **`XlType`:** an Excel-compatible Java value model and runtime that handles coercion, blank and error propagation, dates, arrays and built-in functions.
- **LLM-based synthesis:** the LLM first generates Excel-function utilities, then sheet and component code in dependency order.
- **Cell-to-code traceability:** a mapping $\phi$ links every formula cell to the Java computation that produces it.
- **Targeted repair:** a mismatch frontier and sub-formula localization find the responsible computation, so only that computation is repaired. Cumulative regression checks then verify the repair against all previously passing scenarios.

Values recalculated in Excel serve as the **test oracle**. The framework is compared with two agentic generate-test-repair baselines on **8 industrial insurance workbooks** (89,568 formula cells, 221 company-supplied scenarios).

This repository contains the prompts, evaluation outputs and analysis artifacts for all three approaches and all three research questions (RQ1–RQ3).

---

## Compared approaches

| Approach                                         | Description                                                                                                                                                                                                                                                                                   | Prompts                                             | Results                                                                    |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------- |
| **A1: Global agentic repair**                    | Follows the execution-feedback self-repair and verbal-reflection paradigms (Self-Debugging, Reflexion, Self-Edit). A synthesis agent generates the complete Java program. A critic then uses compilation, runtime and validation feedback to drive whole-program or large-scope regeneration. | [`prompts/A1/`](prompts/A1/README.md)               | [`results/rq2_results_and_logs/A1/`](results/rq2_results_and_logs/A1/)     |
| **A2: Role-specialized agentic workflow**        | Follows role-based multi-agent frameworks (MetaGPT, ChatDev, Self-Collaboration). An orchestrator coordinates extraction, semantic analysis, formula grouping, Java generation, testing and repair agents.                                                                                    | [`prompts/A2/`](prompts/A2/Prompt_Templates.md)     | [`results/rq2_results_and_logs/A2/`](results/rq2_results_and_logs/A2/)     |
| **Ours: Structured validation-driven synthesis** | Dependency analysis, loop-aware IR, `XlType` runtime, cell-to-code traceability and localized repair.                                                                                                                                                                                         | [`prompts/Ours/`](prompts/Ours/Prompt_Templates.md) | [`results/rq1_results_and_logs/Ours/`](results/rq1_results_and_logs/Ours/) |

**Fairness controls:** All approaches used the same GPT-5.6-sol deployment, workbooks, scenario partition, Excel-derived oracle, matching predicate, Java environment and budget. They also received the same instruction not to hard-code scenario inputs or expected outputs. See [`environment/model_and_runtime_config.md`](environment/model_and_runtime_config.md).

### How the prompts differ

|                   | A1                                                                                                | A2                                                                              | Ours                                                                                       |
| ----------------- | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Unit of synthesis | Whole Java program in one generation (P1 + P2)                                                    | One formula or cluster at a time (S1–S6), rendered into Jinja templates         | One sheet or loop block from pseudocode (T2), using an LLM-built `ExcelUtils` library (T1) |
| Repair            | Whole-program regeneration from compile errors (P3), unmatched cells (P4) or counterexamples (P5) | Failure attributed to a decision level and the owning agent re-prompted (R0–R4) | Localized: first mismatch per column (T3) or compiler/runtime errors (T4)                  |

---

## Experimental protocol

### Scenario partition

For each workbook, the company-supplied scenario suite $T$ was randomly split once into two disjoint subsets:

| Setting                | Folder name                   | Share                | Use                                                                                                                                                   |
| ---------------------- | ----------------------------- | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **In-loop** $T_{in}$   | `in_loop` / `InLoopResults`   | ~20% (45 scenarios)  | Available during synthesis and repair.                                                                                                                |
| **Held-out** $T_{out}$ | `out_loop` / `OutLoopResults` | ~80% (176 scenarios) | Never seen during synthesis. Once an approach satisfied $T_{in}$, its program was **frozen** and evaluated once on $T_{out}$, with no further repair. |

### Oracle and pass criterion

For each scenario, its inputs are applied to the workbook and Microsoft Excel recalculates all formulas. A scenario **passes** only if the Java program compiles, runs successfully, and **every formula cell** matches Excel under the type-aware predicate ≈:

- numbers and dates match within configurable tolerances;
- strings, Booleans and Excel error codes must match exactly;
- blanks and arrays follow Excel-compatible normalization.

---

## Workbooks (Table I)

| WB  | Folder name                             | Formula cells | Cross-sheet deps. | Max. nesting | Loop cells | Scenarios | $T_{in}$ | $T_{out}$ |
| --- | --------------------------------------- | ------------- | ----------------- | ------------ | ---------- | --------- | -------- | --------- |
| A   | `sip_cal`                               | 1,917         | 3                 | 2            | 1,912      | 20        | 4        | 16        |
| B   | `mortgage_Calculator`                   | 2,159         | 1                 | 3            | 2,159      | 24        | 5        | 19        |
| C   | `SurrenderCharge_FreeAmount`            | 164           | 5                 | 8            | 32         | 25        | 5        | 20        |
| D   | `PMI_DB_Life`                           | 73            | 2                 | 9            | 0          | 30        | 6        | 24        |
| E   | `RiderQuarterlyFeeCalc`                 | 88            | 6                 | 8            | 0          | 20        | 4        | 16        |
| F   | `PolicyValue_CashValue_MRCV_PremiumTax` | 7,036         | 10                | 4            | 6,704      | 28        | 6        | 22        |
| G   | `LifeInsuranceCalculator`               | 16,464        | 3                 | 4            | 15,311     | 34        | 7        | 27        |
| H   | `gaulEngine`                            | 61,667        | 8                 | 5            | 57,431     | 40        | 8        | 32        |
|     | **Total**                               | **89,568**    | –                 | –            | **83,549** | **221**   | **45**   | **176**   |

The table also maps each paper ID (A–H) to its folder name. Folder names differ in capitalization between approaches. For example, the Ours and ablation results use lowercase names such as `gaulengine`.

> **Formula cells vs. compared cells:** Table I gives the static formula-cell count of each workbook. The number of cells compared per scenario can be lower, because input-dependent tables (e.g. schedules whose length depends on the term) have fewer active rows in some scenarios. Each approach compares the cells its harness evaluates for that scenario, so per-scenario cell counts differ between approaches and from Table I.

---

## Results

### RQ1: Functional fidelity and robustness (Table II)

In the table, W is WBPass: 1 if the approach passes every in-loop scenario of the workbook. P/F is the number of passed/failed held-out scenarios. NA means no program was produced because the workbook exceeded the model context.

| WB        | $T_{in}$ | $T_{out}$ | A1 W  | A1 P   | A1 F   | A2 W  | A2 P    | A2 F   | **Ours W** | **Ours P** | **Ours F** |
| --------- | -------- | --------- | ----- | ------ | ------ | ----- | ------- | ------ | ---------- | ---------- | ---------- |
| A         | 4        | 16        | 1     | 16     | 0      | 1     | 13      | 3      | 1          | 16         | 0          |
| B         | 5        | 19        | 1     | 8      | 11     | 1     | 14      | 5      | 1          | 19         | 0          |
| C         | 5        | 20        | 1     | 16     | 4      | 1     | 20      | 0      | 1          | 20         | 0          |
| D         | 6        | 24        | 1     | 5      | 19     | 1     | 17      | 7      | 1          | 24         | 0          |
| E         | 4        | 16        | 1     | 12     | 4      | 1     | 12      | 4      | 1          | 16         | 0          |
| F         | 6        | 22        | 1     | 14     | 8      | 1     | 22      | 0      | 1          | 22         | 0          |
| G         | 7        | 27        | 1     | 13     | 14     | 1     | 27      | 0      | 1          | 27         | 0          |
| H         | 8        | 32        | 0     | NA     | NA     | 1     | 32      | 0      | 1          | 32         | 0          |
| **Total** | **45**   | **176**   | **7** | **84** | **60** | **8** | **157** | **19** | **8**      | **176**    | **0**      |

All values in this table can be reproduced from the files in this repository (see _Where to read each Table III number_ below).

Held-out pass rates were **47.7% (A1)**, **89.2% (A2)** and **100% (Ours)**.

### RQ2: Failure characteristics (Table III)

The 79 baseline held-out failures trace back to just **13 root defects**: 7 in A1 and 6 in A2. Per-failure labels are in [`analysis/Defect_Analysis_RQ2.xlsx`](analysis/Defect_Analysis_RQ2.xlsx); the underlying failing cells are in the A1 comparison reports and the A2 `reports/failed_cells.json` under [`results/rq2_results_and_logs/`](results/rq2_results_and_logs/).

| ID  | Defect category                  | Mode   | A1     | A2     |
| --- | -------------------------------- | ------ | ------ | ------ |
| D1  | Hard-coded reference data        | Silent | 19     | 5      |
| D2  | Lookup replaced by approximation | Silent | 14     | 0      |
| D3  | Unhandled atypical input         | Crash  | 15     | 5      |
| D4  | Omitted calculation logic        | Silent | 2      | 5      |
| D5  | Floating-point rounding          | Silent | 8      | 0      |
| D6  | Non-Excel comparison semantics   | Silent | 2      | 4      |
|     | **Total**                        |        | **60** | **19** |

74.7% of these failures produced **silently wrong values** rather than crashes.

### RQ3: Component contribution (Table IV)

Each variant removes one component from our pipeline. Cost columns are ratios of the variant's costs, summed over the 8 workbooks, to those of the full pipeline. Details and per-workbook results: [`results/rq3_ablation_study/`](results/rq3_ablation_study/README.md).

| Variant                 | WBPass | Time × Full | Tokens × Full | Repairs × Full |
| ----------------------- | ------ | ----------- | ------------- | -------------- |
| Full pipeline           | 8/8    | 1.00        | 1.00          | 1.00           |
| − Fault localization    | 7/8    | 1.71        | 1.12          | 1.22           |
| − Multi-scenario repair | 7/8    | 0.92        | 0.89          | 0.78           |
| − XlType                | 7/8    | 1.26        | 1.78          | 6.97           |
| − Loop inference        | 4/8    | 1.73        | 6.54          | 7.68           |

---

## Repository structure

```
CAIN27_validation_artifacts/
├── README.md                              ← this file
├── environment/
│   └── model_and_runtime_config.md        model deployment, Java/runtime and oracle settings
├── prompts/
│   ├── A1/                                P1 system, P2 generation, P3–P5 repair (+ README)
│   ├── A2/Prompt_Templates.md             S1–S6 synthesis, R0–R4 repair, configuration
│   └── Ours/Prompt_Templates.md           T1–T2 synthesis, T3–T4 repair, placeholders, limits
├── analysis/
│   └── Defect_Analysis_RQ2.xlsx                 RQ2 root-defect label for each baseline held-out failure
└── results/
    ├── rq1_results_and_logs/
    │   └── Ours/                          Ours, Table III
    │       ├── in_loop/<wb>/  logs/{summary.txt, execution_log.json}, output/<block>_diff_set_<n>.csv, <block>_result.json
    │       └── out_loop/<wb>/ same structure
    ├── rq2_results_and_logs/              baselines, Table III and Table II
    │   ├── A1/
    │   │   ├── InLoopResults/<wb>/   final_logs.json, scenario_comparisons/set_<n>/set_<n>_comparison_report.xlsx
    │   │   └── OutLoopResults/<wb>/  same structure (no gaulEngine: NA in Table III)
    │   └── A2/
    │       ├── in_loop/runs/<wb>/    summary.json, reports/{passed,failed}_cells.json, logs/<wb>_<timestamp>.jsonl
    │       └── out_loop/runs/<wb>/   same structure
    └── rq3_ablation_study/                Ours ablations, Table IV
        ├── without_fault_localization/<wb>/   logs/, output/ (same format as Ours)
        ├── without_multi_scenario_repairs/<wb>/
        ├── without_xltype/<wb>/
        └── without_loop_inference/<wb>/
```

Each results folder has its own `README.md` describing its file formats.

> The A1 and A2 baseline results are stored under `rq2_results_and_logs/` because RQ2 analyses their failures; they also provide the A1 and A2 columns of Table III (RQ1).

### Where to read each table number

| Table               | Approach      | Source                                                                                                                                                                                                      |
| ------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| III (W, in-loop)    | A1            | `results/rq2_results_and_logs/A1/InLoopResults/<wb>/final_logs.json` → `scenario_pass_count` / `scenario_total_count`                                                                                       |
| III (P/F, held-out) | A1            | `results/rq2_results_and_logs/A1/OutLoopResults/<wb>/final_logs.json`, same fields                                                                                                                          |
| III                 | A2            | `results/rq2_results_and_logs/A2/{in_loop,out_loop}/runs/<wb>/summary.json` → `test_sets_passed` / `test_sets_total`                                                                                        |
| III                 | Ours          | `results/rq1_results_and_logs/Ours/{in_loop,out_loop}/<wb>/output/*_diff_set_<n>.csv`: a scenario passes if every row with `scn_id = set_<n>` has `status = MATCH`                                          |
| II                  | A1, A2        | `analysis/Defect_Analysis_RQ2.xlsx`                                                                                                                                                                         |
| IV                  | Ours variants | `results/rq3_ablation_study/<variant>/<wb>/logs/summary.txt` → `Final status`, `total_tokens`, `Overall Repairs`, `[TIME]` lines; full-pipeline reference from `results/rq1_results_and_logs/Ours/in_loop/` |

---
