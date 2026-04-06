# Chapter 10: Classes

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008)
> **When to read this:** Before designing a new Apex class, refactoring a "God class" service, splitting a bloated trigger handler, or deciding how to carve responsibilities across multiple `.cls` files.

## Key Principles

1. **Classes follow a standard organization:** constants first, then static variables, then instance variables, then public methods, then the private helpers that each public method calls (stepdown rule — reads like a newspaper article).
2. **Keep variables and helpers private.** Loosen encapsulation only as a last resort — and only when a test in the same package needs access. Prefer package/protected over public.
3. **Classes should be small. Then smaller than that.** With functions we count lines; with classes we count *responsibilities*.
4. **If you can't give a class a concise, non-weasel name, it's too big.** Names containing `Processor`, `Manager`, `Super`, or `Helper` almost always hint at aggregated responsibilities.
5. **The 25-word rule:** You should be able to describe a class in about 25 words without using "if," "and," "or," or "but." An "and" in the description is a code smell — the class probably has two responsibilities.
6. **Single Responsibility Principle (SRP):** A class should have one, and only one, *reason to change*. SRP gives you both a definition of responsibility and a yardstick for class size.
7. **Many small classes over a few large ones.** A system of small, cohesive classes has no more moving parts than a system of large classes — but it's organized like a labeled toolbox instead of a junk drawer.
8. **High cohesion:** A class is cohesive when most of its methods use most of its instance variables. When classes lose cohesion, split them. Keeping functions small and parameter lists short naturally causes instance variables to proliferate — that's a signal to extract a new class, not a reason to despair.
9. **Organize for change — Open/Closed Principle (OCP):** New features should arrive as new subclasses/implementations, not edits to existing classes. Every time you "open up" a working class to modify it, you take on regression risk.
10. **Isolate from change — Dependency Inversion Principle (DIP):** Depend on abstractions, not on concrete details. If `Portfolio` depends directly on `TokyoStockExchange`, you can't test it without network calls. Introduce a `StockExchange` interface and inject the concrete implementation.

## Detailed Notes

### Class Organization (Stepdown Rule)

Within a class, order members so a reader can descend from high-level intent to low-level detail:

1. Public static final constants
2. Private static variables
3. Private instance variables (public instance variables are almost always wrong)
4. Public methods — each one immediately followed by the private helpers it calls

The goal: the file reads top-to-bottom like an article, not a dictionary you have to jump around in.

### Encapsulation

Keep fields and helpers private by default. The only reason to loosen encapsulation in the book is testing, and even then, look for a way to maintain privacy first (e.g., extract a collaborator you can test directly). "Loosening encapsulation is always a last resort."

### Classes Should Be Small

The book's running example is `SuperDashboard`, a Swing class exposing ~70 public methods. Even the trimmed-down five-method version (`getLastFocusedComponent`, `setLastFocused`, `getMajorVersionNumber`, `getMinorVersionNumber`, `getBuildNumber`) is still too big — it has two reasons to change:

- Tracking version information (changes when the product ships)
- Managing Swing components (changes when GUI code changes)

Extracting the three version methods into a `Version` class yields a single-responsibility construct with high reuse potential.

### The Single Responsibility Principle

SRP is simple to state and widely violated. Why? Because "getting software to work" and "making software clean" are two different activities, and once the code passes its tests most developers move on to the next feature rather than returning to break the overstuffed class apart.

Martin's counter-argument to the "too many small classes is hard to navigate" complaint: a system of small classes has no more moving parts than a system of few large classes. The question is whether you want a toolbox with many labeled drawers or one big drawer with everything in it.

### Cohesion

A maximally cohesive class has every method using every instance variable. That's rarely achievable or even desirable, but high cohesion is a target. The `Stack` example in Listing 10-4 uses two instance variables (`topOfStack`, `elements`) — `push` and `pop` use both, `size` uses only one. That's acceptable and cohesive.

When you extract one piece of a large function into a helper and find yourself passing four arguments through, promote those variables to instance variables. That makes the extraction easy — but it may also lower cohesion, and *that's a signal*: those variables and the methods that share them probably belong in a new class.

### Maintaining Cohesion Results in Many Small Classes

The `PrintPrimes` refactor (Listings 10-5 through 10-8) walks through splitting a single 60-line function into three classes:

- `PrimePrinter` — handles invocation/execution environment (would change if converted to a SOAP service)
- `RowColumnPagePrinter` — formats numbers into paginated rows and columns (would change if the output format changed)
- `PrimeGenerator` — generates primes (would change if the algorithm changed)

Note how the refactored code is *longer* but easier to understand: more descriptive names, function and class declarations acting as commentary, whitespace preserving readability. The refactor was done incrementally, with a test suite verifying the precise behavior of the original at every step — it was not a rewrite.

### Organizing for Change (Open/Closed Principle)

Listing 10-9 shows an `Sql` class that must be "opened up" every time a new statement type is added — a clear SRP and OCP violation. Listing 10-10 refactors it into an abstract `Sql` base class with concrete subclasses `CreateSql`, `SelectSql`, `InsertSql`, `UpdateSql`, `PreparedInsertSql`, etc., plus utility classes `Where` and `ColumnList`. Adding an `update` statement becomes: write a new `UpdateSql` class. No existing class is touched. No regression risk.

### Isolating from Change (Dependency Inversion)

If `Portfolio` constructs its own `TokyoStockExchange`, it's impossible to test deterministically — prices change every five minutes. Extract a `StockExchange` interface, make `TokyoStockExchange` implement it, and inject it into `Portfolio`'s constructor. Tests can now provide a `FixedStockExchangeStub` that always returns $100 per share of MSFT. The test asserts $500 for five shares.

This is DIP in action: `Portfolio` depends on the abstract concept (ask for a price) rather than the concrete implementation (call the Tokyo exchange API).

## Salesforce/Apex Caveats

**Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### Hard limits that shape Apex class design

- **Single class file limit:** 1 million characters per Apex class (but practically, you'll hit compile/readability issues long before that). Methods, inner classes, and the class itself count toward this.
- **Deploy limits:** 10,000 code units per deployment. If you naively apply "many tiny classes" to a large org, you can make deployments fragile and slow.
- **75% test coverage mandate:** Every class that runs in production must have tests covering at least 75% of its lines, and every trigger must have *some* coverage. Splitting a class into ten tiny classes means ten times as many tests to maintain. Factor test cost into your SRP decisions.
- **Compile dependencies propagate:** Unlike Java, Apex doesn't have package-private. A public class is visible org-wide. Test and production classes deploy together and share the namespace.

### SRP is great, but respect the "one trigger per object" rule

Salesforce strongly advises (and most teams enforce) one trigger per SObject. That single trigger is typically a thin dispatcher, and the real logic lives in a `TriggerHandler` or domain class. This layers interestingly with SRP:

- The trigger itself: one reason to change (the object it fires on).
- The handler: can legitimately have multiple responsibilities (before insert, after insert, before update, after update, etc.) — or you can decompose those into helper classes per operation.
- A common clean pattern: `AccountTriggerHandler` delegates to `AccountValidationService`, `AccountEnrichmentService`, `AccountSharingService`, each with a single reason to change.

Do NOT write multiple triggers for one object in an attempt to satisfy SRP. Order of execution becomes non-deterministic and you'll chase ghosts for years.

### Bulkification beats SRP when they conflict

Clean Code says "many small methods doing one thing." Apex says "process in bulk, never do SOQL/DML in a loop." These principles can collide:

```apex
// CLEAN CODE instinct (SRP-ish, but wrong for Apex)
public class AccountProcessor {
    public void process(Account a) {
        // one account, one responsibility
        update a; // DML in a loop if called repeatedly — governor-limit disaster
    }
}

// APEX-CORRECT (bulk-first, even if the method looks less "pure")
public class AccountProcessor {
    public void process(List<Account> accounts) {
        // process the whole collection, DML once at the end
        update accounts;
    }
}
```

The bulk signature WINS. Resist the urge to extract "a method that handles one record" as the public API. Per-record helpers are fine as *private* utilities inside a bulk-aware public method.

### Dependency Inversion in Apex: awkward but possible

Apex supports interfaces (`public interface StockExchange { Money currentPrice(String symbol); }`), so the book's `Portfolio`/`TokyoStockExchange` example translates directly. You can inject a stub in a test class with no framework.

What Apex does NOT have:
- Default methods on interfaces (pre-API 56 was even worse — no default methods at all; newer versions allow virtual classes but not true interface defaults)
- Generics beyond `List<T>`, `Set<T>`, `Map<K,V>`
- Annotations beyond built-ins (`@AuraEnabled`, `@InvocableMethod`, `@TestVisible`, `@IsTest`, `@Future`, etc.) — you cannot define your own
- Reflection in the Java sense
- DI frameworks like Spring — you roll your own with factory patterns or Custom Metadata Types

A realistic Apex DIP pattern:

```apex
// abstraction
public interface ExternalPricingService {
    Decimal getPrice(String symbol);
}

// concrete production implementation
public class BloombergPricingService implements ExternalPricingService {
    public Decimal getPrice(String symbol) {
        // ... real callout here
        return 42.00;
    }
}

// the class under design depends on the interface, not the concrete
public with sharing class Portfolio {
    private final ExternalPricingService pricingService;

    // constructor injection — the test will pass in a stub
    public Portfolio(ExternalPricingService svc) {
        this.pricingService = svc;
    }

    public Decimal value(List<Holding> holdings) {
        // loop over holdings, total the market value
        Decimal total = 0;
        for (Holding h : holdings) {
            total += h.shares * pricingService.getPrice(h.symbol);
        }
        return total;
    }
}

// in the test class
@IsTest
private class PortfolioTest {
    // stub that returns a fixed price — no real callout
    private class FixedPriceStub implements ExternalPricingService {
        public Decimal getPrice(String symbol) {
            return 100;
        }
    }

    @IsTest
    static void fiveSharesOfMsftShouldBe500() {
        Portfolio p = new Portfolio(new FixedPriceStub());
        List<Holding> holdings = new List<Holding>{ new Holding('MSFT', 5) };
        System.assertEquals(500, p.value(holdings));
    }
}
```

This is as close to DIP as you get in Apex and it's genuinely useful — especially for isolating HTTP callouts, which would otherwise require `HttpCalloutMock` scaffolding in every test.

### `with sharing` is part of the class's identity

Every production Apex class should declare `with sharing`, `without sharing`, or `inherited sharing`. This is not a Clean Code concern — it's a security mandate. When designing for SRP, remember that the sharing context is part of the class's responsibility: a class that legitimately needs `without sharing` (e.g., an allow-listed integration writer) should be small and tightly scoped so its elevated privilege is easy to audit.

### Class explosion hurts more in Apex than in Java

- Every class is a separate metadata file (`.cls` + `.cls-meta.xml`).
- SFDX project deploys must ship them all together.
- Code review diffs get noisier.
- Package installation takes longer.

This doesn't mean "don't split classes" — it means you should split with *intent*. The `PrintPrimes` refactor is a great example of a split that pays for itself. Splitting a 40-line utility into six classes each with one method is cargo-cult SRP, and on Apex it bites harder.

## Key Takeaways

- **Count responsibilities, not lines.** If you can't describe an Apex class in 25 words without saying "and," split it.
- **Name matters.** `AccountManager` probably does too much. `AccountBillingSplitter` probably doesn't.
- **SRP applies to trigger handlers, service classes, and selectors — but the public API must stay bulk-aware.** Process collections, not single records.
- **Use DIP with Apex interfaces to isolate callouts, queries, and external dependencies from your test suite.** It's the single biggest clean-code win available to you on this platform.
- **The one-trigger-per-object rule is non-negotiable.** Keep triggers thin and push logic into handler classes, which you can then decompose with SRP.
- **Don't split for splitting's sake.** Every new class is another file, another test, another metadata item, another row in the deployment manifest. Make each split earn its keep.
- **`with sharing` is a first-class design decision**, not an afterthought — factor it into how you carve responsibilities.
