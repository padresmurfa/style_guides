### **Python Unit Testing Style Guide**

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Python.

---

## **Target Audience**

The primary target audience for this document are Large Language Models in the role of coding assistants. By following
these guidelines, LLMs can expect to create highly structured, readable and mainatainable unit and (moderately) integrated tests.

These practices are however quite heavy for a human coder, thus it is recommended that the human upload this style
guide to their LLM of choice, along with the source code that they wish to create unit tests for, and request that the
LLM create unit tests that fully cover the source code in question.

It is important that the human review the test functions carefully, as ultimately it is the human, not the LLM, that is
responsible for the quallity of the code.

---

## **1. Test Function Naming**
- Each test function **must start with** `test_`.
- The function name should be **descriptive and readable**, clearly reflecting what the test verifies.
- **Separate tests** into distinct functions rather than combining multiple test cases into one.
- **Avoid `pytest.mark.parametrize` unless it significantly improves clarity**.

✅ **Good Example:**
```python
def test_process_data_returns_correct_value():
```
🚫 **Bad Example:**
```python
def check_process_data():  # ❌ Missing test_ prefix
def test_process():  # ❌ Too vague
def test_multiple_cases():  # ❌ Tests multiple things at once
```

---

## **2. Test File Organization**
- **Each test file should cover only a single system-under-test (SUT).**
- **When testing a complex class, each function in the class may have its own test file.**
- This allows tests to be run with **coverage enabled** while focusing on the specific part of the SUT being verified.
- Avoid **mixing** tests for different parts of a system in the same file, as this can lead to **false positives in test coverage reports**.

✅ **Good Example:**
```
tests/
│── test_authenticate.py    # Covers `authenticate()`
│── test_process_request.py # Covers `process_request()`
│── test_service.py         # Covers a complex class's methods individually
```
🚫 **Bad Example:**
```
tests/
│── test_all_services.py  # ❌ Mixed tests in one file
```

---

## **3. Test Structure**
Each test function follows a structured format, **separated by clear comments**:

### **Standard Sections:**
1. **GIVEN**  
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given_`.
   - **Fixtures** should be named `fixture_*` and typically assigned to `given_*` variables.
   - If a fixture is used **directly in assertions**, it may be assigned to `expected_*`.

2. **MOCKING** *(if applicable)*  
   - Define and configure **mock objects**.
   - Mock variables must be named `mock_*`.
   - **Use `@patch` decorators before the function** rather than `with patch` inside.

3. **WHEN**  
   - Perform the action or function call being tested.
   - Assign the result to `actual_*` variables.

4. **EXPECTATIONS**  
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected_*` variables.

5. **THEN**  
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected_*` variables.

6. **BEHAVIOR** *(if applicable)*  
   - Contains assertions for **mock interactions** (e.g., `mock_api_call.assert_called_once()`).
   - This section **must be present if mocks are used**.

---

## **4. Fixtures**
- **Fixture functions must be prefixed with `fixture_`.**
- Assign them to `given_` variables in most cases.
- Fixtures **may be used directly in `expected_*` variables** if they represent expected outputs.
- Use `@pytest.fixture` for **reusable test setup logic**.

✅ **Good Example:**
```python
import pytest

@pytest.fixture
def fixture_database():
    return {"users": ["Alice", "Bob"]}

def test_fetch_user_from_database(fixture_database):
    """
    Ensure fetch_user correctly retrieves a user.

    GIVEN a database fixture
    WHEN fetch_user is called
    THEN it should return the expected user.
    """
    # GIVEN
    given_database = fixture_database
    given_user_id = "Alice"

    # WHEN
    actual_user = fetch_user(given_database, given_user_id)

    # EXPECTATIONS
    expected_user = "Alice"

    # THEN
    assert actual_user == expected_user
```

🚫 **Bad Example:**
```python
def test_fetch_user_from_database(fixture_database):
    assert fetch_user(fixture_database, "Alice") == "Alice"  # ❌ No structured sections
```

---

## **5. Using `@patch` for Mocking**
- **Always prefer `@patch` decorators before the function** instead of `with patch` inside.
- This ensures better readability and reduces indentation.
- **Assertions on mock interactions go in the BEHAVIOR section**.

✅ **Good Example:**
```python
from unittest.mock import patch

@patch("my_module.some_dependency")
def test_service_calls_dependency(mock_some_dependency):
    """
    Ensure the service correctly calls its dependency.

    GIVEN a service instance
    WHEN the service processes a request
    THEN it should call the dependency.
    """
    # GIVEN
    given_service = MyService()
    given_input = "data"

    # WHEN
    given_service.process(given_input)

    # EXPECTATIONS
    expected_call_arg = "data"

    # THEN
    assert mock_some_dependency.call_args[0][0] == expected_call_arg

    # BEHAVIOR
    mock_some_dependency.assert_called_once()
```

🚫 **Bad Example:**
```python
def test_service_calls_dependency():
    with patch("my_module.some_dependency") as mock_some_dependency:
        service = MyService()
        service.process("data")
        mock_some_dependency.assert_called_once()  # ❌ Mocking inside the function
```

---

## **6. Docstrings**
- Every test function **must have a docstring**.
- The **first line** should be a **one-liner** summarizing the test.
- The **one-liner should, by default, rephrase the function name into readable English**.
- The **rest of the docstring** should follow the **GIVEN, WHEN, THEN** format.

✅ **Good Example:**
```python
def test_process_data_returns_correct_value():
    """
    Ensure process_data correctly processes input.

    GIVEN a valid input value
    WHEN process_data is called
    THEN it should return the expected processed value.
    """
```

🚫 **Bad Example:**
```python
def test_process_data_returns_correct_value():
    """Test process_data."""  # ❌ Too vague
```

---

## **7. Assertions & Variable Naming**
- **Expected values** must be stored in `expected_*` variables **before** assertions.
- **Actual results** must be stored in `actual_*` variables.
- **Never assert against literals directly**—use `expected_*` variables.

✅ **Good Example:**
```python
expected_result = 42
assert actual_result == expected_result
```
🚫 **Bad Example:**
```python
assert actual_result == 42  # ❌ No expected variable
```

---

## **8. Avoiding TestCase Classes**
- **Do not use `unittest.TestCase`**.
- **No `setUp` or `tearDown` methods**—each test should be fully independent.
- **Use simple functions** instead of classes.

✅ **Good Example (Preferred Style with pytest):**
```python
import pytest

def test_some_function():
    """
    GIVEN a condition
    WHEN something happens
    THEN the expected result should occur.
    """
    # Test logic
```

🚫 **Bad Example (Avoid `TestCase` Classes):**
```python
import unittest

class TestSomeFunction(unittest.TestCase):
    def test_case_1(self):
        # ❌ Uses TestCase (not preferred)
```

---

## **9. Handling Exceptions (`pytest.raises`)**
- **When testing exceptions, the `THEN` section must be placed before `WHEN`**.
- **This is because the test's primary condition is that the SUT raises the expected exception**, and the actual execution occurs inside the indented block.
- **Use `pytest.raises` with `match=` where appropriate** to verify the error message.
- **Alternatively, prefer having the SUT raise custom exception classes instead of generic ones**. This allows the test suite to **catch them directly instead of relying on `match=`**.

✅ **Good Example:**
```python
import pytest

def test_divide_raises_exception():
    """
    Ensure divide() raises an exception when dividing by zero.

    GIVEN a zero denominator
    THEN a ZeroDivisionError should be raised
    WHEN divide is called.
    """
    # GIVEN
    given_numerator = 10
    given_denominator = 0

    # THEN
    with pytest.raises(ZeroDivisionError):
        # WHEN
        divide(given_numerator, given_denominator)
```

# Python Unit Testing Style Guide Additions

These additions extend the base style guide to cover the nuances of tests that blend both mocking and integration-style approaches. The goal is to ensure tests are human-readable, clearly structured, and maintain a high level of clarity regarding the flow of data, the use of mocks, and the expectations for the system under test.

---

## 10. Test Structure & Section Labels

- **Clear Sectioning:**  
  Every test function should be explicitly divided into sections using comments. At a minimum, include:

  - **GIVEN:**  
    - Describe the initial conditions, input data, and preconditions.
    - Use the prefix `given_` for variables representing these inputs.

  - **MOCKING & SETUP:** *(if applicable)*  
    - If both mocks and real setup are required, this section should contain both.
    - If a sufficient amount of both is present, split them into separate **MOCKING** and **SETUP** sections.
    - **MOCKING:** Used when creating mock objects; all mock variables must be prefixed with `mock_`.
    - **SETUP:** Used for real objects that define the context in which the system under test is exercised.
    - Mocks and setup objects should be clearly documented to describe their role in the test.

  - **WHEN:**  
    - Perform the action or function call under test.
    - Use the prefix `actual_` for results produced from this action.
    - If the **THEN** and **BEHAVIOR** sections benefit from it, define additional `actual_` variables after exercising the SUT (e.g., `actual_status = actual_response.status_code`).

  - **EXPECTING:**  
    - Define the expected outcome(s) (e.g., expected status codes, expected data, expected error messages).
    - Use the prefix `expected_` for these values.

  - **THEN:**  
    - Write assertions comparing the actual values to the expected values.
    - **Every assertion must include a custom error message** if it is possible to create a better human-readable message than the default.
    - The assertion should be formatted with the assertion itself on one line ending with a comma and a line continuation symbol, followed by an indented custom message on the subsequent line.
    - In cases where exception handling is tested with `pytest.raises`, and further assertions are needed after the `with` block, open a new **THEN** section.
    - If additional `actual_` variables need to be defined after a `with pytest.raises` block, open a new **WHEN** section for them.

  - **BEHAVIOR:** *(Optional)*  
    - When mocks are used, include additional assertions to verify that mocked dependencies were called correctly.

*Example structure:*

```python
def test_example(...):
    """
    One-line summary of what is being tested.

    GIVEN some initial state,
          and possibly some mocks set up,
    WHEN a specific action is taken,
    THEN the actual result matches the expected result,
         and the mocked dependencies are called correctly.
    """
    # GIVEN
    given_input = ...  # e.g., given_request, given_feature_flag

    # MOCKING & SETUP
    # (if applicable) Setup any mocks here with clear comments on their purpose

    # WHEN
    actual_result = function_under_test(given_input)
    actual_status = actual_result.status_code

    # EXPECTING
    expected_result = ...

    # THEN
    assert actual_result == expected_result,
        f"Expected {expected_result}, but got {actual_result}"
```

---


## 11. Variable Naming Conventions

**Prefixes:**
- `given_`: For input values, initial conditions, or preconditions.
- `actual_`: For results or outputs produced by the system under test.
- `expected_`: For the expected outcomes.
- `mock_`: For mock objects used in testing.

**Fixtures:**
- Always name fixtures with the prefix `fixture_` (e.g., `fixture_dummy_request`, `fixture_feature_flag`).
- Fixtures should only be used by assigning them once to a variable in the **GIVEN** section, or once in the **EXPECTING** section.
- Elsewhere, refer to the `given_` or `expected_` variables instead of the fixture directly.

## 12. Mixing Integration and Mocking Approaches

### Integration vs. Unit Tests:

**Integration Tests:**
- When it is acceptable to use the database and actual objects (e.g., using a real request factory or model instances), write tests that closely mirror production code with minimal mocking.

**Unit Tests with Mocks:**
- When isolating a unit (such as an endpoint that involves external dependencies), use mocks for those dependencies.
- Always include a **MOCKING & SETUP** section if mocks are applied.
- Ensure that mocks are only used where they add clarity or improve test speed.
- If integration tests become too slow, consider supplying both integrated and unit tests.
  - Mocked tests will run much faster and can be part of the local dev cycle.
  - Integration tests can be moved to a CI/CD pipeline if they are too slow for local development.


---

## 13. Documentation and Readability

### Docstrings:
- Every test function must have a clear docstring.
- The first line should be a one-liner summary that rephrases the function name in plain English.
- The remainder of the docstring should follow the **GIVEN, WHEN, THEN** (and optionally **MOCKING & SETUP, BEHAVIOR**) structure.

### Endpoint and Behavior Clarity:
- In the docstring, clearly mention which endpoint or method is being tested (e.g., `list`, `list_v2`, `retrieve`).
- Indicate whether the test uses real database objects or mocks.

---

## 14. Custom Assertion Messages

### Mandatory Custom Messages:
- Every assertion must include a custom error message that clearly states:
  - The expected value.
  - The actual value.

Example:

```python
assert actual_status_code == expected_status_code, (
    f"Expected status {expected_status_code}, but got {actual_status_code}"
)
```

---

## 15. Patch Decorator Usage

### Decorator Placement:
- Always use `@patch` decorators above the test function rather than using context managers inside the test body.
- This reduces indentation and makes the test flow more linear.

### Ordering of Patches:
- The order of the patch decorators should match the order of parameters in the test function.
- Include a comment (in the **MOCKING & SETUP** section) describing what each patch is replacing and why.

---

## 16. Consistent Test Naming

### Descriptive Names:
- Test names should be descriptive, following the pattern `test_<endpoint>_<expected_behavior>`.

Example:
```python
test_list_returns_feature_flag_detail_serializer_data
```

### Clarity:
- The name should provide a clear indication of what is being tested (e.g., the method, the condition, and the expected outcome).

---

## 17. Grouping and Organization

### Logical Grouping:
- Group tests by the system-under-test (e.g., all tests for `FeatureFlagViewSet` in one file).
- Within the file, consider grouping tests by the method or endpoint (e.g., tests for `list`, then `list_v2`, then `retrieve`).
- If a test file becomes too long, consider splitting it into multiple files (e.g., one per method or endpoint). This improves code coverage analysis by allowing focused test execution and avoiding false positives from unrelated tests.

### Inline Comments:
- Use inline comments or section dividers in the test file to separate tests that rely on heavy mocking from those that are more integration-style.

---
