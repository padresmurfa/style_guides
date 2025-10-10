# Django Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Django using Django's built-in `TestCase`, the Django test client, and pytest-django.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Module Organization**
Consistent module placement makes it immediately obvious where a test lives relative to the production code, so reviewers can locate the subject under test without scanning multiple folders. Misaligned modules cause confusion when navigating between apps and tests, especially in projects where Django's `AppConfig` isolates concerns.
- Every Django app must contain its tests in a dedicated `tests/` package rather than a single `tests.py` file.
- Within the `tests/` package, mirror the app's source tree: `tests/models/` for model tests, `tests/views/` for views, `tests/forms/` for forms, etc.
- Keep a **one-to-one mapping between modules and Django components** so moving a source file preserves the test module location.
- Avoid combining tests for different apps or components in one module—create separate modules instead.

✅ **Good Example:**
```text
my_app/
├── models.py
├── views/
│   └── product.py
└── tests/
    ├── models/
    │   └── test_product.py
    └── views/
        └── test_product.py
```
🚫 **Bad Example:**
```text
my_app/
├── views/
│   └── product.py
└── tests.py  # ❌ All tests lumped together
```

---

## **2. Test Class Naming and Files**
Consistent test class names make it immediately obvious which production code a suite exercises, so reviewers can trace failures quickly and spot coverage gaps. Without a naming convention teams waste time hunting for relevant tests, duplicate effort, and risk overlooking scenarios because responsibilities are spread across ambiguously labeled files.
- Each test module must contain one primary `TestCase`-derived class whose name **ends with** `Tests`.
- If a class exercises a whole Django component (e.g., a model or view), it must be named `<ComponentName>Tests`.
- If a class exercises a single method or handler, it must be named `<ComponentName><MethodName>Tests`.
- Method-focused test classes are **optional** patterns. Prefer them when they clarify intent; otherwise keep everything in a single `<ComponentName>Tests` class when the component under test is small.
- Test class names should not contain underscores and must inherit from `django.test.TestCase`, `SimpleTestCase`, or `pytest-django` equivalents (e.g., plain functions using fixtures may exist but follow the same naming rules in docstrings and comments).
- **Each test class should cover a single class, view, form, serializer, or utility.**
- **A test module's file name must be of the form `test_<component>[_<method>].py`**, and every component of the file name must appear in the class name exactly once and in the same order.
- Organize test modules into packages that mirror the app structure to keep discovery tools and human navigation aligned.

✅ **Good Examples:**
```python
class ProductModelTests(TestCase):
    ...

class ProductListViewGetTests(TestCase):
    ...
```
🚫 **Bad Example:**
```python
class ProductTests(TestCase):  # ❌ Ambiguous responsibility
    ...

class WorksAsIntended(TestCase):  # ❌ Missing Tests suffix and unclear target
    ...
```

---

## **3. Test Method Naming**
Clear method names document the behavior under test and the expected outcome, which helps maintainers understand intent without rereading the implementation. Vague or inconsistent names hide gaps in coverage, encourage multi-purpose tests, and make regression triage harder when a failing test name does not reveal what broke.
- Each test method **must start with** `test_`.
- Use underscores to separate major phrases in the test method name.
- Test method names should **not contain consecutive underscores**.
- If multiple production methods are being tested within the same test class, the test method **must start with** `test_<method_name>_`.
- If a single production method is being tested in the file, omit the method name from the test method.
- The method name should be **descriptive and readable**, reflecting the behavior under test.
- Favor verbosity over brevity—short names rarely communicate enough context to make the test self-documenting.
- Test method names must describe the **specific branch of execution** they cover (e.g. `test_get_returns_200_when_user_authenticated`).
- **Separate tests** into distinct methods rather than combining multiple test cases.
- **Avoid parameterized tests** unless they significantly improve clarity and do not introduce database reuse issues.
- Any component found in the test class name **must NOT be duplicated in the test method name**.

✅ **Good Example:**
```python
def test_post_redirects_to_detail_when_form_valid(self):
    ...
```
🚫 **Bad Example:**
```python
def test_view(self):  # ❌ Too vague
    ...

def test_multiple_cases(self):  # ❌ Tests multiple things at once
    ...
```

---

## **4. Test Method Sectioning**
Standardized sections carve complex tests into digestible steps, making it easier to see how inputs flow through factories, the Django ORM, and the system under test. Without this structure tests devolve into monolithic blocks where intent, setup, and verification intermingle, obscuring bugs and encouraging brittle copy-paste patterns.
Each test method follows a structured format, **separated by clear comments**:
- Use the order presented below wherever possible.
- Empty sections should be omitted.
- When a section becomes large or involves multiple distinct steps, split it into smaller sub-sections with human-readable comment headers (e.g. `# SETUP: Configure authenticated client`).
- Section comment headers should be written as full descriptive sentences (e.g. `# GIVEN: An authenticated customer`).

### **Standard Sections:**

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given_`.
   - Factories (`factory_boy`, `model_bakery`) should assign created objects to `given_` variables.
   - Use Django's `override_settings` context managers here to configure environment prerequisites.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define, in which case omit it.

2. **MOCKING**
   - Define and configure **mock objects** using `unittest.mock.patch` or pytest monkeypatching.
   - Mock variables must be named `mock_`.
   - Prefer decorators (`@patch`) over context managers for readability.
   - Configure external service replacements (e.g., email backends, requests to third-party APIs) here.

3. **SETUP**
   - Prepare the environment that interacts with Django (e.g., instantiate `Client`, configure `RequestFactory`, create URL kwargs).
   - Variables created here should be prefixed with `env_` (e.g., `env_client`, `env_request`).
   - Assign mock objects to environment variables in this section (e.g., `env_mailer = mock_mailer`).

4. **SYSTEM UNDER TEST**
   - Assign the system under test to a variable named `sut` when using callable utilities or class-based views (e.g., `sut = ProductListView.as_view()`).
   - For request/response tests, this section may simply note the view or client being exercised.

5. **WHEN**
   - Perform the action being tested (e.g., calling a view, invoking a model method).
   - Assign the result to `actual_` variables.
   - Wrap database commits in transactions only if the code under test requires it.

6. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected_` variables.
   - Include expected HTTP status codes, template names, JSON payloads, queryset contents, etc.

7. **THEN**
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected_` variables.
   - Use Django's assertion helpers (e.g., `self.assertTemplateUsed`, `self.assertContains`) within this section.

8. **BEHAVIOR**
   - Contains assertions for **mock interactions** (e.g., `mock_send_mail.assert_called_once()`).
   - This section **must be present if mocks are used**.

---

## **5. Database Usage**
- Derive from `django.test.TestCase` when the test touches the database; use `SimpleTestCase` for pure unit tests.
- Always create models within the GIVEN section using factories or explicit constructors; avoid relying on global fixtures that hide dependencies.
- Use `setUpTestData` for expensive fixtures that can be shared across methods but keep per-test state within the GIVEN section to prevent leakage.
- Never mutate class-level fixtures inside tests—clone or copy objects instead.
- Use `django_assert_max_num_queries` (pytest-django) or custom context managers to enforce query counts when relevant.

---

## **6. Django Test Client and RequestFactory**
- Prefer Django's `Client` for high-level request/response testing and `RequestFactory` for class-based view units.
- Initialize clients in the SETUP section with descriptive names (`env_client = Client()`).
- For authenticated scenarios, create users in GIVEN and authenticate them in SETUP (`env_client.force_login(given_user)`).
- When testing class-based views, build the request with `RequestFactory` in SETUP and assign the view to `sut`.

✅ **Good Example:**
```python
# GIVEN: An authenticated staff user
given_user = UserFactory(is_staff=True)

# SETUP: Configure client
env_client = Client()
env_client.force_login(given_user)

# SYSTEM UNDER TEST
sut = reverse("products:list")

# WHEN
actual_response = env_client.get(sut)
```

---

## **7. Factories, Fixtures, and Data Builders**
- Prefer factories (`factory_boy`, `model_bakery`) over static fixtures to keep tests self-contained and explicit.
- Fixture functions must be prefixed with `fixture_` and return data, not perform assertions.
- When using pytest-django fixtures (e.g., `client`, `db`), assign them to `env_` variables within SETUP to maintain naming consistency.
- Avoid loading Django fixture files (`loaddata`) inside tests; instead, create objects programmatically in GIVEN.

---

## **8. Assertions and Error Handling**
- Use Django's built-in assertion helpers when they improve clarity (`assertTemplateUsed`, `assertRedirects`).
- When asserting on querysets, evaluate them explicitly and assign to `actual_` variables before THEN.
- For exceptions, use `with self.assertRaises(ExpectedException)` inside THEN, and capture the context if verifying message text.
- Prefer `assertJSONEqual` for JSON responses to avoid brittle string comparisons.

---

## **9. Mail, Signals, and Background Tasks**
- Patch external integrations such as email senders, Celery tasks, or signal handlers in the MOCKING section.
- Assert signal registration by connecting temporary receivers within GIVEN and verifying they were triggered in BEHAVIOR.
- When testing asynchronous tasks triggered by views, capture the mock task and verify `.delay()` invocations.

---

## **10. Settings and Environment Overrides**
- Use `@override_settings` or the `settings` fixture to modify Django settings for a test.
- Apply overrides within GIVEN or via decorators at the method/class level; never rely on global settings modifications.
- Reset environment variables using pytest's `monkeypatch` fixture or context managers to prevent leakage between tests.

---

## **11. Breaking the Rules**
The guidelines above are intentionally prescriptive to build reliable habits. Deviate only when doing so makes the test **clearer** and **easier to maintain**.
- When a naming rule conflicts with established project conventions, prefer the existing convention but document the deviation.
- If strict sectioning would create unnecessary boilerplate (e.g., trivial pure functions), you may omit certain sections while keeping the remaining structure explicit.
- Any deviation must be justified with a comment referencing why the standard pattern was not followed.

---

By adhering to these conventions, Django unit tests remain readable, intention-revealing, and straightforward to maintain.
