# Appendix A: Prompt Templates

**T1** generates a utility library of Excel functions in Java. **T2** turns each sheet's pseudocode into Java. **T3** and **T4** repair the generated code: T3 fixes wrong values found by comparing with the workbook, and T4 fixes compiler and runtime errors.

Each prompt is sent as a single user message. Every template asks for exactly one fenced code block in reply, and the tool extracts that block. `{{…}}` marks a value filled in at runtime.

| ID  | Template                 | Stage                 | Output                                 |
| --- | ------------------------ | --------------------- | -------------------------------------- |
| T1  | Excel function synthesis | Preprocessing         | Java class `Test` with one method      |
| T2  | Sheet code synthesis     | Code generation       | Java class for one sheet or loop block |
| T3  | Loop repair              | Value-mismatch repair | Corrected Java class                   |
| T4  | Compile/runtime repair   | Error repair          | Corrected Java class                   |

---

## A.1 Shared rules: JavaXl type model

T2, T3 and T4 all include the following block, referred to as `<<JAVAXL_RULES>>` in the templates below.

```text
## JavaXl Types
- Only record types in {{xltype_pkg}} may be used: {{available_xltypes}}.
- Scalar records expose only value(); XlArray exposes only values() → {{xl_model_class}}[][].
- No other helpers or APIs exist.

## Excel → JavaXl Type Mapping
{{type_mapping}}

## Semantics
- No implicit type coercion; follow Excel semantics only.
- Numbers keep at most 15 significant digits (Excel precision).
- Arrays keep their shape and are never coerced to scalars.
- Dates are numeric serials (XlDateSerial or XlNumber, treated identically); no Java date APIs.

## Excel Functions
- Use the helper class {{excel_functions_file}}; do not redefine its methods:
{{functions_list}}
```

---

## A.2 T1: Excel function synthesis

T1 generates a Java implementation of one Excel function. The tool then compiles it and tests it against values calculated by Excel. If it fails, the tool repairs it for up to 7 rounds. All functions that pass are merged into the `ExcelUtils` class.

````text
You are a senior Java engineer specializing in Excel-compatible calculation engines.
Implement the following Excel function in Java using the JavaXl type model.

Function: {{func_name}}
Description: {{description}}
Syntax: {{syntax}}

Examples (illustrative only; they must not override Excel semantics):
{{test_examples}}

## Contract
- One public class `Test` with exactly one public method `{{func_name}}`.
- Signature matches the Excel syntax; all parameters and the return type are {{xl_model_class}}.
- Variable-argument functions take a single {{xl_model_class}} parameter; Java varargs are forbidden.
- At most one private helper method; no fields, inner classes or external libraries.
- Javadoc only on {{func_name}}, with @param and @return.

## Types
- Use only {{available_xltypes}} from {{xltype_pkg}}; no implicit coercion.
{{type_mapping}}

## Excel Semantics
- Always return a valid, non-null {{xl_model_class}}; never throw exceptions (catch and log them).
- Array-returning functions always return XlArray; empty arrays are valid results.
- Optional parameters are always in the signature; a null argument means
  "omitted → apply Excel default". No overloading, and no branching on argument count.
- Numeric equality must follow Excel semantics, not Java `==`; no BigDecimal.
- Boolean TRUE/FALSE branches must be explicit and match Excel short-circuit behaviour.
- Dates are serial numbers. Preserve the 1900 leap-year bug (serial 60 = 1900-02-29).
  Use serial arithmetic only; no Java date/time APIs.

## Output
Return only the Java source in a single ``` block:
package {{xlfuncs_pkg}};
import {{xltype_pkg}}.*;
public class Test { /* {{func_name}} */ }
````

---

## A.3 T2: Sheet code synthesis

T2 translates the pseudocode generated for one sheet, or for one loop block within a sheet, into Java.

````text
You are a senior Java engineer specializing in Excel-compatible calculation engines.
Translate the pseudocode below into a complete Java program using the JavaXl type model.

## Rules
- Public class name: {{class_name}}.
- Keep all identifiers exactly as in the pseudocode.
- Keep bounded loops as `for` loops; never use while(true), do-while or break-driven loops.
- Preserve the order of statements inside each iteration: it encodes formula
  dependencies (rows may depend on the previous row).
- Represent sheets as HashMap<String, HashMap<String, {{xl_model_class}}>>.
- Handle missing or null values safely.

## Reference Classes
- Other sheet classes exist in the same package; instantiate them and call
  initialize_map(). Do not re-implement them.

<<JAVAXL_RULES>>

## Pseudocode
{{pseudocode}}

## Output
Return only the full Java source in a single ``` block.
````

---

## A.4 T3: Loop repair

T3 repairs a generated loop in which each iteration corresponds to one Excel row. The model is told to fix the column where the error starts, so that the columns depending on it become correct as well.

````text
You are an expert Java developer. Repair the loop below so that its output matches the
Excel formulas. Each loop iteration corresponds to one Excel row.

## Repair Rules
- Root cause first: most mismatches are consequences of an earlier column. Fix only the
  method that computes that column; do not patch downstream columns.
- Formula-local: repair according to the Excel formula of the mismatched cell, not the
  wrong Java value.
- First-row seed: if the first data-row formula differs from the second, add an explicit
  first-row branch, e.g. if (row_index.equals("<first_row>")) { ... }.
- Blanks: XlText("") is an Excel blank and is not 0. A previous-row reference in the first
  row must evaluate exactly as Excel does (0, "" or error).

<<JAVAXL_RULES>>

## Detected Mismatches (Excel rows + first mismatch per column)
{{issues}}

## Current Source Code
{{source_code}}

## Constraints
- Do not change imports, class or package names, method signatures, or the loop structure.
- Do not hard-code expected values, except literal or seed formulas.

## Output
Return the full corrected Java source in a single ``` block, with no explanation.
````

---

## A.5 T4: Compile/runtime repair

T4 applies the smallest change that removes compiler or runtime errors. The tool runs it at most 7 times and stops early if the same errors repeat.

````text
You are a senior Java engineer. Fix the compiler/runtime errors below with minimal changes.

## Rules
- Do not rename, add or remove classes, methods, fields, variables or imports.
- Do not modify the reference sheet classes or {{excel_functions_file}}.
- Keep the logic and data-dependency order; reorder statements only to fix dependency order.
- Fix only type mismatches, null handling and data-access errors; comment each fix inline.

<<JAVAXL_RULES>>

## Source Code
{{source_code}}

## Errors (up to 15)
{{errors}}

## Output
Return only the full corrected Java source in a single ``` block.
````

---

## A.6 Placeholders

| Placeholder                                      | Meaning                                                                        |
| ------------------------------------------------ | ------------------------------------------------------------------------------ |
| `{{func_name}}`, `{{description}}`, `{{syntax}}` | Excel function metadata                                                        |
| `{{test_examples}}`                              | Up to 2 input/output examples calculated by Excel, truncated to 150 characters |
| `{{xl_model_class}}`                             | `XlType` when the type model is on, otherwise `Object`                         |
| `{{available_xltypes}}`, `{{type_mapping}}`      | Available JavaXl record types and the Excel → JavaXl type mapping              |
| `{{excel_functions_file}}`, `{{functions_list}}` | `ExcelUtils` and the signature, Javadoc and an example for each of its methods |
| `{{pseudocode}}`                                 | Pseudocode for one sheet or loop block                                         |
| `{{issues}}`                                     | Snippet of the Excel loop table plus the first mismatch in each column         |
| `{{source_code}}`, `{{errors}}`                  | Current Java source; parsed `javac` errors or stack traces                     |

## A.7 Reproducibility

- **Model:** [GPT-5.6-sol], accessed through the Azure OpenAI Chat Completions API.
- **Limits:** 7 repair rounds for each function and each code block.
