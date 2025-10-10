# Elixir Unit Testing Style Guide

This document adapts the **C# Unit Testing Style Guide** for the Elixir ecosystem and ExUnit. The goal is to provide **clear, structured, and maintainable** guidance for developers and Large Language Models (LLMs) generating Elixir unit tests.

---

## **Target Audience**

These conventions target developers and LLMs collaborating on Elixir projects. They intentionally err on the side of explicitness so that automatically generated tests stay trustworthy. Human reviewers should always validate generated tests before merging.

---

## **1. Test Module Organization**
Consistent module names help reviewers map a test module to its production counterpart without guesswork. Misaligned module names slow navigation, confuse IDE tooling, and obscure intent.
- Every test file must define a module named `<ProductionModule>Test` (e.g., `MyApp.Accounts.User` → `MyApp.Accounts.UserTest`).
- Mirror the production module's namespace exactly before appending the `Test` suffix. Nesting must match directories one-to-one.
- Avoid generic names such as `MyAppTest` or `UnitTests` that do not indicate the subject under test.

✅ **Good Example:**
```elixir
defmodule MyApp.Accounts.UserTest do
  use ExUnit.Case, async: true
end
```
🚫 **Bad Example:**
```elixir
defmodule AccountsTest do
  use ExUnit.Case
end
```

---

## **2. Test File Naming & Layout**
File names should make it trivial to locate the test for a given module and keep ExUnit's automatic discovery predictable.
- Name test files using snake_case with `_test.exs` suffix: `user_test.exs`, `user_controller_test.exs`, etc.
- Keep one public test module per file. When testing multiple sections of a module, split them into multiple files (e.g., `user_auth_test.exs`, `user_validation_test.exs`).
- Organize test files into directories that mirror the production code's directory structure (e.g., `lib/my_app/accounts/user.ex` → `test/my_app/accounts/user_test.exs`).

---

## **3. Describe Blocks & Context Grouping**
`describe` blocks make the subject under test explicit and group related scenarios. Overusing them or using vague descriptions hides coverage gaps.
- Use `describe` blocks to group tests for a specific function or public behavior. Name the block after the function, including arity: `describe "create_user/1" do`.
- Avoid nesting more than two levels of `describe`; deep nesting hurts readability. Prefer additional modules or files when context becomes complex.
- Do not leave tests floating outside of a `describe` block unless the module under test exposes a single public entry point.

✅ **Good Example:**
```elixir
describe "create_user/1" do
  test "persists a user when params are valid" do
    ...
  end
end
```
🚫 **Bad Example:**
```elixir
# Lacks a describe block and uses a vague test name
test "works" do
  ...
end
```

---

## **4. Test Naming Conventions**
Readable test names act as living documentation. They should capture the scenario, action, and expected outcome.
- Use the `test "..." do` macro with a descriptive string; prefer `"function_name does something when condition"` phrasing.
- Keep names in sentence case and avoid trailing punctuation. Use "returns", "raises", or "sends" verbs to signal outcomes.
- Include the predicate or branch under test: `"returns {:error, :not_found} when user is missing"`.
- Do not reuse redundant phrases already present in the `describe` block. Combine them to read naturally: `describe "create_user/1"; test "returns {:error, changeset} when params are invalid"`.
- Avoid referencing implementation details such as helper functions or private modules. Focus on observable behavior.

---

## **5. Test Method Structure & Sectioning**
Structured sections keep setup, execution, and assertions easy to scan. Adopt the GIVEN/SETUP/SUT/WHEN/EXPECTATIONS/THEN pattern with Elixir-friendly naming.
- Separate sections using full-sentence comments (e.g., `# GIVEN: a valid user params map`).
- Name variables using snake_case prefixes: `given_params`, `mock_repo`, `env_conn`, `sut`, `actual_result`, `expected_response`.
- When pattern matching on results, bind matches to `actual_*` variables first, then assert on them.
- Do not merge sections. If a section would be empty, omit it entirely rather than leaving a placeholder comment.
- Favor pipelines inside sections but avoid spanning multiple sections with a single pipeline; reassign to a new variable when transitioning sections.

### **Standard Sections**
1. **GIVEN** – inputs, fixtures, and constants. Assign to `given_*` variables.
2. **MOCKING** – configure mocks or fakes (`mock_*` variables). Use libraries like [Mox](https://hexdocs.pm/mox) sparingly and document expectations clearly.
3. **SETUP** – prepare the runtime environment (`env_*` variables), build connections, and assign mock implementations to interfaces.
4. **SYSTEM UNDER TEST** – assign the struct, module, or anonymous function under test to `sut` (or `sut_*` for multiple subjects).
5. **WHEN** – invoke the system under test, binding outputs to `actual_*` variables. Treat `assert_raise`/`catch_throw` wrappers as part of this section and store the exception in an `actual_exception` variable.
6. **EXPECTATIONS** – derive expected values and store in `expected_*` variables before comparing.
7. **THEN** – perform assertions using ExUnit macros (`assert`, `refute`, `assert_receive`). Never assert directly against literals; reference `expected_*`.
8. **LOGGING** – verify log output using `ExUnit.CaptureLog` when applicable.
9. **BEHAVIOR** – verify mock interactions (`Mox.verify_on_exit/1`, explicit counters) when mocks are present.

✅ **Good Example:**
```elixir
test "returns {:ok, user} when params are valid" do
  # GIVEN: valid registration params
  given_params = Fixtures.user_params()

  # SETUP: capture log messages to assert trace output
  {:ok, env_log} = ExUnit.CaptureLog.start_link()

  # SYSTEM UNDER TEST: create a function reference for readability
  sut = &Accounts.create_user/1

  # WHEN: call the subject with the params
  actual_result = sut.(given_params)

  # EXPECTATIONS: expected tuple result
  expected_result = {:ok, %User{email: given_params.email}}

  # THEN: compare actual and expected values
  assert actual_result == expected_result

  # LOGGING: ensure the branch emitted an info log
  assert capture_log(env_log) =~ "user_created"
end
```

🚫 **Bad Example:**
```elixir
test "works" do
  assert {:ok, _} = Accounts.create_user(%{})
end
```

---

## **6. Fixtures & Factories**
Shared fixtures prevent copy-paste data and make changes easy to propagate.
- Collect reusable fixtures in dedicated modules (e.g., `MyApp.Fixtures.User`). Provide helper functions like `user_params/0`, `user/1`.
- Instantiate fixtures inside the GIVEN or SETUP section. When customization is needed, start from a base fixture and override keys instead of recreating maps manually.
- Prefer factories over Mox when you only need data. Reserve mocks for behavior verification.
- Document non-trivial fixtures with moduledocs to aid discoverability.

✅ **Good Example:**
```elixir
# test/support/fixtures/user.ex
defmodule MyApp.Fixtures.User do
  def user_params(attrs \\ %{}) do
    Map.merge(%{email: "test@example.com", name: "Test"}, attrs)
  end
end
```

---

## **7. Mocking & Fakes**
Mocks should model collaborator contracts, not implementation trivia.
- Prefer explicit behaviours and [Mox](https://hexdocs.pm/mox) for dependency substitution. Avoid monkey patching modules.
- Define mocks in the `MOCKING` section and name them `mock_*`. For Mox, call `stub/3` or `expect/3` with clear arguments and return values.
- Always pair `Mox.expect/3` with `verify_on_exit!/1` in the BEHAVIOR section (or via `setup :verify_on_exit!`). Do not leave expectations implicit.
- Use fakes (hand-written modules) when they convey expectations more clearly than a mock or when you need stateful assertions.
- Assign mocks/fakes to `env_*` variables in SETUP before injecting into the SUT.

✅ **Good Example:**
```elixir
# MOCKING: expect the mailer to deliver once
mock_mailer = expect(MyApp.MailerMock, :deliver, fn _ -> :ok end)

# SETUP: inject the mock into the context
env_notifier = mock_mailer

# BEHAVIOR: verify expectations
verify_on_exit!(MyApp.MailerMock)
```

---

## **8. Assertions & Pattern Matching**
Assertions should be precise and intention revealing.
- Use `assert actual == expected` with named variables rather than literals. Pattern match assignments in the WHEN section, not inside assertions.
- Prefer `assert {:ok, value} = actual_result` to destructure success tuples, then follow with specific assertions in THEN.
- Use `assert_receive/3`, `refute_receive/3`, and `assert_called/3` (from `Mox` or custom fakes) in the BEHAVIOR section only.
- Avoid chaining multiple assertions in a single pipeline; separate them for clarity and better failure messages.
- When asserting floating point values, use `assert_in_delta/3` with a named delta variable.

---

## **9. Exception Handling**
Treat exceptions and exits as first-class outcomes.
- Wrap calls expected to raise with `assert_raise/2` or `assert_raise/3` inside the WHEN section, capturing the result in `actual_exception`.
- Compare exception types or messages in THEN using `expected_*` variables instead of literals.
- For processes that crash, use `catch_exit` or `Process.flag(:trap_exit, true)` in SETUP and assert on captured exits in THEN.

✅ **Good Example:**
```elixir
# WHEN: capture the exception raised by the SUT
actual_exception = assert_raise Ecto.NoResultsError, fn ->
  Accounts.get_user!(given_id)
end

# EXPECTATIONS
expected_exception = Ecto.NoResultsError

# THEN
assert actual_exception.__struct__ == expected_exception
```

---

## **10. Setup & Teardown**
`setup` callbacks reduce repetition but can hide critical details if overused.
- Prefer local fixtures inside tests. Use `setup` only for truly shared initialization that is stable across every test in the `describe` block.
- Return a map with keys named after the section prefixes (`given_*`, `env_*`) so pattern matching stays consistent.
- Avoid mutating state in `setup`. Use `setup_all` only for expensive, read-only fixtures.
- Always document side effects in setup callbacks with comments describing why they are safe for every test in the group.

✅ **Good Example:**
```elixir
describe "create_user/1" do
  setup :verify_on_exit!

  setup do
    {:ok, given_params: Fixtures.user_params()}
  end

  test "returns {:ok, user}", %{given_params: given_params} do
    ...
  end
end
```

---

## **11. Async Tests & Concurrency**
ExUnit's async mode speeds up suites but introduces shared state hazards.
- Default to `use ExUnit.Case, async: true` for pure modules. Opt out (`async: false`) when tests touch the database without sandboxing, rely on shared ETS tables, or mutate global process state.
- Prefer sandboxed repositories (Ecto SQL Sandbox) and supervised processes scoped per test to keep async safe.
- Use unique resource names per test (e.g., via `System.unique_integer/0`) to avoid collisions.

---

## **12. Doctests & Integration with ExUnit**
Doctests ensure documentation stays accurate but should complement, not replace, unit tests.
- Include `doctest MyApp.Module` at the top of the test module when the module exposes simple, composable functions.
- Keep doctests focused on straightforward success paths; reserve edge cases and failure modes for explicit `test` blocks.
- When doctests require setup helpers, expose them as public helper functions rather than relying on module attributes or cross-test state.

---

## **13. Breaking the Rules**
These conventions prioritize clarity, explicit data flow, and reliable verification. Deviate only when a rule makes the test harder to read or maintain. If you must break a rule, leave a comment explaining why—future reviewers (and LLMs) will thank you.

