# AI Usage Statement

This document explicitly defines the boundaries of how Artificial Intelligence was utilized during the development and documentation of the OPFF (Open Pixmap File Format) project. 

## Architectural & Engineering Control

The human author maintained absolute, deterministic control over the project's engineering, logic, and philosophy at all times. The AI was strictly guided by the author and acted solely as a text-formatting assistant. 

**Every architectural and engineering decision is strictly human-authored:**

* The 64-byte frozen core mandate.

* The O(1) boundary calculations and parser logic.

* The "Implementation Friction" heuristic.

* The extension model and namespace rules (`OPFX`, `OPF*`).

* The hardware alignment constraints and atomic rejection matrix.

## What AI Was Used For

AI was utilized exclusively in the documentation phase to assist with technical writing:

* **Textual Refinement:** Converting the author's raw architectural concepts, design rules, and origin story into clean, professional prose.

* **Syntax and Grammar:** Ensuring correct grammar, spelling, and sentence structure across the documentation files.

* **Markdown Formatting:** Structuring the `README.md`, `ORIGIN.md`, and `PHILOSOPHY.md` files with proper headers, lists, and layout for maximum readability.

## What AI Did NOT Do

* **Zero Code Generation:** The AI did not write, structure, or dictate any of the source code. The Rust implementation, zero-copy parser logic, memory mapping, and validation checks are 100% human-authored.

* **Zero System Design:** The AI did not invent the format, its physical boundaries, its limitations, or its applications. It merely transcribed and formatted the strict architectural laws provided directly by the human author.