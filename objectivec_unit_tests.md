# Objective-C Unit Testing Style Guide

This document defines **best practices** for writing **clear, structured, and maintainable** unit tests in Objective-C using **XCTest** together with helpers like **OCMock**, custom fakes, and fixture factories.

---

## **Target Audience**

The primary audience is developers and Large Language Models (LLMs) that author Objective-C unit tests. Following these conventions yields suites that are **predictable, review-friendly, and easy to maintain**. Human review is still essential—always read generated tests closely before relying on them.

---

## **1. Test Module Organization**
Objective-C projects often mirror production module and folder structures inside the test bundle. Keeping the test hierarchy aligned with the production target allows reviewers to navigate quickly and makes Xcode's auto-generated group mapping accurate.
- Create a **test bundle** that mirrors the production target name with a `Tests` suffix (e.g. `MyApp` → `MyAppTests`).
- Inside the bundle, mirror the production folder structure. If production code lives under `Sources/Networking`, create `Tests/Networking`.
- The root umbrella header for a module (if used) should import the corresponding production header and append `Tests` (e.g. `#import "MyAppTests-Swift.h"`).
- Avoid placing unrelated test files into catch-all groups such as "Misc" or "Utilities"—instead, organize them by production module.

✅ **Good Example:**
```
MyApp/
└── Sources/
    └── Services/
        └── UserService.h/.m
MyAppTests/
└── Services/
    └── UserServiceTests.m
```
🚫 **Bad Example:**
```
MyAppTests/
└── Misc/
    └── test_user.m   // ❌ No structural relationship to production folder
```

---

## **2. Test Class Naming and Files**
Explicit class names make it obvious which production types each test covers. XCTest discovers test classes via `@interface` declarations, so naming inconsistencies slow down reviews and confuse tooling.
- Each test file must declare a **single Objective-C test class** that inherits from `XCTestCase`.
- Test class names **must end with** `Tests` (e.g. `UserServiceTests`).
- If the class covers a single Objective-C class, name it `<ClassName>Tests`.
- If the class targets a specific category, selector, or region, append those segments in PascalCase (e.g. `UserServiceLoginTests`).
- Keep one logical subject per file. If you need a second focus area, create another test file and class.
- File names must match the class name exactly, including suffixes (e.g. `UserServiceLoginTests.m`).
- Do not include underscores or spaces in test class names.

✅ **Good Example:**
```objectivec
@interface UserServiceTests : XCTestCase
@end
```
🚫 **Bad Example:**
```objectivec
@interface UserService_Test : XCTestCase // ❌ Underscore in class name
@end
```

---

## **3. Region-Focused Test Classes**
Objective-C projects frequently group related methods in categories (`UserService+Authentication.h`). When production code uses categories or pragma marks to separate responsibilities, mirror that organization to keep the test suite navigable.
- Introduce `<CategoryName>` or `<PragmaMarkName>` segments only when the production code exposes the same PascalCase identifier.
- Create a dedicated test class per production region or selector group, keeping each in its own file.
- Avoid lumping unrelated regions together simply because they share dependencies—favor narrow focus for clarity.
- If helper methods exist solely to support a public API, test that public API rather than the helpers directly.

✅ **Good Example:**
```
Sources/
│── UserService.h/.m           // Contains "#pragma mark - Authentication"
Tests/
│── UserServiceAuthenticationTests.m
```
🚫 **Bad Example:**
```
Tests/
│── UserServiceAuthAndProfileTests.m // ❌ Two pragma mark regions in one class
```

---

## **4. Test Method Naming**
XCTest discovers tests whose instance methods begin with `test`. Consistent, descriptive names function as documentation for the behavior under test.
- Every test method must begin with `- (void)test` and use PascalCase after the prefix (e.g. `- (void)testFetchUsers_WhenCacheIsEmpty_ReturnsServerResult`).
- Use underscores to separate major phrases **only if** it improves readability; do not use consecutive underscores.
- If the test class covers multiple production selectors, include the selector name immediately after `test` (e.g. `testLogin_WhenPasswordInvalid_ShowsError`).
- When a test class covers a single selector, omit the selector name and describe the scenario directly (e.g. `testWhenPasswordInvalid_ShowsError`).
- Prefer verbose method names that communicate behavior, branch, and outcome. Avoid vague names like `testWorks`.
- Each behavior should have its own test method—do not combine multiple execution paths.
- Avoid parameterized helper tests; XCTest's data-driven APIs (e.g. `XCTContextRunActivity`) are acceptable only when they clarify intent.
- Do not repeat class-level focus words in method names. If the class is `UserServiceLoginTests`, omit `Login` from the method names unless necessary for clarity.

✅ **Good Example:**
```objectivec
- (void)testLogin_WhenPasswordInvalid_ShowsError
{
    // ...
}
```
🚫 **Bad Example:**
```objectivec
- (void)testLogin
{
    // ❌ Too vague
}
```

---

## **5. Test Method Sectioning**
Structured sections make Objective-C tests self-explanatory, especially when mixing Swift bridging headers, dependency injection, or asynchronous APIs. Use full-sentence comments to signal the role of each block. Empty sections may be omitted.

Each test method follows the canonical order below:
1. **GIVEN**
   - Declare initial state and inputs. Variables in this section must use the `given` prefix (`NSString *givenUsername`).
   - Fixture classes should be named `Fixture...` and assigned to `given` variables before use.
   - Do not merge GIVEN with other sections.
2. **MOCKING**
   - Create and configure mock or fake objects. Name them `mock...` or `fake...`.
   - Prefer constructor injection or dependency passing during object creation.
   - Place MOCKING before SETUP unless a clear exception is documented.
   - Do not merge this section with others. Split into subheaders when configuration is complex (e.g. `// MOCKING: Configure network client stub`).
3. **SETUP**
   - Prepare environment objects that will be supplied to the system under test (SUT). Name them with the `env` prefix (`id<Logger> envLogger`).
   - When constants are important for comprehension, define them in GIVEN and reference them here.
   - Assign mocks/fakes to `env` variables in this section before injecting them into the SUT.
   - Do not merge SETUP with other sections; use subheaders for multi-step setup.
4. **SYSTEM UNDER TEST**
   - Instantiate the object under test, assigning it to `sut` (or `sutSomething` for multiple instances).
   - Never refer to `mock*` variables directly in this section—use the `env` wrappers from SETUP.
5. **WHEN**
   - Execute the behavior under test. Store results in `actual` variables (`NSError *actualError`).
   - If you need environment values later, capture them into `actual` variables here.
   - Do not merge WHEN with other sections.
6. **EXPECTATIONS**
   - Compute expected values and assign them to `expected` variables (`NSString *expectedMessage`).
   - This section appears after WHEN and before THEN, and must not reference `actual` variables.
7. **THEN**
   - Perform assertions comparing `expected*` and `actual*` values using XCTest macros.
   - Never assert against literals directly; always use `expected*` variables.
8. **LOGGING**
   - Verify log output captured through fakes or observers. Prefer log assertions over elaborate mock verifications when possible.
9. **BEHAVIOR**
   - Verify interactions with mocks (`OCMock` expectations, fake assertions). Required whenever mocks are used.

✅ **Good Example:**
```objectivec
- (void)testLoadProfile_WhenNetworkSucceeds_ReturnsDecodedUser
{
    // GIVEN: A valid user identifier
    NSString *givenUserID = @"123";

    // MOCKING: Provide a fake network client response
    id mockClient = OCMClassMock([UserClient class]);
    OCMStub([mockClient fetchUserWithID:givenUserID completion:[OCMArg invokeBlockWithArgs:[self fixtureUserJSON], [NSNull null], nil]]);

    // SETUP: Promote mocks into environment dependencies
    UserClient *envClient = mockClient;

    // SYSTEM UNDER TEST: Create the user service
    UserService *sut = [[UserService alloc] initWithClient:envClient];

    // WHEN: Load the profile
    XCTestExpectation *actualExpectation = [self expectationWithDescription:@"Fetch completes"];
    __block User *actualUser = nil;
    [sut loadProfile:givenUserID completion:^(User *user, NSError *error) {
        actualUser = user;
        [actualExpectation fulfill];
    }];
    [self waitForExpectations:@[actualExpectation] timeout:1.0];

    // EXPECTATIONS: Define expected decoded user
    User *expectedUser = [self fixtureUser];

    // THEN: Assert the decoded user matches expectations
    XCTAssertEqualObjects(expectedUser.userID, actualUser.userID);

    // BEHAVIOR: Ensure client was invoked once
    OCMVerify([mockClient fetchUserWithID:givenUserID completion:[OCMArg any]]);
}
```

🚫 **Bad Example:**
```objectivec
- (void)testLoadProfile
{
    id client = OCMClassMock([UserClient class]); // ❌ No sectioning
    UserService *service = [[UserService alloc] initWithClient:client]; // ❌ Direct mock usage
    // ...
}
```

---

## **6. Fixtures: Best Practices**
Reusable fixtures keep Objective-C tests realistic and reduce duplication, especially when dealing with model objects or JSON fixtures.
- Prefer fixtures over inline literals. Store reusable objects or data in a shared `Fixtures` helper class or category.
- Instantiate fixtures in the GIVEN or SETUP sections and adjust them for the scenario.
- Expose fixture factories via class methods for clarity (`+ (User *)userFixture`).
- Keep fixture helpers in the test target to avoid polluting production headers.

✅ **Good Example:**
```objectivec
@interface Fixtures : NSObject
@end

@implementation Fixtures

+ (User *)userFixture
{
    User *user = [[User alloc] initWithID:@"123" name:@"Lee"];
    return user;
}

+ (NSDictionary *)userJSONFixture
{
    return @{ @"id": @"123", @"name": @"Lee" };
}

@end

- (void)testLoadProfile_WhenUserMissing_ReturnsNil
{
    // GIVEN
    User *givenMissingUser = nil;
    NSDictionary *givenResponse = @{};

    // SYSTEM UNDER TEST
    UserService *sut = [[UserService alloc] init];

    // WHEN
    User *actualUser = [sut decodeUserFromResponse:givenResponse];

    // EXPECTATIONS
    User *expectedUser = givenMissingUser;

    // THEN
    XCTAssertEqualObjects(expectedUser, actualUser);
}
```
🚫 **Bad Example:**
```objectivec
- (void)testLoadProfile_WhenUserMissing_ReturnsNil
{
    // GIVEN
    NSDictionary *givenResponse = @{ @"id": [NSNull null], @"name": @"" }; // ❌ Inline magic values
    // ...
}
```

---

## **7. Mocking: Best Practices**
Thoughtful mocking keeps Objective-C tests deterministic without hiding implementation details.
- Do not mock what you can replace with a fixture or simple fake object.
- Only mock collaborators directly used by the SUT. Dependencies should have their own focused tests.
- Favor `OCMock` or lightweight fake objects for dependency injection.
- Assertions about mocks belong in the BEHAVIOR section.
- Never pass `mock*` objects directly into the SUT—wrap them in `env*` variables first.
- Use strict expectations whenever possible (`OCMExpect`, `OCMVerifyAll`) to surface unexpected calls immediately.

### **7.1 Test Fakes**
Handwritten fakes often produce clearer tests than deeply configured mocks.
- Name fake instances with the `fake` prefix and build them in MOCKING.
- Assign fakes to `env` variables in SETUP before injecting into the SUT.
- Provide assertion helpers on fakes to keep THEN/BEHAVIOR sections readable.
- Keep fake implementations near the tests that rely on them, and document their intended scope.

✅ **Good Example (Fake Logger):**
```objectivec
@interface FakeLogger : NSObject <Logger>
@property (nonatomic, strong, readonly) NSMutableArray<NSString *> *entries;
- (void)assertLoggedMessage:(NSString *)expectedMessage;
@end

@implementation FakeLogger
- (instancetype)init
{
    if (self = [super init]) {
        _entries = [NSMutableArray array];
    }
    return self;
}
- (void)log:(NSString *)message
{
    [self.entries addObject:message];
}
- (void)assertLoggedMessage:(NSString *)expectedMessage
{
    XCTAssertTrue([self.entries containsObject:expectedMessage]);
}
@end
```

---

## **8. Assertions & Variable Naming**
Consistent naming clarifies which values represent inputs versus observed outcomes.
- Assign expected values to `expected*` variables in the EXPECTATIONS section.
- Assign all observed results to `actual*` variables in WHEN.
- Never assert directly against literals, `given*`, or `env*` variables—always go through `expected*` or `actual*`.
- It is acceptable to assert directly on `mock*` variables inside BEHAVIOR (e.g. `OCMVerifyAll(mockClient)`).

✅ **Good Example:**
```objectivec
NSNumber *expectedCount = @3;
XCTAssertEqualObjects(expectedCount, actualCount);
```
🚫 **Bad Example:**
```objectivec
XCTAssertEqualObjects(@3, actualCount); // ❌ Literal in assertion
```

---

## **9. Exception Handling (`XCTAssertThrows`)**
Treat exception capture as part of WHEN so the flow remains explicit.
- When using `XCTAssertThrowsSpecific` or similar, store the thrown object in an `actual*` variable when possible.
- For APIs that only offer block-based assertions, document in comments that the block belongs to WHEN, then assert on the exception type or message in THEN using `expected*` variables.

✅ **Good Example:**
```objectivec
- (void)testDivide_WhenDenominatorZero_RaisesException
{
    // GIVEN
    NSInteger givenNumerator = 10;
    NSInteger givenDenominator = 0;

    // WHEN
    NSException *actualException = nil;
    XCTAssertThrowsSpecificNamed({
        actualException = [Calculator divide:givenNumerator by:givenDenominator];
    }, NSException, NSInvalidArgumentException);

    // EXPECTATIONS
    NSString *expectedName = NSInvalidArgumentException;

    // THEN
    XCTAssertEqualObjects(expectedName, actualException.name);
}
```

---

## **10. Using `setUp` / `tearDown` Methods**
Lifecycle hooks can hide important setup. Use them sparingly.
- Prefer explicit setup inside each test using fixtures.
- Use `- setUp` or `- tearDown` only for repeated initialization that is truly shared across all tests in the class.
- Do not instantiate new dependencies inside `setUp` that you mutate later; keep shared objects immutable or recreate them per test to avoid state leakage.

✅ **Good Example:**
```objectivec
- (void)setUp
{
    [super setUp];
    self.givenURLSession = [NSURLSession sharedSession];
}
```
🚫 **Bad Example:**
```objectivec
- (void)setUp
{
    self.service = [[UserService alloc] init]; // ❌ Hides per-test context
}
```

---

## **11. Organizing Tests in Objective-C Modules**
Namespace equivalents in Objective-C include prefixes and directory structure.
- Group related test classes in folders that match the production module or prefix.
- Maintain naming parity with production headers (e.g. `ABCUserService` → `ABCUserServiceTests`).
- Use `#import` statements at the top of each test file to import the production header under test, not the umbrella header unless necessary.

✅ **Good Example:**
```objectivec
#import "UserService.h"

@interface UserServiceTests : XCTestCase
@end
```

---

## **12. Managing Copy-Paste Setup**
Duplicating setup often keeps tests clearer than over-abstracting shared helpers.
- Start with a "happy path" test and copy its structure when creating variants; adjust the GIVEN/MOCKING sections and update the section comments to highlight differences.
- Refactor into helpers only when the abstraction improves readability for every caller.
- Keep helpers near the tests that use them and document their intent.

---

## **13. Deterministic Tests and Dependency Injection**
Deterministic tests build trust. Inject controllable collaborators rather than relying on global singletons or system state.
- Isolate side effects (time, randomness, file system) behind protocols or blocks passed into the SUT.
- Avoid calling `NSDate date`, `NSUUID UUID`, or singletons directly—inject substitutes in tests.
- For asynchronous APIs, use `XCTestExpectation` with deterministic triggers instead of relying on timed waits.

✅ **Example:**
```objectivec
@protocol Clock <NSObject>
@property (nonatomic, readonly) NSDate *now;
@end

- (void)testInvoiceDate_UsesInjectedClock
{
    // GIVEN
    id mockClock = OCMProtocolMock(@protocol(Clock));
    NSDate *givenNow = [NSDate dateWithTimeIntervalSince1970:0];
    OCMStub([mockClock now]).andReturn(givenNow);

    // SETUP
    id<Clock> envClock = mockClock;

    // SYSTEM UNDER TEST
    InvoiceService *sut = [[InvoiceService alloc] initWithClock:envClock];

    // WHEN
    NSDate *actualDate = [sut generateInvoiceDate];

    // EXPECTATIONS
    NSDate *expectedDate = givenNow;

    // THEN
    XCTAssertEqualObjects(expectedDate, actualDate);

    // BEHAVIOR
    OCMVerify([mockClock now]);
}
```

---

## **14. Using Comments in Tests**
Human-readable comments provide context for future maintainers.
- Annotate each test class with an Objective-C documentation comment (`/** ... */` or `///`) explaining the subject under test.
- Comment each test method with a summary of the scenario it covers.
- Section headers should be full sentences that explain intent, not terse labels.

✅ **Good Example:**
```objectivec
/// Covers the authentication flows of UserService
@interface UserServiceAuthenticationTests : XCTestCase
@end

@implementation UserServiceAuthenticationTests

/// Ensures invalid credentials surface an error dialog
- (void)testLogin_WhenCredentialsInvalid_ShowsError
{
    // GIVEN: The user enters an incorrect password
    // ...
}
```

---

## **15. Increasing Test Clarity with Section Comments**
Dense Objective-C code benefits from narrative comments that guide the reader.
- When configuring mocks, creating fixtures, or asserting complex behavior, split each responsibility into its own section and describe it with a full-sentence comment (`// MOCKING: Configure API client to return success`).
- Keep comments synchronized with the code; update them when logic changes.

---

## **16. SETUP / SUT Dependency Rule**
Maintain a strict separation between dependency preparation and SUT construction.
- SETUP must initialize all collaborators and expose them via `env*` variables.
- The SYSTEM UNDER TEST section must construct the SUT using only `env*` variables.
- Never inject `mock*` or `fake*` variables directly into the SUT.

---

## **17. Breaking the Rules**
The rules above are prescriptive to produce consistent, reviewable tests. Occasionally, exceptions improve clarity.
- Deviation is acceptable when it simplifies the test or makes deterministic behavior clearer.
- Document the rationale inline with a comment when breaking a rule.
- Revisit exceptions periodically; once constraints disappear, restore the standard structure.

---

By following this style guide, Objective-C test suites remain reliable, expressive, and easy to navigate, enabling developers and LLMs alike to collaborate effectively on high-quality code.
