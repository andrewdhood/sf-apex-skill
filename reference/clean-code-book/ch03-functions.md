# Chapter 3: Functions

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008)
> **When to read this:** Before writing any non-trivial Apex method — and especially before writing a trigger handler, a batch `execute`, an invocable method, or any method that touches SOQL or DML.

## Key Principles

- **Functions should be small. Then smaller than that.** Rarely more than 20 lines; often 2-5.
- **Do One Thing. Do it well. Do it only.** If you can extract a named function from it, it was doing more than one thing.
- **One level of abstraction per function.** Never mix high-level policy with low-level detail in the same function.
- **Follow the Stepdown Rule:** Read the code as a top-down narrative where each function leads to the next level of abstraction.
- **Switch statements should be buried in a single abstract factory and hidden behind polymorphism** — tolerated once, never repeated.
- **Use descriptive names.** Long descriptive names beat short cryptic names. Don't fear renaming.
- **Prefer zero arguments; one is OK; two is harder; three should be avoided; more than three requires argument objects.**
- **No flag arguments.** A boolean parameter shouts "this function does more than one thing."
- **No side effects.** A function that claims to do X must not silently also do Y.
- **Output arguments are confusing.** Use return values or member state.
- **Command-Query Separation.** A function either *does* something or *answers* something. Not both.
- **Prefer exceptions to returning error codes.** Extract try/catch bodies into their own functions. Error handling is one thing.
- **Don't Repeat Yourself.** Duplication is the root of much evil.
- **Structured programming (single return, no break/continue) is unnecessary in small functions.** Multiple returns are fine when they clarify intent. `goto` is still out.
- **You write clean functions by drafting them messy and then refactoring while tests stay green.**

## Detailed Notes

### Small!

- The first rule is functions should be small. The second rule is they should be smaller than that.
- Functions should hardly ever be 20 lines long. Two, three, or four lines is common and good.
- Each function should tell a story and lead to the next one in compelling order.

### Blocks and Indenting

- Blocks inside `if`, `else`, `while` should be one line long — probably a function call.
- The called function gets a nicely descriptive name, adding documentary value.
- Functions should not be large enough to hold nested structures. **Indent level should not exceed one or two.**

### Do One Thing

**FUNCTIONS SHOULD DO ONE THING. THEY SHOULD DO IT WELL. THEY SHOULD DO IT ONLY.**

How do you know a function is doing one thing? If every step is at **one level of abstraction below** the function's name, it's doing one thing. You can describe it as a "TO paragraph":

> TO RenderPageWithSetupsAndTeardowns, we check to see whether the page is a test page and if so, we include the setups and teardowns. In either case we render the page in HTML.

Another test: you cannot meaningfully extract another function from it whose name is not simply a restatement of the implementation.

**Sections within a function** (e.g., declarations, initializations, sieve) are a symptom of doing more than one thing. If you can carve it into labeled sections, split it into separate functions.

### One Level of Abstraction per Function

Mixing high-level (`getHtml()`), mid-level (`String pagePathName = PathParser.render(pagePath);`), and low-level (`.append("\n")`) in one function is confusing. Readers can't tell which expressions are essential concepts and which are details. Mixed abstraction levels are like broken windows: they attract more mess.

### Reading Code from Top to Bottom: The Stepdown Rule

Code should read like a top-down narrative. Every function is followed by those at the next level of abstraction. You descend one level at a time.

Writing functions that stay at a single level of abstraction is hard to learn but essential. Read through the refactored `SetupTeardownIncluder` (Listing 3-7) to see the Stepdown Rule in action.

### Switch Statements

- Switch statements inherently do N things. It's hard to make them small and hard to make them do one thing.
- A switch violates Single Responsibility (more than one reason to change) and Open/Closed (must be modified when new cases are added).
- **Acceptable usage:** appear only once, used to create polymorphic objects, hidden inside an Abstract Factory, and the rest of the system uses the resulting polymorphism.

```java
// Bad: switch scattered across many functions
public Money calculatePay(Employee e) {
    switch (e.type) {
        case COMMISSIONED: return calculateCommissionedPay(e);
        case HOURLY:       return calculateHourlyPay(e);
        case SALARIED:     return calculateSalariedPay(e);
    }
}

// Good: switch hidden in factory
EmployeeFactory factory;
Employee e = factory.makeEmployee(record);
e.calculatePay();   // polymorphic dispatch
e.isPayday();
e.deliverPay(pay);
```

### Use Descriptive Names

- A long descriptive name beats a short enigmatic name or a long comment.
- Spend time naming; modern IDEs make renaming trivial. Try several names and read the code with each.
- Choosing good names helps you clarify the design in your head and often triggers favorable restructuring.
- Be consistent: `includeSetupAndTeardownPages`, `includeSetupPages`, `includeSuiteSetupPage`, `includeSetupPage` — consistent phraseology lets the sequence tell a story.

### Function Arguments

The ideal number of arguments: **zero (niladic)**. Next: **one (monadic)**. Then: **two (dyadic)**. Three arguments (triadic) should be avoided. More than three (polyadic) needs very special justification — and still shouldn't be used.

Arguments cost conceptual power:
- They are at a different level of abstraction than the function name and force you to know details you don't want to know at that moment.
- They make testing combinatorially harder.

**Common monadic forms:**
- Asking a question: `boolean fileExists("MyFile")`
- Transforming and returning: `InputStream fileOpen("MyFile")`
- Event (input only, no return, state change): `void passwordAttemptFailedNtimes(int attempts)` — use with care, make it clear to the reader.

**Avoid:** monads that use output arguments instead of return values. `StringBuffer transform(StringBuffer in)` is better than `void transform(StringBuffer in, StringBuffer out)`.

**Flag arguments are ugly.** Passing a boolean is a loud admission that the function does more than one thing. Split `render(boolean isSuite)` into `renderForSuite()` and `renderForSingleTest()`.

**Dyadic functions** are harder than monads. Even `assertEquals(expected, actual)` has an ordering problem you must learn. Consider: making one argument a field, extracting a class, or converting to a monad on a new object.

**Triads** are significantly harder than dyads. `assertEquals(message, expected, actual)` is a classic trap — readers initially mistake `message` for `expected`.

**Argument objects:** when you need >2 arguments, wrap related ones into a class.
```java
Circle makeCircle(double x, double y, double radius);  // not great
Circle makeCircle(Point center, double radius);         // better
```

**Argument lists:** `String.format(String format, Object... args)` is effectively dyadic. Variadic functions can still be monadic, dyadic, or triadic — don't hand them more.

**Verbs and Keywords:** a monad should form a verb/noun pair with its argument (`write(name)`, better `writeField(name)`). The *keyword form* encodes argument names into the function name: `assertExpectedEqualsActual(expected, actual)` beats `assertEquals(expected, actual)` for readability.

### Have No Side Effects

A side effect is a lie. Your function promises to do one thing but secretly also does another. Example:

```java
public boolean checkPassword(String userName, String password) {
    // ... validates password ...
    if (...) {
        Session.initialize();   // ← side effect! not in the name
        return true;
    }
    return false;
}
```

The side effect creates a **temporal coupling** — the function can only safely be called at certain times. If you must have a temporal coupling, put it in the name (`checkPasswordAndInitializeSession`) — though that violates "Do One Thing" and you should probably split.

**Output arguments** are a specific kind of side effect. `appendFooter(s)` — is `s` the thing getting a footer appended to it, or the footer being appended? Ambiguous. In OO, prefer `report.appendFooter()`. **If your function must change state, have it change the state of its owning object.**

### Command Query Separation

Functions should either:
- **Do** something (a command), or
- **Answer** something (a query)

Not both. `public boolean set(String attribute, String value)` is confusing — readers can't tell if `if (set("username", "unclebob"))` asks "was it previously set?" or "did we successfully set it?" Split into a query and a command:

```java
if (attributeExists("username")) {
    setAttribute("username", "unclebob");
}
```

### Prefer Exceptions to Returning Error Codes

Returning error codes is a subtle CQS violation — commands become expressions in `if` conditions. It forces callers to handle errors immediately, creating deeply nested structures.

```java
// Bad: nested error checks
if (deletePage(page) == E_OK) {
    if (registry.deleteReference(page.name) == E_OK) {
        if (configKeys.deleteKey(page.name.makeKey()) == E_OK) {
            logger.log("page deleted");
        } else { ... }
    } else { ... }
} else { ... }

// Good: exceptions separate happy path from error path
try {
    deletePage(page);
    registry.deleteReference(page.name);
    configKeys.deleteKey(page.name.makeKey());
} catch (Exception e) {
    logger.log(e.getMessage());
}
```

**Extract try/catch blocks.** They are ugly and mix error processing with normal processing. Pull the try and catch bodies into their own functions:

```java
public void delete(Page page) {
    try {
        deletePageAndAllReferences(page);
    } catch (Exception e) {
        logError(e);
    }
}
```

**Error handling is one thing.** If `try` appears in a function, it should be the first word and there should be nothing after the `catch`/`finally` blocks.

**`Error.java` dependency magnet:** an enum of error codes becomes a class that everyone imports. Adding a new error forces recompile/redeploy of everything. Exceptions are derivatives of the exception class and can be added without breaking dependents.

### Don't Repeat Yourself

Duplication may be the root of all evil in software. Much of software engineering history — structured programming, OO, AOP, normalization in databases — is about eliminating duplication. Even subtle duplication (the same algorithm scattered across four cases) bloats code, multiplies the cost of change, and multiplies the chance of omission errors.

### Structured Programming

Dijkstra said functions should have one entry and one exit (one `return`, no `break`/`continue`, no `goto`). In small functions, this is unnecessary — multiple returns often make code clearer. In large functions, it helps. `goto` is bad at all sizes.

### How Do You Write Functions Like This?

You don't. You write them long and clumsy first, with long argument lists, duplicated code, bad names. Then you **refactor with a suite of unit tests protecting you**: extract methods, rename, eliminate duplication, shrink, reorder. Nobody writes clean functions on the first draft.

## Salesforce/Apex Caveats

**⚠ Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

This is the chapter where Clean Code and Apex collide most violently. Read carefully.

### 1. DML must happen before callouts — ordering beats "Do One Thing"

Salesforce forbids DML after an HTTP callout in the same transaction. Clean Code would push callout logic and DML logic into separate "one thing" functions and let the caller compose them. That composition is dangerous because the order matters: a future developer refactoring for "clarity" can break transactions by flipping the call order. Document the ordering in comments, and consider a single orchestration method (intentionally not "doing one thing") that enforces the sequence. **Platform wins.**

### 2. No SOQL, DML, or callouts inside loops — hoist them out, even at the cost of function purity

Clean Code would put `accountToUpdate = queryAccount(id)` in a helper and call it inside the loop. In Apex, that is an automatic failure at bulk scale — every iteration burns a SOQL query, and you hit the 100 SOQL limit on the 101st record. **Always hoist queries above the loop, collect IDs into a Set, and query once.** Your "helper function" should take `Set<Id>` and return `Map<Id, SObject>`. This directly conflicts with "functions take 0-1 arguments and do one thing on one item," but it is non-negotiable.

```apex
// WRONG — Clean Code "one thing per function" trap
for (Id accId : accountIds) {
    Account a = queryAccount(accId);   // SOQL in loop
    updateAccount(a);                   // DML in loop
}

// RIGHT — bulkified, hoisted
Map<Id, Account> accts = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :accountIds]
);
List<Account> toUpdate = new List<Account>();
for (Account a : accts.values()) {
    // begin massaging each account in memory
    a.Name = a.Name + ' (updated)';
    toUpdate.add(a);
}
update toUpdate;   // one DML for the whole collection
```

### 3. Bulkification mandates `List<>` arguments — not monads

Clean Code wants functions to take one item. Apex triggers, invocable methods, batch `execute`, and Queueable handlers *must* take collections. Writing `handleAccount(Account a)` and calling it in a trigger loop is a bulkification bug waiting to happen. **Write `handleAccounts(List<Account> accts)` — polyadic by necessity.** This is not a violation of Clean Code so much as a different optimization frontier: in Java, you optimize for readability of single-item flow; in Apex, you optimize for governor-limit safety at bulk scale.

### 4. Trigger handlers cannot be "small" in the Clean Code sense

A `beforeUpdate` handler often needs to: gather related record IDs, query related records in bulk, iterate, apply business rules, collect DML targets, and call DML. That is 5+ concerns. You cannot break them into 5 top-level functions without duplicating the iteration. The compromise: keep the orchestration method short (~20 lines), push each concern into a helper method that takes collections, and resist the temptation to sprinkle SOQL across helpers.

### 5. Switch statements on SObject record types and picklists are common and often justified

Clean Code says bury every switch in an abstract factory. In Apex, you cannot easily create polymorphic subclasses per Record Type — RecordType is a runtime property, not a subclass. A `switch on opportunity.StageName` or `switch on recordTypeDevName` is idiomatic Apex. Try to centralize it in one strategy-selector method, but do not contort yourself into a factory hierarchy that fights the platform's data model.

### 6. Exceptions vs error codes — Apex sides firmly with exceptions

Apex has rich exception types (`DmlException`, `QueryException`, custom `extends Exception`), and the platform's Database methods return `SaveResult[]` which can look like error codes. Prefer real exceptions. When you must use `Database.insert(records, false)` (partial success), loop the results and throw your own exception or aggregate errors — don't let error codes leak upward.

### 7. `System.debug()` not `System.out.println()`

Java's `System.out.println` is stdout. Apex uses `System.debug()` which writes to the debug log and supports log levels. Any Clean Code example showing println should be read as debug in Apex.

### 8. Command-Query Separation holds, but watch `Database.insert` and `upsert`

`Database.insert(records)` is a command (mutates), a query (returns SaveResults), *and* it has side effects on the input list (populates Id fields). This is an unavoidable violation of CQS baked into the platform. Treat it as a command and read the results as necessary output — not as a CQS failure to "fix."

### 9. Flag arguments creep in via `Database.insert(records, allOrNone)`

The platform uses boolean flags extensively. When you write wrapper methods, resist the temptation to pass the flag through. Prefer `upsertAllOrNone(records)` and `upsertPartial(records)` as separate methods.

### 10. Side-effect "No No" meets trigger context automatically

In trigger context, `Trigger.new` and `Trigger.old` are implicit side-effect channels. A handler method that modifies a field on a record in `Trigger.new` is a side effect relative to its signature but is the explicit point of `before` triggers. Document it. Name the method `setDefaultsOnInsert(List<Account> newAccounts)` so the side effect is at least in the name.

### 11. `with sharing` / `without sharing` / `inherited sharing` is not a Clean Code concern — but it is non-negotiable

Every class that touches data should declare its sharing mode. Default (undeclared) inherits from the caller, which is a bug farm. **Always specify sharing explicitly.** This isn't about function design but it belongs on every class-level checklist.

### 12. 75% test coverage is platform-mandated

Clean Code treats tests as craftsmanship. Salesforce treats tests as a deployment gate. Functions must be structured to be testable not just for quality but because you literally cannot deploy otherwise. Keep methods small enough to mock or stub their dependencies (dependency injection or TestVisible stubs).

### 13. Governor limits may force duplication that Clean Code would forbid

Sometimes the DRY fix (a helper method that does a SOQL query) defeats bulkification. Sometimes the cleanest-looking code hits CPU time limits because extra method calls add up at scale. When DRY and governor limits conflict, **measure and bulkify first, then extract helpers within the bulk-safe pattern.** Don't DRY at the cost of blowing SOQL limits.

### 14. Method-local variables and hot loops

Clean Code is unconcerned with allocation; Apex has CPU time limits (10s sync, 60s async). Creating lots of short-lived objects in a 50,000-row batch iteration can bite. When profiling shows CPU pressure, consider inlining small helpers — but only after measuring.

## Key Takeaways

Write small, single-purpose functions with descriptive names, few arguments, no flag parameters, no side effects, and clear separation between commands and queries. Use exceptions instead of error codes and extract try/catch bodies into their own functions. Draft ugly code first, then refactor with tests protecting you. **But in Apex, governor limits and bulkification override the "Do One Thing" principle whenever they conflict:** hoist SOQL and DML out of loops even when that makes a function do "more than one thing," always operate on `List<>` inputs in triggers and invocables, and accept that trigger handlers will be larger than Clean Code ideals. Salesforce platform rules — DML-before-callouts, no SOQL in loops, `with sharing`, 75% coverage, `System.debug` — always win when they conflict with Clean Code style.
