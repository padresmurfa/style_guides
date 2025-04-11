# 🔪 JavaScript Unit Testing Style Guide

This guide defines **best practices** for writing **structured, clear, and maintainable** JavaScript unit tests. It is designed for use with Jest (or similar frameworks like Vitest), and targets **developers and LLMs** writing or assisting with automated tests.

---

## 1. ✅ Test File Naming

- Use `test_<file_under_test>.js` or `<module>.test.js`
- Only cover a single **unit under test (UUT)** per test file.
- Place test files inside a `tests/` or `__tests__/` directory.

✅ Good:
```
tests/
├── test_authenticate.js
├── test_userService.js
```

🚫 Bad:
```
tests/
├── all_tests.js
```

---

## 2. ✅ Test Function Naming

- Use `test_<method>_<behavior>` or a natural sentence-style name.
- All test functions must start with `test_` when using manual naming, or use Jest's `test()`/`it()` with a clear description.

✅ Good:
```js
test('loginUser returns user token when credentials are valid', ...)
```

🚫 Bad:
```js
test('should work')
```

---

## 3. ✅ Test Structure & Sections

Each test is split into clearly labeled sections as **comments**, with optional docstrings for clarity.

### Standard Sections (in order):

```js
// GIVEN
// SETUP
// MOCKING
// SYSTEM UNDER TEST
// WHEN
// EXPECTATIONS
// THEN
// BEHAVIOR
```

Use this layout *even for simple tests* to ensure consistency.

✅ Example:
```js
test('loginUser returns token on success', async () => {
  // GIVEN
  const givenEmail = 'user@example.com';
  const givenPassword = 'secure123';

  // MOCKING
  const mockAuthService = {
    login: jest.fn().mockResolvedValue('token-abc')
  };

  // SYSTEM UNDER TEST
  const sut = new LoginController(mockAuthService);

  // WHEN
  const actualToken = await sut.login(givenEmail, givenPassword);

  // EXPECTATIONS
  const expectedToken = 'token-abc';

  // THEN
  expect(actualToken).toBe(expectedToken);

  // BEHAVIOR
  expect(mockAuthService.login).toHaveBeenCalledWith(givenEmail, givenPassword);
});
```

---

## 4. ✅ Variable Naming

| Purpose        | Prefix         | Example               |
|----------------|----------------|------------------------|
| Input data     | `given_`       | `givenEmail`           |
| Expected value | `expected_`    | `expectedToken`        |
| Actual result  | `actual_`      | `actualResponse`       |
| Mocks          | `mock_`        | `mockAuthService`      |
| System under test | `sut`      | `sut = new LoginController(...)` |

**Do not assert directly on literals** — always use `expected_*` and `actual_*`.

---

## 5. ✅ Grouping Tests

Use `describe()` to group tests logically by method or behavior:

```js
describe('loginUser()', () => {
  test('returns token when successful', ...)
  test('throws error for invalid password', ...)
});
```

---

## 6. ✅ Docstrings & Comments

Each test should have a leading comment or docstring summarizing what it tests. Use:

```js
// ------------------------------------------------------------------------------------------------------------
// Verifies loginUser returns a token when credentials are valid.
//
// GIVEN a valid email and password
// WHEN loginUser is called
// THEN it returns the expected token and calls AuthService.
```

Use this above the `test()` block.

---

## 7. ✅ Mocking Practices

- Use `jest.fn()` or `jest.mock()` for mocking dependencies.
- Place all mocks in the **MOCKING** section.
- Verify mock behavior in the **BEHAVIOR** section.
- Avoid mocking things you don’t need to.

---

## 8. ✅ Exception Testing

- Place **THEN** section **before** the **WHEN** section.
- Use `expect().toThrow()` or `await expect().rejects.toThrow()` for async code.

✅ Example:
```js
test('throws error for invalid password', async () => {
  // GIVEN
  const givenEmail = 'user@example.com';
  const givenPassword = 'wrongpassword';

  // MOCKING
  const mockAuthService = {
    login: jest.fn().mockRejectedValue(new Error('Invalid password'))
  };

  // SYSTEM UNDER TEST
  const sut = new LoginController(mockAuthService);

  // THEN
  await expect(sut.login(givenEmail, givenPassword)).rejects.toThrow('Invalid password');

  // BEHAVIOR
  expect(mockAuthService.login).toHaveBeenCalled();
});
```

---

## 9. ✅ Separation of Concerns

- Never combine multiple `WHEN` or `THEN` clauses in one test.
- Write one test per distinct behavior.
- Avoid parameterized tests unless clarity benefits significantly.

---

## 10. ✅ Fixtures

If using fixtures, define reusable factories:

```js
function createUserFixture(overrides = {}) {
  return {
    id: 'u123',
    name: 'Test User',
    email: 'test@example.com',
    ...overrides
  };
}
```

Use it in GIVEN:
```js
const givenUser = createUserFixture({ email: 'alt@example.com' });
```

---

## 11. ✅ Avoid Global Setup

- Do not use `beforeEach()` unless the setup is identical across all tests in the file.
- Prefer explicit **GIVEN/SETUP** inside each test function.

---

## 12. ✅ Project Structure for Tests

```
project-root/
├── src/
│   ├── login.js
│   └── ...
├── tests/
│   ├── test_login.js
│   └── ...
```

---

## 13. ✅ Assertions

Use:
```js
expect(actual_foo).toEqual(expected_foo);
```

Avoid:
```js
expect(actual_foo).toEqual('literal'); // ❌ Bad
```

Include custom messages if the framework allows:
```js
expect(actual).toBe(expected); // add message in comment or output if needed
```

---

## Summary

This guide blends the best practices from your C#, Python, and C++ conventions into a unified **JavaScript test style**. Use this for consistent, high-quality, and LLM-friendly test writing across your Vercel app or any Node.js-based JavaScript project.