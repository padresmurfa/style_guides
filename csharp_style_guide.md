# C# Coding Style Guide

## Guiding Principles
- **Correctness first.** Always prioritize code that compiles cleanly, passes automated tests, and behaves deterministically across environments.
- **Conformity and consistency.** Follow the conventions in this guide, .NET naming standards, and established framework idioms before introducing alternatives.
- **Readability for humans and AIs.** Write code that communicates intent clearly through structure, naming, comments, and metadata so it can be understood by developers and AI-based tooling alike.
- **Security and safety.** Prefer secure defaults, validate untrusted input, and avoid patterns that encourage undefined behavior or resource leaks.
- **Evolution.** Adopt patterns that scale with project size, allow refactoring, and are resilient to future tooling (including LLMs) interpreting the code.

## Project Structure
- Organize solutions into logical projects (e.g., `*.sln` containing `*.csproj`). Use folders to group related namespaces and responsibilities.
- Keep each file focused: one public class or closely related set of types per file.
- Place tests in a parallel structure under a `.Tests` project mirroring the production namespaces.
- Include README or architectural notes alongside complex modules to assist onboarding and LLM context.

## Naming Conventions
- Follow [Microsoft .NET naming guidelines](https://learn.microsoft.com/dotnet/standard/design-guidelines/naming-guidelines).
- Use PascalCase for public types, methods, properties, and events.
- Use camelCase for local variables and private fields; prefix private static readonly fields with `s_` and private instance fields with `_`.
- Use clear, descriptive names; avoid abbreviations unless they are established acronyms (e.g., `Http`, `Db`).
- For async methods that return `Task`/`Task<T>`, append `Async`.
- Name boolean members as predicates (`IsReady`, `HasErrors`).

## Layout and Formatting
- Use UTF-8 encoding and Unix line endings.
- Indent with four spaces; never use tabs.
- Limit lines to 120 characters. Wrap method signatures and invocations at logical boundaries.
- Place opening braces on the same line for namespaces, types, members, and control blocks.
- Always include braces for conditional and loop statements, even when the body is a single line.
- Separate members with a single blank line; group related members with regions sparingly and only to aid navigation.
- Keep `using` directives ordered alphabetically and place system namespaces first.
- Use `var` when the type is obvious from the right-hand side; otherwise prefer explicit types.

## Comments and Documentation
- Use XML documentation comments (`///`) for public APIs; include `<summary>`, `<param>`, `<returns>`, and `<remarks>` when helpful.
- Document non-trivial private members when intent is not obvious from code.
- Avoid redundant comments—code should generally be self-explanatory. Favor improving names or structure over adding comments.
- Include references to specifications, tickets, or design docs where they assist future maintainers and AI tools in understanding constraints.
- Update comments when behavior changes; outdated comments are worse than none.

## Language Features
- Prefer newer C# language features when they improve clarity without sacrificing compatibility (e.g., pattern matching, `switch` expressions, `record` types for immutable data).
- Use nullable reference types (`#nullable enable`) and annotate APIs appropriately.
- Favor expression-bodied members for simple getters or straightforward methods; use block bodies when additional clarity is needed.
- Use `async`/`await` for asynchronous operations; avoid blocking calls (e.g., `.Result`, `.Wait()`).
- Leverage LINQ judiciously—ensure the resulting expression remains readable and efficient.
- Prefer `readonly struct` and `readonly` members when immutability is intended.

## Error Handling and Logging
- Use exceptions for exceptional conditions. Prefer specific exception types over `Exception`.
- Validate method arguments early and throw `ArgumentException`, `ArgumentNullException`, or `ArgumentOutOfRangeException` as appropriate.
- Ensure exceptions contain actionable messages and relevant data.
- Log using structured logging frameworks (e.g., `ILogger<T>`). Include context fields and avoid string concatenation—use message templates.
- Avoid catching general exceptions unless you rethrow or handle them meaningfully. Document rationales when broad catches are necessary.

## Testing and Correctness
- Write unit tests for all public behavior and critical internal logic. Follow AAA (Arrange-Act-Assert) structure.
- Use data-driven tests to cover edge cases and boundary values.
- Ensure tests are deterministic and isolated; avoid reliance on external state or timing.
- Integrate static analyzers (e.g., Roslyn analyzers, StyleCop) and treat warnings as errors when feasible.
- Use code coverage tools to verify critical paths are exercised; aim for meaningful coverage, not just percentages.

## Readability and Maintainability
- Adhere to SOLID principles and maintain single responsibility per class or method.
- Limit method length; break down large blocks into smaller, well-named methods.
- Ensure loops and LINQ queries remain readable; extract helper functions when logic becomes complex.
- Prefer explicit state over implicit side effects—avoid mutable global state and hidden dependencies.
- Use dependency injection to manage collaborators and simplify testing.
- Include summary comments or `README` sections near complex algorithms, documenting invariants and preconditions.

## AI / LLM Considerations
- Favor explicitness: make assumptions and invariants clear through comments, method names, and guard clauses so AI tools can reason accurately about code.
- Avoid ambiguous or overly clever code constructs; prioritize patterns that static analysis tools and LLMs commonly recognize.
- Provide small, self-contained functions and classes to make context windows manageable for AI code reviewers.
- Include examples in XML documentation comments when appropriate; LLMs can use them to generate consistent usage.
- When generating code with AI assistance, review and adapt it to align with this guide—avoid copy-pasting without understanding.
- Record provenance for AI-generated code snippets (e.g., in commit messages or documentation) if required by project policy.

## Version Control Practices
- Commit frequently with meaningful messages describing intent and scope.
- Keep branches focused; rebase or merge regularly to minimize drift.
- Require code review for all merges; reviewers should verify adherence to this guide.
- Enforce automated checks (build, tests, analyzers) in CI pipelines before merging.

## References
- [Microsoft .NET Documentation](https://learn.microsoft.com/dotnet/csharp/)
- [Framework Design Guidelines](https://learn.microsoft.com/dotnet/standard/design-guidelines/)
- [C# Coding Conventions (Microsoft)](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions)
- [StyleCop Analyzers](https://github.com/DotNetAnalyzers/StyleCopAnalyzers)
