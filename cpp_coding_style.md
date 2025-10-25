# C++ Coding Style Guide

This style guide sets expectations for human developers and AI coding assistants working in this repository. It complements project-specific documentation (such as `cpp_source_code.md`) and focuses on correctness, conformity to modern C++ best practices, and readability. Whenever conflicts arise between this guide and project requirements, defer to project requirements while documenting the deviation.

## 1. Core Principles

1. **Correctness First**  
   - Favor clarity over micro-optimizations unless performance requirements are documented.  
   - Treat compiler warnings as errors; enable the highest warning level available on the toolchain (`-Wall -Wextra -Wpedantic` or equivalent).  
   - Prefer static analysis tools (e.g., `clang-tidy`, `cppcheck`) and address findings promptly.  
   - Write deterministic code: avoid undefined behavior, data races, and reliance on unspecified order of evaluation.
2. **Conformity and Consistency**  
   - Target the project-specified C++ language version; when unspecified, default to C++20.  
   - Follow the Standard Library and widely adopted guidelines (ISO C++ Core Guidelines, LLVM, Google) when they improve safety and maintainability.  
   - Keep style consistent within each module; when introducing new patterns, document them clearly.
3. **Readable for Humans and AIs**  
   - Structure code and comments so that both human reviewers and AI tools can infer intent.  
   - Provide explicit rationale for non-obvious logic, invariants, and concurrency contracts.  
   - Prefer declarative names that reflect domain concepts, enabling better semantic understanding by tools.

## 2. Language and Library Usage

- **Smart Pointers and RAII**: Own resources via RAII types (`std::unique_ptr`, `std::shared_ptr`, custom guards). Avoid naked `new`/`delete`.  
- **Const-Correctness**: Use `const` and `constexpr` aggressively to express invariants; prefer `[[nodiscard]]` for functions where ignoring the result is a bug.  
- **Initialization**: Use uniform initialization (`{}`) unless copy-initialization is required. Prefer constructor member initializer lists.  
- **Error Handling**: Use exceptions for recoverable errors when exceptions are enabled; otherwise, use `expected`-style return values. Never swallow errors—log or propagate them.  
- **Concurrency**: Prefer higher-level abstractions (`std::jthread`, executors) over raw threads. Document synchronization strategies explicitly.  
- **Templates and Concepts**: Use templates judiciously. Constrain templates with `concepts` or `static_assert` to prevent hard-to-diagnose substitution errors.

## 3. Naming and Structure

- **Files**: Use lower snake case for file names (`unicode_utils.cpp`). Place declarations in headers (`.hpp`/`.h`) and definitions in sources (`.cpp`).  
- **Namespaces**: Mirror the directory hierarchy. Use anonymous namespaces for internal linkage. Avoid `using namespace` in headers.  
- **Types**: Use `PascalCase` for class/struct/enum names.  
- **Functions and Variables**: Use `lower_snake_case`. Member variables may include a short suffix (`_` or `_id`) if the project already uses it.  
- **Constants**: Use `UPPER_SNAKE_CASE`.  
- **Enumerations**: Prefer `enum class` for type safety.  
- **Layouts**: Place related declarations together (type definitions, then constructors, then public methods, followed by protected and private sections).

## 4. Formatting and Layout

- **Indentation**: Four spaces per level; never use tabs.  
- **Line Length**: 100 characters maximum; break lines thoughtfully using hanging indents for parameter lists and chained calls.  
- **Braces**: Adopt the Allman style for function definitions and control blocks. Single-statement bodies may omit braces only when the statement fits on one line and the intent is obvious—otherwise, include braces.  
- **Spacing**: Use one space around binary operators and after commas. Avoid trailing whitespace.  
- **Includes**: Use include guards or `#pragma once`. Order headers as follows: related header, standard library, third-party, project headers. Keep each group alphabetized.

## 5. Documentation and Comments

- **Doxygen-Friendly**: Use triple-slash `///` comments for functions, classes, and public members. Include `\brief`, `\param`, `\return`, and `\throws` as appropriate.  
- **Rationale Comments**: Document algorithmic decisions, invariants, and thread-safety guarantees. Mention preconditions and postconditions explicitly.  
- **TODOs**: Prefix outstanding work with `TODO(name):` and reference tracking issues where possible.  
- **Examples for AI**: Provide small self-contained code examples in comments or markdown docs when a pattern is non-trivial. This assists automated tools in applying the style.

## 6. Testing and Verification

- **Unit Tests**: Every new feature or bug fix requires unit tests using the project-approved framework. Tests must cover success and failure paths.  
- **Regression Tests**: Add regression tests for fixed bugs with descriptive names referencing the issue ID.  
- **Fuzzing and Property Testing**: When dealing with parsing, serialization, or external inputs, add fuzz or property-based tests.  
- **Continuous Integration**: Keep builds green. Ensure that changes compile across supported platforms/compilers and that static analyzers remain clean.

## 7. Interaction with AI and LLM Tools

- **Prompt Context**: When using AI assistance, provide relevant headers, interfaces, and test cases to prevent hallucinated APIs.  
- **Verification**: Validate AI-generated code through compilation, tests, and manual review. Do not accept changes without understanding them.  
- **Diff Hygiene**: Ensure AI-assisted edits preserve surrounding context, include necessary headers, and maintain formatting.  
- **Metadata**: When committing AI-generated code, record manual verification steps in commit messages or review comments.

## 8. Code Review Checklist

- Are interfaces minimal, stable, and documented?  
- Does the implementation respect RAII, const-correctness, and move semantics?  
- Are preconditions validated and errors propagated?  
- Are unit tests comprehensive and readable?  
- Is the code free of data races and UB under the chosen memory model?  
- Is the style consistent with this guide and module conventions?  
- Are comments and documentation accurate, up to date, and helpful for both humans and AI tools?

## 9. Glossary

- **RAII**: Resource Acquisition Is Initialization; tie resource lifetime to object lifetime.  
- **UB**: Undefined Behavior; any operation for which the standard imposes no requirements.  
- **LLM**: Large Language Model; AI system capable of generating or assisting with code.  
- **Allman Style**: Bracing style where the opening brace is on its own line aligned with the controlling statement.

Adhering to this guide ensures that C++ code remains correct, maintainable, and accessible to both human collaborators and AI-assisted tooling.
