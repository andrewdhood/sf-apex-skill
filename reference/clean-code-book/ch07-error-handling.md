# Chapter 7: Error Handling

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008), Chapter 7 by Michael Feathers
> **When to read this:** Before writing any Apex that catches exceptions, throws errors, returns nulls, or does DML with partial-success semantics.

Error handling is important, *but if it obscures logic, it's wrong*. The goal of this chapter is code that is both clean and robust — code that handles errors with grace and style without letting error handling dominate the reader's view of what the code actually does.

---

## Key Principles

1. **Use exceptions rather than return codes.** Return codes clutter the caller; the caller must check for errors immediately after every call, and it's easy to forget. Throwing an exception separates the happy path from the error path.
2. **Write your try-catch-finally statement first.** Exceptions define a scope. Starting with the try-catch-finally forces you to think about what the caller should expect no matter what goes wrong.
3. **Use unchecked exceptions.** Checked exceptions violate the Open/Closed Principle — a change at a low level can force signature changes all the way up the call stack. Encapsulation breaks because every function in the throw path must know about the low-level exception.
4. **Provide context with every exception.** A stack trace tells you where, not why. Always include a message that names the operation that failed and the type of failure.
5. **Define exception classes in terms of the caller's needs.** The most important question is *how they are caught*, not where they came from. Often a single exception class with distinguishing info on it is better than a deep hierarchy.
6. **Wrap third-party APIs.** Wrapping minimizes dependency on the vendor, gives you an API you control, and makes it easier to mock the third party when testing your own code.
7. **Define the normal flow.** Push error detection to the edges. Wrap external APIs, translate to your own exceptions, and let the bulk of your code read like a clean unadorned algorithm. Use the SPECIAL CASE PATTERN to encapsulate exceptional behavior in an object so the caller never has to deal with it.
8. **Don't return null.** Every `null` return creates work for callers and one missed check sends the app out of control. Prefer exceptions or a special-case object. If a third-party API returns null, wrap it.
9. **Don't pass null.** Passing null into methods is worse than returning it. Forbid it by default; then a null in an argument list is clearly a bug.

---

## Detailed Notes

### Use Exceptions Rather Than Return Codes
Return codes force callers to interleave error-checking with business logic, and callers routinely forget to check. Exceptions let the calling code remain clean; the two concerns — the algorithm and the error handling — become separable and can be understood independently.

### Write Your Try-Catch-Finally Statement First
A `try` block is like a transaction — the `catch` has to leave the program in a consistent state no matter what happens in the try. Start by writing a test that expects an exception for a bad input, then stub the method, then wrap the risky work in try-catch-finally so the scope is defined before you write the body. This is a TDD-style approach to error handling: force exceptions in tests, then build the try-block's "normal" behavior between them.

### Use Unchecked Exceptions
C#, C++, Python, and Ruby all do fine without checked exceptions. The price of checked exceptions is that a low-level change forces every method between the throw site and the catch site to be modified — either to catch the new exception or to append a `throws` clause. This cascades up through the system, breaks encapsulation, and is rarely worth the benefit. Checked exceptions can be useful when writing a critical library, but in general application development the dependency costs outweigh the benefits.

### Provide Context with Exceptions
Each exception you throw should contain enough context to determine the source and the intent of the failed operation. A stack trace can tell you where something failed, not what you were trying to do. Create informative messages and pass them along. If you log in `catch`, pass enough detail to log meaningfully.

### Define Exception Classes in Terms of a Caller's Needs
When you classify exceptions, the most important concern is *how they will be caught*. If three different exceptions from a third party library all get the same "log the error and report the port error" treatment, that's duplication — and a sign you should wrap the library and translate its exceptions to a single type. Wrapping third-party APIs is a best practice because:
- You minimize dependencies on the vendor.
- You can swap libraries later without rewriting all callers.
- Wrapping gives you a seam for mocking when you test your own code.
- You get an API that matches your design instead of the vendor's.
Use a single exception class for a single area of code unless you genuinely want to catch one type and let another pass through.

### Define the Normal Flow
Once you push error detection to the edges, most of your code should read as an unadorned algorithm. But sometimes you don't actually want to abort — you want a default. Don't use a try-catch to handle routine absence:
```
// awkward
try {
    MealExpenses expenses = expenseReportDAO.getMeals(employee.getID());
    m_total += expenses.getTotal();
} catch (MealExpensesNotFound e) {
    m_total += getMealPerDiem();
}
```
Instead, use the SPECIAL CASE PATTERN: change the DAO to always return a `MealExpenses` object; when there are no meals, return one that yields the per-diem as its total. The client code collapses to a single line with no exceptional path.

### Don't Return Null
Returning null creates cascades of null checks at every caller. A missing null check becomes a `NullPointerException` somewhere unrelated. If you're tempted to return null, throw an exception instead, or return a special-case object (e.g., an empty collection). Wrap third-party APIs that return null so your code never sees them.

### Don't Pass Null
Returning null is bad. Passing null is worse. You will get NPEs deep inside methods that had no idea they were receiving garbage. Throwing `InvalidArgumentException` from every method adds ceremony without fixing the problem. Assertions document intent but still blow up at runtime. The rational approach: *forbid passing null by default*. A null in an argument list then clearly indicates a bug.

---

## Salesforce/Apex Caveats

**Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### Apex has two exception families, not one
- **`AuraHandledException`** — MUST be used when throwing from `@AuraEnabled` methods consumed by LWC or Aura. Any other exception type will reach the client as `"Script-thrown exception"` with no useful detail. When you throw an `AuraHandledException`, set the message via the constructor AND call `setMessage()` on the thrown instance — a known platform quirk where the constructor message does not make it to `error.body.message` in LWC.
- **Standard exceptions** (`DmlException`, `QueryException`, `LimitException`, `CalloutException`, plus your own `extends Exception`) — use in batch, queueable, scheduled, trigger, and internal service layers where the caller is Apex.

When a service method might be called from both a Batchable and an `@AuraEnabled` method, throw a custom domain exception from the service and let the LWC-facing wrapper catch it and rethrow as `AuraHandledException`. Never make the service layer itself throw `AuraHandledException`.

### `Database.insert(records, false)` conflicts with "prefer exceptions"
This is the biggest Clean Code collision in the platform. When you use partial-success DML:
```apex
// remember that allOrNone=false returns a result array instead of throwing
Database.SaveResult[] results = Database.insert(recordsToInsert, false);
for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        // we have to walk the error array — no exception was ever raised
        for (Database.Error err : results[i].getErrors()) {
            System.debug('Row ' + i + ' failed: ' + err.getStatusCode() + ' - ' + err.getMessage());
        }
    }
}
```
This is return-code style error handling, not exception style. You do it anyway because the business requirement of "insert the good rows, log the bad ones" is common in integrations and bulk loads. Do not try to "fix" this by wrapping in try/catch with `allOrNone=true` — you will lose good rows.

**The Clean Code-compatible approach is to encapsulate the partial-success handling in a single place.** Build a `DmlResultHandler` or similar utility that takes the SaveResult array and a "context" string, and funnels failures to a logging framework (e.g., nebula-logger, or your own Error_Log__c object). From the caller's perspective, the result array becomes a single method call with no branching. The SPECIAL CASE PATTERN applies here — wrap the awkward API.

### Checked vs unchecked — Apex gives you what you want
Apex does not have checked exceptions in the Java sense. Every custom exception extending `Exception` is effectively unchecked. Feathers' advice lands naturally in Apex: define a thin hierarchy of custom exceptions rooted in a few domain classes, throw them freely, and let the caller decide whether to catch.

### `with sharing` is not an error-handling concern, but it looks like one
DML and SOQL will throw `System.QueryException` or `System.DmlException` when a user lacks object or field permissions. Do not catch these and silently fall back to "no results" — that hides a privilege bug. Let them propagate, or catch and rethrow as a domain-specific `InsufficientAccessException` so the caller can distinguish "no data" from "not allowed to see data."

### Governor limits throw exceptions that you CANNOT catch
`System.LimitException` (the one raised when you exceed SOQL queries, DML rows, CPU time, heap, callouts, etc.) is uncatchable in most cases. You will see `try { ... } catch (Exception e) { ... }` blocks that appear to handle it, but the framework re-raises. Do not rely on catching limit exceptions — prevent them through bulkification, `Limits` class pre-checks, and careful design.

### Providing context — use `addError()` in triggers
In a trigger, `record.addError('Meaningful message here')` is how you communicate failures to the UI, integration callers, and the platform's rollback machinery. It is NOT the same as throwing. `addError()` marks the record as failed but allows processing of other records in the batch to continue; throwing an exception rolls back the entire trigger execution. Pick the right one:
- **`addError()`** — per-record validation failure, bulk operations should continue processing the good rows.
- **`throw new ...`** — framework-level failure, there is no sane way to continue.

### Don't return null — return empty collections
Apex is kind about this: every SOQL query that returns zero rows gives you an empty `List`, never null. Follow the platform's lead in your own code:
```apex
// good — always returns a list
public List<Account> findEligibleAccounts(Set<Id> ids) {
    if (ids == null || ids.isEmpty()) {
        return new List<Account>();
    }
    return [SELECT Id, Name FROM Account WHERE Id IN :ids WITH USER_MODE];
}
```
However: individual SObject fields CAN be null, and `Map.get(key)` returns null for missing keys. You still have to null-check reads; just don't propagate nulls outward.

### Don't pass null — use method overloads
Apex does not support default parameter values. The Clean Code way to avoid null-as-a-flag is overloading:
```apex
// begin with the no-frills version
public void processOrders(List<Order__c> orders) {
    processOrders(orders, false);
}
// fuller version for callers who need it
public void processOrders(List<Order__c> orders, Boolean sendNotifications) {
    // ...
}
```
This beats `processOrders(myOrders, null)` at every call site.

### Try-catch-finally in a trigger context
Triggers run inside a transaction. A thrown exception rolls back everything since the last savepoint. If you need partial rollback semantics, use `Database.setSavepoint()` / `Database.rollback(sp)` inside a try/catch. This is rare and usually indicates design smell — prefer bulkified validation that prevents the error in the first place.

---

## Key Takeaways

- Prefer exceptions over return codes — EXCEPT when Apex forces you into `Database.insert(records, false)` for partial success. In that case, wrap the awkward API in a helper so callers see a clean interface.
- Throw `AuraHandledException` (and only `AuraHandledException`) from `@AuraEnabled` methods. Set the message twice (constructor + `setMessage()`) to work around the LWC serialization quirk.
- Push error detection to the edges. Wrap third-party APIs (REST callouts, managed packages, external libraries) in your own classes that translate their exceptions to your domain exceptions.
- Never return null from a list-returning method. Return `new List<SObject>()` or a special-case object.
- Don't pass null as a flag. Use method overloads instead.
- In triggers, distinguish `addError()` (per-record validation, bulk continues) from `throw` (whole-transaction abort).
- Governor-limit exceptions are uncatchable in practice. Prevent them; don't try to recover from them.
- Test code should force exceptions and verify the error path — this is where "write your try-catch-finally first" pays off.
