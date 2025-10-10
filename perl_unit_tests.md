# Perl Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Perl using the Test
2 ecosystem (`Test::More`, `Test::Deep`, `Test::MockModule`, etc.).

---

## **Target Audience**

The primary target audience for this document is developers and Large Language Models (LLMs) acting as coding assistants. By
following these guidelines, test suites stay **predictable, reviewable, and friendly to continuous integration**.

These practices lean toward automation-friendly rigor. Humans may choose to relax specific rules, but LLM-generated code **must**
follow them precisely so reviewers can immediately validate intent and coverage.

---

## **1. Test File Organization**
Consistent test file layouts make it obvious where each scenario lives. Perl test harnesses (`prove`, `dzil test`, `ExtUtils::Makemaker`
`make test`) rely on `.t` files in the `t/` directory, so keep files predictable.
- Place every unit test script under `t/unit/` (or the repository's agreed test root) using folders that mirror the production
  module's namespace. For example, `lib/My/App/Service.pm` maps to `t/unit/My/App/Service.t`.
- If a module exposes multiple complex functions or roles, split them into focused files such as `t/unit/My/App/Service/process_request.t`.
- Avoid mixing unrelated modules in a single `.t` file. One file per production module (or per complex function) keeps failure
  triage straightforward.
- Name helper libraries under `t/lib/` with the same package names as production code but suffixed with `::TestHelpers` to make
their intent obvious.

✅ **Good Example:**
```
lib/
│── My/App/Service.pm
│
t/
│── unit/
│   │── My/App/Service.t
│   │── My/App/Service/process_request.t
│
│── lib/
    │── My/App/Service/TestHelpers.pm
```
🚫 **Bad Example:**
```
t/
│── all_tests.t   # ❌ Mixed modules and behaviors in one file
```

---

## **2. Test Package Declaration and Imports**
Explicit packages and imports make dependencies obvious to reviewers and tools. Skipping them encourages implicit globals that
are hard to reason about.
- Each `.t` file must start with:
  - `use strict;`
  - `use warnings;`
  - A single package declaration that mirrors the target module with `::Tests` appended (e.g., `package My::App::Service::Tests;`).
- Load `Test2::V0` (preferred) or `Test::More` with an explicit test plan (using `done_testing` when the number of assertions is
dynamic).
- Import the production module using its full package name (e.g., `use My::App::Service qw(process_request);`).
- Load helper modules (`Test::MockModule`, `Test::Deep`, `Test::Warnings`, etc.) at the top-level so readers see every tool in use.

✅ **Good Example:**
```perl
package My::App::Service::Tests;

use strict;
use warnings;

use Test2::V0;
use My::App::Service qw(process_request);
use Test::MockModule;
```
🚫 **Bad Example:**
```perl
use Test::More tests => 3;    # ❌ Missing strict/warnings and package alignment
use lib 'lib';                 # ❌ Implicit namespace relationship
```

---

## **3. Test Subroutine Naming and Files**
Perl tests typically live at the script level. Encapsulating each scenario in a subroutine keeps structure manageable when LLMs
or humans need to add new coverage.
- Wrap every scenario in a named `sub test_<Behavior>` and call it from the bottom of the file, maintaining the call order.
- Subroutine names must start with `test_` and use snake_case segments to describe the behavior (e.g., `test_process_request_returns_success`).
- Use a single `.t` file per test package; do not define multiple packages in one file.
- Keep helper subs outside the `test_` prefix (e.g., `_build_request_fixture`).

✅ **Good Example:**
```perl
sub test_process_request_returns_success {
    ...
}

subtest 'process_request returns success' => \&test_process_request_returns_success;
```
🚫 **Bad Example:**
```perl
test_process_request();  # ❌ Not encapsulated; unclear behavior description
```

---

## **4. Test Method Sectioning**
Standard sections ensure each test reads like a narrative. Align the comment headers and variable naming with the structure used
in other language guides so cross-language teams can review quickly.

Each test subroutine must follow the sections below, separated by `#` comments written as **full sentences**:

1. **GIVEN**
   - Establish inputs, fixtures, and data prerequisites.
   - Name scalars `my $given_input`, arrays `my @given_items`, hashes `my %given_options`.
   - Build reusable fixtures via helper subs stored under `t/lib` where feasible.

2. **MOCKING** *(if applicable)*
   - Set up mocks using `Test::MockModule`, `Test::MockObject`, or `Test2::Tools::Mock`.
   - Name variables `my $mock_logger`, `my $mock_http_client`, etc.
   - Configure behavior before moving on to SETUP.

3. **SETUP**
   - Prepare environment scaffolding that is not strictly input data (e.g., instantiate the SUT, wire dependencies, seed global
     state).
   - Use the prefix `$env_*`, `@env_*`, `%env_*` for environment variables. Assign mocks to these environment variables before
     handing them to the system under test.

4. **SYSTEM UNDER TEST**
   - Instantiate or select the SUT in a variable named `$sut`. Multiple systems require `$sut_primary`, `$sut_secondary`, etc.
   - Do not pass mock objects directly; always inject the `$env_*` variables.

5. **WHEN**
   - Execute the behavior under test. Store results in `$actual_*` variables (e.g., `$actual_response`).
   - Capture exceptions using `dies { ... }` or `exception { ... }` and assign them to `$actual_exception`.

6. **EXPECTATIONS**
   - Declare expected values with `$expected_*`, `@expected_*`, `%expected_*`. Compute them before assertions.

7. **THEN**
   - Use assertion functions (`is`, `like`, `cmp_deeply`, etc.) comparing `$actual_*` to `$expected_*`.
   - Never assert directly against literals or `given`/`env` variables.

8. **LOGGING** *(if applicable)*
   - Verify captured log output. Prefer fakes that record messages over complex regex assertions.

9. **BEHAVIOR** *(if applicable)*
   - Verify interactions with mocks (`$mock_logger->called_ok(...)`). Every expectation registered in MOCKING must be verified here.

Empty sections may be omitted, but the order above must be preserved for sections that appear.

✅ **Good Example:**
```perl
sub test_process_request_returns_success {
    # GIVEN: a valid request and service configuration
    my $given_request = fixture_request();

    # SETUP: initialize a service instance
    my $env_config = fixture_config();

    # SYSTEM UNDER TEST: create the service
    my $sut = My::App::Service->new( config => $env_config );

    # WHEN: processing the request
    my $actual_response = $sut->process_request($given_request);

    # EXPECTATIONS: determine the expected payload
    my $expected_status = 'success';

    # THEN: the response should include the success status
    is( $actual_response->{status}, $expected_status, 'returns success' );
}
```
🚫 **Bad Example:**
```perl
my $service = My::App::Service->new();
my $res = $service->process_request({});
ok( $res->{status} eq 'success' );   # ❌ Missing structure and expected variable
```

---

## **5. Documentation and POD within Tests**
Short narratives improve readability when tests are complex.
- Precede each `subtest` with a brief documentation block or inline comment summarizing the scenario.
- Use POD (`=head2`) sparingly for large files where navigation is difficult; otherwise prefer comment headers.
- Ensure the comments match the GIVEN/WHEN/THEN story so future readers trust the description.

---

## **6. Fixtures: Best Practices**
Reusable fixtures keep data consistent across tests and reduce duplication.
- Store fixture builders in `t/lib/.../Fixtures.pm` modules exposing functions like `fixture_request()` or `fixture_customer()`.
- Call fixture subs in the GIVEN section and assign the result to `$given_*` variables.
- When a fixture requires per-test customization, clone it first (e.g., `my %given_request = %{ fixture_request() };`).
- Avoid editing shared fixture state in place; copy data to keep tests isolated.

✅ **Good Example:**
```perl
package My::App::Fixtures;
use Exporter 'import';
our @EXPORT_OK = qw(fixture_request fixture_config);

sub fixture_request {
    return {
        id      => 'REQ-123',
        payload => { user_id => 'user-42' },
    };
}

sub fixture_config {
    return {
        retries => 3,
        logger  => undef,
    };
}
```

```perl
# GIVEN: a default request and configuration
my $given_request = My::App::Fixtures::fixture_request();
my $env_config    = My::App::Fixtures::fixture_config();
```
🚫 **Bad Example:**
```perl
# GIVEN
my $given_request = { id => 'REQ-123', payload => { user_id => 'user-42' } }; # ❌ Inline literals duplicated in many tests
```

---

## **7. Mocking and Fakes**
Deliberate mocking policies prevent brittle tests.
- Prefer real collaborators or fixtures when behavior is simple. Mock only direct dependencies of the SUT.
- Use `Test::MockModule` or `Test2::Tools::Mock` to replace subroutines. Always localize changes (`$mock->redefine(...)`) and
  restore them in scope (`$mock->unmock_all` in an END block if necessary).
- Name mock instances `$mock_*` and assign them to `$env_*` variables before injecting into the SUT.
- Verify mock expectations in the BEHAVIOR section using `called_ok`, `call_count_is`, or custom helper methods.
- For complex interactions, write fakes (small packages inside `t/lib`) that record behavior and provide assertion helpers.
- Fakes must expose verification methods (e.g., `$fake_logger->assert_info_logged(...)`) called during BEHAVIOR.

✅ **Good Example:**
```perl
# MOCKING: capture log calls with a fake logger
my $mock_logger = Test::MockModule->new('My::App::Logger');
$mock_logger->redefine( info => sub { push @captured_logs, \@_ } );

# SETUP: expose logger via env variable
my $env_logger = My::App::Logger->new();

# SYSTEM UNDER TEST
my $sut = My::App::Service->new( logger => $env_logger );

# WHEN
$sut->process_request($given_request);

# BEHAVIOR
is( scalar @captured_logs, 1, 'logger called once' );
```
🚫 **Bad Example:**
```perl
My::App::Logger::info = sub { ... }; # ❌ Global override with no scope or verification
```

---

## **8. Assertions & Variable Naming**
Naming conventions signal intent and keep assertions readable.
- Always compare `$actual_*` against `$expected_*` using `is`, `is_deeply`, or `cmp_deeply`.
- Avoid inline literals in assertions unless the literal *is* the expected variable (`my $expected_status = 'success';`).
- When verifying complex structures, prefer `cmp_deeply` with `hash(..., array_each(...))` to highlight intent.
- Use descriptive test messages explaining the expectation; they appear in TAP output and debugging sessions.

✅ **Good Example:**
```perl
my $expected_status = 'success';
is( $actual_status, $expected_status, 'returns success status' );
```
🚫 **Bad Example:**
```perl
ok( $actual_status eq 'success', 'returns success status' ); # ❌ Inline literal and weak diagnostic
```

---

## **9. Exception Handling**
Handle exceptions explicitly so the WHEN/THEN structure stays obvious.
- Use `exception { ... }` from `Test2::V0` or `dies_ok { ... }` from `Test::More` to capture exceptions.
- Assign the result to `$actual_exception` during WHEN, then assert on its type or message in THEN.
- Never combine the call and assertion in a single statement; reviewers need a dedicated EXPECTATIONS/THEN section.

✅ **Good Example:**
```perl
# WHEN: invoking the dangerous operation
my $actual_exception = exception { $sut->divide($given_numerator, $given_denominator) };

# EXPECTATIONS: expect a divide-by-zero error class
my $expected_message = qr/denominator must not be zero/;

# THEN: verify the error message
like( $actual_exception, $expected_message, 'fails with divide-by-zero error' );
```
🚫 **Bad Example:**
```perl
dies_ok { $sut->divide(1, 0) };   # ❌ No stored exception or explicit expectation
```

---

## **10. Shared Setup (`BEGIN`, `END`, `INIT`)**
Perl provides many global hooks; use them sparingly to avoid hidden coupling.
- Avoid `BEGIN`/`INIT` blocks for test setup. Prefer explicit GIVEN/SETUP code inside each test.
- Use `END` blocks only to clean up global state modified in the test file (e.g., removing temporary directories).
- When repeated setup is unavoidable, extract helper subs or modules instead of relying on global hooks.

---

## **11. Writing Subtests**
`subtest` can organize related scenarios but must not hide the GIVEN/WHEN/THEN structure.
- Wrap each `test_` subroutine in a `subtest` call from the main script. The subroutine body still uses the section headers.
- Never nest more than one level of `subtest`; deeply nested TAP output is difficult to debug.
- Provide descriptive subtest names that mirror the subroutine name.

✅ **Good Example:**
```perl
subtest 'process_request returns success' => \&test_process_request_returns_success;
```
🚫 **Bad Example:**
```perl
subtest 'process request' => sub {
    ok(...);            # ❌ Missing structured sections inside
};
```

---

## **12. Breaking the Rules**
Deviations should be rare and must be documented inline.
- If a test cannot follow the section order (e.g., because the SUT is a compile-time import), add a comment explaining why.
- Any intentional guideline violation must include `# WHY: ...` immediately above the deviation.
- Pull requests that skip these annotations will be rejected during review.

---

## **13. Quick Checklist**
Before finalizing a test, confirm the following:
- [ ] File lives under `t/unit/...` and mirrors the module path.
- [ ] Package name matches the module plus `::Tests`.
- [ ] `use strict;` and `use warnings;` are present.
- [ ] Each scenario lives in a `test_` subroutine called via `subtest`.
- [ ] Sections follow GIVEN → MOCKING → SETUP → SUT → WHEN → EXPECTATIONS → THEN → LOGGING → BEHAVIOR.
- [ ] Variables use `given_`, `env_`, `actual_`, `expected_`, `mock_`, `sut` prefixes appropriately.
- [ ] Assertions compare expected vs. actual variables with meaningful messages.
- [ ] Mocks and fakes are verified in BEHAVIOR.
- [ ] Exceptions are captured and asserted explicitly.
- [ ] Any rule break is annotated with `# WHY:`.

Adhering to these conventions keeps Perl unit tests predictable, review-friendly, and ready for automation.
