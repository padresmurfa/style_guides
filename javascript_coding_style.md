# JavaScript Coding Style Guide

This style guide defines expectations for writing JavaScript that is correct, consistent, and readable. It is tailored for human developers and AI-assisted contributors alike. Each section explains the intent behind the guidelines so that automated tools and code reviewers can apply them consistently.

## 1. Core Principles

1. **Correctness first**: Never sacrifice accuracy for brevity. Prefer explicit, predictable code and defensive checks.
2. **Conformity**: Follow shared conventions to ensure code is easy to navigate and review across teams.
3. **Readability**: Write code that clearly communicates intent, including to future maintainers and automated systems.
4. **Trust but verify**: Treat AI-generated code as a draft. Run linters, type checkers, and tests to validate behavior.
5. **Document decisions**: When deviating from this guide for valid reasons, add comments or documentation explaining the rationale.

## 2. Language Level and Compatibility

- Target ECMAScript 2020 features unless project requirements dictate otherwise.
- Prefer modern syntax (e.g., `const`, `let`, arrow functions) while maintaining compatibility with your build tooling and supported environments.
- Avoid experimental features (`stage < 4`) unless explicitly approved and transpiled with tests.
- Ensure dependencies are evergreen and audited; pin versions for build tools to guarantee reproducible results.

## 3. Files, Modules, and Imports

- Use one module per file. Export only what is required by consumers.
- Default exports are acceptable for a single primary export; otherwise, use named exports.
- Order imports: standard libraries, third-party modules, internal modules, relative modules. Within each group, sort alphabetically.
- Avoid wildcard imports. Import only what you use to minimize bundle size and improve static analysis.
- Ensure side effects are explicit; do not rely on import execution order to set up state.

## 4. Naming Conventions

- Use `camelCase` for variables and functions, `PascalCase` for classes and React components, and `SCREAMING_SNAKE_CASE` for constants.
- Choose descriptive names over abbreviations unless well-established (e.g., `err`, `req`).
- Use verbs for functions (`fetchUser`, `calculateTotal`) and nouns for variables and classes (`user`, `Invoice`).
- Prefix asynchronous functions that return promises with contextual verbs to clarify intent (`loadUser`, `saveSettings`).

## 5. Formatting and Layout

- Follow a consistent formatter such as Prettier with 2-space indentation and 100-character line width.
- Place braces on the same line as control statements. Always include braces, even for single-line blocks.
- Terminate statements with semicolons to avoid automatic semicolon insertion pitfalls.
- Use trailing commas in multi-line literals and parameter lists to improve diff clarity.
- Separate logical sections of code with blank lines to emphasize structure. Avoid more than one blank line consecutively.

## 6. Comments and Documentation

- Write JSDoc (or TypeScript-style) comments for exported functions, classes, and modules, including parameter and return descriptions.
- Explain *why* something is implemented a certain way when it is not obvious. Avoid repeating what the code already states.
- Update comments when code changes. Stale documentation erodes trust.
- For AI-generated suggestions, annotate with `@generated` or similar tags only when the output is fully reviewed and tested.

## 7. Types and Static Analysis

- Use TypeScript or JSDoc types to describe public APIs, function signatures, and complex data structures.
- Prefer explicit type annotations for exported entities, class properties, and complex inference cases.
- Enable strict type-checking options (`noImplicitAny`, `strictNullChecks`) to catch correctness issues early.
- Run static analysis tools (ESLint, TypeScript compiler) as part of continuous integration.

## 8. Control Flow and Error Handling

- Prefer early returns to reduce nesting depth.
- Handle errors explicitly. Use `try/catch` for operations that may throw, and surface meaningful error messages.
- Reject promises with Error instances, not plain strings.
- Do not suppress errors unless there is a documented fallback; log with context if an error is intentionally ignored.
- Use `async/await` for asynchronous logic. Chain `then`/`catch` only when a sequential flow is simpler and still readable.

## 9. Data Structures and Immutability

- Use `const` by default. Switch to `let` only when reassignment is required. Avoid `var` entirely.
- Favor immutable updates for arrays and objects (`map`, `filter`, spread syntax) to prevent accidental shared state.
- Use `Map`, `Set`, and other built-in data structures when they better represent intent than plain objects.
- Avoid mutating input parameters; clone or copy as needed.

## 10. Functions and Classes

- Keep functions small and focused; limit to one responsibility.
- Use default parameters and object destructuring to clarify optional inputs.
- Avoid arrow functions for class methods when `this` binding is required.
- Prefer composition over inheritance. Use classes only when modeling shared behavior/state is necessary.
- Document invariants and state transitions in classes to help humans and AI understand lifecycle expectations.

## 11. Asynchronous Code and Concurrency

- Always handle rejected promises. Use `Promise.allSettled` when processing independent tasks.
- When running parallel tasks, limit concurrency with utility helpers to avoid overwhelming resources.
- Timeouts and retries must be explicit and configurable.
- For long-running processes, provide cancellation hooks (AbortController) or clean-up logic.

## 12. Testing and Verification

- Write unit tests for pure logic and integration tests for side effects. Ensure tests cover edge cases and failure modes.
- Test asynchronous code with real timers or properly mocked utilities.
- Prefer deterministic tests; avoid global state leaks and time-dependent behavior without proper control.
- Validate AI-assisted contributions against the test suite before merging.

## 13. Security and Reliability Considerations

- Escape or sanitize user-supplied input in DOM manipulations and queries.
- Use parameterized queries or query builders for database access.
- Treat data from AI tools as untrusted; validate before executing or storing.
- Never commit secrets or access tokens. Use environment variables or secure vaults.
- Implement logging with context while avoiding sensitive data exposure.

## 14. Performance and Optimization

- Optimize only after measuring; use profiling tools and performance budgets.
- Prefer lazy evaluation for expensive computations that may not be needed.
- Use memoization judiciously, ensuring cached data remains valid.
- Document performance trade-offs explicitly to help future maintainers and AI reviewers.

## 15. Collaboration and Code Review

- Keep pull requests focused and small to simplify review.
- Include code examples in documentation and pull requests that follow this style guide.
- Encourage reviewers—human or AI—to comment on deviations from the guide and suggest corrections.
- Resolve review feedback through follow-up commits or documentation updates.

## 16. Accessibility and Internationalization

- For front-end code, use semantic HTML and ARIA attributes where appropriate.
- Ensure keyboard accessibility for interactive components.
- Externalize user-facing strings for localization; avoid hard-coded text.
- Support right-to-left layouts if required by product scope.

## 17. Tooling Recommendations

- Configure ESLint with a ruleset aligned to this guide (`eslint:recommended`, `plugin:import/errors`, `plugin:react/recommended` if applicable).
- Use Prettier for formatting, integrated with ESLint via `eslint-plugin-prettier` or `prettier-eslint`.
- Set up Husky or similar git hooks to run linting and tests on commit.
- Use Dependabot or Renovate to monitor dependency updates and security advisories.

## 18. AI and LLM-Specific Guidance

- Prompt AI systems with explicit references to this guide to ensure consistent outputs.
- Review AI-generated code line-by-line. Confirm that variable naming, formatting, and control flow adhere to this guide.
- Prefer deterministic prompts and temperature settings when possible to maintain uniform code style.
- When integrating AI-generated snippets, document the prompt context in comments or pull request descriptions for traceability.
- Continuously retrain or adjust AI tools based on review feedback to minimize repetitive corrections.

## 19. Example Template

```javascript
import { fetchUserProfile } from "../services/user-service";

/**
 * Retrieves the current user profile, ensuring defaults are applied.
 * @param {string} userId - The unique identifier for the user.
 * @returns {Promise<UserProfile>} A promise that resolves with the profile.
 */
export async function loadUserProfile(userId) {
  if (typeof userId !== "string" || userId.trim() === "") {
    throw new TypeError("userId must be a non-empty string");
  }

  const profile = await fetchUserProfile(userId);

  return {
    ...profile,
    locale: profile.locale ?? "en-US",
    theme: profile.theme ?? "light",
  };
}
```

This example demonstrates the conventions described above, including type checks, structured imports, documentation, and immutable updates.

## 20. Continuous Improvement

- Revisit this style guide periodically to incorporate community best practices and project-specific insights.
- Encourage feedback from all contributors, tracking proposed revisions in version control.
- Update examples and tooling recommendations as the ecosystem evolves.

By following this guide, teams—human and AI alike—can deliver JavaScript that is dependable, consistent, and easy to maintain.
