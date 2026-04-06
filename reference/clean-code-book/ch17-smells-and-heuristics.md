# Chapter 17: Smells and Heuristics

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008)
> **When to read this:** When reviewing Apex code (your own or someone else's) and you need a vocabulary for what's wrong — or when you're writing new code and want a checklist of things to not do.

---

## Key Principles

Chapter 17 is Martin's catalog of ~70 code smells, each with a category prefix (C = Comments, E = Environment, F = Functions, G = General, J = Java, N = Names, T = Tests). The chapter is meant to be read top-to-bottom once as an education and then used as a reference.

The meta-principle: **craftsmanship comes from values, not rules.** The list exists to imply a value system, not to be mechanically applied.

This distillation covers the ~25-30 smells most relevant to Apex development, grouped by theme. Java-specific smells (J1-J3) are noted but not expanded because Apex doesn't have imports or interface-inherited constants in the same way. Every smell gets: **name, problem in one sentence, fix in one sentence, Apex relevance.**

---

## Detailed Notes (grouped by theme)

### Theme 1: Comments Should Help, Not Hurt

**C1 — Inappropriate Information**
- **Problem:** Comments carrying metadata (author, change history, SPR numbers) that belongs in version control clutter the file.
- **Fix:** Put metadata in git. Reserve comments for technical explanation.
- **Apex relevance:** Direct hit. Salesforce developers love stamping `// Modified by X on Y for ticket Z` in Apex headers. Stop. Git blame exists.

**C2 — Obsolete Comment**
- **Problem:** A comment that has drifted from the code it describes actively misleads readers.
- **Fix:** Update it or delete it the moment you spot it.
- **Apex relevance:** Common after refactors. The field name changed, but the ApexDoc still references the old name.

**C3 — Redundant Comment**
- **Problem:** A comment that merely restates what the code already says (`i++; // increment i`) wastes the reader's attention.
- **Fix:** Delete it, or rewrite the code so it tells the story the comment was trying to tell.
- **Apex relevance:** ApexDoc blocks that just mirror the method signature (`@param accountId the account id`) are this smell. If the name already says it, the comment is noise.

**C4 — Poorly Written Comment**
- **Problem:** A comment worth writing is worth writing well; rambling, ungrammatical, or vague comments insult the reader.
- **Fix:** Edit them like you'd edit prose. Be brief and precise.
- **Apex relevance:** The "disembodied narrator" style (`// begin sorting through accounts`, `// remember that we put xyz here for this reason`) is this smell's opposite — purposeful, conversational, precise.

**C5 — Commented-Out Code**
- **Problem:** Dead code in comments rots. Nobody knows if it's important, so nobody deletes it.
- **Fix:** Delete it immediately. Git remembers.
- **Apex relevance:** Extremely common in Apex. Delete it. If you're nervous, make a git branch first.

### Theme 2: Functions Should Be Small and Honest

**F1 — Too Many Arguments**
- **Problem:** Functions with 4+ parameters are hard to call, hard to read, and usually hint at a missing abstraction.
- **Fix:** Group related params into an object, or split the function.
- **Apex relevance:** Strong. Apex methods taking seven scalar arguments are begging for a wrapper class or a custom DTO.

**F2 — Output Arguments**
- **Problem:** Arguments that a function *modifies* violate reader expectations (we expect arguments to be inputs).
- **Fix:** Return a new value, or have the method modify `this`.
- **Apex relevance:** In Apex this appears as "trigger handler methods that mutate the `Trigger.new` list AND take a second list to fill in." Prefer returning a new list, or mutate `Trigger.new` exclusively.

**F3 — Flag Arguments**
- **Problem:** A boolean parameter loudly announces the function does two things.
- **Fix:** Split into two functions with clear names (`render()` / `renderForSuite()`, not `render(boolean)`).
- **Apex relevance:** `processAccounts(List<Account> accts, Boolean isUpdate)` should become `processAccountsForInsert` and `processAccountsForUpdate`.

**F4 — Dead Function**
- **Problem:** A method no one calls is cognitive debt on every future reader.
- **Fix:** Delete it.
- **Apex relevance:** Doubly important in Apex because dead methods still count against org code coverage calculations and can mislead coverage reports.

### Theme 3: Duplication Is The Root Of All Evil (G5)

**G5 — Duplication**
- **Problem:** Every copy-paste is a missed abstraction and a future bug (fix one, forget the other).
- **Fix:** Extract a method, a class, a template method, or a strategy.
- **Apex relevance:** This is probably the single most important smell for Apex. Trigger handlers copy-pasted across objects, utility methods duplicated in three classes, the same SOQL query in fifteen places — all this smell. Build a shared service class. Build a selector layer. Use the Template Method pattern when trigger frameworks have the same shape but different logic.
- **Martin's hierarchy of duplication (from obvious to subtle):**
  1. Identical blocks of pasted code → extract method
  2. `switch/case` or `if/else` chains repeated everywhere → replace with polymorphism
  3. Modules with similar algorithms but different details → Template Method or Strategy pattern

### Theme 4: Dead Weight

**G9 — Dead Code**
- **Problem:** Code that can never execute (unreachable branches, never-thrown exceptions caught, methods nobody calls) rots because nobody updates it.
- **Fix:** Delete on sight.
- **Apex relevance:** Extremely relevant. Apex often accumulates dead code from deprecated features, experimental triggers, and old test methods. And — critically — dead Apex classes still exist in the org until explicitly deleted, potentially with security implications if they expose `webservice` or `@AuraEnabled` methods.

**G12 — Clutter**
- **Problem:** Empty constructors, unused variables, meaningless comments, empty catch blocks — all noise that dilutes signal.
- **Fix:** Remove anything that doesn't earn its keep.
- **Apex relevance:** Empty catch blocks in Apex are particularly dangerous because they silently swallow governor-limit exceptions, DML exceptions, and callout failures. Never write `catch (Exception e) {}`.

### Theme 5: Placement and Coupling

**G6 — Code at Wrong Level of Abstraction**
- **Problem:** Mixing high-level concepts with low-level implementation details in one class (e.g., a generic `Stack` interface with a `percentFull()` method that only makes sense for bounded stacks).
- **Fix:** Move the detail-level code to a derived or specialized class. Keep the base class pure.
- **Apex relevance:** Applies to service classes that mix "business workflow" with "SOQL query construction." Split them — service layer and selector layer.

**G13 — Artificial Coupling**
- **Problem:** Putting a constant, enum, or utility function in a class just because it was convenient at the moment, even though it has nothing to do with that class.
- **Fix:** Put it where a reader would expect it.
- **Apex relevance:** Apex developers often dump constants into whatever class they were editing. Create a dedicated `Constants` class (or better, feature-specific constant holders) rather than scattering `public static final Integer BATCH_SIZE = 200;` across unrelated classes.

**G14 — Feature Envy**
- **Problem:** A method that reaches into another class's accessors/mutators more than its own data — it "envies" the other class and "wishes it were there."
- **Fix:** Move the method to the class whose data it actually uses.
- **Apex relevance:** Classic symptom: a utility class that takes an sObject and pulls eight fields off it to compute a result. That logic probably belongs on a wrapper class around the sObject, or on a dedicated service that at least takes typed parameters. Feature envy is especially sneaky in Apex because sObjects are public-field bags and every method can touch every field.

**G17 — Misplaced Responsibility**
- **Problem:** Code put in a place that's convenient for the writer but surprising to the reader.
- **Fix:** Follow the principle of least surprise. Put code where the reader will look for it.
- **Apex relevance:** "Where does the Account validation logic live?" If the answer is "in the `OpportunityTriggerHandler` because I was editing that file when I needed it," that's misplaced responsibility.

**G22 — Make Logical Dependencies Physical**
- **Problem:** Code assumes something about another module without asking for it explicitly (e.g., hardcoding a constant that should come from the other module).
- **Fix:** Turn the logical assumption into an explicit method call or parameter.
- **Apex relevance:** If your batch class hardcodes `private static final Integer BATCH_SIZE = 200;` because that's what `Database.executeBatch` defaults to, you've got a logical dependency. Reference the actual value.

### Theme 6: Conditionals and Control Flow

**G23 — Prefer Polymorphism to If/Else or Switch/Case**
- **Problem:** `switch/case` on a type code tends to repeat everywhere you deal with that type, fanning out the knowledge.
- **Fix:** Use polymorphism. Martin's "One Switch" rule: there may be no more than one switch per type selection; that one switch should create polymorphic objects that take the place of other switches.
- **Apex relevance:** Apex supports interface polymorphism. If you have `if (recordType == 'Consumer') { ... } else if (recordType == 'Business') { ... }` repeated across three classes, define an interface and two implementations. Use a factory to select the implementation once.

**G28 — Encapsulate Conditionals**
- **Problem:** Complex boolean expressions inline (`if (timer.hasExpired() && !timer.isRecurrent())`) force the reader to decode intent.
- **Fix:** Extract into a well-named method (`if (shouldBeDeleted(timer))`).
- **Apex relevance:** Direct hit. Apex triggers often have 4-clause boolean conditions. Name them.

**G29 — Avoid Negative Conditionals**
- **Problem:** `if (!buffer.shouldNotCompact())` is harder to parse than `if (buffer.shouldCompact())`.
- **Fix:** Flip the sense of the predicate.
- **Apex relevance:** Minor but applies.

**G30 — Functions Should Do One Thing**
- **Problem:** A function doing three things at three levels of detail hides all three.
- **Fix:** Extract each thing into its own well-named function.
- **Apex relevance:** This is probably the second-most-important smell for Apex after G5. Apex trigger handlers routinely grow to 300-line methods that query, filter, validate, transform, and DML all in one block. Break them up.

**G34 — Functions Should Descend Only One Level of Abstraction**
- **Problem:** A function that mixes "what I'm doing" with "how HTML tags are constructed" forces the reader to context-switch mid-read.
- **Fix:** Extract the lower-level operations into helper methods so the top-level function reads at a single level.
- **Apex relevance:** Direct hit. A method called `sendWelcomeEmail(User u)` should not contain `String body = '<html><body>Welcome ' + u.FirstName + '</body></html>';`. The template construction is a lower level; extract it.

### Theme 7: Magic Values and Named Constants

**G25 — Replace Magic Numbers with Named Constants**
- **Problem:** Raw numbers (and strings) in code don't explain themselves and can't be safely changed.
- **Fix:** Hide them behind a named constant. Note: "magic number" applies to any opaque token, including string literals like `'John Doe'` or hardcoded IDs.
- **Apex relevance:** Very strong. Hardcoded Profile IDs, RecordType DeveloperNames, Custom Metadata Type names — all should be named constants or, better, fetched from `Schema` describe calls or custom metadata.

```apex
// bad
if (u.ProfileId == '00e1x000000abcd') { ... }

// good
private static final Id READ_ONLY_PROFILE_ID = [
    SELECT Id FROM Profile WHERE Name = 'Read Only' LIMIT 1
].Id;
```

**G16 — Obscured Intent**
- **Problem:** Dense, cryptic expressions (`m_otCalc()`, Hungarian notation, magic numbers crammed together) hide what the code does.
- **Fix:** Use explanatory variables, descriptive names, and named constants.
- **Apex relevance:** Direct hit.

**G19 — Use Explanatory Variables**
- **Problem:** Complex calculations in-line force the reader to hold too many intermediate values in mind.
- **Fix:** Break the calculation into named intermediate variables.
- **Apex relevance:** Always helpful. Apex debug logs become much more useful when intermediate values have names.

### Theme 8: Consistency and Conventions

**G11 — Inconsistency**
- **Problem:** Doing the same thing two different ways in different places (naming `processVerificationRequest` once and `handleDeletionRequest` elsewhere) makes code harder to navigate.
- **Fix:** Pick a convention and stick to it.
- **Apex relevance:** Naming handler methods consistently (`handleBeforeInsert`, `handleAfterInsert`, etc.) makes trigger frameworks predictable.

**G24 — Follow Standard Conventions**
- **Problem:** Ignoring team or industry conventions creates friction for every reader who already knows them.
- **Fix:** Adopt common conventions where they exist.
- **Apex relevance:** For Apex specifically: handler/service/selector separation, `*Test` or `*_Test` for test classes, trigger-per-object, bulk-safe patterns. If your org has SFDX-style layout, follow it.

### Theme 9: Naming Things Well

**N1 — Choose Descriptive Names**
- **Problem:** Names are 90% of what makes code readable. Sloppy names force readers to trace implementation.
- **Fix:** Take the time to choose good names. Re-evaluate them as meanings drift. Names are too important to treat carelessly.
- **Apex relevance:** Apex code lives in an org for *years*. The name you pick today will be read by someone in 2030. Invest in it.

**N2 — Choose Names at the Appropriate Level of Abstraction**
- **Problem:** Names that leak implementation details (`dial(phoneNumber)` on a cable modem that doesn't dial) box you in.
- **Fix:** Name by intent, not mechanism.
- **Apex relevance:** Classic case: `SendgridEmailSender` — the name couples every caller to Sendgrid. Better: `EmailService` with a configurable provider.

**N4 — Unambiguous Names**
- **Problem:** Names that describe behavior in broad, vague terms force readers to read the body.
- **Fix:** Use longer, more precise names. Length is acceptable when it buys clarity.
- **Apex relevance:** `doProcessing()` is a smell. `recalculateAccountHealthScores()` is not, even though it's longer.

**N5 — Use Long Names for Long Scopes**
- **Problem:** Short names like `i` are fine for 5-line loops but lose meaning in 500-line methods.
- **Fix:** The longer the scope of a name, the longer and more precise the name should be.
- **Apex relevance:** Trigger handler class-level variables should be fully descriptive. Local loop counters in a 3-line `for` loop can be `i`.

**N7 — Names Should Describe Side Effects**
- **Problem:** A method named `getOos()` that *creates* the OOS if it doesn't exist is hiding a side effect behind a "get" name.
- **Fix:** Rename to reflect what it does (`createOrReturnOos()`).
- **Apex relevance:** Strong. Apex methods that say `getX()` and also perform DML are a bug magnet. Use `getOrCreateX()` or split into two methods.

### Theme 10: Precision and Correctness

**G3 — Incorrect Behavior at the Boundaries**
- **Problem:** Edge cases (empty lists, nulls, max values, zero) often break in ways intuition doesn't catch.
- **Fix:** Write a test for every boundary. Don't rely on intuition.
- **Apex relevance:** Very strong. Apex governor limits *are* boundaries: what happens at 10,001 rows? 200 records in a bulk trigger? A map with null keys? Null sObject fields? Test them.

**G4 — Overridden Safeties**
- **Problem:** Turning off compiler warnings, failing tests, or safety checks is borrowing from the future at high interest.
- **Fix:** Don't. Fix the underlying issue.
- **Apex relevance:** Turning off `@isTest(SeeAllData=true)` fear of refactoring, disabling validation rules to get data in, using `without sharing` just to get past a sharing check — all this smell. Figure out the real problem.

**G26 — Be Precise**
- **Problem:** Laziness about nulls, concurrent updates, floating-point currency, and "probably just one match" assumptions causes real bugs.
- **Fix:** Be explicit. Check for null. Use integers (or a `Money` class) for currency. Handle the concurrency case.
- **Apex relevance:** Direct hit on several Apex sharp edges:
  - **Null checks:** Apex will happily throw `NullPointerException` on `account.Parent.Name` if Parent is null.
  - **Currency:** Use `Decimal`, not `Double`. Salesforce provides it specifically for this reason.
  - **"First match" queries:** `[SELECT Id FROM Account WHERE Name = :x LIMIT 1]` assumes uniqueness. If Name isn't actually unique you'll silently process the wrong record. Either enforce uniqueness or query all and handle multiples.
  - **Row locks:** If two transactions can update the same record, assume they will.

### Theme 11: Tests

**T1 — Insufficient Tests**
- **Problem:** "Looks like enough" is not a coverage metric. A test suite should cover everything that could plausibly break.
- **Fix:** Keep asking "what could break?" until you can't think of anything more.
- **Apex relevance:** Salesforce requires 75% code coverage to deploy. That number is a *floor*, not a target. Cover boundary conditions, bulk scenarios (200 records), governor limit scenarios, sharing scenarios (run-as user), and negative paths.

**T2 — Use a Coverage Tool**
- **Problem:** You can't improve what you can't see.
- **Fix:** Use the coverage tool your platform provides.
- **Apex relevance:** Salesforce Dev Console and `sfdx force:apex:test:run` both report coverage per class. Use them to find gaps.

**T3 — Don't Skip Trivial Tests**
- **Problem:** "Too trivial to test" means zero future protection against regressions.
- **Fix:** Write them anyway. They're cheap and their documentary value is high.
- **Apex relevance:** Test the obvious stuff. A one-line getter is worth a one-line test.

**T5 — Test Boundary Conditions**
- **Problem:** Middle-of-range cases are usually fine; boundaries are where bugs live.
- **Fix:** Write explicit tests for zero, one, many, max, null.
- **Apex relevance:** Especially bulk boundaries: 0 records, 1 record, 200 records (trigger chunk size), 201 records, 10,001 records (SOQL row limit).

**T6 — Exhaustively Test Near Bugs**
- **Problem:** Bugs congregate. Finding one in a function means there are probably more.
- **Fix:** When you fix a bug, exhaustively test the surrounding code.
- **Apex relevance:** Strong. If a trigger handler had a null bug, there are probably five more.

**T9 — Tests Should Be Fast**
- **Problem:** Slow tests get skipped under deadline pressure, which defeats their entire purpose.
- **Fix:** Do whatever you must to keep them fast.
- **Apex relevance:** Apex test runs in Salesforce are notoriously slow because of DML overhead. Techniques:
  - Use `@TestSetup` to create shared data once per class instead of per method.
  - Avoid unnecessary SOQL in tests.
  - Use `Test.loadData()` sparingly.
  - Mock callouts with `HttpCalloutMock`, never hit real endpoints.
  - Avoid `@isTest(SeeAllData=true)` — it slows tests and makes them brittle.

### Theme 12: Java-Specific (Noted Only)

**J1: Avoid Long Import Lists by Using Wildcards** — Not applicable; Apex has no imports.
**J2: Don't Inherit Constants** — Not applicable; Apex interfaces can't define constants the way Java's could pre-Java-8.
**J3: Constants versus Enums** — Partially applicable; Apex supports enums since very early API versions. Use them when you have a fixed set of named values. Unlike Java enums, Apex enums cannot have methods or fields, so the "rich enum" pattern from the book doesn't work.

---

## Salesforce/Apex Caveats

**⚠ Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

The smells in this chapter are remarkably platform-neutral. Most apply to Apex essentially verbatim. A few have platform-specific twists worth calling out:

### Where Apex Reality Complicates These Smells

- **F1 Too Many Arguments:** Apex has hard limits on method parameter counts, but the soft limit (readability) kicks in first. Wrap sObjects in wrapper classes rather than passing seven fields as separate scalars. Custom metadata types are excellent for reducing parameter counts.

- **G5 Duplication:** Trigger frameworks (fflib, sfdc-trigger-framework, Metadata Trigger Framework) exist *specifically* to kill this smell at the trigger layer. If you're writing raw triggers with duplicated dispatch logic, adopt a framework.

- **G9 Dead Code:** Apex has a platform-specific wrinkle — deleting a class in a production org is a metadata deployment, not a local change. Dead code accumulates because deleting it is annoying. Do it anyway; code you can't delete is a security surface.

- **G18 Inappropriate Static:** In Apex, static is the default for class-level methods in many patterns (trigger handlers often have static methods, service classes often do too). Apex static behaves differently from Java static — no true singletons across transactions because static state resets per transaction. This changes the "prefer nonstatic for polymorphism" calculus: if you need polymorphism in Apex, you need instance methods and an interface.

- **G25 Magic Numbers:** In Apex, hardcoded IDs (Profile, RecordType, User) are a catastrophic version of this smell because IDs differ between sandboxes and production. **Never hardcode Salesforce IDs.** Query them by DeveloperName, use custom metadata, or use `Schema.SObjectType...getRecordTypeInfosByDeveloperName()`.

- **G23 Prefer Polymorphism:** Works fine in Apex via interfaces. The only gotcha is that Apex interfaces can't have default methods (pre API 58) or constants, so your abstractions are leaner than Java's.

- **G30 Functions Should Do One Thing:** Watch out for the "trigger handler god method" anti-pattern where one method in a handler does querying, filtering, validation, transformation, and DML. Split aggressively. Bulk-safety still matters: each split method should itself be bulk-safe.

- **N1 Descriptive Names:** Apex object and field API names are limited to 40 characters and must end in `__c` (custom) — this bleeds into variable naming conventions. Don't let the platform's quirks make your in-code names cryptic.

- **T1-T9 Tests:** The Salesforce-specific test smells not in Martin's list but worth knowing:
  - **No assertions in a test** — a test that just runs code without asserting outcomes passes even when logic is wrong. Salesforce will give you coverage credit, but you've tested nothing.
  - **`SeeAllData=true`** — couples tests to org state and is slow.
  - **Unbounded `Test.setMock()` state** — mock state can leak between assertions.
  - **Not using `Test.startTest()/stopTest()` for async** — async tests need it to execute queued work.

- **G26 Be Precise — currency:** Use `Decimal`, not `Double`. Use `Currency` field types on objects. Salesforce handles multi-currency at the platform level; respect it.

### Smells Martin Didn't Cover That Are Critical in Apex

Martin's list is Java-centric. Apex has its own additional smells that don't appear in the book:

- **SOQL inside a for loop** — guaranteed governor limit failure. The single most important Apex-specific smell.
- **DML inside a for loop** — same.
- **Hardcoded IDs** — different IDs in different orgs.
- **`without sharing` as a default** — violates least privilege.
- **Trigger without bulk-safety** — works in a test with one record, explodes under load.
- **Empty `catch (Exception e)` blocks** — swallow governor errors silently.
- **`SeeAllData=true`** — couples tests to org data.
- **Static variable "caches" that cross transactions** — they don't; statics reset per transaction.
- **Recursive trigger guards using static booleans without careful scope** — can cause missed updates in legitimate re-entry cases.

These are not in Martin's catalog but belong in any Apex code review vocabulary.

---

## Key Takeaways

1. **Learn the vocabulary.** "That's G5 duplication" or "that's G23, we should use polymorphism" is faster and clearer than hand-waving.
2. **The most important smells for Apex are:** G5 (Duplication), G9 (Dead Code), G25 (Magic Numbers — especially hardcoded IDs), G30 (Functions Should Do One Thing), F3 (Flag Arguments), N1 (Descriptive Names), C5 (Commented-Out Code), and T1 (Insufficient Tests).
3. **Java-specific smells mostly don't apply** — Apex has no imports, different static semantics, and different enum capabilities.
4. **Martin's list doesn't cover Apex's unique smells.** SOQL-in-loop, hardcoded IDs, `without sharing` by default, empty catch blocks, and static-variable caching assumptions are Apex-specific and just as important as anything in the book.
5. **The point is values, not rules.** The list implies a value system — care about readers, care about precision, care about change tolerance. Internalize the values and the specific smells follow.
6. **Use this document as a reference during code review.** When something smells wrong, find the category code here, name it out loud, and suggest the fix.
