# Chapter 2: Meaningful Names

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008), chapter by Tim Ottinger
> **When to read this:** Before naming any Apex class, method, variable, SObject alias, or custom field — and any time you feel tempted to write `acc`, `temp`, `data`, `info`, or `list1`.

## Key Principles

- **Names should reveal intent.** If a name needs a comment to explain what it means, the name is wrong.
- **Avoid disinformation.** Don't use words whose established meanings conflict with yours (don't call a `Set` an `accountList`).
- **Make meaningful distinctions.** `a1`, `a2`, `a3` and `ProductInfo` vs `ProductData` are noise, not distinction.
- **Use pronounceable, searchable names.** If you can't say it out loud, you can't discuss it. If you can't grep for it, you can't find it.
- **Avoid encodings.** No Hungarian notation, no `m_` prefixes, no `IShapeFactory` interface prefixes.
- **Classes are nouns; methods are verbs.** `Customer`, `Account` vs `postPayment`, `deletePage`, `save`.
- **One word per concept.** Don't mix `fetch`, `retrieve`, and `get` for the same idea across classes.
- **Don't pun.** Don't reuse the same word (`add`) for semantically different operations.
- **Use solution-domain names when programmers will read the code; use problem-domain names when only domain experts will.**
- **Add meaningful context via containing classes/namespaces; don't add gratuitous context via prefixes.**

## Detailed Notes

### Use Intention-Revealing Names

A name should answer: why does this exist, what does it do, how is it used? `int d; // elapsed time in days` is wrong — write `int elapsedTimeInDays` instead.

Before:
```
public List<int[]> getThem() {
    List<int[]> list1 = new ArrayList<int[]>();
    for (int[] x : theList)
        if (x[0] == 4)
            list1.add(x);
    return list1;
}
```
After (by renaming only):
```
public List<int[]> getFlaggedCells() {
    List<int[]> flaggedCells = new ArrayList<int[]>();
    for (int[] cell : gameBoard)
        if (cell[STATUS_VALUE] == FLAGGED)
            flaggedCells.add(cell);
    return flaggedCells;
}
```

### Avoid Disinformation

- Do not use words with established alternate meanings (`hp`, `aix`, `sco`).
- Do not call a collection a `List` unless it is actually a `List`. Prefer `accountGroup`, `bunchOfAccounts`, or just `accounts`.
- Beware names that differ only in small ways (`XYZControllerForEfficientHandlingOfStrings` vs `XYZControllerForEfficientStorageOfStrings`).
- Never use lowercase `l` or uppercase `O` as variable names — they look like `1` and `0`.

### Make Meaningful Distinctions

- Number-series names (`a1, a2, ..., aN`) are noninformative. Rename to `source` and `destination`.
- Noise words (`Info`, `Data`, `an`, `the`, `variable`, `table`) add nothing. `ProductInfo` vs `ProductData` is a distinction without a difference. `NameString` is worse than `Name`.
- `getActiveAccount()`, `getActiveAccounts()`, `getActiveAccountInfo()` — how is anyone supposed to know which to call?
- Distinguish names so the reader can tell what the difference *is*.

### Use Pronounceable Names

Programming is a social activity. You must be able to say the name aloud in a code review. `genymdhms` (generation year month day hour minute second) is indefensible; `generationTimestamp` is obvious.

### Use Searchable Names

- Single-letter names and magic numbers cannot be found with search tools.
- The number `7` returns thousands of matches; `MAX_CLASSES_PER_STUDENT` returns the one you want.
- The letter `e` is the most common letter in English and a terrible variable name.
- Rule: **the length of a name should correspond to the size of its scope**. Single-letter names are OK *only* as local variables inside short methods (loop counters `i`, `j`, `k`).

### Avoid Encodings

- **Hungarian notation:** Dead. Modern IDEs and strongly typed languages render it obsolete. `phoneString` is wrong the moment someone changes the type.
- **Member prefixes (`m_`):** Unnecessary. Classes should be small enough that members are obvious.
- **Interface prefixes (`IShapeFactory`):** Prefer unadorned interface names. If you must encode something, encode the *implementation* (`ShapeFactoryImpl`), not the interface.

### Avoid Mental Mapping

Don't make readers translate `r` in their head into "the lowercased version of the URL with host and scheme removed." Clarity is king. Professionals write code others can understand.

### Class Names and Method Names

- Class names: nouns or noun phrases. `Customer`, `WikiPage`, `Account`, `AddressParser`. Avoid `Manager`, `Processor`, `Data`, `Info`. A class name should not be a verb.
- Method names: verbs or verb phrases. `postPayment`, `deletePage`, `save`.
- Accessors, mutators, and predicates: prefixed with `get`, `set`, `is` per JavaBean standard.
- When constructors are overloaded, use static factory methods with descriptive names: `Complex.FromRealNumber(23.0)` is better than `new Complex(23.0)`. Consider making the constructor private to enforce use of the factory.

### Don't Be Cute

`HolyHandGrenade()` might be funny; `DeleteItems()` is clear. Don't use `whack()` for `kill()` or `eatMyShorts()` for `abort()`. Clarity over entertainment. Say what you mean.

### Pick One Word per Concept

Pick one word for one abstract concept and stick with it across the codebase. Having `fetch`, `retrieve`, and `get` on equivalent methods across different classes is confusing. Having a `controller`, a `manager`, and a `driver` that do similar things is confusing. **A consistent lexicon is a great boon to programmers.**

### Don't Pun

Don't reuse the same word for two different purposes. If most of your `add` methods combine two values into a new one, don't then name a method `add` that inserts a single item into a collection — use `insert` or `append`.

### Use Solution Domain Names

Programmers will read the code. Use CS terms, algorithm names, pattern names, math terms. `AccountVisitor` means a lot to anyone who knows the Visitor pattern. `JobQueue` is clearer than a domain-speak alternative.

### Use Problem Domain Names

When there is no programmer-ese for what you're doing, use the name from the problem domain. At least a maintainer can ask a domain expert. Separating solution-domain from problem-domain concepts is part of good design.

### Add Meaningful Context

Most names are not meaningful by themselves. Place them in context via classes or namespaces.

- `firstName`, `lastName`, `street`, `houseNumber`, `city`, `state`, `zipcode` — together they suggest an address, but `state` seen alone is ambiguous.
- Solution: create an `Address` class. Now `state` in context is obvious.
- Fallback: prefixes like `addrState`. Not as good as a class, but better than nothing.

### Don't Add Gratuitous Context

- In an app called "Gas Station Deluxe," don't prefix every class with `GSD`. You're fighting your IDE's autocomplete.
- Shorter names are better than longer ones, as long as they are clear.
- `accountAddress` is a fine *instance* name but a poor *class* name. `Address` is fine for a class; if you need to distinguish, use `PostalAddress`, `MAC`, `URI`.

## Salesforce/Apex Caveats

**⚠ Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

1. **SObject API names have platform-mandated suffixes.** Custom objects end in `__c`, custom fields end in `__c`, namespace-scoped metadata has `namespace__` prefixes. These are *not* encodings you can avoid — they are required by the platform. Do not fight them. `Account.Custom_Field__c` is correct; `Account.CustomField` is not valid.

2. **Trigger handler and test class conventions may look like "encodings" but are standard.** `AccountTriggerHandler`, `AccountTriggerHandlerTest`, `AccountService`, `AccountSelector` — these suffixes communicate architectural role in a platform where classes can't be organized into real packages or namespaces the way Java can. The `_Test` or `Test` suffix is *strongly conventional* in Apex. Follow the convention. Clean Code's "no encodings" rule yields to platform convention here.

3. **`Manager`, `Processor`, `Handler`, `Service` are discouraged by Clean Code but common in Apex.** Names like `AccountTriggerHandler` or `OpportunityService` are idiomatic in Apex because they mirror the Enterprise Patterns (fflib / Apex Common) that most serious Apex orgs use. Clean Code would call these noise words. In Apex, they communicate architectural role. Prefer them when they do real work; avoid them when they are fig leaves for "I don't know what this class does."

4. **`get`/`set` prefixes matter extra in Apex because of Visualforce and LWC binding.** Apex properties use `get`/`set` not just as convention but as the syntax required for Visualforce getter/setter binding. Don't rename them away.

5. **SOQL alias/relationship names are the one place where terse is acceptable.** `SELECT Id, (SELECT Id FROM Opportunities) FROM Account` — the subquery's implicit alias is fine; you don't need to rename SObject relationship names. But when you assign query results to variables, use meaningful names: `accountsWithOpps`, not `results`.

6. **Standard object names (`Account`, `Contact`, `Lead`, `Opportunity`) are the problem domain.** Don't rename them in variable names. `Account acc = ...` is worse than `Account account = ...`, but renaming `Account` to `Customer` in your variable name is disinformation — it implies a different object.

7. **Test method naming has a Salesforce convention.** `testMethodName_condition_expectedResult` or `givenX_whenY_thenZ`. Long descriptive test names are *good* in Apex because Apex Test Runner output shows them and you want to diagnose failures quickly. Clean Code agrees with long descriptive names; Apex amplifies the need.

8. **Avoid `list` / `map` / `set` as variable names — they are reserved type-ish keywords.** `List<Account> accountList` is tolerated and common in Apex despite Clean Code's rule against encoding type. The reason is pragmatic: Apex developers are often reading quickly through code that mixes collection types, and the suffix helps. It's not forbidden here, just discouraged when a better problem-domain name is available.

9. **Namespace collisions with managed packages matter.** If your org has managed packages with common class names, your naming must not collide. This is a platform-specific form of "add meaningful context" — sometimes you need a prefix not for the reader but for the compiler.

## Key Takeaways

Names are the single largest lever for readability in any codebase, and Apex is no exception. Make every name reveal intent; avoid disinformation, noise words, and cute jokes; use one word per concept across the whole codebase. In Apex, respect platform naming requirements (`__c`, namespace prefixes) and idiomatic architectural suffixes (`Service`, `Selector`, `TriggerHandler`) — these are not encodings to be stripped but load-bearing convention. Long descriptive names pay off especially in test methods, where Apex Test Runner output depends on them. When you feel a name is wrong, rename it now — the cost of renaming in a modern IDE is trivial, and the cost of leaving a bad name in place compounds forever.
