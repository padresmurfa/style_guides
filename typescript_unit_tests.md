# ⚙️ TypeScript Unit Testing Style Guide

This guide defines **best practices** for writing clear, structured, and maintainable **unit tests in TypeScript**, particularly when using **Jest** (or frameworks like Vitest). It aligns with the conventions from your C++, C#, and JavaScript guides.

---

## 1. 🎯 Target Audience

- Developers and Large Language Models (LLMs) writing or maintaining unit tests.
- Style emphasizes **readability**, **strict structure**, and **self-explanatory naming**.

---

## 2. ✅ Test File Naming

- Use `test_<module>.ts` or `<module>.test.ts`
- Place all tests under a top-level `tests/` or `__tests__/` directory.
- Each file should cover a single **system-under-test (SUT)**.

✅ Good:
```
tests/
├── test_authentication_service.ts
├── test_project_controller.ts
```

🚫 Bad:
```
tests/
├── everything.test.ts  // ❌ Too broad
```

---

## 3. 🥪 Test Function Naming

- Use `test_<function>_<behavior>` for `function`-style tests.
- Prefer descriptive sentence-style names in `test()` or `it()` blocks.
- All test names must express intent clearly.

✅ Good:
```ts
test('authenticateUser returns valid token for correct credentials', () => { ... });
```

🚫 Bad:
```ts
test('should work', () => { ... });
```

---

## 4. 🧱 Test Structure & Sections

Every test must follow a strict sectioned structure, separated with **comments**:

```ts
// GIVEN
// SETUP
// MOCKING
// SYSTEM UNDER TEST
// WHEN
// EXPECTATIONS
// THEN
// BEHAVIOR
```

Even simple tests must use this format to ensure consistency across the codebase.

---

## 5. 🏏 Variable Naming

| Purpose        | Prefix         | Example                   |
|----------------|----------------|----------------------------|
| Input data     | `given_`       | `given_email`             |
| Setup items    | `env_`         | `env_config`, `env_mock`  |
| Mocks          | `mock_`        | `mock_http_service`       |
| SUT instance   | `sut`          | `sut = new AuthService()` |
| Actual result  | `actual_`      | `actual_response`         |
| Expected value | `expected_`    | `expected_status_code`    |

✅ All assertions must compare `actual_` to `expected_`—**never assert against literals or `given_` variables directly**.

---

## 6. 🥪 Example Test Structure

```ts
test('authenticateUser returns valid token on success', async () => {
  // GIVEN
  const given_email = 'user@example.com';
  const given_password = 'securepassword';

  // MOCKING
  const mock_auth_client = {
    login: jest.fn().mockResolvedValue({ token: 'abc123' })
  };

  // SYSTEM UNDER TEST
  const sut = new AuthService(mock_auth_client);

  // WHEN
  const actual_token = await sut.authenticateUser(given_email, given_password);

  // EXPECTATIONS
  const expected_token = 'abc123';

  // THEN
  expect(actual_token).toBe(expected_token);

  // BEHAVIOR
  expect(mock_auth_client.login).toHaveBeenCalledWith(given_email, given_password);
});
```

---

## 7. 🥪 Exception Testing

- The **THEN** section must come **before** the **WHEN** section.
- Use `await expect(...).rejects.toThrow(...)` for async functions.

✅ Example:
```ts
test('throws error when credentials are invalid', async () => {
  // GIVEN
  const given_email = 'user@example.com';
  const given_password = 'wrongpass';

  // MOCKING
  const mock_auth_client = {
    login: jest.fn().mockRejectedValue(new Error('Invalid credentials'))
  };

  // SYSTEM UNDER TEST
  const sut = new AuthService(mock_auth_client);

  // THEN
  await expect(
    // WHEN
    sut.authenticateUser(given_email, given_password)
  ).rejects.toThrow('Invalid credentials');

  // BEHAVIOR
  expect(mock_auth_client.login).toHaveBeenCalledWith(given_email, given_password);
});
```

---

## 8. 🥪 Test Grouping with `describe()`

Group tests using `describe()` blocks, with the group named after the method or scenario under test.

✅ Example:
```ts
describe('authenticateUser()', () => {
  test('returns token on success', ...)
  test('throws on bad credentials', ...)
});
```

---

## 9. 🔁 Reusable Fixtures

- Define fixtures as factory functions and reuse them across tests.
- Fixtures should live in a `fixtures/` directory if shared.

✅ Example:
```ts
function createUser(overrides: Partial<User> = {}): User {
  return {
    id: 'user-123',
    email: 'default@example.com',
    password: 'hashed',
    ...overrides
  };
}
```

---

## 10. ❌ Avoid Global Setup

- Avoid `beforeEach()` unless identical across all tests.
- Prefer inlined **GIVEN / SETUP** in each test.

---

## 11. 💬 Docstrings & Comments

Above every test, include a **visual separator** and a short **docstring** explaining the test:

```ts
// ------------------------------------------------------------------------------------------------------------
// Verifies that authenticateUser returns a valid token when credentials are correct.
//
// GIVEN valid email and password,
// WHEN authenticateUser is called,
// THEN the expected token is returned and the login method is called.
```

---

## 12. ✅ Custom Assertion Messages

Use Jest's default error output or add a comment next to the assertion for context.

✅ Example:
```ts
expect(actual_token).toBe(expected_token); // should match the token from mock
```

---

## 13. 🥪 BEHAVIOR Assertions for Mocks

- Always verify mock interactions separately in the **BEHAVIOR** section.
- Never mix them into **THEN**.

---

## 14. 📁 Project Layout

```
project/
├── src/
│   ├── auth_service.ts
│   └── ...
├── tests/
    ├── test_auth_service.ts
    └── fixtures/
        └── users.ts
```

---

## 15. 🔒 Summary

| Guideline              | Rule                                                                 |
|------------------------|----------------------------------------------------------------------|
| Naming                 | Use `given_`, `actual_`, `expected_`, `mock_`, `sut` prefixes         |
| Structure              | Always follow `GIVEN → MOCKING → SUT → WHEN → EXPECTATIONS → THEN → BEHAVIOR` |
| Literals               | Never assert directly on literals                                     |
| Comments               | Use heavy dividers + structured docstrings                            |
| Behavior checks        | Always separate mock assertions in BEHAVIOR section                   |
| Exception test order   | Place THEN before WHEN                                                |

