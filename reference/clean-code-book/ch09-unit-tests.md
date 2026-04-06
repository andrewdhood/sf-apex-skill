# Chapter 9: Unit Tests

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008), Chapter 9
> **When to read this:** Before writing or refactoring any `@isTest` class. Especially before arguing about whether Apex test coverage "counts as testing."

Most programmers have adopted automated testing but missed the subtler points. This chapter argues that test code is *as important as production code*, that clean tests require the same thought and care, and that dirty tests are worse than no tests at all.

---

## Key Principles

1. **The Three Laws of TDD.** Write no production code until you have a failing test. Write no more of a test than is sufficient to fail. Write no more production code than is sufficient to pass. This locks you into a thirty-second cycle and produces tests that cover virtually all of your production code.
2. **Test code is just as important as production code.** It is NOT a second-class citizen. The dirtier your tests, the harder they are to change. Eventually the cost of maintenance is so high the team discards the suite — and their production code rots soon after.
3. **Tests enable the -ilities.** Without tests, fear of breaking things prevents cleanup, refactoring, and architectural improvement. With tests, fear disappears and improvement becomes possible. **Tests enable change.**
4. **Readability, readability, readability.** It matters even more in tests than in production code. Clean tests say a lot with few expressions.
5. **Build a domain-specific testing language.** Evolve helper functions that make tests more convenient to write and easier to read. These become a specialized API used only by tests.
6. **Dual standard.** Test code runs in a test environment; it can relax about CPU/memory efficiency but must NEVER relax about cleanness.
7. **Minimize asserts per test; test one concept per test.** "One assert per test" is a useful guideline. The better rule is "one concept per test." Several related assertions verifying a single concept are fine; scattered assertions testing unrelated things are not.
8. **F.I.R.S.T.** Clean tests are Fast, Independent, Repeatable, Self-validating, and Timely.

---

## Detailed Notes

### The Three Laws of TDD
- **First Law:** You may not write production code until you have written a failing unit test.
- **Second Law:** You may not write more of a unit test than is sufficient to fail — and not compiling is failing.
- **Third Law:** You may not write more production code than is sufficient to pass the currently failing test.

Followed strictly, the cycle is about thirty seconds long. Tests and production code grow together, with tests just seconds ahead. You produce dozens of tests a day, thousands a year, covering virtually all of your production code.

### Keeping Tests Clean
Martin tells a story of a team that decided test code did not need the same rigor as production code. "Quick and dirty" became their watchword. As production code evolved, tests had to change with it — and the dirtier the tests, the harder they were to change. Eventually the cost of maintaining the tests became the single biggest developer complaint, and the team discarded the suite entirely. Without tests, they lost the ability to know their changes worked. Defect rates rose, they stopped cleaning production code out of fear, and production rotted. Their test effort *had* failed them — but only because they let their tests get dirty. Clean tests don't fail teams.

**Moral:** test code is as important as production code. It requires design, thought, and care.

### Tests Enable the -ilities
The counterintuitive point: unit tests are what keep your code *flexible, maintainable, and reusable*. The higher your coverage, the less your fear of change. You can improve architecture and design with near-impunity. Without tests, every change is a possible bug, and fear prevents improvement.

### Clean Tests
Three things make a clean test: readability, readability, and readability. The same traits that make production code readable — clarity, simplicity, density of expression — matter more in tests. A reader should be able to tell within seconds what the test sets up, what it does, and what it verifies.

Martin refactors a three-test FitNesse example from a noisy pile of crawler setup and response casting into three clean tests that follow the **BUILD-OPERATE-CHECK** pattern:
- **Build** the test data.
- **Operate** on it.
- **Check** the expected result.

Each section is visually distinct and uses only helper methods that read like domain language (`makePages`, `submitRequest`, `assertResponseIsXML`, `assertResponseContains`).

### Domain-Specific Testing Language
The clean tests above demonstrate building a domain-specific testing language. Rather than using raw APIs directly, you build a set of helper functions that make tests convenient to write and easy to read. This testing API is *not* designed up front — it evolves by continual refactoring of test code that has accumulated obfuscating detail. Refactor tests the same way you refactor production code.

### A Dual Standard
The test environment is not the production environment. There are things you might never do in production that are perfectly fine in a test — things involving CPU or memory efficiency. Martin gives the example of a `getState()` method that concatenates strings in a loop (using `+=` instead of `StringBuffer`) — inefficient, but fine in a test. The dual standard applies to efficiency only. **It never applies to cleanness.**

### One Assert per Test
"One assert per test" is an appealing guideline. It forces each test to reach a single, easily understood conclusion. Where the single-assert rule doesn't naturally fit, you can split one test into two — a `testGetPageHierarchyAsXml` that asserts the response is XML, and a separate `testGetPageHierarchyHasRightTags` that asserts the tags. This often produces duplication, which you can address with a Template Method or a `@Before` setup.

Martin admits the rule is a guideline, not dogma: the real principle is to *minimize* asserts per test. What matters more than asserts-per-test is the next rule:

### Single Concept per Test
Every test function should test a single concept. A test that verifies three different behaviors of `addMonths()` — adding one month to May 31st, adding two months to May 31st, and adding one month twice to May 31st — is testing three concepts. Split it into three tests. The reader should not have to figure out why multiple sections exist in one function.

### F.I.R.S.T.
Clean tests follow five rules:
- **Fast.** Tests should run quickly. Slow tests don't get run, and problems don't get found early.
- **Independent.** Tests should not depend on each other. You should be able to run them in any order. Test A must not set up state that Test B needs.
- **Repeatable.** Tests should be runnable in any environment — production, QA, laptop, on a train with no network. If tests aren't repeatable, you always have an excuse for why they fail.
- **Self-Validating.** Tests have a boolean output: pass or fail. You should not have to read logs or diff files to know.
- **Timely.** Tests should be written *just before* the production code. If you write tests after production, you may find the production code hard to test.

---

## Salesforce/Apex Caveats

**Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### The 75% coverage mandate
Salesforce requires at least 75% code coverage to deploy Apex to production; the team norm should be 90%+. This is not a Clean Code principle — it's a platform requirement. The good news: if you follow the Three Laws of TDD, you will exceed 90% naturally. The bad news: coverage is measured per line, which can encourage gaming ("assert nothing, just invoke the method"). Never write coverage-only tests. A test without meaningful assertions is worse than no test, because it takes up a slot in the suite and can never catch regressions.

### Apex testing primitives you must know
- **`@isTest`** — marks a class or method as a test. Test classes do not count against org code size.
- **`@TestSetup`** — runs once before all test methods in a class. Ideal for building the data fixture every test shares. Data created in `@TestSetup` is automatically rolled back between test methods, and each method gets a fresh copy.
- **`Test.startTest()` / `Test.stopTest()`** — resets governor limits between setup and the actual operation being tested, AND forces asynchronous work (future, queueable, batchable) to execute synchronously before `stopTest()` returns. Always put the code under test between these calls.
- **`@isTest(SeeAllData=false)`** — the default. Tests should NOT see org data. Do not set `SeeAllData=true` except for specific Apex tests that require org configuration (e.g., Pricebook2 for Opportunity tests before Salesforce added `Test.getStandardPricebookId()`).
- **`System.runAs(user)`** — executes a block as a specific user. Required for testing sharing rules, profile permissions, and `with sharing` behavior.

### "One assert per test" fights governor limits
This is the biggest Clean Code collision in Apex testing. A single `@isTest` method is expensive to set up: you insert Accounts, Contacts, Opportunities, custom objects, Pricebook entries, etc., each consuming DML rows and SOQL queries against the per-test governor budget. Splitting one test with five asserts into five tests with one assert each means **five times the setup cost**.

Two reconciliation strategies:
1. **Use `@TestSetup` to amortize fixture cost across many small tests.** `@TestSetup` runs once per class but the data is automatically rolled back and re-inserted for each method — meaning the DML happens repeatedly but not in the test method itself. Your individual test methods stay cheap.
2. **Prefer "one concept per test" over strict "one assert per test."** Martin himself admitted the one-assert rule is a guideline. In Apex, a "concept" is a test scenario — given this setup, when I do this operation, these N related things should be true. Asserting all N in one method is acceptable; testing five unrelated scenarios in one method is not.

### The Single Concept per Test rule, Apex-style
A clean Apex test has three clearly marked sections, same as Martin's BUILD-OPERATE-CHECK:
```apex
@isTest
static void recalculatesTotalAfterDiscount() {
    // BUILD — build test data (or pull it from @TestSetup)
    Account acc = [SELECT Id FROM Account LIMIT 1];
    Opportunity opp = TestDataFactory.buildOpp(acc.Id, 1000);
    insert opp;

    // OPERATE — the thing under test
    Test.startTest();
    OpportunityService.applyDiscount(opp.Id, 0.10);
    Test.stopTest();

    // CHECK — the assertions for this one concept
    Opportunity updated = [SELECT Amount, Discount__c FROM Opportunity WHERE Id = :opp.Id];
    System.assertEquals(900, updated.Amount, 'Discount should reduce amount by 10%');
    System.assertEquals(0.10, updated.Discount__c, 'Discount field should be stored');
}
```

### Domain-specific testing language in Apex: the TestDataFactory
Martin's "domain-specific testing language" is exactly what a `TestDataFactory` class is. Build helper methods that hide the verbose construction of SObjects with all their required fields:
```apex
// instead of this noise in every test method:
Account a = new Account(Name='Test', BillingCountry='US', Type='Customer', ...);
Contact c = new Contact(FirstName='X', LastName='Y', AccountId=a.Id, Email='x@y.com', ...);

// have this:
Account a = TestDataFactory.customerAccount();
Contact c = TestDataFactory.primaryContact(a.Id);
```
The factory becomes the testing DSL. Keep it well-organized, with one method per meaningful fixture shape, and never let it sprawl into "God factory" territory.

### F.I.R.S.T. mapped to Apex
- **Fast** — Apex tests are slower than most unit tests because every one hits a real database. Mitigate by minimizing DML, reusing `@TestSetup`, and avoiding `SeeAllData=true`. Aim for individual methods under a few seconds.
- **Independent** — Apex tests ARE independent by default: all DML is rolled back at the end of each test, so state cannot leak. The platform enforces this. Take advantage of it.
- **Repeatable** — Apex tests are deterministic as long as you don't use `SeeAllData=true` and don't rely on `System.now()`, `Date.today()`, or sequential Auto-Number values without injection. Inject "now" through a `SystemAdapter` class if test logic depends on time. For callouts, use `Test.setMock`.
- **Self-validating** — `System.assert`, `System.assertEquals`, and `System.assertNotEquals` already give you pass/fail output. Always provide a message as the third argument so failure reports are readable.
- **Timely** — The Three Laws of TDD work in Apex, but the fast feedback loop is handicapped by deploy times. Use scratch orgs with source tracking (`sf project deploy start --source-dir force-app`) to keep the cycle short, and run tests via `sf apex run test --class-names MyClassTest` to avoid the full run.

### Mocking is hard in Apex
Martin's chapter assumes Mockito-grade mocking is available. It isn't in Apex. Options:
- **Interface + hand-rolled test double** — simplest, most explicit, no dependencies. Preferred for most cases.
- **`System.StubProvider` / Stub API** — built-in mocking via `Test.createStub()`. Works on any non-system class without requiring an interface. Verbose, but no external dependency.
- **`fflib-apex-mocks`** — open-source Mockito port. Powerful but adds a dependency and is non-trivial to learn.
- **`Test.setMock(HttpCalloutMock.class, ...)`** — use for EVERY callout test. There is no substitute.

Design for testability: constructor-inject dependencies so tests can swap in fakes. Avoid static method calls to service classes from within other service classes — they're hard to mock. Use an `Application` factory or a simple setter-injection pattern.

### Three Laws of TDD vs. the save cycle
The thirty-second cycle Martin describes is painful in Apex because saving forces a compile against a live org. Practical advice:
- Use scratch orgs or sandboxes you can push to quickly.
- Use the Apex Replay Debugger and the VS Code Salesforce Extension to run only the test you're working on.
- Accept that your cycle is closer to two minutes than thirty seconds, and batch your tight loops to fit.

### Test code quality still matters (maybe more)
Apex developers are notorious for treating test code as a compliance checkbox. The result is exactly what Martin warned about: tests that nobody understands, nobody maintains, and nobody trusts. When production code changes, the tests are "fixed" by commenting out assertions. Don't do this. **Keep the tests clean.** Refactor them the same way you refactor production code. Delete tests that are no longer meaningful rather than leaving them as dead weight. Use meaningful method names (`recalculatesTotalAfterDiscount` beats `testMethod1`). Apply the same style rules to test code that you apply to production code.

---

## Key Takeaways

- Salesforce mandates 75% coverage (90%+ preferred). This is non-negotiable but not a substitute for good tests.
- Test code is NOT a second-class citizen. Refactor tests, name them meaningfully, and delete dead ones.
- Use `@TestSetup` to amortize fixture cost so you can afford many small single-concept tests.
- Always put the code under test between `Test.startTest()` and `Test.stopTest()` — it resets limits AND forces async work to run.
- Build a `TestDataFactory` as your domain-specific testing language. Never paste SObject construction across every test method.
- Follow BUILD-OPERATE-CHECK visually in every test.
- "Single concept per test" beats strict "one assert per test" in Apex, given governor limit economics. Group related assertions for one scenario; split unrelated ones.
- F.I.R.S.T. maps cleanly to Apex EXCEPT "Fast" — Apex tests are DB-bound and slow. Minimize DML, reuse setup, and don't use `SeeAllData=true`.
- Apex has no Mockito. Use interfaces + test doubles, or the built-in Stub API, or `fflib-apex-mocks`. Always use `Test.setMock` for callouts.
- `System.runAs()` is mandatory for testing sharing and permission behavior. Do not rely on test runs as the default "System" admin to validate security.
- Every assert should have a message as the third argument so failure reports are readable.
