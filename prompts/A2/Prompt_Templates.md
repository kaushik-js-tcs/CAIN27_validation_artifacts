# Appendix B: Prompt Templates of Baseline A2 (Role-Specialized Agentic Workflow)

A2 is a role-based multi-agent pipeline. An orchestrator activates the agents in a fixed phase order:

`extraction → semantic → clustering → mapping → generation → test_runner → (repair) → governance`

Only the **semantic**, **clustering** and **mapping** agents and the **repair cycle engine** call the LLM. Extraction (workbook parsing, dependency graph) is deterministic. Generation renders the Java source deterministically from Jinja templates, using the descriptors and expressions the LLM produces. `{{…}}` marks a value filled in at runtime.

| ID            | Template                                          | Agent / stage          | LLM output                                                                         |
| ------------- | ------------------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------- |
| **Synthesis** |                                                   |                        |                                                                                    |
| S1            | Sheet role classification                         | Semantic               | Structured: `role`, `role_confidence_score`, `role_reasoning`, `has_role_conflict` |
| S2            | Formula intent classification                     | Semantic               | Structured: `formula_category`, `inferred_intent`, `confidence_score`, `reasoning` |
| S3            | Cluster responsibility check                      | Clustering             | Tool call `submit_responsibility_evaluation`                                       |
| S4            | Strategy and method descriptor                    | Mapping                | Structured, two phases: class + strategy, then method descriptor(s)                |
| S5            | Formula → Java expression                         | Mapping                | Tool call `translate_formula` / `submit_translations`                              |
| S6            | Method-body authoring                             | Mapping                | Tool call `author_method`                                                          |
| **Repair**    |                                                   |                        |                                                                                    |
| R0            | Reconsider decision (value mismatch)              | Repair engine          | Revised decision and reasoning                                                     |
| R1            | Reconsider strategy (compile or structural error) | Repair engine          | Revised strategy                                                                   |
| R2            | Reconsider formula category                       | Repair engine          | Revised category                                                                   |
| R3            | Find missing dependency edge                      | Repair engine          | Missing edge                                                                       |
| R4            | Re-translation with failure feedback              | Mapping (S5/S6 re-run) | Corrected expression or body                                                       |

---

## B.1 Shared blocks

S5 and S6 include these blocks, referred to below as `<<HELPERS>>`, `<<LOOKUP_HELPERS>>` and `<<FORBIDDEN>>`.

```text
HELPER FUNCTIONS (use exactly as shown; never call Excel function names):
  getD(map, "Sheet!Cell") -> double     getS(map, "Sheet!Cell") -> String
  roundTo(x, n) [ROUND]   pmt(rate, nper, pv) [PMT]
  edate_serial(serial, m) [EDATE]   edate_month(serial)   edate_year(serial)
  Math.abs / max / min / pow / sqrt / floor / ceil     (int)(x) [INT / TRUNC]

LOOKUP / SEARCH HELPERS (use for any VLOOKUP / XLOOKUP / INDEX-MATCH; never invent your own):
  _resultLookup(map, "Sheet", "keyCol", "retCol", key, minRow)          search a computed sheet
  _resultLookupStr(...)                                                  same, text key
  _resultLookupMulti(map, "Sheet", keyCols[], keyVals[], "retCol", ...)  multi-criteria
  _stLookup(TABLE_ID, key, colIndex) / _stLookupStr / _stLookupStrByStr  static lookup tables
  _stMatchCol(TABLE_ID, textKey)                                         column match

FORBIDDEN:
  Excel function names (SUM, VLOOKUP, INDEX, MATCH, SUMIF, ISERROR, TEXT, LEFT, ...)
  .isEmpty() / .isBlank() / .equals() / .contains() on doubles
  IFERROR as a call (translate as a ternary guard); bare Sheet!Cell strings
```

Cell-reference conventions:

| Context                    | Excel reference | Java                                           |
| -------------------------- | --------------- | ---------------------------------------------- |
| Loop row (current / prior) | `B5`, `B4`      | `cur_B{{var_suffix}}`, `prior_B{{var_suffix}}` |
| Fixed same-sheet cell      | `Sheet1!B5`     | `getD(result, "Sheet1!B5")` / `getS(...)`      |
| Inputs sheet               | `Inputs!C21`    | `getD(inputs, "Inputs!C21")`                   |
| Another computed sheet     | `Calc!D7`       | `getD({{result_map_var}}, "Calc!D7")`          |
| Blank numeric test         | `B4=""`         | `cur_B{{s}} == Double.MAX_VALUE`               |

---

## B.2 Synthesis templates

### S1: Sheet role classification (Semantic agent)

Used for sheets the deterministic classifier cannot label with confidence ≥ 0.7 (or for every sheet under the `llm_first` strategy).

```text
You are classifying an Excel sheet's role for Java code generation.
Sheet: '{{sheet_name}}' in workbook '{{workbook_id}}'
Formula count on sheet: {{formula_count}}
Sample formulas: {{up to 5 cell: formula lines}}
Respond with:
- role: one of [input, reference, calculation, output, staging, unknown]
- role_confidence_score: float 0.0-1.0
- role_reasoning: explain your classification (min 30 chars)
- has_role_conflict: true if the sheet mixes roles
```

### S2: Formula intent classification (Semantic agent)

```text
You are classifying an Excel formula for Java code generation.
Workbook: {{workbook_id}}   Sheet: {{sheet_name}} (role: {{sheet_role}})
Cell: {{cell_address}}   Formula: {{formula}}   Normalised (R1C1): {{r1c1_formula}}
Respond with:
- formula_category: one of [arithmetic, conditional, lookup, aggregation,
  date_time, text, error_handling, unknown]
- inferred_intent: plain-English description (min 20 chars)
- confidence_score: float 0.0-1.0
- reasoning: (min 30 chars)
Use error_handling only when the primary purpose is to catch errors (IFERROR, ISERROR, IFNA).
```

### S3: Cluster responsibility check (Clustering agent)

Formulas are grouped deterministically from the dependency graph (strongly connected subgraphs and formula groups). The LLM checks each cluster against the Single Responsibility Principle; a negative verdict splits the cluster.

```text
[system]
You are evaluating an actuarial spreadsheet formula cluster for the Single
Responsibility Principle. Cluster: {{cluster_ref}}. Total formulas: {{n}}
(sample of {{k}} shown). Sheets: {{sheet_names}}.
You may call get_cluster_formulas to inspect more, then call submit_responsibility_evaluation.
[user]
Sample formulas: {{sample}}
Does this cluster compute exactly one coherent business concept, or does it mix
unrelated calculations?
```

### S4: Strategy and method descriptor (Mapping agent, two-phase)

A deterministic rule pre-selects a strategy. Each strategy corresponds to a Jinja template: `direct_expression`, `sequential_table`, `conditional_branch`, `lookup_map`, `lookup_navigable_map`, `stream_aggregation`, `date_arithmetic`, `string_operation`, `error_wrapper`, `calculation_sheet`, `inputs_resolver`.

```text
Phase 1:
You are mapping Excel formulas to a Java component.
Workbook: {{workbook_id}}   Cluster: {{cluster_id}}   Cluster responsibility: {{srp_statement}}
Pre-selected strategy: {{strategy}} — {{strategy_description}}
Member formulas ({{count}}): {{formula_lines}}
Respond with: java_mapping_strategy (must be '{{strategy}}'), class_name,
division_present, policy_question_ids, confidence, reasoning.

Phase 2 (one per formula, or one batched call per formula group):
Strategy confirmed: {{strategy}}   Class: {{class_name}}
Produce ONE method descriptor for: {{cell}} [{{category}}]: {{formula}} — intent: {{intent}}
Fields: method_name, computation_description,
  return_type   (double | String | boolean | int | long | LocalDate | Optional<Double> |
                 List<Double> | Map<String,Object>),
  parameter_types, strategy-specific fields.
[On compile-failure retry, the compiler error, truncated to 600 chars, is added.]
```

### S5: Formula → Java expression (Mapping agent)

There are three variants: a **loop** variant for row-iteration tables, a **standalone** variant for fixed-row cells, and an **inputs** variant for parameter sheets. Formulas can be translated one at a time (`translate_formula`) or in batches (`submit_translations`).

```text
[system]
You are a Java code generator. Translate Excel formulas to Java expressions
{{for use inside a row-iteration loop | as standalone fixed-row expressions}}.
<<HELPERS>>  <<LOOKUP_HELPERS>>  <<FORBIDDEN>>
CELL REFERENCE RULES / BLANK SENTINEL rules (see B.1)
TRANSLATION EXAMPLES:
  =IF(B5>0,B5*0.1,0)  -> (cur_B<s> > 0 ? cur_B<s> * 0.1 : 0.0)
  =ROUND(B5*C5,2)     -> roundTo(cur_B<s> * cur_C<s>, 2)
  =IFERROR(B5/C5,0)   -> (cur_C<s> != 0.0 ? cur_B<s> / cur_C<s> : 0.0)
MIXED-TYPE (String return): IF(c,0,"N/A") -> (c ? String.valueOf(0.0) : "N/A")
Return ONLY the Java expression. Call read_cell_range for more context, then translate_formula.

[user]
Formula: {{formula}}   Return type: {{return_type}}
Sheet: {{sheet}}, loop rows {{prior_row}}..max, sample row {{row}}, var_suffix {{s}}
Result maps of other sheets: {{result_map_vars}}   Named ranges: {{named_ranges}}
Column types: {{col_types}}
Inputs constants (inline as literals): {{constants}}
Inputs runtime refs (use getD): {{runtime_refs}}
Static lookup tables: {{table_id, range, headers, preview rows}}
Row-relative inputs refs: {{getD(inputs, "Inputs!C" + (row + k))}}
[REPAIR FEEDBACK, see R4]
```

Tools: `read_cell_range(sheet, start_cell, end_cell)`, `parse_formula_tree(formula)`, `classify_range(...)`, `validate_expression(index, expression)` (batch mode), plus the submit tool. Each call allows up to 6 tool-use turns.

### S6: Method-body authoring (Mapping agent)

S6 is the fallback when a formula cannot be written as a single expression.

```text
[system]
You are a Java code generator. The formula below could NOT be expressed as a single
Java expression — author a full METHOD BODY (statements) instead.
You MAY use local variables and loops. The body MUST END with a `return` of the
declared type. No signature, modifiers or braces — statements only.
Read inputs with getD/getS(inputs, ...), earlier cells with getD/getS(result, ...).
<<HELPERS>>  <<LOOKUP_HELPERS>>  <<FORBIDDEN>>
[user]  same context as S5  ->  call author_method(java_body)
```

---

## B.3 Repair templates

After generation, the test runner compiles the Java code, runs it on every in-loop scenario, and compares each cell with Excel. Failing cells are traced back through the traceability graph (cell → Java component → cluster → formula). The repair engine assigns the failure to a decision level and re-prompts the agent that made that decision. Every repair prompt ends with `<<HISTORY>>`:

```text
The following approaches were already attempted and did NOT resolve the failure:
  - Cycle {{i}}: "{{reasoning}}" -> {{outcome}}
You MUST produce a genuinely different approach — do not repeat any of the above.
```

### R0: Reconsider decision (value mismatch)

```text
[system] You are part of an Excel-to-Java translation system. A previous automated
decision produced incorrect output. Reconsider your answer in light of the test failure.
[user] Original question: {{question}}   Original reasoning: {{reasoning}}
Original confidence: {{confidence}}   Rendered Java method: {{java_method}}
This method produced {{actual}} when {{expected}} was expected — reconsider {{field}}.
<<HISTORY>>
```

### R1: Reconsider strategy (compile or structural error)

```text
[system] ... A strategy selection produced a compile failure or structural mismatch.
Reconsider the strategy in light of the error.
[user] Formula: {{formula}}   Parse tree: outermost function={{fn}}, arg count={{n}}
Strategy selected: {{strategy}}   Tier-1 rule that fired: {{rule}}
Compile error | Runtime error: {{error}}
Strategy {{strategy}} produced this error — what strategy should this formula use?
<<HISTORY>>
```

### R2: Reconsider formula category

```text
[system] ... A formula classification may be incorrect. Reconsider whether the
assigned category is right.
[user] Formula: {{formula}}   Category assigned: {{category}}
Rendered Java method: {{java_method}}   Test inputs: {{inputs}}
Given these inputs, this method returned {{actual}} not {{expected}} — reconsider
whether category {{category}} is correct.
<<HISTORY>>
```

### R3: Find missing dependency edge

```text
[system] ... A formula dependency edge may be missing or incorrect. Identify what edge is wrong.
[user] Formula: {{formula}}   Cell references found by parser: {{refs}}
Dependency edges in graph: {{edges}}   Harness parameter sources: {{params}}
Missing output cell address: {{cell}}
Formula references {{refs}}, graph has these edges — what edge is missing?
<<HISTORY>>
```

### R4: Re-translation with failure feedback (S5/S6 re-run)

When the failure is attributed to a translated expression, S5 or S6 is run again with this section added:

```text
PREVIOUS ATTEMPT FAILED:
  Expression produced: {{prior_expression}}
  Failure: {{failure_feedback}}
  Prior failed attempts: - Cycle {{i}}: expr={{expr}} -> {{outcome}}
You MUST produce a different expression that fixes the above failure.
Do NOT repeat the previous expression.
```

For compile failures, the blocked Java component is superseded, its cluster is reset, and S4 and S5 run again with the compiler error hint.

---

## B.4 Configuration used in the experiments

| Parameter                     | Value                                                                                                       |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Model                         | GPT-5.6-sol (same Azure OpenAI deployment as A1 and Ours)                                                   |
| Execution mode                | `auto` (no human in the loop)                                                                               |
| Classification strategy       | `deterministic_first` (LLM only below the confidence threshold); `llm_first` for SurrenderCharge_FreeAmount |
| Confidence threshold          | 0.7                                                                                                         |
| Max repair cycles             | 7                                                                                                           |
| Repair prioritisation         | `proximity_first`                                                                                           |
| Tool-use turns per S5/S6 call | up to 6                                                                                                     |
| Ground-truth backend          | `xlwings` (Microsoft Excel recalculation)                                                                   |
