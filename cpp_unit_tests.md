## C++ Unit Testing Style Guide

This document defines best practices for writing clear, structured, and maintainable unit tests in C++. It applies to tests written using popular frameworks (such as GoogleTest and GoogleMock) but is generally applicable to any C++ testing framework. The guide emphasizes the use of lower_snake_case for function names, variable names, and test identifiers (with UPPER_SNAKE_CASE reserved for constants) and prescribes a consistent, ordered structure for each test.

### 1. Target Audience

- **Primary Audience:**  
  - Large language models (LLMs) acting as coding assistants.
  - Developers writing and maintaining C++ unit tests.
- **Goal:**  
  - Ensure tests are self‑explanatory, well‑documented, and easy to maintain.
  - Promote consistency across test files and suites.

### 2. Test Function and Macro Naming

- **Descriptive Names in lower_snake_case:**  
  Test functions or test case names (using macros such as `TEST()` or `TEST_F()`) must use lower_snake_case.  
  **Example:**
  ```cpp
  TEST(math_tests, divide_by_zero_throws_exception)
  ```
- **Constants:**  
  Use UPPER_SNAKE_CASE for any constants used in tests (e.g. `EXPECTED_RESULT`).

### 3. Test File Organization

- **Single SUT per File:**  
  Each test file should cover only a single system‑under‑test (SUT) or a closely related set of functionalities.
- **Directory Structure:**  
  Place all test files in a dedicated `tests/` directory. When testing an entire SUT (for example, a class), create a folder (e.g. `tests/file_processor/`) to contain all tests for that SUT.
- **File Naming:**  
  Use lower_snake_case for test file names (e.g. `math_tests.cpp`, `file_processor_tests.cpp`).

### 4. Test Docstrings

- **Placement:**  
  Place the test’s docstring immediately above the `TEST()` or `TEST_F()` macro.
- **Format:**  
  Use single‑line comments (`//`) for the docstring. The docstring must start with a visual separator line exactly as follows:
  ```cpp
  // ------------------------------------------------------------------------------------------------------------
  ```
  After the separator, provide a human‑readable one‑liner summary that describes what the test verifies. If the one‑liner is too long, word‑wrap it as needed (without breaking too early) so that it remains clear. Then, after a blank line, list the GIVEN, WHEN, THEN components using indentation to clearly associate subsequent lines with each section.

**Example:**
```cpp
// ------------------------------------------------------------------------------------------------------------
// Verifies that dividing by zero causes an exception to be thrown.
//
// GIVEN a numerator and a zero denominator,
//       and an invalid division scenario,
// WHEN divide() is called,
// THEN std::invalid_argument is thrown.
TEST(math_tests, divide_by_zero_throws_exception) {
    // Test body...
}
```

### 5. Test Body Structure

Each test function should include the following sections in order:

- **// GIVEN**  
  Set up initial conditions and inputs. Variables in this section are prefixed with `given_`.
- **// MOCKING** (or **// MOCKING & SETUP**)  
  Create and configure mock objects if applicable. *Mock variables must be prefixed with `mock_`.* If non‑mocked objects are also created here, they have no special prefix. If extensive setup is needed, split these into separate sections.
- **// WHEN**  
  Execute the function or method under test. Store outputs in variables prefixed with `actual_`.
- **// EXPECTING**  
  Define the expected outcomes in variables prefixed with `expected_`. This section comes between WHEN and THEN.
- **// THEN**  
  Write assertions comparing actual results to expected outcomes. Always include a custom error message explaining the expected versus actual values.
- **// BEHAVIOR** (if applicable)  
  For tests using mocks, include additional assertions (for example, via `EXPECT_CALL`) to verify that the mocks were called as expected.

### 6. Test Groups

- **Test Group Naming:**  
  When multiple tests share common setup/teardown logic, group them in a test group. Name test group classes using lower_snake_case with a `_test_group` suffix.

**Example:**
```cpp
class file_processor_test_group : public ::testing::Test {
protected:
    void SetUp() override {
        given_file_path = "test_data.txt";
        file_processor = new FileProcessor(given_file_path);
    }
    
    void TearDown() override {
        delete file_processor;
    }
    
    std::string given_file_path;
    FileProcessor* file_processor;
};
```
Then use:
```cpp
TEST_F(file_processor_test_group, process_file_returns_correct_results) {
    // Test body follows the GIVEN, MOCKING, WHEN, EXPECTING, THEN order.
}
```

### 7. Mocks and Dependency Injection

- **Declaration Order:**  
  In the MOCKING (or MOCKING & SETUP) section, create mock objects first, then real objects that depend on them.

**Example:**
```cpp
// MOCKING & SETUP
Mock_dependency mock_dependency;
service service_under_test(&mock_dependency);
```
Place the MOCKING section immediately after the GIVEN section and before the WHEN section.

### 8. Assertions and Custom Messages

- **Assertion Macros:**  
  Use the assertion macros provided by your framework (e.g. `EXPECT_EQ`, `ASSERT_TRUE`).
- **Custom Messages:**  
  Always supply a custom error message that clearly explains the expected and actual values.

**Example:**
```cpp
// EXPECTING
int expected_result = 100;

// WHEN
int actual_result = compute_value();

// THEN
EXPECT_EQ(actual_result, expected_result)
    << "Expected compute_value() to return " << expected_result
    << ", but got " << actual_result;
```

### 9. Exception Testing

- **EXPECT_THROW Usage:**  
  Use `EXPECT_THROW` with inline comments to clearly indicate the WHEN and THEN parts (in that order). For clarity, you may also define an expected exception variable.

**Example:**
```cpp
EXPECT_THROW(
    // WHEN
    {
        length_in_grapheme_clusters_to_length_in_code_units(given_string, given_string_length, given_position, given_clusters_to_process);
    },
    // THEN
    invalid_arguments_logic_error
) << "Expected invalid_arguments_logic_error when starting position is negative";
```

### 10. Inline Comments and Section Dividers

- **Within Test Functions:**  
  Use simple one‑line comments (e.g. `// GIVEN`, `// WHEN`, `// EXPECTING`, `// THEN`, `// BEHAVIOR`) to separate logical sections.
- **At File Level:**  
  Outside individual test functions, you may use heavy section divider lines (e.g. using `// ------------------------------------------------------------------------------------------------------------`) with vertical space before and after to separate major sections.

### 11. File Organization for SUT Tests

- **Group by SUT:**  
  When testing all functions in a particular SUT (e.g. a class), create a dedicated folder (e.g. `tests/file_processor/`) and place all related test files in that folder.
- **Multiple Files:**  
  If the SUT is complex and requires several test files, group them in the same directory for clarity and maintainability.

### 12. General Guidelines for Unit Tests

- **Readability is Paramount:**  
  Write tests that are clear and easy to understand. Use extra whitespace, clear section labels, and descriptive variable names.
- **Review and Refactor:**  
  Even if tests are generated by automated tools or LLMs, human review is essential to ensure correctness and clarity.
- **Consistency:**  
  Adhere strictly to these guidelines to maintain a uniform test codebase.
