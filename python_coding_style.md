# Python Coding Style Guide

## Purpose and Scope
This guide defines a Python coding standard that prioritizes correctness, conformity, and readability for both human contributors and AI-assisted tools. It complements PEP 8, PEP 257, and other community best practices while clarifying expectations that help reviewers, linters, and large language model (LLM) code generators produce consistent, high-quality code.

## Core Principles
1. **Correctness First**
   - Prefer explicit, deterministic behavior over clever shortcuts.
   - Validate assumptions with assertions, type hints, and tests.
   - Handle edge cases defensively, especially around external inputs and concurrent code.
2. **Conformity to Standards**
   - Align with PEP 8 for formatting, PEP 257 for docstrings, and PEP 484 for typing unless this guide specifies otherwise.
   - Use automated tooling (formatters, linters, type checkers, test suites) to enforce consistency.
   - Favor widely adopted libraries and language features to reduce maintenance risk.
3. **Readable for Humans and AIs**
   - Write code, comments, and commit messages that communicate intent clearly.
   - Use descriptive names, structured documentation, and regular patterns so tooling and AI suggestions can align with project norms.

## Project Structure and Modules
- Organize packages logically with clear `__init__.py` files, minimizing implicit side effects at import time.
- Keep module responsibilities narrow; split large modules when public APIs exceed ~400 lines or serve multiple domains.
- Prefer explicit relative imports within packages (e.g., `from .submodule import ClassName`).

## Naming Conventions
- Modules and packages: `lowercase_with_underscores`.
- Classes and exceptions: `CapWords`.
- Functions, methods, and variables: `lowercase_with_underscores`.
- Constants: `UPPERCASE_WITH_UNDERSCORES` defined at module scope.
- Private helpers: prefix with a single underscore.

## Formatting Rules
- Use spaces over tabs; indent with four spaces.
- Limit lines to 88 characters to align with Black while allowing docstrings up to 100 characters when clarity improves.
- Break long expressions using parentheses instead of backslashes.
- Place one blank line between logical sections inside functions.
- Keep imports grouped by standard library, third-party, and local packages with blank lines separating each group.

## Typing and Interfaces
- Annotate all public functions, methods, and class attributes with precise type hints.
- Use `typing` and `collections.abc` abstractions (`Sequence`, `Iterable`, `Mapping`) instead of concrete container types when the contract allows.
- Prefer `Protocol` or `TypedDict` to document structural expectations and data shapes.
- Avoid `Any` unless it is truly necessary; document why when used.

## Functions and Methods
- Keep functions short and focused; when a function exceeds ~25 logical lines or handles multiple concerns, refactor into helper functions or classes.
- Use keyword-only arguments for parameters beyond basic inputs to improve clarity and support LLM code generation.
- Document return values explicitly, even when returning `None`.

## Classes and Data Models
- Favor `@dataclass` for simple data containers; ensure default values are immutable or created via `field(default_factory=...)`.
- When overriding methods, call `super()` unless there is a documented reason not to.
- Keep inheritance hierarchies shallow; prefer composition over inheritance to minimize complexity.

## Error Handling
- Raise specific exceptions that describe the failure and include actionable messages.
- Catch only exceptions you can handle; re-raise with context using `raise ... from err`.
- Provide fallback or recovery strategies when interacting with external systems (files, networks, subprocesses).

## Documentation and Comments
- Use docstrings on all public modules, classes, and functions following PEP 257 conventions.
- The first line of a docstring should be a short summary; include argument, return, and exception sections using Google or reStructuredText style consistently across the project.
- Prefer comments that explain *why* rather than *what*. If using an AI tool, ensure generated comments reflect actual behavior and edit for accuracy.

## Testing and Verification
- Write unit tests alongside new features; target high-importance branches with parametrized tests to increase coverage.
- Use property-based tests (`hypothesis`) for critical logic and boundary conditions.
- Add regression tests for every bug fix before changing implementation code.
- Automate test execution in CI and run locally before merging.

## Concurrency and Asynchronous Code
- For async code, use `asyncio` conventions: avoid mixing blocking calls with async functions without `await`-compatible wrappers.
- Use high-level concurrency primitives (`asyncio.TaskGroup`, `concurrent.futures.ThreadPoolExecutor`) instead of manual thread management when possible.
- Guard shared mutable state with locks or immutability and document concurrency expectations.

## Logging and Instrumentation
- Use the standard `logging` module; configure loggers at module level using `logger = logging.getLogger(__name__)`.
- Log actionable information at INFO and WARN levels; reserve DEBUG for noisy diagnostics.
- Ensure sensitive data is never logged. Document any redaction strategies.

## Dependencies and Environment
- Pin direct dependencies and specify Python version support in `pyproject.toml` or `requirements.txt`.
- Prefer standard library modules when functionality overlaps with third-party packages.
- Document any required environment variables or services in README or deployment guides.

## AI-Generated Code Guidance
- Treat AI-generated snippets as suggestions: review for correctness, security, and licensing before inclusion.
- Run static analysis and tests after integrating AI-produced changes.
- Ensure prompts and commit messages anonymize sensitive project details.
- Update docstrings and comments to reflect final human-reviewed behavior, not intermediate AI drafts.

## Code Review Expectations
- Submit pull requests with focused scope, clear summaries, and references to tests executed.
- Respond to review feedback promptly; prefer follow-up commits over force pushes unless instructed.
- Reviewers should verify adherence to this guide, request clarifications for ambiguous sections, and add automated checks when manual review reveals repeated issues.

## Tooling and Automation
- Adopt formatters (Black), import sorters (isort), and linters (ruff, flake8) with configurations checked into version control.
- Use pre-commit hooks to run formatting, linting, and type checking before pushes.
- Configure CI pipelines to block merges when style or quality checks fail, ensuring long-term consistency.

## Continuous Improvement
- Revisit this style guide quarterly or after significant language/tooling updates.
- Capture deviations explicitly in project documentation when exceptions are required.
- Encourage knowledge sharing: document patterns, anti-patterns, and learnings from production incidents.

