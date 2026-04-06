# Chapter 12: Emergence

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008), chapter by Jeff Langr
> **When to read this:** When you want a short, actionable checklist for "is my Apex design simple enough?" or when refactoring an existing class and trying to decide how far to push the cleanup.

## Key Principles

Kent Beck's **Four Rules of Simple Design**, in order of importance:

1. **Runs all the tests.**
2. **Contains no duplication.**
3. **Expresses the intent of the programmer.**
4. **Minimizes the number of classes and methods.**

Rules 2 through 4 are applied during refactoring. Rule 1 is the prerequisite that makes refactoring safe.

## Detailed Notes

### Rule 1: Runs All the Tests

A design must produce a system that *acts as intended*. You can have a beautiful design on paper, but if you can't verify the system works, the design is questionable. A system that cannot be verified should arguably never be deployed.

The hidden benefit: making your system testable *pushes* you toward better design. Testable classes tend to be small and single-purpose (conforming to SRP). Tight coupling makes tests hard to write, so writing tests drives you toward DIP, dependency injection, interfaces, and abstraction.

> Writing tests leads to better designs.

### Rules 2-4: Refactoring

Once you have tests, you're empowered to keep the code clean. For every few lines of code you add, pause, reflect on the new design, and clean it up if it degraded — run the tests to confirm you didn't break anything. *The tests eliminate the fear that cleaning up the code will break it.*

During refactoring, apply the full body of good-design knowledge: increase cohesion, decrease coupling, separate concerns, modularize, shrink functions and classes, choose better names. And apply the final three rules:

### Rule 2: No Duplication

Duplication is the primary enemy of a well-designed system. It adds work, adds risk, adds complexity. It manifests as:

- Lines of code that look exactly alike (obvious)
- Lines of code that are similar and can be massaged to look more alike (harder)
- **Duplication of implementation** — the subtle form

Langr's collection-class example: `int size()` and `boolean isEmpty()` could each track their own state (a counter and a boolean). Or `isEmpty()` can be defined in terms of `size()`:

```java
boolean isEmpty() {
    return 0 == size();
}
```

One source of truth. That's what "no duplication" means at the smallest scale.

He then shows an image-manipulation example where `scaleToOneDimension` and `rotate` both contain:

```java
image.dispose();
System.gc();
image = newImage;
```

Extract that into `private void replaceImage(RenderedOp newImage)` and call it from both. The methods become one-liners.

As you extract commonality at this tiny level, you start recognizing SRP violations. The extracted `replaceImage` might belong in a different class entirely. Someone else on the team may see further reuse. "Reuse in the small" causes system complexity to shrink dramatically.

The **TEMPLATE METHOD pattern** is the standard tool for removing higher-level duplication. Example: `VacationPolicy` with `accrueUSDivisionVacation()` and `accrueEUDivisionVacation()` — largely identical except for the legal-minimums calculation. Refactor to:

```java
abstract public class VacationPolicy {
    public void accrueVacation() {
        calculateBaseVacationHours();
        alterForLegalMinimums(); // the only variable part
        applyToPayroll();
    }
    private void calculateBaseVacationHours() { /* ... */ }
    abstract protected void alterForLegalMinimums();
    private void applyToPayroll() { /* ... */ }
}

public class USVacationPolicy extends VacationPolicy {
    @Override protected void alterForLegalMinimums() { /* US logic */ }
}

public class EUVacationPolicy extends VacationPolicy {
    @Override protected void alterForLegalMinimums() { /* EU logic */ }
}
```

Subclasses fill in the "hole" in the algorithm — the only information that isn't duplicated.

### Rule 3: Expressive

Code should clearly express the intent of its author. The majority of software cost is long-term maintenance, and the clearer the code, the less time future readers spend understanding it. That reduces defects and shrinks maintenance cost.

Ways to be expressive:

- **Choose good names.** You should be able to hear a class or function name and not be surprised by what it does.
- **Keep functions and classes small.** Small units are easier to name, write, and understand.
- **Use standard nomenclature.** Design pattern names (COMMAND, VISITOR, DECORATOR, TEMPLATE METHOD) are compact communication tools. Naming a class after a pattern it implements tells other developers a lot in few words.
- **Write well-crafted unit tests.** Tests are *documentation by example*. Reading a class's tests should give you a quick understanding of what the class does.
- **Take pride in your workmanship.** Care is a precious resource. Spend a little time with each function and class — choose better names, split large functions, take care of what you've created. The most likely next person to read the code is *you*.

### Rule 4: Minimal Classes and Methods

This rule is a counterweight to the first three. Eliminating duplication, expressing intent, and SRP can all be taken too far — you could end up with dozens of tiny classes and methods that hurt more than they help.

Resist pointless dogma: rules like "every class must have an interface" or "fields and behavior must always be in separate classes." Be pragmatic.

> Although it's important to keep class and function count low, it's more important to have tests, eliminate duplication, and express yourself.

This rule is explicitly the **lowest priority** of the four.

## Salesforce/Apex Caveats

**Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### Rule 1 (Runs All the Tests) is literally a platform mandate

The book presents "runs all the tests" as the most important rule. On Salesforce, it's not advice — it's enforced by the runtime:

- **75% code coverage minimum for production deploys.** You physically cannot push code to production without tests covering 75% of it.
- **Every trigger must have at least 1% coverage** (practically: every trigger must be exercised by a test).
- **All tests in the org must pass** when you deploy — not just the tests for the classes you're changing. A test failing in a distant, untouched module will block your deployment.
- **`@IsTest` classes do not count against the character limit** of the classes they test and do not consume production code coverage.

This means the book's "runs all the tests" rule is enforced for you, free of charge. Embrace it. When you refactor, run `sfdx force:apex:test:run` (or the MDAPI/tooling equivalent) before every commit.

### Rule 2 (No Duplication) collides with bulkification sometimes

Clean Code says "eliminate duplication." Apex says "don't do SOQL or DML in a loop." Occasionally these conflict:

```apex
// Tempting: extract a helper "per-record" method (no duplication between callers)
private void processOne(Account a) {
    // does its own SOQL — governor-limit disaster if called in a loop
    Contact c = [SELECT Id FROM Contact WHERE AccountId = :a.Id LIMIT 1];
    // ... logic
}
```

This helper looks clean but cannot be called from a loop without blowing the SOQL limit. The correct Apex pattern is to do bulk queries up front, build maps, and then iterate:

```apex
// Bulk-friendly: query once, iterate over the results
public void process(List<Account> accounts) {
    // gather all contact ids in a single query
    Map<Id, Contact> primaryContactByAccount = new Map<Id, Contact>();
    for (Contact c : [
        SELECT Id, AccountId FROM Contact
        WHERE AccountId IN :accounts
    ]) {
        primaryContactByAccount.put(c.AccountId, c);
    }

    // now iterate — no SOQL in the loop
    for (Account a : accounts) {
        Contact c = primaryContactByAccount.get(a.Id);
        // ... logic using c
    }
}
```

There's some "duplication" of intent here (the query is hardcoded rather than delegated to a per-record helper), but the bulk pattern is non-negotiable. Use private helpers for *calculations* inside the loop, not for *queries*.

### Rule 2 applied well: Trigger Handler base classes

The best Apex application of "no duplication" is a base `TriggerHandler` class (e.g., the Kevin O'Hara pattern). Every trigger handler inherits from it, getting bypass-checking, recursion-prevention, and dispatch logic for free. Subclasses override `beforeInsert`, `afterInsert`, etc. This is TEMPLATE METHOD from the book, applied directly.

### Rule 3 (Expressive) — Apex-specific naming conventions

On Salesforce, "expressive" means:

- **Class suffixes signal role.** `AccountSelector`, `AccountService`, `AccountDomain`, `AccountTriggerHandler`, `AccountController`, `AccountRepository`. The suffix tells reviewers where the class fits in the architecture.
- **Custom object/field API names should be self-explanatory.** `Annual_Revenue__c` is good. `Field1__c` is malpractice.
- **Test method names should read as specifications.** `shouldThrowWhenAccountIsNull()`, `shouldPersistSingleRecordWhenBulkProcessing()`, `shouldBypassValidationWhenRunAsIntegrationUser()`. Use `System.assertEquals` with a meaningful message.
- **Design pattern names** still work: `AccountVisitor`, `AccountObserver`, `OpportunityFactory`.
- **`@TestVisible`** is your tool for keeping production methods private while still letting tests see them. Use it instead of loosening visibility to `public`.

### Rule 4 (Minimal Classes and Methods) bites harder in Apex

In Java/JVM systems, having 500 tiny classes is nearly free. In Apex it's expensive:

- Every class is a `.cls` metadata file.
- Deploys take longer.
- Every class needs its own test coverage tally.
- Code reviews get noisy.
- The org's class count limit (though large) is finite.
- Package installations and upgrades touch every file.

So Apex developers should lean *harder* toward rule 4 than Java developers. When tempted to extract a 5-line helper into its own class, ask: does this class have a real reason to exist independently, or am I cargo-culting SRP? The book itself says this is the lowest-priority rule — honor that.

### The Template Method pattern works beautifully in Apex

The `VacationPolicy` example ports cleanly:

```apex
public abstract class VacationPolicy {
    // the algorithm skeleton — same for all regions
    public void accrueVacation() {
        calculateBaseVacationHours();
        alterForLegalMinimums(); // the variable step
        applyToPayroll();
    }

    // shared logic — subclasses don't see or override these
    private void calculateBaseVacationHours() {
        // compute base hours based on tenure
    }

    // the one thing that actually differs between regions
    protected abstract void alterForLegalMinimums();

    private void applyToPayroll() {
        // push the accrued hours into Payroll__c
    }
}

public class USVacationPolicy extends VacationPolicy {
    protected override void alterForLegalMinimums() {
        // US-specific minimum enforcement
    }
}

public class EUVacationPolicy extends VacationPolicy {
    protected override void alterForLegalMinimums() {
        // EU-specific minimum enforcement
    }
}
```

Note Apex-specific syntax: `protected override`, and the base class must be `public abstract` (or `virtual`). `private` methods are invisible to subclasses in Apex just as in Java, so the "hooks" the subclass fills in must be `protected`.

### Tests as documentation — especially important on Salesforce

The book's point that "well-written unit tests act as documentation" is extra-valuable on Salesforce because:

- New team members often inherit orgs with little written documentation.
- Declarative config (Flows, Validation Rules, Workflow) can change silently — the only record of *expected* Apex behavior is often in the test class.
- Test methods can be read by admins/architects who don't write Apex daily.

Invest in test method names that read like requirements. Use `System.assertEquals(expected, actual, 'explanation of why this should be true')`. Future you will thank present you.

## Key Takeaways

- **Rule 1 is free on Salesforce — the platform enforces it.** 75% coverage minimum, all tests must pass on deploy. Use that discipline to drive refactoring.
- **Kill duplication, but not at the cost of bulkification.** Extract calculation helpers, not per-record SOQL/DML helpers.
- **Use the Template Method pattern for region/tenant/variant logic** — `accrueVacation`, `calculateTax`, `formatStatement`, etc. It's the single most useful GoF pattern in Apex.
- **Expressive means Apex-convention names** (`AccountSelector`, `AccountService`, `AccountTriggerHandler`) plus descriptive test method names that read as specifications.
- **`@TestVisible` is the Apex answer to "loosening encapsulation for tests."** Prefer it over making things `public`.
- **Lean hard on rule 4 in Apex.** Every class is a metadata file and a code-review headache. Small-but-purposeful beats tiny-and-dogmatic.
- **Your test class is your documentation.** Write it that way.
- **The four rules, Apex-flavored, in priority order:**
  1. Tests pass (platform-enforced)
  2. No duplication — except where bulkification demands it
  3. Expressive names and Apex conventions
  4. Keep class count reasonable — deploy friction is real
