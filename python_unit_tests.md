### **Python Unit Testing Style Guide**

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Python using `pytest`
or the built-in `unittest` framework.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By
following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may feel heavy for human-written tests but serve as an excellent guide for generating automated tests with LLM
assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Package Organization**
Consistent module paths make it obvious where a test lives relative to the production code, allowing reviewers to find the
subject under test without jumping between folders. Misaligned packages cause confusion when navigating between source and
tests, especially when IDEs rely on mirrored paths for discovery.
- Place tests in a `tests/` package (or equivalent) that mirrors the source package structure.
- For a module `my_project/services/payment.py`, place tests in `tests/services/test_payment.py`.
- Keep a **one-to-one mapping between folders and packages** so moving a file preserves the import path.
- Avoid combining tests for different packages into a single directory—mirror the source layout instead.

✅ **Good Example:**
```
my_project/
│── services/
│   └── payment.py
tests/
│── services/
│   └── test_payment.py
```
🚫 **Bad Example:**
```
tests/
│── test_everything.py  # ❌ Mixed modules in one file
```

---

## **2. Test Module and Class Naming**
Consistent module names make it immediately obvious which production code a suite exercises, so reviewers can trace failures
quickly and spot coverage gaps. Without a naming convention, teams waste time hunting for relevant tests and risk overlooking
scenarios because responsibilities are spread across ambiguously labeled files.
- Each test module must start with `test_` and mirror the production module name (e.g. `payment.py` → `test_payment.py`).
- When targeting a single class or function in a large module, append a descriptive suffix (e.g. `test_payment_processor.py`).
- Limit each test module to a single class, region, or function under test. Split into multiple files when covering unrelated
APIs.
- If you use `unittest.TestCase`, ensure the class name ends with `Tests` (e.g. `class PaymentProcessorTests(unittest.TestCase)`).
- Prefer plain functions unless `TestCase` provides a clear benefit; do not mix helper classes for unrelated APIs.
- Test class names should not contain underscores. Use CamelCase + `Tests` suffix.

✅ **Good Examples:**
```python
class PaymentProcessorTests(unittest.TestCase):
    ...

class PaymentProcessorRefundTests(unittest.TestCase):
    ...
```
🚫 **Bad Example:**
```python
class payment_processor(unittest.TestCase):  # ❌ Missing Tests suffix and wrong casing
```

---

## **3. Test Function Naming**
Clear function names document the behavior under test and the expected outcome. Vague or inconsistent names hide gaps in
coverage, encourage multi-purpose tests, and make regression triage harder when a failing test does not reveal what broke.
- Each test function **must start with** `test_`.
- Use lowercase with underscores to separate phrases (snake_case).
- If multiple production functions are tested within the same module, the test name **must start with**
  `test_<function_name>_`.
- If the entire module targets a single production function, omit the function name to avoid duplication with the module name.
- Test names must describe the **specific branch of execution** they cover (e.g. `test_process_payment_returns_receipt_when_card_is_valid`).
- Avoid consecutive underscores and do not duplicate phrases already present in the test module filename.
- **Separate tests** into distinct functions rather than combining multiple scenarios into one.
- **Avoid parametrization** unless it significantly improves clarity or removes obvious duplication.

✅ **Good Example:**
```python
def test_process_payment_returns_receipt_when_card_is_valid():
    ...
```
🚫 **Bad Example:**
```python
def test_process_payment():  # ❌ Too vague
def test_multiple_cases():    # ❌ Combines many scenarios
```

---

## **4. Test Function Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through fixtures,
mocks, and the system under test. Without structure, tests devolve into monolithic blocks where intent, setup, and verification
intermingle. The ordering rules below are intentionally strong—deviate only with compelling justification.

Each test function follows a structured format, **separated by clear, descriptive comments**:
- Section headers use uppercase keywords followed by a colon sentence (e.g. `# GIVEN: a user with no roles`).
- Empty sections should be omitted rather than left blank.
- When a single section becomes large or involves multiple responsibilities, split it into sub-sections with descriptive
  headers (e.g. `# SETUP: configure request context`).
- Section comments should be meaningful sentences, not terse labels.

### **Standard Sections:**

1. **GIVEN**
    - Set up initial conditions and inputs.
    - Variables declared here use the `given_` prefix.
    - Instantiate fixtures or helper factories in this section; fixture classes should be named `Fixture*`.
    - The GIVEN section must not be merged with any other section and should appear first when present.

2. **CAPTURE**
    - Declare containers for values captured from collaborators (e.g., through mocks, fakes, or fixtures like `caplog`).
    - Name these placeholders with the `capture_` prefix to make their purpose obvious.
    - Limit this section to declarations; configure how values are captured inside MOCKING or SETUP.
    - Skip this section entirely when no captured data is needed.

3. **MOCKING**
    - Define and configure mock objects.
    - Mock variables must be named `mock_*` (or `fake_*` for handwritten fakes).
    - Prefer decorator-based patching (`@patch`) or fixture-based mocks over inline context managers.
    - Keep mocking logic separate from other sections; large mocks may be split into sub-sections.

4. **SETUP**
    - Prepare the environment around the system under test (e.g. wiring dependencies, building contexts, assigning mocks to
      collaborator slots).
    - Variables created here use the `env_` prefix and may reference `given_*` and `mock_*` values.
    - Assign mock objects to their concrete interfaces in this section (e.g. `env_repository = mock_repository`).
    - Ensure SETUP follows whichever of GIVEN, CAPTURE, and MOCKING are present; omit unused sections without reordering those that remain.
    - The SETUP section must not be merged with other sections.

5. **SYSTEM UNDER TEST**
   - Instantiate or select the system under test and assign it to a variable named `sut` (or `sut_*` when multiple systems are
     required).
   - The SUT should depend on `env_` variables, not raw `mock_*` instances.
   - This section stands alone and is not merged with others.

6. **WHEN**
    - Execute the action being tested.
    - Assign return values and side effects to `actual_*` variables.
    - If later sections need environment values, capture them into `actual_*` variables instead of reusing `env_*` directly.
    - When collaborators captured values into `capture_*` placeholders, move them into `actual_*` variables in this section before asserting.
    - Do not merge WHEN with other sections.

7. **EXPECTATIONS**
   - Declare expected values before performing assertions.
   - Expected values must be assigned to `expected_*` variables and should not reference `actual_*` names.
   - This section always appears between WHEN and THEN and is never merged with other sections.

8. **THEN**
   - Perform assertions comparing `actual_*` results to `expected_*` values.
   - Never assert directly against literals—use an `expected_*` variable, even for simple values.
   - Keep assertions focused on verifying outcomes, not behaviors of mocks.

9. **LOGGING** *(if applicable)*
   - Verify behavior observable through captured logs (e.g. via `caplog`).
   - Prefer log assertions over mock verifications when they clearly demonstrate the branch exercised.
   - Use helper utilities or fixtures to assert on log structure rather than parsing raw tuples inline.

10. **BEHAVIOR** *(if mocks are used)*
   - Assert interactions with mocks or fakes (e.g. `mock_client.assert_called_once_with(...)`).
   - Every mock configured for verification must be asserted in this section; unused mocks should be removed or converted to
     fixtures.
   - Keep interaction assertions out of the THEN section to avoid mixing concerns.

✅ **Good Example:**
```python
from decimal import Decimal

def test_process_payment_returns_receipt_when_card_is_valid(mock_gateway):
    """Ensure process_payment issues receipts for valid cards."""

    # GIVEN: a valid payment request
    given_order_id = "order-123"
    given_amount = Decimal("25.00")

    # MOCKING: configure the payment gateway
    mock_gateway.charge.return_value = "charge-456"

    # SETUP: expose collaborators via env_* variables
    env_gateway = mock_gateway

    # SYSTEM UNDER TEST: build the processor with dependencies
    sut = PaymentProcessor(gateway=env_gateway)

    # WHEN: processing the payment
    actual_receipt = sut.process_payment(order_id=given_order_id, amount=given_amount)

    # EXPECTATIONS: define expected receipt contents
    expected_receipt = Receipt(id="charge-456", order_id=given_order_id, amount=given_amount)

    # THEN: verify the receipt matches expectations
    assert actual_receipt == expected_receipt

    # BEHAVIOR: gateway charge invoked exactly once
    mock_gateway.charge.assert_called_once_with(order_id=given_order_id, amount=given_amount)
```

🚫 **Bad Example:**
```python
from decimal import Decimal

def test_process_payment(mock_gateway):
    sut = PaymentProcessor(gateway=mock_gateway)
    assert sut.process_payment("order-123", Decimal("25.00")) == Receipt(...)
    # ❌ No sections, literal assertions, mock never verified
```

---

## **5. Fixtures: Best Practices**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping tests focused on behavior instead
of plumbing. Inlining bespoke objects everywhere invites duplication, increases maintenance costs when models evolve, and makes
it harder to reason about cascading changes.
- **Prefer fixtures or factory helpers over one-off inline data.**
- Fixture functions must be prefixed with `fixture_` and should live in a `conftest.py` or dedicated fixture module.
- Assign fixtures to `given_*` variables before use. When a fixture represents an expected output, assign it to `expected_*`.
- Use factory helpers (e.g. `FixtureOrder.create()`) to encapsulate complex setup logic. Name fixture classes with a `Fixture`
  prefix.
- Avoid mutating shared fixtures; if state must change, return a new copy per test using fixtures with function scope.

✅ **Good Example:**
```python
from decimal import Decimal

import pytest

@pytest.fixture
def fixture_order():
    return FixtureOrder.create(order_id="order-123", amount=Decimal("25.00"))

def test_submit_order_enqueues_job(fixture_order, queue_client):
    # GIVEN: a fully populated order
    given_order = fixture_order

    # SETUP: configure environment dependencies
    env_queue_client = queue_client

    # SYSTEM UNDER TEST: create the service
    sut = OrderService(queue_client=env_queue_client)

    # WHEN: submitting the order
    sut.submit_order(given_order)

    # BEHAVIOR: ensure enqueue was invoked
    queue_client.enqueue.assert_called_once()
```

🚫 **Bad Example:**
```python
def test_submit_order_enqueues_job(queue_client):
    given_order = {"id": "order-123", "amount": "25.00"}  # ❌ Inline magic values
    OrderService(queue_client=queue_client).submit_order(given_order)
    queue_client.enqueue.assert_called_once()
```

---

## **6. Mocking: Best Practices**
Disciplined mocking keeps tests readable and trustworthy by ensuring only true collaborators are simulated and expectations are
explicit. Over-mocking creates brittle tests that break on harmless refactors, mask missing coverage of dependencies, and clutter
SUT construction with boilerplate.
- **Never mock what you do not have to**—prefer real or fixture-backed collaborators where practical.
- **Only mock dependencies the SUT uses directly.** Downstream collaborators deserve their own tests.
- Use `autospec=True` when patching to catch drift between mocks and real signatures.
- Do not pass `mock_*` objects directly into the SUT; expose them via `env_*` assignments in SETUP first.
- Assertions on mock interactions belong in the BEHAVIOR section, not in THEN.
- Configure mocks to fail fast when unexpected calls occur (e.g. using `side_effect=AssertionError(...)` or strict doubles).
- When verifying asynchronous code, use `AsyncMock` and assert awaited calls explicitly (e.g. `mock_client.send.assert_awaited_once()`).

### **6.1 Test Fakes (a.k.a. Dummies)**
Test fakes are lightweight implementations you author specifically for the test suite. They shine when mocking frameworks become
gnarly—especially for complex interaction verification or structured log inspection—because you can write intention-revealing
code without juggling patch objects.
- Create fakes when configuring or verifying mocks would be noisy, repetitive, or brittle.
- Name fake instances with the `fake_*` prefix and construct them in the MOCKING section.
- Assign each fake to an `env_*` variable in SETUP before injecting it into the SUT.
- Provide helper assertion methods on the fake itself (e.g. `fake_logger.assert_logged(...)`) and invoke them in LOGGING or
  BEHAVIOR sections.
- Keep fake implementations close to the tests they support to ensure they stay in sync with the production interfaces.

✅ **Example: Logging with a Fake**
```python
class FakeLogger:
    def __init__(self):
        self._entries = []

    def info(self, message, **kwargs):
        self._entries.append((message, kwargs))

    def assert_logged(self, expected_message, **expected_kwargs):
        assert len(self._entries) == 1
        actual_message, actual_kwargs = self._entries[0]
        assert actual_message == expected_message
        assert actual_kwargs == expected_kwargs


def test_process_branch_logs_trace_message():
    # GIVEN: a branch that triggers logging
    given_branch = "bar"

    # MOCKING: create a fake logger
    fake_logger = FakeLogger()

    # SETUP: expose fake through env_* variable
    env_logger = fake_logger

    # SYSTEM UNDER TEST
    sut = BranchProcessor(logger=env_logger)

    # WHEN: processing the branch
    sut.process_branch(given_branch)

    # LOGGING: verify trace message
    fake_logger.assert_logged("Foo(branch=bar): the foo is quite barred", branch=given_branch)
```

---

## **7. Breaking the Rules**
There will be rare cases where strict adherence to the structure above reduces clarity. When deviating:
- Document the rationale with an inline comment above the deviation.
- Keep deviations localized—do not let one exception justify loosening discipline across the module.
- Ensure reviewers can still trace setup, execution, and verification without hunting through the test body.

---

By embracing these conventions, your Python unit tests will read like executable documentation: they highlight intent, isolate
behaviors, and make failures easy to diagnose.

