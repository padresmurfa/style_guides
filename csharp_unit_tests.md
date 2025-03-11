# C# Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in C# using xUnit, NUnit, or MSTest.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Method Naming**
- Each test method **must start with** `Test_`.
- Test method names should note contain consequtive underscores.
- The method name should be **descriptive and readable**, reflecting the behavior under test.
- **Separate tests** into distinct methods rather than combining multiple test cases.
- **Avoid parameterized tests** unless they significantly improve clarity.

✅ **Good Example:**
```csharp
[Test]
public void Test_ProcessData_ReturnsCorrectValue()
```
🚫 **Bad Example:**
```csharp
[Test]
public void CheckProcessData() // ❌ Missing Test_ prefix
[Test]
public void Test_Process() // ❌ Too vague
[Test]
public void Test_MultipleCases() // ❌ Tests multiple things at once
```

---

## **2. Test Class Organization**
- **Each test class should cover a single class or method under test**.
- **When testing a complex class, each method may have its own test class**.
- This ensures that tests are **modular, easy to locate, and maintainable**.

✅ **Good Example:**
```
Tests/
│── AuthenticationTests.cs    // Covers `Authenticate()`
│── RequestProcessingTests.cs // Covers `ProcessRequest()`
│── ServiceTests.cs           // Covers multiple service methods separately
```
🚫 **Bad Example:**
```
Tests/
│── AllServicesTests.cs  // ❌ Mixed tests in one file
```

---

## **3. Test Structure**
Each test method follows a structured format, **separated by clear comments**:

### **Standard Sections:**

The following sections are defined for a structured test. Where possible, the order of sections should be as it is presented below. Empty sections should be omitted. In rare cases, it may be justifiable to merge sections, using combined names such as *MOCKING % SETUP*, but try to avoid this.

1. **GIVEN**  
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given_`.
   - when test fixtures are used, they should be allocated in this section
   - **Test Fixtures** should be named `Fixture_*` and assigned to `given_*` variables.
   - The GIVEN section should always be the first section in a test, unless there is nothing to define in GIVEN, in which case it should be omitted.

1. **SETUP**
   - perform all test initialization that prepares the non-mocked environment (e.g., creating HttpContext, configuring dependency injection, and setting up request parameters)
   - variables created in this section should be prefixed with `env_*` (e.g. env_httpContext, env_actionContext)
   - when constants or variables are needed in this section that are important for the overall comprehensibility of the test, they should be declared in variables in the GIVEN section instead
   - the SETUP section should follow the GIVEN section, unless there is nothing to setup in SETUP, in which case it should be omitted
   - the SETUP section should not refer to mock_ variables. If it needs to do so, and this cannot be solved by moving mocked setup into the MOCKING section, then consider moving the MOCKING section above the SETUP section

1. **MOCKING**
   - Define and configure **mock objects**.
   - Mock variables must be named `mock_*`.
   - **Use constructor injection for dependencies**, or use mocking frameworks like Moq.
   - The MOCKING section should follow the SETUP section, or be omitted if no mocking is required. Under special circumstances described above, it may be placed before the SETUP section.

1. **SYSTEM UNDER TEST**  
   - Assigns the system under test to a variable named `sut`
   - if multiple systems are being tested for some reason, assign them to separate`sut_*` variables.

1. **WHEN**  
   - Perform the action or method call being tested.
   - Assign the result to `actual_*` variables.
   - If the test needs to use an `env_*` in later sections, then assign them to `actual_*` variables in WHEN and use those instead.

1. **EXPECTATIONS**  
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected_*` variables.
   - The EXPECTATIONS section should be strictly placed after WHEN and before THEN
   - This section should not refer to any `actual_*` variables

1. **THEN**  
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected_*` variables.

1. **BEHAVIOR**
   - Contains assertions for **mock interactions** (e.g., `mockService.Verify(x => x.DoSomething(), Times.Once());`).
   - This section **must be present if mocks are used**.

✅ **Good Example:**
```csharp
[Test]
public void Test_ProcessData_ReturnsCorrectValue()
{
    // GIVEN
    var given_input = "sample";
    var given_service = new DataService();
    
    // WHEN
    var actual_result = given_service.ProcessData(given_input);
    var actual_sideEffect = Foo.Bar();
    
    // EXPECTATIONS
    var expected_result = "processed_sample";
    var expected_sideEffect = "fubar";
    
    // THEN
    Assert.AreEqual(expected_result, actual_result);
    Assert.AreEqual(expected_sideEffect, actual_sideEffect);
}
```

---

## **4. Mocking & Fixtures: Best Practices**
- **Prefer fixtures over mocks**- preferring to use shared, reusable fixtures rather than creating one-off test data. When using fixtures, instantiate the fixture in the GIVEN section, and modify it as needed to fit the test's needs. Fixtures help keep your tests DRY and consistent, and allows them to work on more accurate test data.
- **Never mock what you don't have to**—prefer real instances where practical.
- **Only mock things that the system-under-test uses directly**-this ensures that your test exercises the SUT properly, without falling into the trap of combinatorial execution path counts as call-depth increases. The systems that your SUT uses directly should be covered by their own direct unit tests. There is no excuse for untested code in the era of AI assisted coding.
- **Use Moq or built-in mocking frameworks** for dependency injection.
- **Assertions on mock interactions go in the BEHAVIOR section**.


✅ **Good Example:**
```csharp
[Test]
public void Test_ServiceCallsDependency()
{
    // GIVEN
    var mock_dependency = new Mock<IDependency>();
    var given_service = new MyService(mock_dependency.Object);
    
    // WHEN
    given_service.DoWork();
    
    // BEHAVIOR
    mock_dependency.Verify(x => x.PerformTask(), Times.Once());
}
```

🚫 **Bad Example:**
```csharp
[Test]
public void Test_ServiceCallsDependency()
{
    var mock_dependency = new Mock<IDependency>();
    var service = new MyService(mock_dependency.Object);
    service.DoWork();
    mock_dependency.Verify(x => x.PerformTask(), Times.Once()); // ❌ No structured comments
}
```

---

## **5. Assertions & Variable Naming**
- **Expected values** always assign input values (from the GIVEN section) to `expected_*` variables in the EXPECTATIONS section if you intend to assert on them in the THEN or BEHAVIOR sections.
- **Actual results** all actual results must be assigned to `actual_*` variables in the WHEN section.
- **Never assert against literals directly** — use `expected_*` variables.
- **Never assert against `given_*` variables directly** — assign them to`expected_*` variables in the EXPECTATIONS section and use those.
- **Never assert against `env_*` variables directly** — assign them to`actual_*` variables exclusively in the WHEN section and use those.
- **Never assert against `mock_*` variables directly in the THEN section** — assign them to`expected_*` variables in the EXPECTATIONS section and use those.
- **It is acceptable to assert against `mock_*` variables directly in the BEHAVIOR section** .
- To reiterate: **Never assert directly against given_* or env_* variable** — always use the `expected_*` or `actual_*` variables in the THEN section.
- To reiterate: **This also holds true for mocks** — always use the `expected_*` or `actual_*` variables in the BEHAVIOR section when asserting on the behavior of mock objects, which must be named `mock_*`.
- To reiterate once more: assertions should never ever ever refer to `given_*` or `env_*` values, and should only refer to `mock_*` variables in the BEHAVIOR section.

✅ **Good Example:**
```csharp
var expected_result = 42;
Assert.AreEqual(expected_result, actual_result);
```
🚫 **Bad Example:**
```csharp
Assert.AreEqual(42, actual_result); // ❌ No expected variable
```

---

## **6. Exception Handling (`Assert.Throws`)**

When using a mechanism such as Assert.Throws, which places the exception name that is being asserted against
above the code block that exercises the system under test.

i.e.
- **The THEN section should be placed before the WHEN section** when testing exceptions.
- **Use `Assert.Throws` to verify expected exceptions.**

✅ **Good Example:**
```csharp
[Test]
public void Test_DivideByZero_ThrowsException()
{
    // GIVEN
    var given_numerator = 10;
    var given_denominator = 0;

    // THEN
    Assert.Throws<DivideByZeroException>(() =>
    {
        // WHEN
        Math.Divide(given_numerator, given_denominator);
    });
}
```

---

## **7. Using `[SetUp]` Methods**
- **Avoid `[SetUp]` for simple tests.**
- Use `[SetUp]` only for **repeated initialization**.

✅ **Good Example:**
```csharp
[SetUp]
public void Setup()
{
    _service = new MyService();
}
```
🚫 **Bad Example:**
```csharp
[SetUp]
public void Setup()
{
    var service = new MyService(); // ❌ Redundant per-test instantiation
}
```

---

## **8. Organizing Tests in Namespaces**
- **Group related test classes into namespaces**.
- Ensure **each namespace matches the SUT structure**.

✅ **Good Example:**
```csharp
namespace Tests.Services
{
    [TestFixture]
    public class UserServiceTests { ... }
}
```
🚫 **Bad Example:**
```csharp
namespace Services.Tests // ❌ Inconsistent naming
```
