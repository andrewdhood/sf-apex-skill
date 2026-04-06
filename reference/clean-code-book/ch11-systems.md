# Chapter 11: Systems

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008), chapter by Dr. Kevin Dean Wampler
> **When to read this:** When deciding how to wire dependencies in a large Apex codebase, when tempted to do construction logic inside business-logic methods, when thinking about cross-cutting concerns (logging, security, transactions), or when architecting a new Salesforce integration layer.

## Key Principles

1. **Separate constructing a system from using it.** The startup/wiring process is a distinct concern from the runtime logic — don't mix them.
2. **Lazy initialization inside business methods is a smell.** It hardcodes dependencies, makes testing harder, and violates SRP by forcing one method to both configure and use an object.
3. **Move construction to `main` (or its equivalent).** Dependencies should flow from `main` toward the application — the application should know nothing about how it was built.
4. **Factories** let the application stay in control of *when* an object is created while keeping the *how* elsewhere.
5. **Dependency Injection (DI)** is Inversion of Control applied to dependency management — the class is passive, dependencies are pushed in by an "authoritative mechanism" (main, a DI container, config file).
6. **Scale up by maintaining separation of concerns.** Software architecture can grow incrementally if concerns are properly separated. Big Design Up Front (BDUF) is harmful.
7. **Cross-cutting concerns** (persistence, security, transactions, logging) cut across domain boundaries. Aspect-Oriented Programming (AOP), Java Proxies, and frameworks like Spring/JBoss exist to restore modularity for them.
8. **Test-drive the system architecture.** If domain logic is POJO-based and decoupled from architecture concerns, you can test-drive the architecture itself.
9. **Optimize decision making by postponing decisions to the last responsible moment.** A decoupled POJO system lets you defer choices until you have the most information.
10. **Use standards only when they add demonstrable value.** EJB2 was a cautionary tale about blindly adopting a "standard."
11. **Domain-Specific Languages (DSLs)** raise the abstraction level so code reads like domain expert prose.

## Detailed Notes

### Separate Constructing a System from Using It

Wampler opens with a building analogy: a half-built hotel looks nothing like the finished hotel. Construction is a distinct process from use — cranes and hard hats vs. guests and glass walls. Software startup (wiring dependencies) is fundamentally different from runtime logic, yet most applications mix the two.

The cautionary example:

```java
public Service getService() {
    if (service == null)
        service = new MyServiceImpl(...);  // Good enough default for most cases?
    return service;
}
```

This is LAZY INITIALIZATION and it has problems:
- **Hardcoded dependency** on `MyServiceImpl` and its constructor args — can't compile without them.
- **Testing pain** — if `MyServiceImpl` is heavy, every test must either tolerate it or inject a test double before this method runs.
- **SRP violation** — this method both constructs and uses, so both paths (null and non-null) must be tested.
- **Wrong object for all contexts?** — the method has to "know the global context" to pick the right implementation, which is almost never true.

One lazy-init isn't a problem. But they multiply across an application until the global setup *strategy* is scattered with no modularity.

### Separation of Main

The simplest fix: push all construction to `main` (or modules `main` calls). `main` builds the object graph and passes it to the application, which has no knowledge of `main` or how it was wired. Dependencies point *away from* the application.

### Factories

When the application must control *when* an object is created (e.g., an `OrderProcessing` class creating `LineItem` instances on demand), use the ABSTRACT FACTORY pattern. The factory interface is owned by the application side; the factory implementation lives on the `main` side. The application says "give me a `LineItem`" when it's ready; the factory decides how.

### Dependency Injection

DI is the application of Inversion of Control to dependency management. A class that uses DI is *passive*: it takes no steps to resolve its own dependencies. Instead it exposes constructor args or setters that the DI container (or `main`, or a factory) uses to wire things up.

JNDI lookups are a partial DI — the object still actively asks a registry for a dependency by name. True DI pushes the dependency in without the class asking.

Spring is the best-known Java DI container. You define the object graph in XML (or annotations), then ask the container for a top-level object by name.

Lazy-initialization still has a place *inside* a DI container — most containers won't build an object until it's first needed, and they can construct proxies for lazy-evaluation optimizations. But don't confuse container-level laziness with the per-method lazy-init antipattern above.

### Scaling Up

Cities grow from towns which grow from settlements. At first roads are dirt, then paved, then widened. Nobody would justify a six-lane highway through a tiny town. Software is the same: "right the first time" is a myth. Implement today's stories, refactor, expand.

> **Software systems are unique compared to physical systems. Their architectures can grow incrementally, *if* we maintain the proper separation of concerns.**

The counterexample: EJB1/EJB2. An `Entity Bean` for a `Bank` class required defining a local or remote interface, implementing a heavyweight abstract class with container-mandated lifecycle methods (`ejbActivate`, `ejbPassivate`, `ejbLoad`, `ejbStore`, `ejbRemove`, most of which were empty), and writing XML deployment descriptors. Business logic was tightly coupled to the EJB2 container. Unit testing in isolation was impossible — you had to mock the container (hard) or deploy to a real server (slow). Reuse outside EJB2 was effectively impossible.

Even object-oriented programming broke down: one bean couldn't inherit from another, which led to DTOs ("structs with no behavior") and boilerplate copying data between representations.

### Cross-Cutting Concerns

Persistence, security, logging, transactions — these cut across the natural boundaries of a domain model. You want to persist all your objects using the same strategy, but that strategy has to be threaded through every one of them. The persistence framework might be modular, the domain might be modular, but their *intersection* is the problem.

Aspect-Oriented Programming (AOP) is a general approach to restoring modularity for cross-cutting concerns. In AOP, *aspects* declaratively specify where in the system a particular concern should apply, and the framework (non-invasively) modifies target code to implement it.

### Java Proxies, Pure Java AOP Frameworks, AspectJ

Three Java-specific mechanisms in the chapter:

1. **Java Proxies (JDK dynamic proxies).** Work only with interfaces. You write an `InvocationHandler` that intercepts every call on the proxied interface and decides what to do — delegate, add persistence, add logging, etc. The example is verbose and complex even for a simple case. Proxies are fine for simple per-object wrapping but don't scale to system-wide concerns.
2. **Pure Java AOP frameworks (Spring AOP, JBoss AOP).** Handle proxy boilerplate for you. Business logic is written as POJOs — no dependency on enterprise frameworks. Cross-cutting concerns are configured declaratively (XML or annotations) and the framework creates proxies under the hood. EJB3 moved to this model and it's much cleaner than EJB2.
3. **AspectJ.** A Java language extension with first-class aspect support — the most powerful tool. Handles the 10-20% of cases Spring AOP can't. Adoption cost: new language, new tools. Annotation-form AspectJ (using Java 5 annotations) lowers the barrier.

### Test-Drive the System Architecture

If domain logic is POJOs decoupled from architecture concerns, you can evolve architecture incrementally. Start naively simple, deliver working stories quickly, add infrastructure as you scale. You don't need BDUF. A good API should disappear from view — if architectural constraints are always in the way, the team can't focus on stories.

> **An optimal system architecture consists of modularized domains of concern, each implemented with Plain Old Java (or other) Objects. The different domains are integrated together with minimally invasive Aspects or Aspect-like tools. This architecture can be test-driven, just like the code.**

### Optimize Decision Making

Postpone decisions until the last responsible moment. Premature decisions are made with incomplete information. A modular POJO system preserves optionality.

### Use Standards Wisely

EJB2 got adopted because it was a "standard," even when simpler designs would have worked. Standards are valuable when they accelerate reuse, hiring, and component wiring. They become liabilities when they're imposed on contexts where they don't fit, or when they freeze into shapes misaligned with real needs.

### Systems Need Domain-Specific Languages

DSLs minimize the gap between domain concepts and the code implementing them. A good DSL lets a domain expert read code and recognize their own vocabulary. They raise abstraction above code idioms and design patterns, letting intent live at the right level.

## Salesforce/Apex Caveats

**Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### Big chunks of this chapter DO NOT apply to Apex

- **No `main` method.** Apex has no application entry point you control. Execution starts from a trigger, an `@InvocableMethod`, an `@AuraEnabled` call, a Visualforce controller, a REST endpoint, a batch `execute`, a schedulable `execute`, a queueable `execute`, a platform event trigger, etc. Each transaction is a fresh context — there is no long-lived "main" building an object graph at startup.
- **No Spring, no Guice, no Dagger, no JBoss AOP, no AspectJ.** None of these exist for Apex and none can be ported. The platform does not allow the kind of classpath manipulation or bytecode weaving they depend on.
- **No JNDI, no service locators in the traditional sense.** What Apex offers is Custom Metadata Types, Custom Settings, and Platform Cache — which you can use to simulate a tiny configuration store.
- **No Java Proxies, no reflection-based interception.** Apex has a very limited `Type.forName(...)` mechanism that can instantiate a class by name, which is the *only* runtime reflection you get. There is no `Method.invoke`, no `Proxy.newProxyInstance`.
- **No annotations beyond the built-in set.** You cannot define a custom `@Transactional` or `@Loggable` annotation and have it do anything. The annotations Apex understands are a fixed list.
- **AspectJ is entirely inapplicable.** Cross-cutting concerns must be handled by convention (e.g., every service method manually calls `Logger.log(...)`) or by wrapping collaborators in decorator classes written by hand.

### What DOES translate

**Separating construction from use still matters.** The cautionary `getService()` lazy-init example is just as bad in Apex as in Java — maybe worse, because Apex has no DI container to rescue you. Don't do this:

```apex
public class AccountService {
    private ExternalPricingService pricingService;

    public Decimal valueAccount(Account a) {
        if (pricingService == null) {
            pricingService = new BloombergPricingService(); // hardcoded, untestable
        }
        // ... use it
        return pricingService.getPrice(a.Ticker__c);
    }
}
```

Do this instead:

```apex
public with sharing class AccountService {
    private final ExternalPricingService pricingService;

    // production constructor — uses factory to pick the real implementation
    public AccountService() {
        this(PricingServiceFactory.getInstance());
    }

    // test constructor — lets test code inject a stub
    @TestVisible
    private AccountService(ExternalPricingService svc) {
        this.pricingService = svc;
    }

    public Decimal valueAccount(Account a) {
        // the method is now purely about using the dependency
        return pricingService.getPrice(a.Ticker__c);
    }
}
```

**Factories are the Apex DI story.** Since there's no container, you roll your own:

```apex
public class PricingServiceFactory {
    // the singleton that gets swapped out during tests
    private static ExternalPricingService instance;

    public static ExternalPricingService getInstance() {
        if (instance == null) {
            // look up which implementation to use — hardcoded, Custom Metadata, or Setting
            instance = new BloombergPricingService();
        }
        return instance;
    }

    // test hook — let a test class install a mock
    @TestVisible
    private static void setMock(ExternalPricingService mock) {
        instance = mock;
    }
}
```

This is the standard "Apex DI" pattern. It's crude compared to Spring, but it achieves the goal: production code can run unmodified and tests can inject stubs.

**Custom Metadata Types as a configuration store.** For runtime-selectable implementations (e.g., "use Bloomberg in production, use Mock in sandbox"), store the target class name in a Custom Metadata record and instantiate via `Type.forName`:

```apex
public class PricingServiceFactory {
    public static ExternalPricingService getInstance() {
        // look up which implementation to use for the current org
        Pricing_Config__mdt cfg = Pricing_Config__mdt.getInstance('Default');
        Type t = Type.forName(cfg.Implementation_Class__c);
        return (ExternalPricingService) t.newInstance();
    }
}
```

This is the closest thing Apex has to a Spring `applicationContext.xml`. It works. Use it sparingly — every indirection is a place a deploy can fail.

### Cross-cutting concerns in Apex — the honest approach

Since there's no AOP, cross-cutting concerns are handled three ways:

1. **Base class inheritance (virtual/abstract).** A `TriggerHandler` base class provides common logging, bypass-check, recursion-prevention. All handlers extend it. This is brittle but widely used.
2. **Decorators (hand-written).** Wrap a service class in another class that adds logging/timing/caching. You write the wrapper by hand — there's no framework to generate it.
3. **Convention.** Every service method manually calls `Logger.log('entering method X')`. Ugly but works. Tools like Nebula Logger formalize this with builder patterns.

For transactions, Apex gives you `Database.setSavepoint()` / `Database.rollback(sp)` — you manage transaction boundaries manually inside the method, not declaratively.

### "Separate construction from use" maps to Apex DML-ordering rules

Apex has a hard platform rule: **you cannot perform DML after an HTTP callout in the same transaction** unless the callout comes first. This is enforced by the runtime. It's a construction/use separation the platform forces on you, whether you like it or not.

Practical implication: the startup-phase / runtime-phase split the book advocates maps onto Apex as a *callout-phase / DML-phase* split within a transaction. Do all your external lookups first, gather data, then do all DML at the end. This is also Clean Code in the sense of separating concerns by phase.

### BDUF vs. incremental architecture — Salesforce angle

Salesforce projects often tempt teams into BDUF because the platform's upfront declarative work (objects, fields, profiles, permission sets, sharing rules, page layouts) feels like it should all be designed at once. Resist.

- Design the data model first (objects and relationships) — this really is hard to change later.
- Defer UX/component decisions until you have user stories.
- Defer integration architecture until you know what systems you're integrating with.
- Build automation (Flows, Apex triggers) one story at a time.

The book's "grow from towns to cities" analogy applies, but Salesforce has a few stone-set foundations that can't be retrofitted cheaply. Know which ones they are.

### Domain-Specific Languages in Apex

Apex is not a DSL-friendly language. It has no operator overloading, no macros, no metaprogramming, no custom syntax. What you can do:

- Use **fluent builders** (method chaining returning `this`) to create expressive APIs.
- Use **Flows** as the ultimate Salesforce DSL for declarative business logic.
- Use **Validation Rules** as a declarative DSL for data constraints.
- Use **Formula Fields** as a declarative DSL for derived values.

These declarative tools ARE Salesforce's DSLs. When a non-developer can read/write them, that's the book's "domain expert reads the code" ideal in practice.

## Key Takeaways

- **Separate construction from use — even without a DI container.** Apex lets you do this with factories and `@TestVisible` constructors. It's worth the ceremony.
- **Forget about Spring, JBoss AOP, Java Proxies, and AspectJ.** None of it exists in Apex. Don't try to port it.
- **Custom Metadata Types + `Type.forName` are your configuration container.** Use for runtime-selectable implementations.
- **Cross-cutting concerns are handled by hand in Apex.** Use base classes, decorators, or convention-based logger calls — pick one and stay consistent.
- **The DML-after-callout rule forces construction/use separation at the platform level.** Embrace it: gather external data first, commit to the database last.
- **Don't BDUF the whole org, but DO design the data model carefully.** Some Salesforce decisions really are hard to unwind.
- **Flows, Validation Rules, and Formulas are Salesforce's DSLs.** Prefer them when business users need to read or change the logic — that's the abstraction level the book is pointing at.
- **Keep domain classes POJO-ish: no triggers inside them, no DML, no callouts — just calculations and field manipulation.** The closer you get to pure-data-in, pure-data-out methods, the closer you get to test-drivable architecture on this platform.
