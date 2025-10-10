# Kotlin Unit Testing Style Guide

This document adapts the conventions from the C# unit testing style guide for Kotlin code bases that rely on **JUnit 5**, **Kotest**, or **Kotlin's kotlinx-test APIs**. The goal is to produce **clear, structured, and maintainable** tests that communicate intent, isolate behavior, and remain resilient during refactors.

---

## **Target Audience**

The primary audience is developers and Large Language Models (LLMs) that generate Kotlin unit tests. Following these rules keeps tests discoverable, self-documenting, and trustworthy. Human review remains essential: **never merge generated tests without reading them end-to-end.**

---

## **1. Package Organization**
Consistent packages make it obvious which production type each test exercises and allow IDEs to co-locate sources and tests.
- Mirror the production package exactly and append a `.test` suffix (e.g., `com.example.billing` → `com.example.billing.test`).
- Match the folder hierarchy to the package (e.g., `src/test/kotlin/com/example/billing/test`).
- When testing internal APIs, keep the test class in the same package as the production code and annotate with `@OptIn(ExperimentalCoroutinesApi::class)` or visibility helpers as needed.

✅ **Good Example:**
```kotlin
package com.example.billing.test
```
🚫 **Bad Example:**
```kotlin
package com.example.tests // ❌ Does not mirror the production package
```

---

## **2. Test Class Naming and Files**
Clear naming keeps coverage obvious and avoids sprawling test files.
- Each test file must contain exactly one top-level public test class.
- Test classes must end with `Test` (singular) rather than `Tests` to match Kotlin idioms (`InvoiceServiceTest`).
- If the class covers a specific method or nested responsibility, append that scope (`InvoiceServiceCalculateTotalsTest`).
- File names must match the class name exactly (`InvoiceServiceTest.kt`).
- Avoid underscores or spaces in class names.

✅ **Good Examples:**
```kotlin
class InvoiceServiceTest
class InvoiceServiceCalculateTotalsTest
```
🚫 **Bad Examples:**
```kotlin
class InvoiceServiceTests // ❌ Plural suffix
class InvoiceServiceWorksGreatTest // ❌ Does not map to a concrete responsibility
```

---

## **3. Focused Test Classes**
Break tests into multiple classes when a production type contains distinct regions of behavior.
- Create separate test classes for different responsibilities (e.g., validation vs. persistence) rather than packing everything into one file.
- Nesting test classes with `inner class` or Kotest `context` blocks is allowed for readability, but keep each nested scope focused on a single method or branch.
- Do not reuse scope suffixes for unrelated behaviors.

---

## **4. Test Function Naming**
Test names double as documentation. Use sentence-style names that describe behavior and outcome.
- Every test function must start with `fun test` and use backticks for readable titles, or use CamelCase prefixed with `test_` if backticks are unavailable (e.g., in parameterized providers).
- Preferred pattern: ``@Test fun `processes valid invoices`()``.
- When multiple production methods exist in the same class, start each name with the method under test: ``fun `calculateTotals returns zero for empty list`()``.
- Avoid consecutive underscores and vague names like ``fun `works`()``.
- Keep each test focused on a single branch.

✅ **Good Example:**
```kotlin
@Test
fun `calculateTotals returns zero for empty invoices`() { ... }
```
🚫 **Bad Example:**
```kotlin
@Test
fun testCalculate() { /* ❌ Too vague */ }
```

---

## **5. Test Function Structure**
Consistent sections make tests scannable. Separate logical phases with `//` comments using the template below. Omit empty sections.

1. **GIVEN**
   - Declare inputs, fixtures, and collaborators.
   - Prefix variables with `given` (e.g., `val givenInvoices = listOf(...)`).
   - Instantiate reusable fixtures here.

2. **MOCKING**
   - Configure mocks using libraries like MockK.
   - Name mock variables with the `mock` prefix (`val mockRepository = mockk<InvoiceRepository>()`).

3. **SETUP**
   - Wire mocks into concrete collaborators and prepare environment objects.
   - Use the `env` prefix (`val envRepository: InvoiceRepository = mockRepository`).

4. **SYSTEM UNDER TEST**
   - Assign the object or function under test to `sut` (or `sut*` for multiples).
   - Never pass mocks directly—pass `env*` variables to the constructor or function.

5. **WHEN**
   - Execute the behavior under test.
   - Assign outcomes to `actual*` variables.

6. **EXPECTATIONS**
   - Declare expected values in `expected*` variables before assertions.

7. **THEN**
   - Assert results using assertion libraries (JUnit, Truth, Kotest).
   - Never compare against literals directly; use `expected*` values.

8. **LOGGING** *(optional)*
   - Validate structured logging or metric emission captured via fakes.

9. **BEHAVIOR** *(required when mocks are used)*
   - Verify interactions (e.g., `verify { mockRepository.save(expectedInvoice) }`).

Use descriptive sub-headers when sections grow large (e.g., `// SETUP: configure dispatcher`).

✅ **Good Example:**
```kotlin
@Test
fun `calculateTotals returns zero for empty invoices`() {
    // GIVEN: an empty invoice list
    val givenInvoices = emptyList<Invoice>()

    // SYSTEM UNDER TEST: default service
    val sut = InvoiceService()

    // WHEN: totals are calculated
    val actualTotal = sut.calculateTotals(givenInvoices)

    // EXPECTATIONS: zero total
    val expectedTotal = BigDecimal.ZERO

    // THEN: total matches expectation
    assertThat(actualTotal).isEqualTo(expectedTotal)
}
```

🚫 **Bad Example:**
```kotlin
@Test
fun `calculateTotals`() {
    val invoices = emptyList<Invoice>()
    assertEquals(BigDecimal.ZERO, InvoiceService().calculateTotals(invoices)) // ❌ inline literals, no sections
}
```

---

## **6. Fixtures and Test Data**
Prefer reusable fixtures over ad-hoc literals to keep tests expressive and resilient.
- Place shared fixtures in `object Fixtures` or top-level functions under `src/test/kotlin/.../fixture` packages.
- Give fixture builders verbs: `fun aPaidInvoice(): Invoice`.
- Mutate fixture copies with `copy(...)` rather than building new instances from scratch.
- Instantiate fixtures in GIVEN or SETUP sections.

✅ **Good Example:**
```kotlin
object InvoiceFixtures {
    fun unpaid() = Invoice(id = UUID.randomUUID(), amount = 100.toBigDecimal(), status = InvoiceStatus.PENDING)
}

@Test
fun `markAsPaid updates status`() {
    // GIVEN: a pending invoice
    val givenInvoice = InvoiceFixtures.unpaid()

    // SYSTEM UNDER TEST
    val sut = InvoiceService()

    // WHEN: invoice is marked paid
    val actualInvoice = sut.markAsPaid(givenInvoice)

    // EXPECTATIONS
    val expectedStatus = InvoiceStatus.PAID

    // THEN
    assertThat(actualInvoice.status).isEqualTo(expectedStatus)
}
```

---

## **7. Mocking Guidelines**
Use mocks deliberately to isolate collaborators without overconstraining implementation.
- Favor real collaborators or fixtures when practical; mock only direct dependencies of the SUT.
- Configure mocks in the MOCKING section and expose them via `env*` variables in SETUP before injecting into the SUT.
- For MockK:
  - Use `mockk(relaxed = false)` to fail on unexpected calls.
  - Call `every { ... } returns ...` in MOCKING and `verify { ... }` in BEHAVIOR.
- Prefer test doubles (fakes) when verifying structured logs, metrics, or complex state is easier via handwritten helpers.
- Name fakes with the `fake` prefix and treat them like mocks (constructed in MOCKING, injected via `env*`).

✅ **Good Example:**
```kotlin
@Test
fun `processor saves aggregated invoice`() {
    // GIVEN: aggregated invoice
    val givenInvoice = InvoiceFixtures.unpaid()

    // MOCKING: strict repository mock
    val mockRepository = mockk<InvoiceRepository>(relaxed = false)
    every { mockRepository.save(any()) } returns Unit

    // SETUP: expose dependency
    val envRepository: InvoiceRepository = mockRepository

    // SYSTEM UNDER TEST
    val sut = InvoiceProcessor(envRepository)

    // WHEN
    sut.process(givenInvoice)

    // EXPECTATIONS
    val expectedInvoice = givenInvoice.copy(status = InvoiceStatus.PROCESSED)

    // BEHAVIOR
    verify(exactly = 1) { mockRepository.save(expectedInvoice) }
}
```

🚫 **Bad Example:**
```kotlin
@Test
fun `processor saves invoice`() {
    val repo = mockk<InvoiceRepository>()
    InvoiceProcessor(repo).process(InvoiceFixtures.unpaid())
    verify { repo.save(any()) } // ❌ No structure, `any()` hides expectations
}
```

---

## **8. Assertions & Variable Naming**
Explicit variable roles make failures easy to diagnose.
- Store expected values in `expected*` variables inside EXPECTATIONS.
- Capture results in `actual*` variables during WHEN.
- Avoid asserting against literals or `given*` values directly.
- Prefer expressive assertion libraries (Truth, AssertJ, Kotest matchers) for clear failure messages.
- When asserting collections, use dedicated helpers (`containsExactly`, `hasSize`) rather than manual loops.

---

## **9. Exception Testing**
Treat exception capture as part of the WHEN phase.
- Use `assertFailsWith<T>` or `assertThrows<T>` and store the exception in an `actual*` variable.
- Assert on both type and message (or status codes) in THEN.
- For coroutine APIs, wrap the call in `runTest { ... }` and call `assertFailsWith` inside the coroutine scope.

✅ **Good Example:**
```kotlin
@Test
fun `division throws on zero denominator`() {
    // GIVEN
    val givenNumerator = 10
    val givenDenominator = 0

    // WHEN
    val actualException = assertFailsWith<ArithmeticException> {
        divide(givenNumerator, givenDenominator)
    }

    // EXPECTATIONS
    val expectedMessage = "/ by zero"

    // THEN
    assertThat(actualException.message).isEqualTo(expectedMessage)
}
```

---

## **10. Shared Setup Hooks**
Use lifecycle hooks intentionally.
- Prefer local setup within tests or fixtures over `@BeforeEach` to keep context near assertions.
- When duplication is unavoidable, limit hooks to idempotent initialization (e.g., creating a test dispatcher) and document their intent.
- Avoid mutating shared state across tests; prefer immutable fixtures or new instances per test.

---

## **11. Coroutines and Asynchronous Code**
Kotlin-specific asynchronous patterns need additional discipline.
- Use `runTest` from `kotlinx-coroutines-test` for suspending functions.
- Inject `TestDispatcher` or `TestScope` via constructor parameters and expose them as `envDispatcher`, `envScope` in SETUP.
- Advance virtual time explicitly with `testScheduler.advanceUntilIdle()` inside WHEN.
- Prefer `MockK` suspend mocks (`coEvery`, `coVerify`) for suspend functions and isolate them in the MOCKING/BEHAVIOR sections.

---

## **12. Parameterized and Data-Driven Tests**
Use parameterized tests sparingly and only when they improve clarity.
- Keep each parameter set self-describing; avoid cryptic numeric tuples.
- Name providers with `provide` prefix (`provideInvalidInvoices`).
- When using JUnit 5 `@MethodSource`, locate the provider in the companion object and return a `Stream<Arguments>`.
- Inside parameterized methods, maintain the same GIVEN/WHEN/THEN structure with comments.

---

## **13. Breaking the Rules**
Occasionally a test benefits from deviating (e.g., snapshot tests, DSL-style Kotest specs). When breaking conventions:
- Comment why the exception is necessary.
- Keep as much structure as practical (e.g., use GIVEN/WHEN/THEN inside Kotest `should` blocks).
- Never skip variable naming conventions for expected vs. actual values.

---

## **14. Checklist for Reviewers**
Use this quick scan before merging:
1. **Package mirrors production** with `.test` suffix.
2. **One class per file** with a meaningful `*Test` name.
3. **Test names describe behavior** and start with ``fun `...`()``.
4. **Sections present** with clear comments; mocks verified in BEHAVIOR.
5. **Fixtures reused** instead of literals.
6. **Assertions compare `expected*` to `actual*`**—no inline literals.
7. **Exceptions captured and verified** explicitly.
8. **Coroutine tests use runTest and test dispatchers**.
9. **No hidden shared state** in lifecycle hooks.

Following these guidelines keeps Kotlin unit tests consistent with the rigor of the original C# style guide while embracing Kotlin idioms.
