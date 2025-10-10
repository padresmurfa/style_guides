# Ruby Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Ruby using RSpec.

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By following these guidelines, tests will be **structured, readable, and maintainable**.

These practices may be too detailed for human-written tests but serve as an excellent guide for generating automated tests with LLM assistance. **Review all examples carefully**, as human oversight is crucial to ensure test quality.

---

## **1. Spec File Organization**
Consistent folder organization makes it immediately obvious where a spec lives relative to the production code. Misaligned structure causes confusion when navigating between source and specs, especially in editors that rely on parallel paths for discovery.
- Every spec file lives under `spec/` and mirrors the production folder layout.
- Append `_spec.rb` to every spec filename and ensure the base name matches the file or class under test (e.g., `lib/my_app/services/file_handler.rb` → `spec/my_app/services/file_handler_spec.rb`).
- Keep a **one-to-one mapping between folders and modules** so moving a file preserves the require paths.
- Avoid combining specs for unrelated classes or modules in a single file—create separate files instead.

✅ **Good Example:**
```text
lib/my_app/services/file_handler.rb
spec/my_app/services/file_handler_spec.rb
```
🚫 **Bad Example:**
```text
spec/services_spec.rb # ❌ Multiple classes in one file
```

---

## **2. Describe Blocks and Modules**
Matching `describe` declarations to the class or module under test keeps reviewers oriented and allows tooling to discover specs automatically.
- The top-level `describe` must reference the fully qualified constant under test (e.g., `describe MyApp::Services::FileHandler do`).
- When production code uses nested modules, mirror that structure exactly in the `describe` chain before adding nested contexts.
- Use `describe` for classes/modules and `context` for branching behaviour (conditions, states, inputs).
- Keep **one top-level describe block per file** to prevent interleaving unrelated classes.

✅ **Good Example:**
```ruby
RSpec.describe MyApp::Services::FileHandler do
  # ...
end
```
🚫 **Bad Example:**
```ruby
RSpec.describe 'file handler specs' do # ❌ Not tied to a constant
  # ...
end
```

---

## **3. Example (`it`) Naming**
Clear example descriptions document the behaviour under test and the expected outcome, helping maintainers understand intent without re-reading the implementation.
- Each `it` description must start with the verb `it` implicitly supplies—write in present tense (e.g., `it 'returns the processed value'`).
- Avoid vague phrases like "works" or "does stuff"; describe the branch being exercised (`'returns nil when no record exists'`).
- Prefer longer, expressive descriptions over terse ones.
- Keep **one expectation per example**; if you need to verify multiple branches, create additional examples.
- Avoid shared examples unless they remove substantial duplication and keep intent obvious.

✅ **Good Example:**
```ruby
it 'returns a processed payload when the response is successful' do
  # ...
end
```
🚫 **Bad Example:**
```ruby
it 'works' do
  # ...
end
```

---

## **4. Example Structure**
Standardized sections carve complex specs into digestible steps. Without structure, specs devolve into monolithic blocks where setup and verification intermingle.

Each example must follow the structured format, **separated by clear comments**:
- Where possible, keep the order of sections as presented below.
- Empty sections should be omitted.
- When a section becomes large or involves distinct responsibilities, split it into sub-sections with descriptive headers (e.g., `# SETUP: configure HTTP client`).
- Section comment headers should be written as full descriptive sentences (e.g., `# GIVEN: a valid account and token`).

### **Standard Sections:**

1. **GIVEN**
   - Set up initial conditions and inputs.
   - Assign variables with the prefix `given_`.
   - Test fixture helpers should be named `fixture_*` and assigned to `given_*` variables prior to use.
   - The GIVEN section should always be the first section in an example unless there is nothing to define.
   - **The GIVEN section must not be merged with any other section.**

2. **MOCKING**
   - Define and configure doubles or spies.
   - Mock variables must be named `mock_*`; fake implementations should be named `fake_*`.
   - Prefer `instance_double`/`class_double` over bare `double` to keep type contracts honest.
   - **The MOCKING section must not be merged with any other section.**

3. **SETUP**
   - Prepare the environment that is not strictly input data (e.g., configure dependency injection, initialize request objects).
   - Variables created in this section should be prefixed with `env_`.
   - Assign mocks or fakes to interface-conforming `env_` variables here (e.g., `env_logger = mock_logger`).
   - **The SETUP section must not be merged with any other section.**

4. **SYSTEM UNDER TEST**
   - Instantiate the system under test in a variable named `sut`.
   - If multiple systems are being exercised, assign them to `sut_*` variables (e.g., `sut_service`).
   - The SUT must never reference `mock_*` variables directly; use the `env_*` collaborators established in SETUP.
   - **Do not merge this section with other sections.**

5. **WHEN**
   - Perform the action being tested.
   - Assign results to `actual_*` variables.
   - If the action changes environment state that you intend to assert against later, copy the value into an `actual_*` variable in this section.
   - **Do not merge this section with other sections.**

6. **EXPECTATIONS**
   - Define expected values **before assertions**.
   - Expected values must be assigned to `expected_*` variables.
   - This section must not refer to `actual_*` variables.
   - **Do not merge this section with other sections.**

7. **THEN**
   - Perform assertions comparing actual results to expected values.
   - Use `expect(actual_value).to eq(expected_value)`; never assert directly against literals.
   - **Do not merge this section with other sections.**

8. **LOGGING** *(if applicable)*
   - Verify behaviour that can be validated via captured logs.
   - Prefer asserting on emitted logs over mock verifications when possible, using helpers that inspect recorded entries.

9. **BEHAVIOR** *(if mocks or spies are used)*
   - Assert on interactions, e.g., `expect(mock_service).to have_received(:call).with(expected_args)`.
   - This section **must be present if mocks/spies are used**.
   - Every expectation set up with `allow(...).to receive(...).and_call_original` or `expect(...).to receive(...)` must be verified here.

✅ **Good Example:**
```ruby
it 'returns a processed payload when the response is successful' do
  # GIVEN: a valid payload
  given_payload = { id: 42, status: 'ok' }

  # SYSTEM UNDER TEST: build the processor
  sut = PayloadProcessor.new

  # WHEN: processing the payload
  actual_result = sut.call(given_payload)

  # EXPECTATIONS: define the expected transformation
  expected_result = { id: 42, status: 'processed' }

  # THEN: compare actual to expected
  expect(actual_result).to eq(expected_result)
end
```
🚫 **Bad Example:**
```ruby
it 'returns a processed payload when the response is successful' do
  processor = PayloadProcessor.new
  result = processor.call({ id: 42, status: 'ok' })
  expect(result[:status]).to eq('processed') # ❌ Missing structure and literals in assertions
end
```

---

## **5. Fixtures and Helpers**
Reusable fixtures encourage realistic, DRY data that mirrors production scenarios while keeping specs focused on behaviour instead of plumbing.
- Prefer fixture helpers (`fixture_*` methods or modules) over inline literals for complex data.
- Instantiate fixtures in the GIVEN or SETUP sections and adjust them to fit the example.
- When fixture data is shared across many specs, place helpers in `spec/support/fixtures.rb` (or similar) and require them in `spec_helper.rb`.
- Avoid overusing `let`/`let!` because the indirection hides the flow; prefer explicit locals in each example. If `let` is unavoidable, name it with the same prefixes (`given_`, `env_`, etc.).

✅ **Good Example:**
```ruby
module Fixtures
  module_function

  def fixture_account
    {
      id: SecureRandom.uuid,
      status: 'active'
    }
  end
end

RSpec.describe AccountService do
  it 'returns the account status' do
    # GIVEN: an existing account fixture
    given_account = Fixtures.fixture_account

    # SYSTEM UNDER TEST
    sut = AccountService.new

    # WHEN
    actual_status = sut.status_for(given_account[:id])

    # EXPECTATIONS
    expected_status = 'active'

    # THEN
    expect(actual_status).to eq(expected_status)
  end
end
```

---

## **6. Mocking Best Practices**
Disciplined test doubles keep specs readable and trustworthy by ensuring only true collaborators are simulated and their expectations are explicit.
- **Never mock what you do not have to**—prefer fixtures or real instances where practical.
- **Only mock collaborators that the SUT interacts with directly.** Downstream dependencies should have their own specs.
- Use verifying doubles (`instance_double`, `class_double`, `spy`) instead of plain `double` to catch signature drift.
- Configure doubles in the MOCKING section and expose them via `env_` variables in SETUP before injecting into the SUT.
- Use `allow(...).to receive(...).and_return(...)` for default behaviour and `expect(...).to receive(...)` for mandatory interactions.
- Assertions on double interactions belong in the BEHAVIOR section, not in THEN.

✅ **Good Example:**
```ruby
it 'delegates notification to the mailer' do
  # MOCKING: create a verifying double
  mock_mailer = instance_double(Mailer, deliver: nil)

  # SETUP: expose the mock as an env collaborator
  env_mailer = mock_mailer

  # SYSTEM UNDER TEST
  sut = NotificationService.new(env_mailer)

  # WHEN
  sut.send_welcome('user@example.com')

  # BEHAVIOR
  expect(mock_mailer).to have_received(:deliver).with('user@example.com')
end
```
🚫 **Bad Example:**
```ruby
it 'delegates notification to the mailer' do
  mailer = double('Mailer') # ❌ Not verifying and not clearly named
  NotificationService.new(mailer).send_welcome('user@example.com')
  expect(mailer).to have_received(:deliver) # ❌ Missing setup and behaviour separation
end
```

---

## **7. Test Fakes**
Test fakes are lightweight, purpose-built implementations that can replace complex doubles when mocking becomes noisy or brittle.
- Create fakes when configuring a double would require complex stubs or multiple expectations.
- Name fake instances with the `fake_*` prefix and build them in the MOCKING section.
- Assign each `fake_*` to an `env_` variable in SETUP before handing it to the SUT.
- Provide intention-revealing assertion helpers on the fake itself and invoke them in LOGGING or BEHAVIOR.

✅ **Good Example:**
```ruby
class FakeLogger
  attr_reader :entries

  def initialize
    @entries = []
  end

  def info(message)
    entries << [:info, message]
  end

  def assert_logged(expected_level, expected_message)
    actual_entry = entries.fetch(0)
    expect(actual_entry).to eq([expected_level, expected_message])
  end
end

it 'logs the branch that was processed' do
  # MOCKING: create a fake logger
  fake_logger = FakeLogger.new

  # SETUP
  env_logger = fake_logger

  # SYSTEM UNDER TEST
  sut = BranchProcessor.new(logger: env_logger)

  # WHEN
  sut.process('foo')

  # LOGGING
  fake_logger.assert_logged(:info, 'Processed branch foo')
end
```

---

## **8. Assertions and Variable Naming**
Strict naming patterns for expected and actual values highlight the difference between inputs, outputs, and verifications.
- Assign expected values to `expected_*` variables in the EXPECTATIONS section.
- Assign actual results to `actual_*` variables in the WHEN section.
- **Never assert against literals directly**—always compare `actual_*` to `expected_*`.
- **Never assert against `given_*` or `env_*` variables directly**—copy them into `expected_*` or `actual_*` first.
- Only reference `mock_*` variables in the BEHAVIOR section (or in LOGGING when fakes expose helpers).

✅ **Good Example:**
```ruby
expected_result = 42
expect(actual_result).to eq(expected_result)
```
🚫 **Bad Example:**
```ruby
expect(actual_result).to eq(42) # ❌ Literal in assertion
```

---

## **9. Exception Handling**
Treat exception capture as part of the WHEN stage so control flow remains explicit and assertions stay separate.
- Use `actual_error = expect { ... }.to raise_error(SomeError)` to capture the exception.
- After capturing, assert on the error class or message in the THEN section using the stored `actual_error`.
- Do not assert inside the block passed to `expect { ... }`.

✅ **Good Example:**
```ruby
it 'raises when dividing by zero' do
  # GIVEN
  given_numerator = 10
  given_denominator = 0

  # WHEN
  actual_error = expect {
    MathHelper.divide(given_numerator, given_denominator)
  }.to raise_error(ZeroDivisionError)

  # EXPECTATIONS
  expected_error_class = ZeroDivisionError

  # THEN
  expect(actual_error).to be_a(expected_error_class)
end
```

---

## **10. Hooks and Shared Context**
Being intentional about `before`/`around` hooks ensures shared initialization is truly common while example-specific context stays nearby.
- **Avoid `before` hooks** when locals inside the example would keep setup clearer.
- Use hooks only for repeated initialization that is identical across every example in the group.
- When using hooks, name assigned instance variables with the same prefixes (`@given_`, `@env_`) and immediately copy them into locals at the start of each example.
- Prefer helper methods or modules over implicit shared state.

---

## **11. Breaking the Rules**
These conventions are intentionally strict to produce consistent, reviewable specs. Deviate only when following the rules would materially obscure intent, and document the rationale with an inline comment so future maintainers understand the trade-off.
- If a one-line expectation would be less clear with full section headers, you may omit intermediate sections but must still use prefixed variable names.
- When working with data-driven tests (e.g., shared examples), keep the structure within the shared example even if arguments are passed in.
- Any deviation should be rare, localized, and justified.

---

By adhering to these guidelines, Ruby specs remain expressive, deterministic, and easy to maintain—even as the codebase evolves.
