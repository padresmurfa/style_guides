# CSS Unit Testing Style Guide

This document adapts the structured conventions from the **C# Unit Testing Style Guide** to the CSS ecosystem. It defines **best practices** for writing **clear, deterministic, and maintainable** unit tests for authored or generated CSS. The guidance assumes a JavaScript-based test runner (Jest, Vitest, or Mocha) executing against CSS Modules, design tokens, utility frameworks, or compiled outputs from preprocessors (Sass, Less, PostCSS, Tailwind, etc.).

---

## **Target Audience & Scope**
- Front-end engineers, design system teams, and **LLMs acting as coding assistants** who need to verify CSS correctness.
- Tests focus on **small CSS units**: individual selectors, utility classes, design tokens, mixins/functions, or post-processed bundles.
- Functional or visual regression testing (e.g., Playwright image comparisons) is out-of-scope—keep those in a separate suite.
- Every recommendation below favors readability and reviewer confidence over clever one-liners. Prefer **explicit structure** even when tests seem trivial.

---

## **1. Tooling & Environment**
- Use a Node-based runner with **JSDOM** so selectors and custom properties can be evaluated without a real browser.
- Normalize CSS via **PostCSS** (with the same plugins used in production) before asserting on output. This prevents disagreements caused by missing vendor prefixes or minification.
- Keep **stylelint** rules aligned between production code and tests; lint fixtures as part of the test pipeline.
- When asserting computed styles, rely on helper utilities that call `getComputedStyle` inside JSDOM. For generated CSS files, parse with `css-tree`, `postcss`, or `@parcel/css` rather than string contains checks.

---

## **2. Test File Organization**
Consistent structure makes it obvious which CSS artifact is being validated.
- Mirror the production folder hierarchy. If `src/components/Button/Button.module.css` exists, place tests in `src/components/Button/__tests__/ButtonStyles.test.ts` (or `.js`).
- **One unit under test per file.** Separate tests for global styles, utility classes, and component-scoped styles.
- For design tokens, colocate tests with the token source (e.g., `tokens/__tests__/colorTokens.test.ts`).
- Export reusable fixture builders (HTML snippets, compiled CSS strings) from `test-utils/cssFixtures.ts` to avoid duplication.

---

## **3. Describe & Namespace Structure**
- Wrap scenarios in `describe()` blocks named after the selector, token group, or mixin under test (`describe('.btn-primary', ...)`).
- Nest `describe()` blocks by **state modifiers** (hover, focus-visible, responsive breakpoints) to clarify intent.
- Avoid anonymous test suites. Make the top-level `describe()` align with the production filename or namespace.

---

## **4. Test Naming Conventions**
Borrow the clarity of the C# naming approach and adapt it to JavaScript test syntax.
- Every Jest/Vitest `test`/`it` description must read like a sentence describing behavior: `it('applies focus ring with accessible contrast', ...)`.
- If manually named functions are required (e.g., tape, node:test), start with `test_` and use snake case: `function test_button_applies_disabled_opacity() { ... }`.
- Include the **state** or **media query** you’re validating: `it('applies padding at md breakpoint', ...)`.
- Prefer explicit verbs (`applies`, `removes`, `overrides`) over vague statements (`works`, `renders`).

---

## **5. Sectioned Test Layout**
Replicate the disciplined sections from the C# style guide using comment headers. Apply them even in short tests.

```js
it('applies primary button background', () => {
  // GIVEN: a button rendered with the primary class
  const givenMarkup = '<button class="btn btn--primary">Submit</button>';
  const givenDocument = createDocumentFromMarkup(givenMarkup);

  // SETUP: attach compiled CSS to the document
  const envStyleElement = injectStyles(givenDocument, buttonStyles);

  // SYSTEM UNDER TEST: locate the element under scrutiny
  const sutButton = givenDocument.querySelector('.btn');

  // WHEN: compute styles after injection
  const actualStyles = getComputedStyle(sutButton);

  // EXPECTATIONS: define the values we want to see
  const expectedBackground = 'rgb(0, 95, 204)';

  // THEN: compare actual vs expected
  expect(actualStyles.backgroundColor).toBe(expectedBackground);
});
```

### Section Rules
- **GIVEN** — declare fixtures (`const givenMarkup = ...`). Prefix inputs with `given`. Keep HTML snippets or token names here.
- **MOCKING** — stub browser APIs or feature queries (`window.matchMedia`). Prefix mocks with `mock`. Only include when necessary.
- **SETUP** — attach style sheets, register CSS custom properties, initialize PostCSS processors. Prefix environment helpers with `env`.
- **SYSTEM UNDER TEST** — assign the DOM node, class name, or compiled stylesheet under examination to `sut`.
- **WHEN** — execute the behavior: trigger pseudo-class simulation, run the PostCSS pipeline, or evaluate a mixin.
- **EXPECTATIONS** — store expected tokens (`const expectedColor = 'var(--color-primary)'`).
- **THEN** — perform assertions against `actual*` and `expected*` values.
- **BEHAVIOR** — verify side-effects, such as ensuring a style injection helper was called once. Only present when mocks are used.
- Merge sections only when empty; otherwise, keep the explicit header comments.

---

## **6. Variable Naming**
- Inputs: `given*` (`givenMarkup`, `givenTokenName`).
- Environment/setup artifacts: `env*` (`envStyleElement`, `envMatchMedia`).
- Mocks and spies: `mock*` (`mockGetBoundingClientRect`).
- System under test: `sut*` (usually `sutElement`, `sutStylesheet`).
- Results: `actual*` (`actualBackgroundColor`).
- Expected values: `expected*`.
- Never assert directly against literals—store them in `expected*` variables first.

---

## **7. Working with Fixtures & Compiled Output**
- Store common HTML snippets in tagged template helpers for readability (`buttonFixture()` returns a string or DOM node).
- Use **data attributes** in fixtures to locate elements instead of depending on brittle descendant selectors.
- When testing compiled bundles, parse the CSS AST and assert on rules/selectors instead of raw string indices.
- Keep fixtures representative: include state classes (`is-disabled`), data attributes, and ARIA hooks that influence styles.

---

## **8. Pseudo-classes, Media Queries, and Feature Queries**
- Prefer helper utilities that emulate state: `applyPseudoClass(sutElement, 'hover')` rather than manually toggling classnames.
- For `@media` tests, run the stylesheet through the same build pipeline with controlled `envMatchMedia` mocks to simulate viewport widths.
- When verifying `@supports`, stub `CSS.supports` in the MOCKING section and assert fallback branches in separate tests.

---

## **9. Design Tokens & Theming**
- Treat each token (color, spacing, typography) as a unit test. Assert both the raw token value and its presence in generated CSS custom properties.
- For theme overrides, test default and overridden values separately using nested `describe()` blocks.
- Never hardcode hex codes in assertions; reference a canonical `expected*` token from the design token source.

---

## **10. Snapshot Usage**
- Snapshots are acceptable for **small, stable** CSS fragments (e.g., a single class). Keep them under 30 lines and colocate them with the test file.
- When snapshotting, normalize whitespace and vendor prefixes so diffs highlight semantic changes.
- Pair each snapshot with targeted assertions to ensure failures are meaningful (`expect(actualStyles.color).toBe(expectedColor)` in addition to the snapshot).
- Re-generate snapshots only after a reviewer confirms the change is intentional.

---

## **11. Accessibility & Interaction Assertions**
- Assert that focus indicators meet contrast requirements by checking computed colors against WCAG ratios (use helper functions).
- Verify `prefers-reduced-motion` handling: simulate the media query and ensure animations are disabled in that branch.
- For high contrast modes or forced colors, mock `matchMedia('(forced-colors: active)')` and assert fallback colors.

---

## **12. Limitations of JSDOM & Workarounds**
- Document every limitation you hit (e.g., `getBoundingClientRect` returns zeros). Replace those APIs with deterministic fakes in the MOCKING section.
- When browser-specific behavior cannot be simulated (e.g., subgrid layout), confine assertions to the generated CSS rather than computed layout metrics.
- If a rule depends on cascading order, assert on the order of rules within the AST rather than DOM measurements.

---

## **13. Lint & Build Integration**
- Run `stylelint` and the CSS build step inside tests to catch regressions early. Treat lint warnings as test failures.
- Ensure the test suite uses the same PostCSS config as production; share configuration files to avoid drift.
- When testing Tailwind or utility frameworks, execute the same purge/content pipeline to ensure class availability matches production.

---

## **14. Breaking the Rules (Rarely)**
- Deviate from the section layout only when writing quick smoke tests for third-party CSS where you cannot control build tooling. Document the deviation with a `// NOTE:` comment.
- If performance is a concern (e.g., compiling Tailwind for every test), cache compiled CSS between tests but keep the GIVEN/SETUP/WHEN/THEN commentary.
- Any intentional violations (inline literals in assertions, missing prefixes) must include a `// EXCEPTION:` comment explaining why the rule cannot be followed.

---

Following these conventions keeps CSS unit tests as rigorous and reviewable as their C# counterparts—structured, intention-revealing, and grounded in the real production pipeline.
