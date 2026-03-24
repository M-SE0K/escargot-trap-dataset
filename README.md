# Escargot Trap Dataset

Escargot Trap Dataset is a benchmark designed to evaluate the vulnerability detection capabilities of LLM-based code review bots (e.g., `escargot-review-bot`).

## Overview

This dataset contains synthetic defects and vulnerabilities intentionally injected into the C++ source code of **Escargot**, a lightweight JavaScript engine. 

The injected vulnerabilities are mapped to the **CWE (Common Weakness Enumeration)**. It tests whether an LLM can move beyond simple linter-level syntax checks to identify deep contextual flaws and C++ undefined behaviors (UB).

## Injected Defects (CWE-mapped)

The patch files (`v4.patch` ~ `v8.patch`) include the following C++ specific vulnerabilities:

*   **CWE-401 Memory Leak**: Missing deallocation (`free`, `delete`) for dynamic memory, particularly in exception paths.
*   **CWE-119 / CWE-124 Buffer Overflow/Underflow**: Out-of-bounds memory access due to miscalculated array sizes (e.g., `.resize(len-1)`) or off-by-one errors.
*   **CWE-416 Use After Free / Dangling Pointer**: Returning or reusing a reference/pointer to an object that has been destroyed or gone out of scope.
*   **CWE-457 Uninitialized Variable**: Reading garbage memory due to missing variable or pointer initialization (e.g., missing `nullptr` assignment).
*   **CWE-252 Unchecked Return Value**: Missing null-checks after C-style memory allocations (e.g., `malloc`), leading to potential crashes.
*   **Project Convention Violations**: Violations of the Escargot style guide (e.g., mandatory explicit expressions like `if (ptr != nullptr)`).

## Ground Truth

Each patch comes with a ground-truth JSON file for automated evaluation. The `answers/` directory contains precise file paths, line numbers, and reference review comments that the model is expected to output.

## Evaluation Metrics

This dataset is integrated with an evaluation dashboard to track the following metrics:

*   **True Positive Rate (Recall)**: The accuracy of detecting intentionally injected vulnerabilities.
*   **False Positive Rate (Hallucination)**: The frequency of the model flagging safe, optimized code as vulnerable.
*   **Review Quality**: The appropriate use of C++ and static analysis terminology (e.g., RAII, NRVO, Branch Prediction) in the suggested solutions.

## Directory Structure

To maximize efficiency and ease of integration, the dataset provides Unified Diff (`.patch`) files instead of copying the entire engine source code.

```text
escargot-trap-dataset/
├── README.md
├── LICENSE
├── patches/
│   ├── v4.patch   # Defect injection patches for version 4
│   ├── v5.patch   # Defect injection patches for version 5
│   └── ... (v6 ~ v8)
└── answers/
    ├── v4_expected.json   # Reference ground truth for v4
    ├── v5_expected.json
    └── ... (v6 ~ v8)
```
