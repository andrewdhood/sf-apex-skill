# Chapter 8: Boundaries

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008), Chapter 8 by James Grenning
> **When to read this:** Before integrating Apex with a third-party managed package, an external REST/SOAP endpoint, a platform service that may change (Einstein, Marketing Cloud, Agentforce), or an unfinished internal interface.

We seldom control all the software in our systems. We buy third-party packages, use open source, and depend on other teams. The tension between a provider who wants a *broad* interface and a user who wants a *focused* one causes problems at the boundaries of our systems. This chapter is about keeping those boundaries clean.

---

## Key Principles

1. **Don't pass third-party interfaces around.** An interface like `java.util.Map` (or in our world, a generic `Map<String, Object>` JSON blob from an external API) has far more capability than any one caller needs. If it leaks through the system, every future change to that interface changes every caller.
2. **Hide the boundary inside a class you control.** Keep the third-party interface inside a wrapper class or a close family of classes. Return your own types, not the vendor's types.
3. **Write learning tests to explore third-party APIs.** Instead of fumbling with the new library inside production code, write small unit tests that call the API the way you expect to use it. You learn the API and get a regression harness for free.
4. **Learning tests are better than free.** They cost nothing (you had to learn the API anyway), and they pay ongoing dividends: when the vendor ships a new version, you run your learning tests to immediately detect behavioral differences.
5. **Use code that doesn't yet exist by defining the interface you wish you had.** When another team hasn't finalized their API, define the interface you *want* to call against. Build production code against your wish-interface. Later, write an ADAPTER that bridges your interface to the real one when it arrives.
6. **Clean boundaries depend on things you control, not things you don't.** Wrap or adapt every boundary. Keep the places that touch the third party few and well-contained.

---

## Detailed Notes

### Using Third-Party Code
Providers aim for broad applicability; users want narrow, focused interfaces. This tension creates problems at boundaries. `java.util.Map` is the canonical example: it has a `clear()` method, a `put()` method, a `remove()` method — so any holder of your map can empty it, mutate it, or put in types you didn't intend. `Map<Sensor>` helps with type safety but doesn't shrink the API surface, and if you pass Maps all over the system, a future change to `Map` requires touching every caller.

The cleaner pattern is to wrap the Map inside a class the application calls:
```
public class Sensors {
    private Map sensors = new HashMap();
    public Sensor getById(String id) {
        return (Sensor) sensors.get(id);
    }
}
```
Now the boundary is hidden. The `Sensors` class can evolve, enforce business rules, and change its internal storage without impacting callers. You are NOT being told to wrap every Map this way — the rule is: **don't pass boundary interfaces around your system; keep them inside a class or close family of classes.** Don't return them from public APIs and don't accept them as arguments to public APIs.

### Exploring and Learning Boundaries
Third-party code helps you move faster. Learning it is hard; integrating it is hard; doing both simultaneously is doubly hard. A *learning test* is a controlled experiment: you write tests that call the third-party API the way you expect to use it, and verify your understanding before the API shows up in your production code.

The chapter walks through learning `log4j`: you write a test that tries the most obvious thing, fail, read more docs, try again, fail, try again, succeed. At the end you have a series of small tests that encode your hard-won understanding of the API. That knowledge can then be used to wrap `log4j` in your own logger class so the rest of the application is isolated from `log4j` entirely.

### Learning Tests Are Better Than Free
Learning tests:
- Cost nothing — you had to learn the API regardless.
- Increase understanding through small precise experiments.
- Survive into the future as regression checks. When the vendor ships v2.x, you rerun the learning tests and find out *immediately* what changed.
- Remove fear of upgrades. Without boundary tests, you cling to old versions longer than you should.

Even if you did not write learning tests while learning, write *boundary tests* once you understand the API. They exercise the boundary the same way production code does.

### Using Code That Does Not Yet Exist
Sometimes the other side of the boundary is unknown — maybe the team building that API hasn't defined it yet. Don't let that block you. Define the interface *you wish you had*. Write your client code against that interface. Create a Fake implementation for testing. When the real API lands, write an ADAPTER that translates between your interface and theirs. The adapter becomes the single place where you deal with the real API; your production code is unaffected.

This is also a classic seam for testing — the Fake implementation lets you test your code long before the real dependency exists.

### Clean Boundaries
Change happens at boundaries. Good designs accommodate change without rework. At every boundary:
- Have clear separation and tests that define expectations.
- Avoid letting too much of your code know about third-party particulars.
- Depend on something *you* control, not on something you don't.
- Use wrappers or ADAPTERS. Refer to the third party from as few places as possible.

---

## Salesforce/Apex Caveats

**Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### Boundary #1: REST/SOAP callouts to external systems
This is the most common boundary in Apex, and the one where Clean Code advice applies most directly.

**Do:**
- Create a dedicated callout class (e.g., `StripeGateway`, `TwilioClient`) that owns every `HttpRequest` / `HttpResponse` interaction with that service.
- Return your own domain objects, not `HttpResponse` or `Map<String, Object>` from `JSON.deserializeUntyped()`. The rest of the codebase should never see a raw JSON blob.
- Wrap `CalloutException`, `System.JSONException`, and any non-2xx status as a domain exception (e.g., `StripeGatewayException`) with a meaningful message.
- Make the client class implement a callout interface (e.g., `IStripeGateway`) so tests can substitute a mock.

**Don't:**
- Pass `HttpResponse` objects around the codebase.
- Let `Map<String, Object>` JSON results leak into service or trigger layers.
- Put `Http.send()` calls anywhere outside the dedicated client class.

### Boundary #2: `HttpCalloutMock` and the testing boundary
Salesforce gives you a first-class boundary mock: `Test.setMock(HttpCalloutMock.class, new MyMockResponse())`. Use this in every test that exercises callout code. Your mock classes become the learning tests Grenning describes — encode every response shape you've observed from the vendor (success, 401, 429, malformed JSON, empty body) as a distinct mock, and test each path.

### Boundary #3: Managed packages
A managed package is a third-party library you cannot modify. Treat it exactly like Grenning's `log4j` example:
- Wrap the package's exposed classes in your own service class.
- If the package changes in an upgrade, only your wrapper needs updating.
- Write learning tests against the managed package API (possible via `@TestVisible` wrappers and mock objects) so upgrades are caught immediately.

This is especially important for FinancialForce, nCino, Conga, DocuSign, Field Service, Revenue Cloud, and similar packages whose APIs evolve across versions.

### Boundary #4: Platform APIs that are changing
Salesforce itself is a boundary, and certain areas are high-churn:
- **Einstein / Agentforce / Prompt Builder APIs** — new, evolving fast, frequent method signature changes.
- **Flow Interview invocation from Apex** — signature changes have happened.
- **Messaging APIs (Enhanced LMS, Chat)** — deprecated periodically.
- **Tooling API / Metadata API** — version-bound.
Wrap these. Do not scatter `ConnectApi.ChatterFeeds.postFeedElement(...)` calls across the codebase. Create an adapter class per platform surface and invoke it from there.

### Boundary #5: Using code that does not yet exist — the common Salesforce case
Grenning's "define the interface you wish you had" maps perfectly to Salesforce projects where:
- The integration endpoint is still being built by another team.
- The custom object model isn't finalized but the LWC/Apex work must proceed.
- An approval process or flow has been spec'd but not built.

**The Apex pattern:** define an interface that describes what you want to call, build a Stub implementation, and wire the Stub into your production code via dependency injection (constructor param, factory, or `ApplicationFactory` pattern). When the real thing lands, write an adapter. Your service-layer tests continue to pass without modification.

```apex
// the interface we wish we had
public interface IShippingCalculator {
    Decimal calculate(Id orderId);
}

// stub until the real integration exists
public class StubShippingCalculator implements IShippingCalculator {
    public Decimal calculate(Id orderId) {
        return 0;  // flat zero until the real service is ready
    }
}

// later: real adapter over the vendor API
public class FedExShippingCalculator implements IShippingCalculator {
    public Decimal calculate(Id orderId) {
        // translate to/from the vendor boundary here, and only here
        return FedExWebService.quote(orderId);  // hypothetical
    }
}
```

### Boundary #6: Salesforce core objects as a boundary
This is more philosophical, but worth noting. `Account`, `Contact`, `Opportunity`, `Case`, and your custom objects are themselves a boundary with the Salesforce data layer. You don't usually wrap SObjects in domain classes for every project — it's overkill for small projects — but on large projects the fflib-style "Domain" layer does exactly this: wraps SObjects in domain classes that encapsulate business rules. The trade-off is boilerplate vs. containment of change.

### Mocking boundaries is harder in Apex than in Java
Apex has no Mockito. Your options for mocking at a boundary are:
1. **Interface + test double:** define an interface, write a hand-rolled test implementation, inject it. Simple, explicit, no magic. This is the Grenning-friendly choice for most code.
2. **`System.StubProvider` / Stub API:** the built-in mocking API. Supports mocking any non-system class without needing an interface. Verbose but doesn't require designing for testability up front.
3. **`fflib-apex-mocks`:** the open-source Apex port of Mockito. Powerful but adds a dependency.
4. **`Test.setMock(HttpCalloutMock.class, ...)`:** for HTTP boundaries specifically. Always use this.

For most new code: define the interface, inject the dependency, and write a test double. The interface itself *is* the clean boundary.

### Static Resources and Metadata as boundary data
When a third party defines a data format (e.g., an XML schema for a legacy SOAP service, a CSV layout for a bank file), store sample payloads as Static Resources and load them in test setup. This is a learning-test pattern: you encode the vendor's actual output shape as a test fixture that fails fast if the vendor changes its format.

---

## Key Takeaways

- Wrap every external boundary: callouts, managed packages, evolving platform APIs, and unfinished internal APIs.
- Never let `HttpResponse`, `Map<String, Object>` from JSON.deserializeUntyped, or raw SObjects from a third-party query leak into service and presentation layers. Translate them at the boundary into your own types.
- Use `HttpCalloutMock` for every callout test. Cover success, every failure mode, and every malformed-response shape.
- When another team hasn't built their API yet, define the interface you wish you had. Build a Stub. Adapt to the real thing when it arrives.
- Apex doesn't have Mockito, but interfaces + hand-rolled test doubles are usually enough. Reach for the Stub API or fflib only when the interface approach feels forced.
- Depend on things you control. The fewer places in the codebase that reference a third party, the cheaper every upgrade becomes.
