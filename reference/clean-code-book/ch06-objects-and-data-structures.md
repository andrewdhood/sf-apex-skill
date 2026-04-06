# Chapter 6: Objects and Data Structures

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008)
> **When to read this:** Before designing a new Apex class hierarchy, wrapper, or DTO — especially when deciding whether a concept should be an object (hides data, exposes behavior) or a data structure (exposes data, has no behavior).

## Key Principles

- **Objects hide their data behind abstractions and expose functions that operate on that data.**
- **Data structures expose their data and have no meaningful functions.**
- Objects and data structures are **virtual opposites** — each is good at what the other is bad at.
- **Objects** make it easy to add *new kinds of objects* without changing existing behaviors; they make it *hard* to add new behaviors (every class must change).
- **Data structures** make it easy to add *new behaviors* without changing existing data structures; they make it *hard* to add new data types (every function must change).
- **Mature programmers know the idea that "everything is an object" is a myth.** Sometimes you really do want simple data structures with procedures operating on them.
- **Law of Demeter:** A module should not know about the innards of the objects it manipulates. Talk to friends, not to strangers.

## Detailed Notes

### Data Abstraction

Keeping variables private is not about getters and setters — it is about *hiding implementation*. A class that blindly exposes its private variables through getters and setters is no better than one with public fields. The point of encapsulation is to expose *abstractions* that let callers manipulate the *essence* of the data without knowing how it is stored.

**Concrete vs. abstract example:**

```java
// Concrete — reveals storage
public interface Vehicle {
    double getFuelTankCapacityInGallons();
    double getGallonsOfGasoline();
}

// Abstract — hides storage
public interface Vehicle {
    double getPercentFuelRemaining();
}
```

The abstract version does not tell you whether fuel is measured in gallons, liters, or a calculated percentage. The caller asks for the *essence* of what it needs and the class is free to change how it stores the data.

### Data/Object Anti-Symmetry

Martin shows two versions of a shape system:

**Procedural (data structures + external function):** `Square`, `Rectangle`, `Circle` are plain structs with public fields. A separate `Geometry` class has an `area()` function with a big `if/else if` chain.
- Adding `perimeter()` is trivial: add one function to `Geometry`. Shape classes are unaffected.
- Adding `Triangle` is painful: every function in `Geometry` must change.

**Object-oriented (polymorphic):** `Square`, `Rectangle`, `Circle` each implement a `Shape` interface and provide their own `area()`. No `Geometry` class is needed.
- Adding `Triangle` is trivial: implement `Shape`, existing classes are unaffected.
- Adding `perimeter()` is painful: every shape class must change.

**The takeaway:**
> Procedural code makes it easy to add new functions without changing existing data structures. OO code makes it easy to add new classes without changing existing functions.
>
> Procedural code makes it hard to add new data structures because all the functions must change. OO code makes it hard to add new functions because all the classes must change.

Pick the right tool for the problem. If you anticipate adding new *behaviors* to a stable set of types, procedural/data-structure style is fine. If you anticipate adding new *types* that share a common contract, OO is the right choice.

### The Law of Demeter

> A method *f* of a class *C* should only call methods of:
> - `C` itself
> - An object created by *f*
> - An object passed as an argument to *f*
> - An object held in an instance variable of `C`
>
> The method should **not** invoke methods on objects that are *returned* by any of the allowed functions.

In shorthand: "talk to friends, not to strangers."

### Train Wrecks

A chain of calls like:

```java
final String outputDir = ctxt.getOptions().getScratchDir().getAbsolutePath();
```

looks like coupled train cars. This is sloppy. Prefer splitting it up:

```java
Options opts = ctxt.getOptions();
File scratchDir = opts.getScratchDir();
final String outputDir = scratchDir.getAbsolutePath();
```

Whether this is an *actual* Law of Demeter violation depends on whether `ctxt`, `Options`, and `ScratchDir` are **objects** or **data structures**. If they are objects, they should hide their structure and expose behavior — so reaching through them is a violation. If they are data structures, exposing structure is the whole point and Demeter does not apply.

### Hybrids

Confusion between objects and data structures leads to **hybrids** — classes with private fields, public accessors *and* real behavior methods. These are the worst of both worlds:
- Hard to add new functions (because some of the behavior is already baked into the class).
- Hard to add new data types (because other code is treating the class like a procedural struct).

Hybrids indicate muddled design. Avoid them. Decide whether each class is an *object* (hide data, expose behavior) or a *data structure* (expose data, no behavior) and commit to it.

### Hiding Structure

If `ctxt` is a real object, don't ask it "give me your scratch directory's path." Ask it to *do* the thing you actually wanted:

```java
// Bad — reaches through ctxt to get internals
String outFile = outputDir + "/" + className.replace('.', '/') + ".class";
FileOutputStream fout = new FileOutputStream(outFile);

// Better — tell ctxt what you need
BufferedOutputStream bos = ctxt.createScratchFileStream(classFileName);
```

This lets `ctxt` hide its internals and keeps your code from knowing about `Options`, `ScratchDir`, or file path concatenation.

### Data Transfer Objects (DTOs)

A DTO is a class with public variables and no functions. DTOs are the quintessential data structure. They are useful for:
- Communicating with databases
- Parsing messages from sockets
- The first translation stage between raw data and richer application objects

The "bean" form (private fields, trivial getters/setters, no behavior) is common but usually provides no benefit over plain public fields — it just makes OO purists feel better.

### Active Record

An Active Record is a special form of DTO with navigational methods like `save()` and `find()`. They are typically direct translations of database tables. The mistake people make is putting business rules into Active Records, creating hybrids. The right approach:
- Treat the Active Record as a pure data structure.
- Create *separate* business-rule objects that hold behavior and contain (or wrap) the Active Record.

## Salesforce/Apex Caveats

**⚠ Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### SObjects ARE DTOs/Active Records. This is non-negotiable.

Every `Account`, `Contact`, `Opportunity`, and custom object in Apex is effectively an **Active Record** — a data structure with public fields, `insert`/`update`/`delete`/`upsert` navigational methods, and a direct mapping to a database table. This is the platform's design and you cannot change it.

**Consequences:**
- You *must* access SObject fields directly (`account.Name`, `contact.Email`). There are no getters and there will never be getters.
- `SELECT` queries return data structures, not behaviorally rich objects. You can't hide the field list behind an abstraction.
- Trigger handlers receive `List<Account>` — lists of Active Records — and mutate their fields directly before the platform commits them.

**Consequence for Clean Code's "data/object anti-symmetry":** Apex developers are forced to mix procedural-style code (working on SObject data structures) with object-oriented code (domain services, trigger handlers, helper classes). Don't fight this. The idiomatic pattern is:

1. **SObjects are data structures.** Access fields directly. Do not wrap them in getters.
2. **Service/handler classes are objects.** They hide business logic behind well-named methods and take SObjects as input parameters.
3. **Keep business rules OUT of the SObject layer.** Put them in services, selectors (SOQL wrappers), and domain classes (fflib-style if using Apex Commons).

### SObject field traversal is idiomatic Apex, not a Law of Demeter violation

Apex SOQL returns nested record graphs, and reaching through them is normal, expected, and how Salesforce developers have written code since 2008:

```apex
// This is NOT a Law of Demeter violation in Apex.
// SObjects are data structures, and reaching through them is how they work.
for (Account a : [SELECT Id, Name, (SELECT Id, Email FROM Contacts) FROM Account LIMIT 10]) {
    for (Contact c : a.Contacts) {
        System.debug(c.Email);
    }
}
```

```apex
// Also normal — SObject relationship traversal
String ownerEmail = opportunity.Account.Owner.Email;
```

**Why this is OK:**
- `Account`, `Contact`, `Opportunity`, `User` are data structures (Active Records).
- Clean Code explicitly exempts data structures from the Law of Demeter: "if they are just data structures with no behavior, then they naturally expose their internal structure, and so Demeter does not apply."
- The platform *forces* you to reach through relationship fields because that's the query shape SOQL returns.

**Where Demeter DOES still apply in Apex:** Your own service and handler classes. If you have a `BillingService` that returns an `InvoiceCalculator` that has a `getTaxRate()` method, don't write `billingService.getInvoiceCalculator().getTaxRate()` — ask `billingService` for the tax rate directly. Those are objects with behavior, and they should hide their internals.

### Selector/Service pattern is the Apex answer to data abstraction

Clean Code wants you to hide implementation behind abstractions. In Apex, that looks like:

```apex
// Selector class — hides the SOQL implementation
public with sharing class AccountSelector {
    public static List<Account> getActiveEnterpriseAccounts() {
        return [
            SELECT Id, Name, Industry, AnnualRevenue
            FROM Account
            WHERE IsActive__c = true
              AND Type = 'Enterprise'
            LIMIT 200
        ];
    }
}

// Service class — hides the business logic
public with sharing class AccountRenewalService {
    public static void processRenewals(List<Account> accounts) {
        // ... behavior lives here
    }
}

// Caller doesn't know the field list or the logic
List<Account> targets = AccountSelector.getActiveEnterpriseAccounts();
AccountRenewalService.processRenewals(targets);
```

The selector hides the SOQL (the implementation). The service hides the logic. Callers work at the level of abstraction they care about. This is Martin's data abstraction applied to Salesforce.

### `with sharing` on objects matters

When you write an Apex service class, put `with sharing` on it unless you have a documented, audited reason to use `without sharing`. This is a Salesforce platform concern that Clean Code does not address but must override any stylistic preference:

```apex
public with sharing class AccountRenewalService {
    // ...
}
```

If a method has to run `without sharing` (e.g., a guest-user-facing controller that needs to read records the guest can't see), **isolate that method in a dedicated `without sharing` inner class or separate class** — don't make the whole class `without sharing` for the sake of one method. Also remember: `without sharing` still enforces object-level CRUD and field-level security. It only bypasses row-level sharing.

### Wrapper classes are the one place you build real objects in Apex

LWC, Visualforce, and Apex REST endpoints often need a custom response shape. This is where you write *real* objects:

```apex
public class AccountSummary {
    // keep fields private if the caller should not mutate them
    @AuraEnabled public String accountName;
    @AuraEnabled public Integer openOpportunities;
    @AuraEnabled public Decimal totalRevenue;

    // constructor builds the summary from the raw Account + Opportunities
    public AccountSummary(Account a, List<Opportunity> opps) {
        this.accountName = a.Name;
        this.openOpportunities = opps.size();
        this.totalRevenue = calculateRevenue(opps);
    }

    // private helper hides the calculation detail
    private Decimal calculateRevenue(List<Opportunity> opps) {
        Decimal total = 0;
        for (Opportunity o : opps) {
            if (o.Amount != null) {
                total += o.Amount;
            }
        }
        return total;
    }
}
```

Note: `@AuraEnabled` fields must be public — the framework needs to serialize them. This is a platform constraint that overrides Clean Code's "private fields" preference for LWC response shapes.

### Hybrids happen in Apex when trigger handlers leak business rules into SObjects

Do NOT put business logic into trigger handlers that directly mutates an SObject then immediately queries off its fields and mutates again. That's a hybrid pattern — the handler is treating the SObject as both a data structure (direct field access) and as an object (implicit behavior from re-reading fields). Instead:

1. Let the trigger handler collect what it needs.
2. Pass SObjects into a service class.
3. The service class computes changes and returns them (or applies them via bulk DML).
4. The handler performs the DML.

## Key Takeaways

1. **Objects hide data behind abstractions; data structures expose data and have no behavior. Know which one you're building.**
2. **"Everything is an object" is a myth.** Sometimes you really do want a data structure with procedures operating on it.
3. **SObjects are Active Records / data structures.** Access their fields directly. Reaching through SObject relationships (`account.Contacts[0].Email`) is idiomatic Apex, not a Law of Demeter violation.
4. **The Law of Demeter still applies to your own service and handler classes.** Don't chain calls through real objects — tell them to do the thing you want.
5. **Use the selector/service pattern to get Clean Code's data abstraction benefits in Apex.** Selectors hide SOQL; services hide business logic.
6. **Put `with sharing` on service classes by default.** This is a platform concern that Clean Code doesn't cover but that must override stylistic decisions.
7. **Wrapper classes for LWC/REST responses are where you build real objects in Apex** — but `@AuraEnabled` fields must be public, which is a platform exception to Clean Code's private-field preference.
8. **Avoid hybrids.** Decide whether each class is a data structure or an object and commit to it. Active Records with business rules stapled on are the worst of both worlds — move the behavior to a separate service.
