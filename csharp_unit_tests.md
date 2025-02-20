# C# Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in C# using xUnit, NUnit, or MSTest.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all test functions carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Test Method Naming**
- Each test method **must start with** `Test_`.
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
1. **GIVEN**  
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given_`.
   - **Test Fixtures** should be named `Fixture_*` and assigned to `given_*` variables.

2. **MOCKING** *(if applicable)*  
   - Define and configure **mock objects**.
   - Mock variables must be named `mock_*`.
   - **Use constructor injection for dependencies**, or use mocking frameworks like Moq.

3. **WHEN**  
   - Perform the action or method call being tested.
   - Assign the result to `actual_*` variables.

4. **EXPECTATIONS**  
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected_*` variables.

5. **THEN**  
   - Perform **assertions** comparing actual results to expected values.
   - **Never assert against literals**—always use `expected_*` variables.

6. **BEHAVIOR** *(if applicable)*  
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
    
    // EXPECTATIONS
    var expected_result = "processed_sample";
    
    // THEN
    Assert.AreEqual(expected_result, actual_result);
}
```

---

## **4. Mocking Best Practices**
- **Use Moq or built-in mocking frameworks** for dependency injection.
- **Never mock what you don't own**—prefer real instances where practical.
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
- **Expected values** must be stored in `expected_*` variables **before** assertions.
- **Actual results** must be stored in `actual_*` variables.
- **Never assert against literals directly**—use `expected_*` variables.

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
