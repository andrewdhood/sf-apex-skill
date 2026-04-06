# Chapter 8: Quality and Testing

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When planning a testing strategy, writing Apex unit tests, setting up static analysis, configuring quality gates, or evaluating test tooling for a Salesforce project. This is the most comprehensive single-chapter treatment of Salesforce testing strategy available.

## Key Concepts

### Three Dimensions of Quality

Quality is not a single attribute. Always evaluate code across three dimensions:

1. **Functional quality** -- Does the code do what it is supposed to do? Validated by unit tests, acceptance tests, and manual QA.
2. **Structural quality** -- Is the code well-built internally? Assessed by static analysis (linting, quality gates, security scans), code reviews, and performance testing.
3. **Process quality** -- Is the team following good practices? Measured by code review adoption, test coverage trends, and deployment success rates.

When only one dimension is measured (typically functional via the 75% coverage gate), the other two decay silently. Technical debt accumulates, and if left to grow, it makes software maintenance increasingly difficult, time-consuming, and risky.

### Test Classification Taxonomy

Tests can be classified along three independent axes:

**By scope:**
- **Unit tests** -- Exercise a single unit of code (one method, one class). Should run in under 5 minutes total.
- **Component tests** -- Exercise multiple units together (e.g., a trigger that fires a handler that invokes a service class). Most Apex "unit tests" are actually component tests.
- **Integration tests** -- Exercise the full integrated system, cutting across multiple Salesforce components or external systems. Apex governor limits on CPU, memory, and callouts make true integration tests difficult in Apex.

**By what they evaluate:**
- **Functional tests** -- Does the code produce the correct output? (e.g., "does the discount apply when the contract value exceeds $1M?")
- **Nonfunctional tests** -- Is the code secure, performant, maintainable, and reliable?

**By how they run:**
- **Code-based tests** -- Apex tests, JavaScript/Jest tests for LWC, JavaScript tests for VisualForce static resources.
- **API-based tests** -- Tests that exercise Salesforce APIs. Important for integration scenarios. MuleSoft MUnit is an example.
- **UI tests** -- Tests that automate browser interactions. Use Selenium, Puppeteer, Provar, or Tosca.

**By when they run:**
- **Commit-stage tests (fast tests)** -- Run before/after every commit. Must complete in under 5 minutes. Includes linting and unit tests.
- **Acceptance tests (comprehensive tests)** -- Run before release. May take hours. Includes code-based acceptance tests, UI tests, security scans, and manual QA.

### The Test Pyramid

The test pyramid (introduced by Mike Cohn) defines the ideal ratio of test types:

- **Base (most tests):** Unit tests -- fast, isolated, code-based.
- **Middle (moderate):** Service/component/acceptance tests -- broader scope, still code-based, don't require a UI.
- **Top (fewest tests):** UI tests -- slow, brittle, expensive. Reserve for critical or complex processes that cannot be tested any other way.

Never try to cover every edge case with UI tests. They have vastly more permutations of inputs and outputs, are more expensive to maintain, and a single UI change can cascade across many tests.

### Salesforce's 75% Coverage Rule

Salesforce requires 75% code coverage across all Apex classes and triggers before deploying to production. This is a minimum, not a target.

- Deployments fail if any single Apex class or trigger has 0% coverage OR the overall org coverage drops below 75%.
- The error message is: `Code coverage is insufficient. Current coverage is 74.928...% but it must be at least 75% to deploy.`
- Never push hard for 100% coverage. When Apex code starts becoming more complex and difficult to read or filling up with `Test.isRunningTest()` checks, you have gone too far.
- High code coverage does not guarantee good tests. Low code coverage guarantees you have not written enough tests.
- Flows and Processes also require code coverage if deployed to a production org as part of a CI/CD process and they are deployed as active.

## Detailed Notes

### Fast Tests for Developers

Fast tests are commit-stage tests that should run before and/or after every commit. They consist of static analysis (linting, quality gates) and unit tests. Their purpose is to provide early warning to developers if they break something and to minimize time spent diagnosing the cause.

#### Static Analysis: Linting

Static analysis is automated analysis of source code to identify possible structural faults in performance, style, or security. When performed in real time in the IDE, it is called linting.

**When linting runs:** Continuously in the IDE as the developer works, similar to spell checking. Linters typically deal with one file at a time.

**Where linting runs:** In the IDE, directly on the codebase. No Salesforce environment required. No test data required.

**Linting tools for Salesforce:**

| Language | Tool | Notes |
|----------|------|-------|
| JavaScript (LWC) | ESLint + `eslint-plugin-lwc` | Dominant choice. Highly configurable rules. Easy to write custom rules. |
| JavaScript (Aura) | ESLint + `eslint-plugin-aura` | Same engine, different plugin. |
| Apex | ApexPMD | Most popular static analysis for Apex. Free, open source. VS Code extension by Chuck Jonas. |
| Apex | SonarLint | Linting component of SonarQube. Can run "offline" with built-in rules or "online" connected to a SonarQube instance for synchronized rulesets. |

**Key guidance for linting:**
- Do not expect linting to catch every issue. It enforces general rules ("methods should not have too many parameters"), not specific business logic.
- Linting rules are not usually customized per Salesforce org. Select a subset of rules appropriate for your project and stick with them.
- If certain rules are not helpful and just add noise, remove them. Developers are under no obligation to act on linting feedback.
- If you want to enforce linting rules, use quality gates (see below) rather than relying on developer discipline.
- PMD has a graphical rule designer for creating custom rules (using XPath or Java). ESLint has a much easier custom rule API.

**Running PMD from the command line:**
```bash
pmd -d src -failOnViolation false -f text -language apex -R rulesets/apex/style.xml,rulesets/apex/complexity.xml,rulesets/apex/performance.xml
```

**Running PMD's copy-paste detector (CPD):**
```bash
cpd --files src --language apex --minimum-tokens 100 --format csv
```

#### Static Analysis: Quality Gates

Quality gates apply the same analysis rules as linting but at the scope of a commit or pull request, producing a pass/warn/fail assessment. They enforce rules that linting only suggests.

**When quality gates run:** When a developer makes a commit or creates a pull request. Pull requests are the most common trigger.

**Where quality gates run:** In a CI process or static analysis engine. No Salesforce environment required. No test data required.

**How to establish quality gates:**
- Most static analysis tools (SonarQube, Clayton, Codacy, CodeClimate) provide quality gate functionality when used with pull requests and a CI/CD tool.
- Commercial Salesforce release management tools may include this as a native capability with integrated PMD scans.
- Copado Compliance Hub provides Salesforce-specific security analysis (checking profiles, permission sets, sharing model changes for security breaches).

**Quality gate criteria should be consistent with linting rules.** This ensures alignment across all three levels of static analysis (IDE, pull request, full codebase).

**The "leak period" strategy:** Rather than requiring immediate remediation of all existing issues, enforce coverage/quality rules only on recently changed code. Over time, this ratchets up quality across the entire codebase without requiring a massive remediation effort upfront. One team went from negligible JavaScript coverage to over 50% in a few weeks using this approach with SonarQube.

**A passing quality gate does not guarantee quality.** Static analysis does not guarantee code will execute successfully. It is a useful indication, not a substitute for functional testing and code reviews.

#### Unit Testing

Unit tests are the best-known type of automated test on the Salesforce platform. Every Salesforce developer is familiar with them because of the Apex test runner built into the platform and the 75% coverage requirement.

**How to run unit tests:**

| Technology | Test Engine | Runs Where | Notes |
|------------|------------|------------|-------|
| Apex | Built-in Apex test runner | Salesforce org (scratch org or sandbox) | Proprietary. Cannot run outside Salesforce. |
| JavaScript in VisualForce | Jest (recommended) | Developer workstation or CI | Extract JS into static resources to enable testing outside the VF page. |
| Aura Lightning Components | Lightning Testing Service (Jasmine/Mocha) | Salesforce org (installed as unmanaged package) | Must run inside an org. Cannot run outside of Salesforce. |
| Lightning Web Components | Jest (`@salesforce/sfdx-lwc-jest`) | Developer workstation or CI | Can run outside a browser. Fast. Standards-compliant. |

**When unit tests run:**
- During active development, run the small number of tests directly pertaining to your current work.
- Also run preexisting tests related to your changes to catch regressions.
- Because Apex tests run more slowly than Java or JavaScript equivalents, it may not be practical to rerun every test before every release. Use the `ApexCodeCoverage` table (via Tooling API) or tools like ApexUnit to dynamically determine which tests cover which code.
- The Salesforce extensions for VS Code allow running and monitoring Apex unit tests from inside the IDE.

**Unit testing environments:**
- Apex unit tests can only run in a Salesforce org. Use development scratch orgs during development; CI/CD can create scratch orgs dynamically for test runs.
- JavaScript tests (LWC, static resources) can run locally on a developer workstation or in CI, outside any Salesforce org.
- Aura component tests require a Salesforce org (via Lightning Testing Service). Run them in scratch orgs, not production.
- LWC tests can run outside a browser and are extremely fast.

**Data for unit tests:**

- Tests should create their own test data. Never rely on data in the underlying org.
- Since Spring '12, Apex tests no longer have access to org data by default unless you use `@isTest(SeeAllData=true)`. Avoid `SeeAllData=true` -- it makes tests brittle and org-dependent.
- Tests that do not require access to org data are called "data silo tests." They pass reliably in any org and don't break when other users modify data.
- Apex tests run inside a database transaction. Data created during the test is not persisted after the test finishes.
- JavaScript tests (LWC, static resources) don't run in a transaction. Data they create in an org will remain after the test completes.
- For large volumes of test data, use static resources in CSV format loaded via `@TestSetup`.

**Using the Apex Domain Builder for test data:**

The Apex Domain Builder (github.com/rootstockmfg/domainbuilderframework) provides a fluent syntax for creating test data that is readable and reduces boilerplate:

```apex
@IsTest
private static void easyTestDataCreation() {
    // Given
    Contact_t jack = new Contact_t().first('Jack').last('Harris');

    new Account_t()
        .name('Acme Corp')
        .add( new Opportunity_t()
                    .amount(1000)
                    .closes(2019, 12)
                    .contact(jack))
        .persist();
    // When
    ...
    // Then
    ...
}
```

Required and boilerplate field values are defined centrally in the `_t` classes, so inside tests you only specify the values specific to that test scenario.

**Creating unit tests -- the BDD approach:**

Adopt behavior-driven development (BDD) for structuring tests. Each test method name should describe the expected behavior, beginning with `itShould...`. Each test body follows the Given/When/Then structure (equivalent to the older Arrange-Act-Assert pattern):

```apex
@isTest
static void itShouldUpdateReservedSpotsOnInsert() {
    System.runAs(TestUtil.careerPlanner()) {
        // Given
        Workshop__c thisEvent = TestFactory.aWorkshopWithFreeSpaces();
        Integer initialAttendance = TestUtil.currentAttendance(thisEvent);
        final Integer PRIMARY_ATTENDEES = 3;
        final Integer NUMBER_EACH = 4;

        // When
        Test.startTest();
        TestFactory.insertAdditionalRegistrations(thisEvent, PRIMARY_ATTENDEES,
        NUMBER_EACH);
        Test.stopTest();

        // Then
        Integer expectedAttendance = initialAttendance + PRIMARY_ATTENDEES *
        NUMBER_EACH;
        system.assertEquals(expectedAttendance, TestUtil.
        currentAttendance(thisEvent),
            'The attendance was not updated correctly after an insert');
    }
}
```

**Considerations for unit testing:**

- If a harmless change (like adding a required field) causes many tests to fail, use a central test factory to create objects. The factory defines base objects with all required fields in one place; individual tests modify only the fields relevant to their scenario.
- Never push hard for 100% coverage. The Pareto principle applies: 80% of the progress requires 20% of the effort; the remaining 20% requires 80% of the effort.
- Use `@IsTest(isParallel=true)` on tests that are parallel-safe. Disable it for tests that encounter `UNABLE_TO_LOCK_ROW` errors due to shared record access.
- Parallel test execution is configured at: **Setup > Custom Code > Apex Test Execution > Options** ("Disable Parallel Apex Testing").

### Comprehensive Tests

Comprehensive tests (acceptance tests) are the broader set of tests run before code is released. They focus on thoroughness over speed and include automated functional testing, nonfunctional testing (security, performance, static analysis of the full codebase), and manual QA.

#### Code-Based Acceptance Testing

There is no technological difference between code-based unit tests and code-based acceptance tests. The same Apex/Jest/Mocha engines run both. The difference is purpose and scope:

| Characteristic | Unit Tests | Acceptance Tests |
|---------------|-----------|-----------------|
| Purpose | Fast feedback during development | Comprehensive regression testing before release |
| Scope | Single unit of code | Multiple components, end-to-end scenarios |
| Speed | Under 5 minutes total | May take hours (run in parallel) |
| When | Every commit | Before every release |

**When acceptance tests run:**
1. During development (same as unit tests, but gradually accumulate into the acceptance suite).
2. During deployments -- Salesforce runs Apex tests during deployment. Tests execute in the target org after metadata has been deployed. If any test fails (or coverage drops below 75%), the deployment rolls back atomically. This is one of the platform's most powerful features.
3. Triggered by external processes -- Use `sfdx force:apex:test:run` from CI, or schedule tests using `ApexTestQueueItem`.

**JavaScript tests cannot run inside a Salesforce deployment transaction.** They must run as a separate stage in your CI/CD pipeline. Failing JavaScript tests should block later pipeline stages.

**Acceptance testing environments:**
- Can run in scratch orgs or testing sandboxes.
- For JavaScript and UI tests, scratch orgs are ideal because they start clean. But scratch orgs may not fully resemble production.
- Automated acceptance tests should also be run in a partial or full sandbox to validate behavior in a production-like environment.
- Data for automated tests must be clearly segregated from data used for manual testing.

**Creating acceptance tests:**
- Write unit tests with the mindset that each will become part of the acceptance test suite. Each test is an executable specification: given certain inputs, when a particular action occurs, assert the correct result.
- The BDD/Given-When-Then approach bridges unit and acceptance testing naturally.
- Complex multistep procedures involving multiple triggers, processes, or flows can easily exceed Apex governor limits. UI tests may be better for simulating these scenarios.

**Tracking code coverage:**
- The Salesforce Developer Console and IDEs can show line-level Apex coverage.
- SonarQube and CodeScan can ingest Apex coverage results for trend tracking. Generate coverage JSON with: `sfdx force:apex:test:run -c -d test-results -r json`
- For Jest/JavaScript coverage: `npx jest --ci --coverage` generates lcov.info files. Feed them to SonarQube with: `sonar.javascript.lcov.reportPaths=coverage/lcov.info`

#### UI Testing

UI tests automate browser interactions and verify visual output. Salesforce does not provide a built-in UI test engine.

**Benefits of UI tests on Salesforce:**
- Can test complex action sequences that would exceed Apex governor limits.
- Require less specialized expertise (testers interact with the familiar Salesforce UI).
- Can test things that happen in the browser but not directly inside Salesforce.

**Challenges of UI tests on Salesforce:**
- The Salesforce UI changes at least three times per year (major releases). HTML tags and CSS classes are dynamically generated and change with each page load.
- You cannot rely on DOM IDs or CSS classes for element identification. Salesforce either does not generate IDs or autogenerates them inconsistently.
- UI tests are significantly slower than Apex tests, which are slower than JavaScript tests.
- UI tests are notoriously brittle. "Flappers" (tests that alternate unpredictably between pass and fail) should be removed or fixed immediately because they erode trust in the entire test suite.

**UI testing tools:**

| Tool | Type | Notes |
|------|------|-------|
| Selenium | Open source, code-based | Most popular. Supports multiple languages and browsers. Runs on any platform. |
| Puppeteer | Open source, code-based | Google Chrome only. JavaScript-based. Headless browser. Fast and simple. Handles ~80% of UI testing needs. |
| Provar | Commercial, Salesforce-specific | Record-and-replay. Handles Lightning and Classic. Keeps up with Salesforce UI changes. Best choice if only testing Salesforce. |
| Tricentis Tosca | Commercial | Good for mixed Salesforce + non-Salesforce testing. |
| Copado / AutoRABIT | Commercial, integrated | Built-in Selenium test recorders. Integrated into their release management workflows. |

**Best practices for UI test maintainability:**
- Add explicit `Id` attributes to all custom page elements. Use the naming convention **PageName_FieldName_FieldType** (e.g., `CaseSummary_Owner_Picklist`).
- Build modular test architecture: centralized modules for login, data load, navigation, list views, record pages. This prevents cascading failures when a single UI change occurs.
- Load test data via the API (fast, reliable) rather than typing it into the UI. Focus UI tests on validating complex custom aspects of the UI.
- If possible, share a single consistent test data set between automated UI tests and manual testers so that test data updates only need to happen once.

**When UI tests run:** Not during development. Best suited for post-deployment regression testing. Parallelize them because they take a long time.

**Where UI tests run:** In a scratch org or test sandbox with your customizations deployed and a full set of test data loaded.

#### Nonfunctional Testing

Nonfunctional testing examines the structural characteristics of code: reliability, maintainability, security, performance, and size.

##### Static Analysis: Full Codebase

The third level of static analysis (after linting and quality gates) is assessing the entire codebase and evaluating trends over time. Tools scan for issues ranging from code smells and duplication to security vulnerabilities and excessive complexity.

**Six well-established tools for Salesforce static analysis:**

| Tool | Type | Key Strength |
|------|------|-------------|
| Clayton | Salesforce-only, SaaS | Only tool designed exclusively for Salesforce. Connects to orgs or repos. Rules curated by Salesforce CTAs. Links to Trailhead for remediation. |
| SonarQube | Open-core, self-hosted or SonarCloud SaaS | Tracks issues over time with severity ranking. Supports 25+ languages. Enterprise edition has native Apex rules. |
| CodeScan | SonarQube-based, Salesforce-focused | Largest number of Salesforce-specific quality rules. Cloud and self-hosted options. Supports Apex, Lightning, Visualforce. |
| ApexPMD | Open source, CLI or IDE | Most widely used analysis engine for Apex. Free. Underlies many other tools. Rich configurable ruleset out of the box. Includes CPD (copy-paste detector). |
| Codacy | SaaS | Uses ESLint and PMD under the hood. Adds UI, authentication, dashboards. Enterprise edition supports self-hosting and GitLab. |
| CodeClimate | SaaS | Two products: Quality (static analysis via PMD/ESLint) and Velocity (developer activity metrics like cycle time). |

**Key considerations:**
- Do not be overwhelmed by initial scan results. SonarQube uses the term "kilodays" for estimated remediation time. Focus on trends, not absolute numbers.
- Do not assume a clean scan means good code. Static analysis does not catch logical errors or poor architecture. Code reviews and unit tests are essential complements.
- Use bubble charts or similar visualizations to identify hotspot classes (large, complex files with many issues). Prioritize refactoring these while the responsible developers are still available.

##### Security Analysis

Security analysis tools focus specifically on identifying security vulnerabilities in Apex code.

**CheckMarx:** The dominant security analysis tool for Salesforce. Salesforce maintains a free instance for customers. It can identify cross-file security issues (e.g., SOQL injection where unsanitized text input flows from a VisualForce page through an Apex controller to a service class to a query). It also detects stored XSS attacks. Available as a SaaS tool or with CI plugins (CxSAST) for Jenkins, Bamboo, etc.

**Micro Focus Fortify:** Less Salesforce-specific but provides an on-demand SaaS-based scanner (CheckMarx must be installed on a server). Fortify offers both cloud and on-premise options.

##### Performance Testing

Performance testing evaluates how applications perform under normal or higher-than-normal load. It is an aspect of nonfunctional testing that is distinct from static analysis and security analysis.

**Two primary scalability concerns on Salesforce:**
- **Large Data Volume (LDV)** -- Tens of millions of records on a single object. Causes slow SOQL queries and reports. Can be investigated by developers with sufficient data. Addressed with indexing, skinny tables, query plans, and archiving.
- **Large Traffic** -- Unpredictable when exposing orgs to customers through Sites or Communities. Requires load testing tools.

**Types of performance testing:**
- **Load testing** -- Simulating expected volumes and varying load across the normal range.
- **Stress testing** -- Simulating higher-than-normal loads. Includes soak testing (stable or increasing load over extended time) and spike testing (sudden traffic bursts).

**Performance testing tools:** JMeter (open source, popular), Micro Focus LoadRunner (commercial, well-established), Loader.io (cloud-based, free tier available), Odaseva and OwnBackup (data-focused tools for generating large volumes of sample data).

**Critical rule: Performance testing is prohibited on Salesforce sandboxes except by prior arrangement.** You must plan at least 2 weeks in advance and coordinate with Salesforce support. They can abort tests that affect other customers on the shared infrastructure. Use full or partial copy sandboxes (not scratch orgs).

**Creating performance tests -- the four-step process:**
1. Build a test plan (identify key transactions, data volumes, API/UI endpoints).
2. Run baseline tests (small transaction groups to validate scripts and set expectations).
3. Identify target load (be realistic -- if you expect 200 parallel requests, don't test 10,000).
4. Scale up gradually (start at 50% load, then 75%, then full).

#### Code Reviews

Code reviews are a manual form of nonfunctional testing and one of the most powerful methods of ensuring consistent high-quality code.

**When to perform code reviews:** During development (pair programming), informally after the fact (peer review), or as part of a formal pull request process.

**Key finding from the State of DevOps Reports (summarized in Accelerate):**
- Teams relying on peer review achieved higher software delivery performance than teams requiring external approval.
- External approvals (change advisory boards) were negatively correlated with lead time, deployment frequency, and restore time, and had no correlation with change fail rate.
- Recommendation: Use lightweight change approval processes. Pair programming or intrateam peer reviews bring more educated, contextual review than external reviewers with limited understanding.

**Suggestions for code review quality:**
- Follow the programming language style guide.
- Use descriptive names for methods and variables.
- Do not overdesign.
- Use efficient data structures and algorithms.
- Create proper test classes and modularize code.
- Document complex manual steps, simplify them where possible, and make code self-documenting.
- Keep all elements of the project in version control.

#### Manual QA and Acceptance Testing

Manual testing is mentioned last not because it is unimportant, but because the time and skills of testers are best spent supplementing automated testing -- focusing on exploratory testing and one-off scenarios that do not justify automation.

**Two phases of manual acceptance testing:**
1. **Internal QA** -- Performed by the development or QA team before making functionality available to users. A sanity check that things work as specified. Should be done on work that has already passed all automated tests.
2. **User Acceptance Testing (UAT)** -- Performed by subject matter experts (SMEs) from the business team. Validates that the system behaves correctly under realistic conditions and represents an improvement over what is currently in use.

**QA and UAT environments:**
- QA can shift left into scratch orgs, providing fast feedback in the same environment the developer is using (or a "Review App" scratch org spun up by CI).
- UAT should be done in a production-like environment (partial or full copy sandbox) with familiar data and live integrations.
- Store test data in your code repository so it can be loaded into scratch orgs for QA. This provides a shared, version-controlled, regularly-reset data set.

**Data for UAT:**
- UAT data should match the actual production org. Use partial or full copy sandboxes.
- Configuration data (Products, Pricebooks) is the most critical. Then key records (Accounts, Opportunities) that match the production system.
- Data management tools like OwnBackup and Odaseva can anonymize and import production data for testing.
- Salesforce DX `data:tree:export` can export data collections into version control for loading into scratch orgs.

**The developer-tester relationship:**
- Developers focus on building quickly; testers focus on breaking things until confident nothing will break.
- QA testers hold institutional memory of the most common failures and remain watchful to prevent regressions.
- The longer the time between development and feedback, the less effective that feedback becomes. Automated delivery shortens this feedback loop.

### Test Engines, Environments, and Data Management

**Test engines:** Different test types require different engines. For Apex tests, the engine is the built-in Apex test runner. For manual tests, the "engine" is a human. For all other types, you need external tools (Jest, Selenium, PMD, JMeter, etc.). Some tools (like static analysis tools) can serve at multiple levels -- linting, quality gates, and full codebase scans.

**Test environments -- the "shift left" strategy:** Batch multiple testing demands into as few environments as possible. Short-lived testing environments (scratch orgs, developer sandboxes) should be created and destroyed automatically as part of CI. Long-lived environments for tests requiring external integrations should be created manually from **Environments > Sandboxes** in production.

**Test data management:** Most tests require data. Managing test data is an integral part of test setup. Key principles:
- Code-based tests should create their own data within the test (data silo pattern).
- UI tests and manual tests may share a common data set loaded via API or CSV.
- Use tools to anonymize production data for realistic testing scenarios.
- Maintain test data in version control when possible.

### Summary Table: Test Types at a Glance

| Test Type | Automated? | Environment | Speed | Purpose | Tools |
|-----------|-----------|-------------|-------|---------|-------|
| Linting | Yes | IDE (local) | Real time | Coding style, common faults | PMD, SonarLint, ESLint |
| Quality gates | Yes | CI engine | Fast | Code issue overview, duplicates, trends | PMD, SonarQube, Clayton, Copado |
| Unit tests | Yes | Scratch org, dev sandbox, or local | < 5 min total | Fast feedback for developers | Apex runner, Jest/Mocha |
| Code-based acceptance tests | Yes | Scratch org, test sandbox, CI job | Minutes to hours (parallel) | Comprehensive regression testing | Apex runner, Jest/Mocha |
| UI tests | Yes | Scratch org, test sandbox | Minutes to hours (parallel) | Regression testing critical/complex processes | Selenium, Provar, Puppeteer, Tosca |
| Static analysis (full codebase) | Yes | CI job, static analysis tool | Fast | Tracking trends, identifying quality hotspots | SonarQube, Clayton, CodeScan, PMD, Codacy, CodeClimate |
| Security analysis | Yes | Security analysis tool | Minutes | Identifying security flaws (SOQL injection, XSS) | CheckMarx, Fortify |
| Performance testing | Yes | Full or partial sandbox (scheduled) | Minutes to hours (parallel) | Occasional targeted performance analysis | JMeter, LoadRunner, Loader.io |
| Code reviews | No | In-person or pull requests | Real time | Code quality, shared analysis, learning, collaboration | Fellow developers |
| Manual QA and acceptance tests | No | Testing sandbox | Indefinite | Exploratory testing, getting feedback from users | Mouse, keyboard, monitor, human |

## Key Takeaways

1. **Testing is not just Apex tests.** A mature testing strategy spans ten distinct test types across three quality dimensions. Most Salesforce teams only use one (Apex unit tests) and underinvest in the others.

2. **Start with linting and quality gates -- they are free wins.** Static analysis provides fast, automated code quality feedback with no risk and no test data required. ApexPMD for Apex and ESLint for JavaScript should be in every Salesforce developer's IDE. Quality gates on pull requests enforce the same rules at merge time.

3. **Write tests as executable specifications using BDD.** Name test methods with `itShould...`, structure bodies as Given/When/Then, and use `System.assertEquals` with descriptive failure messages. Each test becomes a living specification of expected behavior.

4. **Never use `@isTest(SeeAllData=true)`.** Tests that depend on org data are brittle, org-dependent, and break unpredictably. Always create your own test data. Use `@TestSetup` for shared data, static resources in CSV format for large volumes, and consider the Apex Domain Builder for readable fluent test data creation.

5. **Use a central test factory for object creation.** When a new required field is added to an object, you want to update one factory method rather than dozens of test methods. The factory defines base objects with all required fields; individual tests modify only what matters to their scenario.

6. **Apex tests run during deployments -- this is one of the platform's best features.** Tests execute in the target org after metadata is deployed. If any test fails or coverage drops below 75%, the deployment rolls back atomically. JavaScript tests cannot run inside a deployment transaction and must be a separate CI pipeline stage.

7. **The 75% coverage rule is a floor, not a ceiling, and not a goal.** Aim for meaningful coverage of business-critical logic. The Pareto principle applies: 80% of the coverage value comes from 20% of the effort. When code becomes contorted just to hit coverage targets, you have gone too far.

8. **Use the "leak period" strategy for quality gates.** Rather than remediating all existing issues, enforce quality rules only on recently changed code. This naturally ratchets up quality over time without requiring a massive upfront investment.

9. **Performance testing on Salesforce requires advance coordination.** You must plan at least 2 weeks ahead and get approval from Salesforce support. Generating unusually high data volumes without permission violates Salesforce's terms of use. Use full or partial copy sandboxes, never scratch orgs.

10. **Prefer peer review over external change approval.** The State of DevOps research found that external approval bodies (change advisory boards) are negatively correlated with delivery performance. Pair programming and intrateam peer reviews provide more educated, contextual review.

11. **Manual QA testers should focus on exploratory testing.** Automate the repetitive regression tests. Reserve human testers for one-off scenarios, edge cases, and UAT with subject matter experts using realistic production-like data.

12. **UI tests are expensive -- use them sparingly.** Reserve UI tests for critical or complex processes that cannot be tested by code alone. Add explicit `Id` attributes to custom page elements (naming convention: `PageName_FieldName_FieldType`), build modular test architecture, and remove "flapper" tests immediately.
